# @Query JPQL vs Native Query 처리 — 두 경로의 내부 동작 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Query`의 JPQL과 `nativeQuery=true`는 내부적으로 어떻게 다르게 처리되는가?
- `@NamedQuery`와 `@Query` 중 어느 것이 먼저 확인되는가?
- Native Query 결과를 엔티티가 아닌 DTO나 인터페이스로 매핑하는 내부 원리는?
- `@Modifying`이 필요한 이유와 `clearAutomatically = true`의 효과는?
- Native Query에서 Pagination을 사용할 때 왜 `countQuery`를 별도로 지정해야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Query Method로 표현할 수 없는 쿼리가 있다

```java
// Query Method로 표현 불가능한 경우들
// 1. 서브쿼리
List<User> findUsersWhoOrderedMoreThan(int count);  // ❌ 불가능

// 2. 집계 함수 + HAVING
List<Object[]> findTopCategories();  // ❌ 불가능

// 3. DB 고유 함수
List<User> findByNameSimilarTo(String pattern);  // ❌ SIMILAR TO는 DB 고유 문법

// 4. 복잡한 JOIN + 조건 혼합
// → @Query로 JPQL 또는 Native Query를 직접 작성해야 함
```

```
@Query의 존재 이유:
  Query Method DSL의 한계를 극복하면서도
  메서드 시그니처와 쿼리를 같은 위치(Repository)에 유지
  → 쿼리 로직의 분산 방지
  → Spring Data의 파라미터 바인딩, 페이징, 트랜잭션 관리 재사용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Modifying 없이 UPDATE/DELETE 쿼리를 실행한다

```java
// ❌ @Modifying 누락 — QueryExecutionRequestException 발생
@Query("UPDATE User u SET u.status = :status WHERE u.lastLoginAt < :date")
void deactivateOldUsers(@Param("status") Status status, @Param("date") LocalDate date);

// ✅ @Modifying 필요 — DML 쿼리임을 명시
@Modifying
@Query("UPDATE User u SET u.status = :status WHERE u.lastLoginAt < :date")
int deactivateOldUsers(@Param("status") Status status, @Param("date") LocalDate date);
// 반환 타입: int (영향받은 행 수) 또는 void
```

### Before: @Modifying 후 영속성 컨텍스트가 자동으로 갱신된다

```java
// ❌ 잘못된 이해:
@Modifying
@Query("UPDATE User u SET u.status = 'INACTIVE' WHERE u.id = :id")
void deactivateUser(@Param("id") Long id);

// 같은 트랜잭션에서 조회 → 1차 캐시(영속성 컨텍스트)에 여전히 ACTIVE 상태
User user = userRepository.findById(id).get();
System.out.println(user.getStatus()); // 여전히 ACTIVE! (캐시 미갱신)

// ✅ clearAutomatically = true 사용
@Modifying(clearAutomatically = true)
@Query("UPDATE User u SET u.status = 'INACTIVE' WHERE u.id = :id")
void deactivateUser(@Param("id") Long id);
// → EntityManager.clear() 호출 → 1차 캐시 전체 비움
// → 이후 조회 시 DB에서 새로 읽어옴
```

### Before: Native Query와 JPQL의 파라미터 바인딩이 동일하다

```java
// ❌ Native Query에서 :name 사용 → 드라이버에 따라 지원 여부 다름
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);
// → Spring Data가 :name → ?로 치환해주지만, 일부 드라이버에서 예외 가능

// ✅ 위치 기반 바인딩이 더 안전
@Query(value = "SELECT * FROM users WHERE name = ?1", nativeQuery = true)
List<User> findByName(String name);

// ✅ 또는 Spring Data의 @Param으로 명확히 바인딩
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);
```

---

## ✨ 올바른 이해와 패턴

### After: JPQL vs Native Query 선택 기준

```
JPQL 사용:
  - 엔티티 객체 조회 (Entity Graph, Lazy Loading 필요)
  - DB 독립적 쿼리 (MySQL, PostgreSQL 등 전환 가능성)
  - Spring Data의 Pagination 자동 count 쿼리 지원 필요
  - Hibernate의 2차 캐시 혜택 필요

