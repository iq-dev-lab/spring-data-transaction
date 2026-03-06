# Query Plan Cache 동작 — JPQL 파싱 비용과 캐시 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hibernate가 JPQL을 파싱해서 `QueryPlan`으로 캐시하는 내부 구조는?
- 파라미터 바인딩 방식(`?` vs `:name`)이 캐시 히트율에 영향을 주는 이유는?
- `hibernate.query.plan_cache_max_size` 설정이 부족할 때 어떤 문제가 발생하는가?
- IN절 파라미터 수가 다르면 왜 다른 캐시 엔트리가 생성되는가?
- QueryPlanCache 미스율이 높을 때 진단하고 튜닝하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: JPQL 파싱은 매번 수행하면 비용이 크다

```java
// JPQL 실행 시마다 파싱이 일어난다면:
for (int i = 0; i < 1000; i++) {
    em.createQuery("SELECT u FROM User u WHERE u.id = :id", User.class)
      .setParameter("id", (long) i)
      .getSingleResult();
    // ANTLR4 파싱 → SQM 생성 → SQL 번역 → PreparedStatement
    // 매번 반복하면 ~수십 ms × 1000 = 수십 초
}

// QueryPlanCache가 없다면 동일 JPQL도 매번 파싱 → 심각한 성능 저하
// → QueryPlanCache: 동일 JPQL 문자열 → 캐시된 QueryPlan 재사용
```

```
QueryPlanCache 동작:
  캐시 키:   JPQL 문자열 (정확히 일치해야 함)
  캐시 값:   HQLQueryPlan (파싱 결과, SQL 번역 결과)
  효과:      파싱 비용 제거, PreparedStatement 재사용
  주의:      동적으로 생성된 JPQL은 캐시 미스 → 파싱 반복
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 파라미터만 다르면 같은 캐시를 사용한다

```java
// ❌ 값을 직접 문자열에 포함 — 쿼리마다 다른 캐시 키
String jpql1 = "SELECT u FROM User u WHERE u.id = 1";
String jpql2 = "SELECT u FROM User u WHERE u.id = 2";
// 완전히 다른 문자열 → 다른 캐시 엔트리 → 매번 파싱

// ✅ 파라미터 바인딩 — 같은 캐시 키
String jpql = "SELECT u FROM User u WHERE u.id = :id";
em.createQuery(jpql).setParameter("id", 1L); // 캐시 저장
em.createQuery(jpql).setParameter("id", 2L); // 캐시 히트!
// 동일 JPQL 문자열 → 동일 캐시 키 → 파싱 생략
```

### Before: IN절에 파라미터 수가 달라도 같은 캐시를 쓴다

```java
// ❌ 잘못된 이해: IN절 파라미터 수 무관하게 캐시 공유

// ✅ 실제: IN절 파라미터 수가 다르면 다른 캐시 키
"SELECT u FROM User u WHERE u.id IN (:ids)"
// ids = [1, 2, 3]   → "... WHERE u.id IN (?1, ?2, ?3)"
// ids = [1, 2]      → "... WHERE u.id IN (?1, ?2)"
// → SQL 문자열이 달라 → 다른 캐시 엔트리!
// → 가변 목록 IN절 → 캐시 미스 반복 → QueryPlanCache 오염

// 해결: @BatchSize의 패딩 전략 (고정 크기 IN절)
// 또는 Spring Data JPA의 IN절 처리
```

---

## ✨ 올바른 이해와 패턴

### After: QueryPlanCache 키 결정 요소

```
캐시 키 = JPQL 문자열 (완전 일치)

캐시 미스 유발 패턴:
  1. 파라미터 값을 문자열에 직접 포함
     "WHERE u.name = 'John'" → 매번 다른 문자열
  
  2. 가변 IN절 파라미터 수
     IN (?1, ?2, ?3) vs IN (?1, ?2) → 다른 캐시

  3. 동적 JPQL 문자열 생성 (매번 다른 조건)
     StringBuilder로 if/else 조건 추가

캐시 히트 보장 패턴:
  1. 파라미터 바인딩 (:name, ?1)
  2. 고정 크기 IN절 (BatchSize 패딩)
  3. 정적 @Query 어노테이션
  4. QueryDSL (JPQLSerializer가 일관된 문자열 생성)
```

---

## 🔬 내부 동작 원리 — QueryPlanCache 소스 추적

### 1. QueryPlanCache 구조

```java
// Hibernate QueryPlanCache — SessionFactory 레벨 (애플리케이션 전체 공유)
public class QueryPlanCache implements Serializable {

    private final SessionFactoryImplementor factory;

