# Query Method 파싱 메커니즘 — PartTree와 findByNameAndAge의 비밀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `findByNameAndAgeGreaterThan(String name, int age)` 메서드 이름에서 JPQL이 생성되는 과정은?
- `PartTree`는 메서드 이름을 어떻게 Subject와 Predicate로 분리하는가?
- Query Method 파싱은 언제 발생하는가 — 컨텍스트 시작 시인가, 첫 호출 시인가?
- 지원하는 키워드와 지원하지 않는 표현의 경계는 어디인가?
- 메서드 이름이 복잡해질 때 `@Query`로 전환해야 하는 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: 단순한 조건 조회마다 JPQL을 직접 작성해야 한다

```java
// JPQL 직접 작성 — 조건이 늘어날수록 쿼리가 산재됨
@Query("SELECT u FROM User u WHERE u.name = :name AND u.age > :age")
List<User> findActiveUsers(@Param("name") String name, @Param("age") int age);

@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

@Query("SELECT u FROM User u WHERE u.status = 'ACTIVE' ORDER BY u.createdAt DESC")
List<User> findActiveOrderByCreatedAt();
```

```
단순한 조건 조회에도 JPQL 문자열을 직접 관리해야 하는 문제:
  오타 → 컴파일 타임에 발견 불가, 런타임 예외
  조건 추가 시 쿼리 문자열 직접 수정
  메서드 이름과 쿼리 의미가 일치하는지 확인 필요

해결:
  메서드 이름 자체를 DSL로 사용 → 파싱해서 JPQL 자동 생성
  findByNameAndAgeGreaterThan → WHERE name = ? AND age > ?
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Query Method 파싱이 요청 시마다 발생한다

```java
// ❌ 잘못된 이해:
// findByNameAndAge()를 호출할 때마다 메서드 이름을 파싱한다
// → 매 요청마다 파싱 비용 발생

// ✅ 실제:
// 파싱은 애플리케이션 시작 시 1회만 발생
// → 결과(RepositoryQuery 객체)가 Map<Method, RepositoryQuery>에 캐시
// → 런타임 호출 시 캐시에서 꺼내 파라미터만 바인딩
```

### Before: 메서드 이름으로 뭐든 표현 가능하다

```java
// ❌ 과도한 기대:
List<User> findByNameContainingIgnoreCaseAndAgeGreaterThanEqualAndStatusInAndCreatedAtBetweenOrderByNameAscAgeDesc(
    String name, int age, List<Status> statuses, LocalDate from, LocalDate to);
// → 동작은 하지만 가독성 0, 유지보수 불가

// ❌ 지원하지 않는 패턴:
// OR 조건과 AND 조건 혼합 우선순위 제어 불가
List<User> findByNameAndStatusOrAgeGreaterThan(...);
// → (name AND status) OR (age > ?) 인지, name AND (status OR age > ?) 인지 모호
// → Spring Data는 왼쪽부터 순서대로 처리 (우선순위 제어 불가)

// ✅ 기준:
// 조건이 2개 이하 → Query Method
// 조건이 3개 이상, OR 혼합, 서브쿼리 → @Query 또는 Specification/QueryDSL
```

---

## ✨ 올바른 이해와 패턴

### After: Query Method는 단순 조건 조회에만 사용하고, 복잡한 경우 @Query로 전환

```java
// ✅ Query Method가 적합한 경우 (단순, 명확)
Optional<User> findByEmail(String email);
List<User> findByStatus(Status status);
boolean existsByEmail(String email);
long countByStatus(Status status);

// ✅ @Query가 적합한 경우 (복잡, 특수)
@Query("SELECT u FROM User u WHERE u.age BETWEEN :min AND :max AND u.status = 'ACTIVE'")
List<User> findActiveUsersInAgeRange(@Param("min") int min, @Param("max") int max);

// ✅ Specification이 적합한 경우 (동적 조건)
Specification<User> spec = where(nameContains(keyword))
    .and(ageGreaterThan(minAge))
    .and(statusIn(statuses));
userRepository.findAll(spec);
```

---

## 🔬 내부 동작 원리 — PartTree 파싱 소스 추적

### 1. Query Method 등록 시점 — 컨텍스트 시작 시

```java
// QueryExecutorMethodInterceptor 생성자 (컨텍스트 시작 시 1회 실행)
QueryExecutorMethodInterceptor(RepositoryInformation repositoryInformation, ...) {

    this.queries = new HashMap<>();

    // Repository 인터페이스의 모든 메서드를 순회
    for (Method method : repositoryInformation.getQueryMethods()) {

        // 각 메서드에 대해 RepositoryQuery 생성 (파싱 발생!)
        RepositoryQuery query = lookupStrategy.resolveQuery(
            method, repositoryInformation, projectionFactory, namedQueries);

        this.queries.put(method, query);  // 캐시에 저장
    }
    // 이후 호출: queries.get(method) → 캐시된 쿼리 실행
}
```

### 2. Query Lookup Strategy — 쿼리 탐색 순서

```java
// JpaQueryLookupStrategy.CREATE_IF_NOT_FOUND (Spring Boot 기본값)
// 탐색 순서:
//   1. @Query 어노테이션 확인
//   2. Named Query 확인 (persistence.xml 또는 @NamedQuery)
//   3. 없으면 메서드 이름으로 생성 (PartTree 파싱)