Native Query 사용:
  - DB 고유 함수 (MySQL의 JSON_EXTRACT, PostgreSQL의 ARRAY_AGG 등)
  - 복잡한 서브쿼리, CTE(WITH 절), Window Function
  - 성능 최적화가 필요한 쿼리 (힌트 사용, 실행 계획 제어)
  - 레거시 DB 스키마로 엔티티 매핑이 어려운 경우
```

---

## 🔬 내부 동작 원리 — @Query 처리 소스 추적

### 1. @Query 감지 → SimpleJpaQuery 생성

```java
// JpaQueryLookupStrategy에서 @Query 어노테이션 감지
@Override
protected RepositoryQuery resolveQuery(JpaQueryMethod method, ...) {

    Query queryAnnotation = method.getQueryAnnotation();

    if (queryAnnotation == null) {
        return null;  // Query Method 방식으로 폴백
    }

    // nativeQuery 여부에 따라 분기
    if (method.isNativeQuery()) {
        return NativeJpaQuery(method, em, queryAnnotation.value(), ...);
    }

    return new SimpleJpaQuery(method, em, queryAnnotation.value(), ...);
}
```

### 2. JPQL 처리 경로 — SimpleJpaQuery

```java
// SimpleJpaQuery — JPQL 문자열을 TypedQuery로 변환
class SimpleJpaQuery extends AbstractStringBasedJpaQuery {

    public SimpleJpaQuery(JpaQueryMethod method, EntityManager em,
                          String queryString, ...) {

        super(method, em, queryString, ...);

        // 시작 시 유효성 검증 (파싱 오류 조기 발견)
        validateQuery(queryString, "Validation failed for query for method %s!", method);
    }

    @Override
    protected TypedQuery<Long> doCreateCountQuery(...) {
        // Pagination 시 count 쿼리 자동 생성
        // "SELECT u FROM User u WHERE u.status = :status"
        // → "SELECT count(u) FROM User u WHERE u.status = :status"
        String countQuery = QueryUtils.createCountQueryFor(queryString, countProjection);
        return em.createQuery(countQuery, Long.class);
    }
}
```

```
JPQL 처리 특징:
  Hibernate가 JPQL → SQL로 변환 (Dialect 사용)
  엔티티 필드명 사용 → DB 컬럼명 매핑 자동 처리
  Hibernate Query Plan Cache 혜택 (JPQL 파싱 결과 캐시)
  페이징 시 count 쿼리 자동 생성
```

### 3. Native Query 처리 경로 — NativeJpaQuery

```java
// NativeJpaQuery — SQL 문자열을 직접 실행
class NativeJpaQuery extends AbstractStringBasedJpaQuery {

    @Override
    protected Query doCreateQuery(JpaParametersParameterAccessor accessor) {

        String sortedQueryString = QueryUtils.applySorting(query, sort, alias);

        // em.createNativeQuery() 사용 → JPQL이 아닌 SQL 직접 전달
        Class<?> typeToRead = getTypeToRead(returnedObjectType);

        if (typeToRead != null) {
            // 반환 타입이 엔티티인 경우
            return em.createNativeQuery(sortedQueryString, typeToRead);
        }

        // 반환 타입이 인터페이스 Projection, DTO인 경우
        return em.createNativeQuery(sortedQueryString);
        // → 결과는 Object[] → Spring Data가 매핑 처리
    }
}
```

```
Native Query 처리 특징:
  SQL 문자열 그대로 DB에 전달 → Hibernate 파싱 최소화
  Hibernate Query Plan Cache 미적용 (SQL은 캐시하지 않음)
  엔티티 매핑: @SqlResultSetMapping 또는 반환 타입 명시 필요
  Pagination: count 쿼리 자동 생성 불가 → countQuery 직접 지정 필요
```

### 4. 파라미터 바인딩 — JPQL vs Native Query

```java
// JPQL 파라미터 바인딩 (위치 기반 vs 이름 기반)
@Query("SELECT u FROM User u WHERE u.name = ?1 AND u.age > ?2")
List<User> findByNameAndAge(String name, int age);
// → query.setParameter(1, name), query.setParameter(2, age)