    // 캐시 구현체 — BoundedConcurrentHashMap
    private final BoundedConcurrentHashMap<KeyT, HQLQueryPlan> queryPlanCache;
    // 키: HQLQueryPlanKey (JPQL 문자열 + ScrollMode + EntityFilter)
    // 값: HQLQueryPlan (파싱된 SQM + 번역된 SQL + 파라미터 바인더)
    // 기본 크기: 2048 (hibernate.query.plan_cache_max_size)

    private final BoundedConcurrentHashMap<String, ParameterMetadataImpl> parameterMetadataCache;
    // 파라미터 메타데이터 캐시
    // 기본 크기: 128 (hibernate.query.plan_parameter_metadata_max_size)

    // JPQL → QueryPlan 조회 또는 생성
    public HQLQueryPlan getHQLQueryPlan(String queryString, boolean shallow,
                                         Map<String, Filter> enabledFilters) {
        HQLQueryPlanKey key = new HQLQueryPlanKey(queryString, shallow, enabledFilters);

        HQLQueryPlan plan = queryPlanCache.get(key);
        if (plan == null) {
            // 캐시 미스 → 파싱 (비용 발생)
            plan = new HQLQueryPlan(queryString, shallow, enabledFilters, factory);
            // → ANTLR4 파싱 → SQM 생성 → SQL 번역

            queryPlanCache.putIfAbsent(key, plan);
        }
        return plan;  // 캐시 히트 → 파싱 없음
    }
}
```

### 2. HQLQueryPlanKey — 캐시 키 구성

```java
// HQLQueryPlanKey — equals/hashCode로 캐시 히트 결정
class HQLQueryPlanKey {
    private final String query;           // JPQL 문자열 (정확히 일치)
    private final boolean shallow;
    private final Set<String> filterNames; // 활성화된 Hibernate Filter
    private final int hashCode;

    @Override
    public boolean equals(Object o) {
        HQLQueryPlanKey that = (HQLQueryPlanKey) o;
        return Objects.equals(query, that.query)  // 문자열 완전 일치
            && shallow == that.shallow
            && Objects.equals(filterNames, that.filterNames);
    }
}

// 공백 하나, 대소문자 차이도 다른 캐시 키!
"SELECT u FROM User u WHERE u.id = :id"  // 캐시 키 A
"select u from User u where u.id = :id"  // 캐시 키 B (다름!)
"SELECT u FROM User u WHERE u.id = :id " // 캐시 키 C (끝 공백 다름!)
```

### 3. IN절 파라미터 수와 캐시 영향

```java
// 가변 IN절의 캐시 문제
// Spring Data JPA findAllById(Iterable<ID> ids) 내부:
public List<T> findAllById(Iterable<ID> ids) {
    List<ID> idList = Streamable.of(ids).toList();
    // idList 크기에 따라 다른 IN절 생성:
    // 3개: WHERE id IN (?1, ?2, ?3)
    // 5개: WHERE id IN (?1, ?2, ?3, ?4, ?5)
    // → 매번 다른 QueryPlan 캐시 엔트리!
}

// Hibernate 6.x의 개선: IN절 패딩
// 실제 파라미터 수 → 다음 단계 크기로 패딩
// [1, 2, 3, 4, 5, 6, 7, 8, 10, 12, 14, 16, 20, 24, 28, 32, ...]
// 3개 → 3개 그대로 (이미 단계 값), 4개 → 4개, 5개 → 6개, ...
// → 적은 캐시 엔트리로 더 많은 케이스 커버

// 설정으로 패딩 활성화 (Hibernate 5.x):
spring.jpa.properties.hibernate.query.in_clause_parameter_padding=true
// Hibernate 6.x: 기본 활성화
```

### 4. QueryPlanCache 크기 부족 시 증상

```java
// 캐시 크기 초과 시: BoundedConcurrentHashMap의 LRU 교체
// → 자주 쓰는 QueryPlan도 교체되면 → 파싱 반복 → CPU 부하 증가

// 증상:
// CPU 사용률 비정상적 증가 (파싱은 CPU 집약적)
// 요청 처리 시간 증가
// GC 압박 (QueryPlan 객체 반복 생성/수집)

// 진단: Hibernate Statistics 활용
SessionFactory sf = emf.unwrap(SessionFactory.class);
Statistics stats = sf.getStatistics();
stats.setStatisticsEnabled(true);

System.out.println("QueryPlan cache hits: " + stats.getQueryPlanCacheHitCount());
System.out.println("QueryPlan cache misses: " + stats.getQueryPlanCacheMissCount());
// 미스율 = misses / (hits + misses)
// 미스율 > 20% → 캐시 크기 증가 또는 동적 쿼리 최적화 검토

// 캐시 크기 조정
spring:
  jpa:
    properties:
      hibernate:
        query:
          plan_cache_max_size: 4096        # 기본 2048
          plan_parameter_metadata_max_size: 256  # 기본 128