public RepositoryQuery resolveQuery(Method method, ...) {

    // 1. @Query 어노테이션 우선
    Query queryAnnotation = method.getAnnotation(Query.class);
    if (queryAnnotation != null) {
        return new SimpleJpaQuery(method, em, queryAnnotation.value(), ...);
    }

    // 2. Named Query 탐색
    String namedQueryName = repositoryInformation.getDomainType().getSimpleName()
                            + "." + method.getName();
    if (namedQueries.hasQuery(namedQueryName)) {
        return new NamedQuery(...);
    }

    // 3. 메서드 이름 파싱 (PartTree)
    return new PartTreeJpaQuery(method, em, ...);
}
```

### 3. PartTree — 메서드 이름 파싱 핵심

```java
// PartTree 생성 — "findByNameAndAgeGreaterThan" 파싱
public PartTree(String source, Class<?> domainClass) {

    // [1] Subject 추출: "find", "read", "get", "count", "exists", "delete"
    // "findByNameAndAgeGreaterThan" → subject="find", predicate="NameAndAgeGreaterThan"
    Matcher matcher = PREFIX_TEMPLATE.matcher(source);
    // PREFIX_TEMPLATE: (find|read|get|query|search|stream|count|exists|delete|remove)(First\d*|Top\d*)?(.*)By(.*)

    this.subject = new Subject(matcher.group(1));   // "find"
    // "Distinct", "Top3", "First5" 등도 여기서 추출

    // [2] Predicate 파싱: "NameAndAgeGreaterThan"
    this.predicate = new Predicate(matcher.group(4), domainClass);
    // "NameAndAgeGreaterThan" → [Part("Name"), Part("AgeGreaterThan")]
}

// OrPart: AND로 연결된 조건들의 묶음
// "NameAndAgeGreaterThan" → OrPart[Part("Name"), Part("AgeGreaterThan")]
// "NameOrEmail" → OrPart[Part("Name")], OrPart[Part("Email")]
```

### 4. Part — 개별 조건 파싱

```java
// Part — 단일 조건 (예: "AgeGreaterThan", "NameContainingIgnoreCase")
class Part {

    private final String property;   // "age"
    private final Type type;         // GreaterThan
    private final IgnoreCaseType ignoreCase;

    public Part(String source, TypeInformation<?> domainType) {
        // 키워드 추출: "AgeGreaterThan" → keyword="GreaterThan", property="Age" → "age"
        // Type.fromProperty()로 키워드 분리

        String[] parts = detectAndSetIgnoreCase(source);
        this.type = Type.fromProperty(parts[0]);   // GreaterThan
        this.property = parts[1];                   // "age"
    }
}

// 지원하는 키워드 → 생성되는 쿼리 조건
// Is, Equals              → WHERE x.field = ?
// Not, IsNot              → WHERE x.field != ?
// Like                    → WHERE x.field LIKE ?
// NotLike                 → WHERE x.field NOT LIKE ?
// StartingWith            → WHERE x.field LIKE ?%
// EndingWith              → WHERE x.field LIKE %?
// Containing              → WHERE x.field LIKE %?%
// GreaterThan             → WHERE x.field > ?
// GreaterThanEqual        → WHERE x.field >= ?
// LessThan                → WHERE x.field < ?
// LessThanEqual           → WHERE x.field <= ?
// Between                 → WHERE x.field BETWEEN ? AND ?
// IsNull, Null            → WHERE x.field IS NULL
// IsNotNull, NotNull      → WHERE x.field IS NOT NULL
// In                      → WHERE x.field IN (?)
// NotIn                   → WHERE x.field NOT IN (?)
// True, False             → WHERE x.field = true/false
// Before, After           → WHERE x.field < / > ? (날짜)
// IgnoreCase              → WHERE LOWER(x.field) = LOWER(?)
// OrderBy...Asc/Desc      → ORDER BY x.field ASC/DESC
```

### 5. PartTree → JPQL 변환 과정

```java
// PartTreeJpaQuery — JPQL 생성 최종 단계
class PartTreeJpaQuery extends AbstractJpaQuery {