@Query("SELECT u FROM User u WHERE u.name = :name AND u.age > :age")
List<User> findByNameAndAge(@Param("name") String name, @Param("age") int age);
// → query.setParameter("name", name), query.setParameter("age", age)

// Native Query 파라미터 바인딩 (SpEL 지원!)
@Query(value = "SELECT * FROM users WHERE created_at > :#{#since}", nativeQuery = true)
List<User> findRecentUsers(@Param("since") LocalDate since);
// :#{#since} → SpEL 표현식으로 파라미터 가공 가능
```

### 5. @Modifying 내부 동작

```java
// @Modifying 처리 — ModifyingQueryExecutor
class JpaQueryExecution {

    // DML 쿼리 실행
    static class ModifyingExecution extends JpaQueryExecution {

        @Override
        protected Object doExecute(AbstractJpaQuery query, ...) {

            // executeUpdate() 사용 (executeQuery가 아님!)
            int rowsAffected = query.createQuery(accessor).executeUpdate();

            // clearAutomatically=true이면 영속성 컨텍스트 클리어
            if (query.getQueryMethod().getClearAutomatically()) {
                em.clear();  // 1차 캐시 전체 비움
            }

            // flushAutomatically=true이면 먼저 flush
            // (DB와의 동기화 후 UPDATE 실행)
            if (query.getQueryMethod().getFlushAutomatically()) {
                em.flush();
            }

            return rowsAffected;
        }
    }
}
```

### 6. Native Query + Pagination — countQuery 필수

```java
// ❌ countQuery 없이 Pagination — HibernateException 발생 가능
@Query(value = "SELECT u.*, r.name as role_name FROM users u JOIN roles r ON u.role_id = r.id",
       nativeQuery = true)
Page<User> findAllWithRole(Pageable pageable);
// → Spring Data가 자동 생성하는 count 쿼리:
// SELECT count(*) FROM (SELECT u.*, r.name as role_name FROM users u JOIN roles r ...)
// → 일부 DB에서 서브쿼리 count 미지원 or 성능 저하

// ✅ countQuery 직접 지정
@Query(
    value = "SELECT u.*, r.name as role_name FROM users u JOIN roles r ON u.role_id = r.id",
    countQuery = "SELECT count(u.id) FROM users u JOIN roles r ON u.role_id = r.id",
    nativeQuery = true
)
Page<User> findAllWithRole(Pageable pageable);
```

### 7. Native Query 결과 매핑 방식 3가지

```java
// 방식 1: 엔티티 직접 매핑 (DB 컬럼 ↔ 엔티티 필드 자동 매핑)
@Query(value = "SELECT * FROM users WHERE status = ?1", nativeQuery = true)
List<User> findByStatus(String status);

// 방식 2: Interface Projection (컬럼 이름 → 게터 메서드 매핑)
public interface UserSummary {
    Long getId();
    String getName();
    // DB 컬럼명: id, name → 자동 매핑
}
@Query(value = "SELECT id, name FROM users WHERE status = ?1", nativeQuery = true)
List<UserSummary> findSummaryByStatus(String status);

// 방식 3: @SqlResultSetMapping (명시적 매핑, 복잡한 경우)
@Entity
@SqlResultSetMapping(
    name = "UserWithOrderCount",
    classes = @ConstructorResult(
        targetClass = UserOrderDto.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "name"),
            @ColumnResult(name = "order_count", type = Integer.class)
        }
    )
)
public class User { ... }

@Query(value = "SELECT u.id, u.name, COUNT(o.id) as order_count " +
               "FROM users u LEFT JOIN orders o ON u.id = o.user_id GROUP BY u.id, u.name",
       nativeQuery = true)