```

### 5. Criteria API vs JPQL — 캐시 관점 차이

```java
// Criteria API: 매 실행마다 새 객체 생성 → 내부적으로 캐시 키 다를 수 있음
// Hibernate는 Criteria 쿼리도 내부적으로 JPQL로 변환 후 캐시 활용
// 단, 동일 Criteria 구조도 객체 동일성이 아닌 구조적 동일성 기반

// JPQL @Query: 고정 문자열 → 캐시 히트율 최고
@Query("SELECT u FROM User u WHERE u.status = :status")
List<User> findByStatus(@Param("status") Status status);
// → 항상 동일 문자열 → 항상 캐시 히트

// QueryDSL: JPQLSerializer가 일관된 문자열 생성
// 조건이 같으면 동일 JPQL → 캐시 히트
queryFactory.selectFrom(user).where(user.status.eq(status)).fetch();
// → "select user from User user where user.status = ?1" (일관된 형식)
```

---

## 💻 실험으로 확인하기

### 실험 1: 캐시 히트/미스 측정

```java
@Test
void queryPlanCacheHitRate() {
    SessionFactory sf = emf.unwrap(SessionFactory.class);
    Statistics stats = sf.getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();

    // 동일 JPQL 100번 실행
    String jpql = "SELECT u FROM User u WHERE u.status = :status";
    for (int i = 0; i < 100; i++) {
        em.createQuery(jpql, User.class)
          .setParameter("status", Status.ACTIVE)
          .getResultList();
    }

    System.out.println("Hits: " + stats.getQueryPlanCacheHitCount());   // 99
    System.out.println("Misses: " + stats.getQueryPlanCacheMissCount()); // 1
    // 첫 번째만 파싱, 이후 99번은 캐시 히트
}
```

### 실험 2: 동적 JPQL 캐시 오염 확인

```java
@Test
void dynamicJpqlCachePollution() {
    stats.clear();

    // 값을 문자열에 포함한 동적 JPQL
    for (int i = 0; i < 10; i++) {
        String jpql = "SELECT u FROM User u WHERE u.id = " + i; // ← 캐시 오염
        em.createQuery(jpql, User.class).getResultList();
    }

    System.out.println("Misses: " + stats.getQueryPlanCacheMissCount()); // 10
    // 매번 다른 문자열 → 10번 모두 캐시 미스 → 10번 파싱
    // 캐시에 10개의 쓸모없는 엔트리 추가 (캐시 오염)
}
```

### 실험 3: IN절 패딩 효과 확인

```yaml
spring.jpa.properties.hibernate.query.in_clause_parameter_padding: true
```

```java
@Test
void inClausePaddingEffect() {
    stats.clear();

    // 다양한 크기 목록으로 findAllById 호출
    userRepository.findAllById(List.of(1L, 2L, 3L));       // 3개
    userRepository.findAllById(List.of(1L, 2L, 3L, 4L));   // 4개
    userRepository.findAllById(List.of(1L, 2L, 3L, 4L, 5L)); // 5개 → 6으로 패딩

    // 패딩 없으면: 미스 3번 (3개, 4개, 5개 각각 다른 캐시)
    // 패딩 있으면: 미스 2~3번 (4개 → 4단계, 5개 → 6단계, 패딩된 값 공유 가능)
    System.out.println("Misses: " + stats.getQueryPlanCacheMissCount());
}
```

---

## ⚡ 성능 임팩트

```
QueryPlan 파싱 비용:
  ANTLR4 파싱 + SQM 생성 + SQL 번역: ~1~10ms (쿼리 복잡도에 비례)
  캐시 히트 시: ~수십 μs (HashMap 조회)
  → 캐시 히트율 99%: 평균 ~수십 μs
  → 캐시 히트율 50%: 평균 ~수 ms (5~100배 차이)

캐시 크기 부족 시 오버헤드:
  LRU 교체 → 이전 캐시 엔트리 GC
  자주 쓰는 쿼리도 교체 → 파싱 반복
  CPU 30~50% 증가 사례 보고 (파싱이 CPU 집약적)

캐시 크기 최적화:
  기본 2048: 정적 쿼리 위주 애플리케이션에서 충분
  4096+: 동적 쿼리 많거나 쿼리 종류 많은 경우
  동적 IN절: 패딩 활성화로 캐시 엔트리 50~80% 감소

IN절 패딩 효과 예시:
  IN절 크기 1~100 무작위 → 패딩 없이 100개 캐시 엔트리
  IN절 패딩 활성화 → 약 15~20개 캐시 엔트리 (2의 제곱수 단계)
  → 캐시 메모리 사용량 감소, 히트율 증가
```

---

## 🤔 트레이드오프

```
캐시 크기 증가:
  ✅ 더 많은 QueryPlan 유지 → 미스율 감소
  ❌ 메모리 사용량 증가 (QueryPlan 객체는 수십 KB)
  → JVM 힙 여유 있으면 2048 → 4096 증가 권장