    private final PartTree tree;
    private final JpaParameters parameters;

    @Override
    protected Query doCreateQuery(JpaParametersParameterAccessor accessor) {

        // PartTree → CriteriaQuery 변환 (JPA Criteria API 사용!)
        JpaQueryCreator creator = new JpaQueryCreator(tree, domainClass, em.getCriteriaBuilder(), ...);
        CriteriaQuery<?> criteriaQuery = creator.createQuery();

        // CriteriaQuery → TypedQuery
        TypedQuery<?> query = em.createQuery(criteriaQuery);

        // 파라미터 바인딩
        parameterBinder.bind(query, accessor);
        return query;
    }
}

// 결과적으로 생성되는 JPQL (개념적)
// findByNameAndAgeGreaterThan("홍길동", 20)
// → SELECT u FROM User u WHERE u.name = ?1 AND u.age > ?2
```

### 6. 파싱 오류 시점 — 시작 시 즉시 실패

```java
// 잘못된 메서드 이름 → 애플리케이션 시작 시 예외 발생
interface UserRepository extends JpaRepository<User, Long> {
    // "Address"는 User 엔티티에 없는 필드
    List<User> findByAddress(String address);  // PropertyReferenceException!
}

// 예외 메시지:
// No property 'address' found for type 'User'!
// Did you mean 'age'? (유사한 필드 제안)
// → 런타임 NPE 대신 시작 시 즉시 발견 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: 생성되는 SQL 확인

```yaml
# application.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

```java
// 각 Query Method가 생성하는 SQL 확인
userRepository.findByName("홍길동");
// Hibernate: SELECT u1_0.id, u1_0.name, u1_0.age, u1_0.email
//            FROM users u1_0
//            WHERE u1_0.name=?

userRepository.findByNameContainingIgnoreCase("길동");
// Hibernate: SELECT ... WHERE lower(u1_0.name) like lower(?) escape '\'

userRepository.findTop3ByStatusOrderByCreatedAtDesc(Status.ACTIVE);
// Hibernate: SELECT ... WHERE u1_0.status=? ORDER BY u1_0.created_at DESC LIMIT 3
```

### 실험 2: PartTree 직접 파싱

```java
// PartTree를 직접 인스턴스화해서 파싱 결과 확인
@Test
void parsePartTree() {
    PartTree tree = new PartTree(
        "findByNameAndAgeGreaterThan",
        User.class
    );

    System.out.println("Subject: " + tree.isDistinct());
    // 출력: false

    tree.getParts().forEach(part -> {
        System.out.println("Property: " + part.getProperty());
        System.out.println("Type: " + part.getType());
    });
    // Property: name,  Type: SIMPLE_PROPERTY (= equals)
    // Property: age,   Type: GREATER_THAN
}
```

### 실험 3: 캐시 확인 — 동일 메서드 반복 호출

```java
@Test
void queryIsCached() {
    // 첫 호출: 캐시된 RepositoryQuery 실행 (파싱은 시작 시 완료)
    long start = System.nanoTime();
    userRepository.findByName("홍길동");
    long first = System.nanoTime() - start;

    // 두 번째 호출: 동일 비용 (파싱 없음, 캐시 히트)
    start = System.nanoTime();
    userRepository.findByName("김철수");
    long second = System.nanoTime() - start;

    // first ≈ second (DB 조회 시간 제외 시)
    // 파싱은 두 번 모두 발생하지 않음
}
```

---

## ⚡ 성능 임팩트

```
파싱 비용 분석:

[시작 시] PartTree 파싱 비용 (1회)
  단순 메서드 (findByName):           ~0.2ms
  복잡한 메서드 (5개 조건, OrderBy):  ~1~2ms
  Repository 1개 × 10 메서드:         ~2~20ms
  → 시작 시간에 미미한 영향

[런타임] 호출당 비용
  캐시 히트 (Map.get):                ~수백 ns
  파라미터 바인딩:                    ~수 μs
  → 실질적으로 무시 가능

주의: IN 절 메서드의 파라미터 수
  findByIdIn(List<Long> ids) → WHERE id IN (?, ?, ?, ...)
  ids 크기가 매 호출마다 다르면 → Query Plan Cache 미스 가능
  → ids 크기를 고정하거나 @Query로 전환 권장
```

---

## 🤔 트레이드오프

```
Query Method 방식의 장점:
  타입 안전 — 존재하지 않는 필드 참조 시 시작 시 즉시 오류
  간결함 — 단순 조건은 메서드 이름만으로 의도 전달
  자동 JPQL 생성 — SQL/JPQL 직접 작성 불필요