List<UserOrderDto> findUsersWithOrderCount();
```

---

## 💻 실험으로 확인하기

### 실험 1: @Modifying + clearAutomatically 차이 확인

```java
@Transactional
@Test
void modifyingClearEffect() {
    // 초기 데이터
    User user = userRepository.save(new User("테스트", Status.ACTIVE));
    Long id = user.getId();

    // JPQL UPDATE (영속성 컨텍스트 우회)
    userRepository.deactivateUser(id);  // @Modifying, clearAutomatically=false

    // 1차 캐시에서 반환 → 여전히 ACTIVE
    User fromCache = userRepository.findById(id).get();
    System.out.println("캐시 상태: " + fromCache.getStatus());
    // 출력: ACTIVE  ← 오래된 캐시!

    // clearAutomatically=true 버전 사용 시
    userRepository.deactivateUserWithClear(id);

    User fromDb = userRepository.findById(id).get();
    System.out.println("DB 상태: " + fromDb.getStatus());
    // 출력: INACTIVE  ← 정확한 상태
}
```

### 실험 2: JPQL vs Native Query 실행 계획 차이

```java
@Test
void queryExecutionPath() {
    // JPQL — Hibernate가 SQL로 변환 (Dialect 적용)
    List<User> byJpql = userRepository.findActiveByJpql();
    // Hibernate: select u1_0.id, u1_0.name, ... from users u1_0 where u1_0.status=?

    // Native Query — SQL 그대로 전달
    List<User> byNative = userRepository.findActiveByNative();
    // Hibernate: SELECT * FROM users WHERE status = ?
    // (Hibernate 변환 없음, 컬럼 별칭 주의)
}
```

### 실험 3: count 쿼리 자동 생성 확인

```java
// JPQL Pagination — count 쿼리 자동 생성
@Query("SELECT u FROM User u WHERE u.status = :status")
Page<User> findByStatus(@Param("status") Status status, Pageable pageable);

// 실행 시 발생하는 쿼리 2개:
// 1. SELECT u1_0.id, ... FROM users u1_0 WHERE u1_0.status=? LIMIT ? OFFSET ?
// 2. SELECT count(u1_0.id) FROM users u1_0 WHERE u1_0.status=?
//    (Spring Data가 SELECT절을 count로 자동 변환)
```

---

## ⚡ 성능 임팩트

```
JPQL vs Native Query 성능 비교:

[쿼리 파싱 비용]
  JPQL: Hibernate가 JPQL → SQL 변환 (Query Plan Cache로 캐시)
        첫 실행: ~수 ms
        이후: 캐시 히트 (~μs)
  Native: 파싱 없음, SQL 그대로 전달
          매 실행: ~μs (파싱 비용 없음)

[결과 매핑 비용]
  JPQL → 엔티티: Hibernate ResultSet → Entity 매핑 (리플렉션)
  Native → 엔티티: 동일
  Native → Interface Projection: JDK Proxy 생성 (소량이면 무시 가능)
  Native → DTO: 생성자 직접 호출 (가장 빠름)

[실질적 차이]
  DB I/O가 90% 이상 → 파싱 비용 차이는 무시 가능
  복잡한 Native Query가 실행 계획 최적화로 JPQL보다 수십 배 빠를 수 있음
  (인덱스 힌트, Window Function 등 DB 고유 최적화 사용 가능)

Pagination count 쿼리 주의:
  전체 데이터 1,000만 건 → count(*) 실행 시간 수 초
  → @CountQuery 별도 지정 또는 Cursor 기반 Pagination 전환 권장
```

---

## 🤔 트레이드오프

```
@Query JPQL:
  DB 독립성 유지           ✅
  Hibernate 최적화 혜택     ✅ (2차 캐시, Lazy Loading)
  복잡한 DB 기능 사용       ❌ (Window Function, CTE 등 불가)
  Pagination count 자동     ✅

@Query nativeQuery=true:
  DB 고유 기능 사용         ✅ (JSON, Array, Window Function)
  성능 최적화 제어           ✅ (쿼리 힌트, 실행 계획 제어)
  DB 독립성                 ❌ (DB 교체 시 쿼리 재작성)
  Pagination count 자동     ❌ (countQuery 직접 지정 필요)
  엔티티 변경 감지          ❌ (Native Query는 영속성 컨텍스트 우회)

@Modifying 사용 원칙:
  UPDATE/DELETE JPQL/Native 쿼리 → @Modifying 필수
  같은 트랜잭션에서 이후 조회 → clearAutomatically = true
  대용량 벌크 업데이트 → flushAutomatically = true 고려
  (미반영 변경사항이 있으면 flush 먼저 실행 후 UPDATE)