IN절 패딩 활성화:
  ✅ 캐시 엔트리 수 감소 → 히트율 증가
  ❌ 불필요한 null 파라미터 전송 → 약간의 네트워크 오버헤드
  ❌ DB에서 null 필터링 부가 연산
  → 대부분의 경우 장점이 단점보다 큼 (활성화 권장)

정적 @Query vs 동적 JPQL:
  @Query: 캐시 히트율 100% (항상 같은 문자열)
  동적 JPQL: 캐시 오염, 메모리 낭비
  → 동적 쿼리는 QueryDSL로 (일관된 JPQL 생성)
  → 정적 쿼리는 @Query (최고 캐시 효율)

파라미터 바인딩 스타일:
  :name (이름 기반): 가독성 좋음, 캐시 동일
  ?1 (위치 기반): 간결, 캐시 동일
  → 캐시 관점에서 동일, 취향/코딩 스타일 따름
```

---

## 📌 핵심 정리

```
QueryPlanCache 구조
  SessionFactory 레벨 (애플리케이션 전체 공유)
  BoundedConcurrentHashMap (기본 2048 엔트리)
  키: JPQL 문자열 + Filter 상태 (정확히 일치해야 히트)

캐시 미스 주요 원인
  파라미터 값을 문자열에 직접 포함
  가변 크기 IN절 (매번 다른 파라미터 수)
  조건에 따라 구조가 다른 동적 JPQL

최적화 방법
  파라미터 바인딩 (:name, ?1)
  IN절 패딩 활성화 (in_clause_parameter_padding=true)
  캐시 크기 조정 (plan_cache_max_size)
  정적 @Query 우선 사용

진단
  Hibernate Statistics: getQueryPlanCacheMissCount()
  미스율 > 20% → 최적화 필요
  CPU 이상 증가 → QueryPlanCache 문제 의심
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 JPQL을 `@Transactional` 메서드와 비트랜잭션 메서드에서 각각 실행하면 동일한 QueryPlanCache 엔트리를 공유하는가?

**Q2.** Hibernate Filter(`@Filter`, `session.enableFilter()`)가 활성화된 상태와 비활성화된 상태에서 같은 JPQL을 실행하면 QueryPlanCache 히트가 발생하는가?

**Q3.** QueryDSL을 사용할 때 같은 쿼리 구조에 대해 항상 동일한 JPQL 문자열을 생성하는가? 동적 조건(where절 조건 추가/제거)이 있을 때는?

> 💡 **해설**
>
> **Q1.** 공유한다. QueryPlanCache는 SessionFactory 레벨에서 관리되며 트랜잭션 상태와 무관하다. 캐시 키는 JPQL 문자열과 활성화된 Hibernate Filter로만 구성되므로, 트랜잭션 유무, readOnly 여부, Isolation Level 등은 캐시 키에 영향을 주지 않는다. 두 메서드가 동일한 JPQL 문자열을 사용하면 항상 같은 QueryPlan을 공유한다.
>
> **Q2.** 공유하지 않는다. `HQLQueryPlanKey`에는 JPQL 문자열 외에 활성화된 Filter 이름 집합이 포함된다. Filter가 활성화된 상태에서 실행한 JPQL과 비활성 상태의 JPQL은 다른 캐시 키를 갖는다. 이유는 Filter가 WHERE 조건에 추가 술어를 삽입하므로 동일 JPQL이라도 실제 실행 SQL이 달라질 수 있기 때문이다. Hibernate는 이를 올바르게 처리하기 위해 Filter 상태를 캐시 키에 포함시킨다.
>
> **Q3.** 같은 쿼리 구조에서는 동일한 JPQL을 생성한다. `JPQLSerializer`는 `QueryMetadata`(조건, 정렬, 프로젝션 등)를 일관된 방식으로 직렬화하므로, 동일한 표현식 트리는 항상 동일한 JPQL 문자열을 생성한다. 단, 동적 조건이 있을 때는 조건의 존재 여부에 따라 생성되는 JPQL 구조가 달라진다. 예를 들어 `where(nameContains(name), statusEq(status))`에서 `name=null`이면 `nameContains(null)=null`이 되어 where절에서 제외되고 `where(statusEq(status))`로 실행된다. 이 경우 `name`이 있는 쿼리와 없는 쿼리는 다른 JPQL 문자열 → 다른 캐시 엔트리가 된다. 이는 의도된 동작이며, 동적 쿼리의 각 조합은 별도 QueryPlan을 가질 수 있다.

---

<div align="center">

**[⬅️ 이전: @BatchSize vs default_batch_fetch_size](./04-batch-size-configuration.md)** | **[다음: Second-Level Cache ➡️](./06-second-level-cache.md)**

</div>