Query Method 방식의 단점:
  복잡한 조건 표현 한계 — OR/AND 우선순위 제어 불가
  가독성 저하 — 조건 3개 이상이면 메서드 이름이 문장 수준
  OR 조건 혼재 — 의도와 다른 결합 순서 위험
  서브쿼리, CASE WHEN — 표현 불가

전환 기준 (실무 가이드):
  조건 1~2개 → Query Method
  조건 3개 이상 → @Query (명시적 JPQL)
  동적 조건 (조건 유무가 런타임에 결정) → Specification 또는 QueryDSL
  복잡한 집계, 서브쿼리 → @Query(nativeQuery=true) 또는 QueryDSL
```

---

## 📌 핵심 정리

```
파싱 시점
  애플리케이션 시작 시 1회 → 결과가 Map<Method, RepositoryQuery>에 캐시
  런타임 호출 시: 캐시에서 꺼내 파라미터만 바인딩

파싱 과정
  메서드명 → PartTree (Subject + Predicate 분리)
  Predicate → OrPart 목록 → Part 목록
  Part → 키워드 + 프로퍼티 추출
  PartTree → CriteriaQuery → TypedQuery (JPA Criteria API 경유)

오류 발견 시점
  잘못된 프로퍼티 참조 → 시작 시 PropertyReferenceException (컴파일 수준 안전성)

지원 키워드 핵심
  Is/Equals, Not, Like, Containing, StartingWith, EndingWith
  GreaterThan(Equal), LessThan(Equal), Between
  IsNull, IsNotNull, In, NotIn, True, False, Before, After
  IgnoreCase, OrderBy, Top/First (개수 제한)

전환 기준
  조건 3개 이상 or OR 혼재 or 동적 조건 → @Query / Specification / QueryDSL
```

---

## 🤔 생각해볼 문제

**Q1.** `findTop5ByStatusOrderByCreatedAtDesc(Status status)` 메서드는 내부적으로 어떻게 `LIMIT 5`를 구현하는가? JPA 표준 방식과 Hibernate의 구현 차이는?

**Q2.** `findByNameIgnoreCaseAndEmail(String name, String email)`에서 `IgnoreCase`는 `name`에만 적용되는가, `email`에도 적용되는가?

**Q3.** `List<User> findByIdIn(List<Long> ids)`를 호출할 때마다 `ids` 리스트의 크기가 다르다. 이것이 Hibernate의 Query Plan Cache에 어떤 영향을 미치는가?

> 💡 **해설**
>
> **Q1.** `PartTree`의 `Subject`가 `Top5`를 파싱하여 `isLimiting() = true`, `maxResults = 5`로 저장한다. `JpaQueryCreator`가 `CriteriaQuery`를 생성할 때 `query.setMaxResults(5)`를 호출한다. JPA 표준 방식인 `TypedQuery.setMaxResults(5)`로 처리되며, Hibernate는 이를 데이터베이스 방언(Dialect)에 맞게 `LIMIT 5`(MySQL), `FETCH FIRST 5 ROWS ONLY`(DB2), `ROWNUM <= 5`(Oracle 구버전) 등으로 변환한다.
>
> **Q2.** `NameIgnoreCase`는 `Name`에만 적용된다. `Part` 파싱 시 `IgnoreCase` 키워드는 해당 Part의 프로퍼티(`name`)에만 결합된다. `email`은 별도의 `Part`로 파싱되며 `IgnoreCase`가 붙지 않는다. 전체 메서드에 IgnoreCase를 적용하려면 `findByNameIgnoreCaseAndEmailIgnoreCase()`로 명시해야 한다. `AllIgnoreCase` 키워드(`findByNameAndEmailAllIgnoreCase`)를 사용하면 모든 조건에 일괄 적용할 수 있다.
>
> **Q3.** Hibernate의 Query Plan Cache는 파라미터 수가 다른 IN 절을 다른 쿼리로 취급한다. `findByIdIn([1,2,3])`과 `findByIdIn([1,2,3,4,5])`는 서로 다른 캐시 엔트리를 생성한다(`WHERE id IN (?,?,?)`와 `WHERE id IN (?,?,?,?,?)` 는 다른 쿼리). 매 호출마다 크기가 다르면 캐시 미스가 반복되어 JPQL 파싱 비용이 누적된다. `hibernate.query.plan_cache_max_size` 기본값(2048)을 초과하면 오래된 항목이 제거되며 메모리 압박도 발생한다. 해결책은 `@BatchSize` 또는 ids를 고정 크기의 청크로 분할하는 것이다.

---

<div align="center">

**[⬅️ 이전: Repository 프록시 생성 과정](./01-repository-proxy-creation.md)** | **[다음: @Query JPQL vs Native Query 처리 ➡️](./03-query-annotation-jpql-native.md)**

</div>