```

---

## 📌 핵심 정리

```
@Query 처리 경로
  @Query(JPQL) → SimpleJpaQuery → em.createQuery() → Hibernate SQL 변환
  @Query(native) → NativeJpaQuery → em.createNativeQuery() → SQL 직접 전달

쿼리 탐색 우선순위 (CREATE_IF_NOT_FOUND 전략)
  1. @Query 어노테이션
  2. @NamedQuery (persistence.xml 또는 @NamedQuery)
  3. PartTree 파싱 (메서드 이름)

@Modifying 필수 조건
  UPDATE / DELETE / INSERT JPQL 또는 Native Query
  반환 타입: int (영향받은 행 수) 또는 void

clearAutomatically = true
  @Modifying 후 같은 트랜잭션에서 조회 시 필수
  em.clear() → 1차 캐시 전체 비움 (성능 영향 주의)

Native Query Pagination
  countQuery 반드시 직접 지정 (자동 생성 쿼리 비효율)
  Interface Projection으로 필요한 컬럼만 선택
```

---

## 🤔 생각해볼 문제

**Q1.** `@Modifying(clearAutomatically = true)` 사용 시 같은 트랜잭션 내의 모든 엔티티가 캐시에서 제거된다. 대용량 배치에서 이것이 문제가 될 수 있는 이유는?

**Q2.** `@Query`의 JPQL에서 `JOIN FETCH`를 사용하면서 동시에 `Pageable`을 파라미터로 받으면 어떤 경고 또는 문제가 발생하는가?

**Q3.** Native Query에서 Interface Projection을 사용할 때, DB 컬럼명과 인터페이스 메서드 이름 매핑 규칙은 무엇인가? 컬럼명이 `user_name`이고 메서드가 `getName()`이면 매핑되는가?

> 💡 **해설**
>
> **Q1.** `em.clear()`는 현재 영속성 컨텍스트의 **모든** 엔티티를 제거한다. 배치 처리에서 청크 단위로 엔티티를 로드하고 처리하는 중간에 `@Modifying`을 호출하면, 이미 로드되어 처리 중이던 다른 엔티티들도 함께 캐시에서 제거된다. 이후 해당 엔티티들에 접근할 때 `LazyInitializationException` 또는 추가 DB 조회가 발생할 수 있다. 배치에서는 `@Modifying(clearAutomatically = false)`를 사용하고, 필요한 경우 수동으로 `entityManager.refresh(entity)`를 호출하거나 트랜잭션을 청크 단위로 분리하는 것이 안전하다.
>
> **Q2.** Hibernate는 `JOIN FETCH`와 `LIMIT/OFFSET`(Pagination)을 함께 사용하면 `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory` 경고를 출력한다. 실제로는 **전체 데이터를 메모리에 로드한 후 애플리케이션 레벨에서 페이징**을 수행한다. 데이터가 많을수록 OOM 위험이 있다. 해결책은 Fetch Join 대신 `@EntityGraph`를 사용하거나, DTO Projection으로 전환하거나, 페이징 쿼리와 데이터 로드 쿼리를 분리(ID로 페이징 후 ID로 Fetch Join)하는 것이다.
>
> **Q3.** 매핑되지 않는다. Native Query + Interface Projection에서 Spring Data는 DB 컬럼 별칭(alias)과 인터페이스 메서드 이름을 직접 대조한다. `user_name` 컬럼과 `getName()` 메서드는 `name`과 `user_name`이 달라 매핑에 실패하고 `null`이 반환된다. 올바른 방법은 SQL에서 별칭을 지정하는 것이다: `SELECT u.user_name AS name FROM users u`. 별칭 `name`이 `getName()`의 `name`과 매칭된다. 대소문자는 무시된다(`created_at` → `getCreatedAt()` 가능).

---

<div align="center">

**[⬅️ 이전: Query Method 파싱 메커니즘](./02-query-method-parsing.md)** | **[다음: Projection — Interface vs DTO ➡️](./04-projection-interface-dto.md)**

</div>
