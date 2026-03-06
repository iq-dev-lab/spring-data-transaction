# JPQL vs Criteria API vs QueryDSL — 세 가지 쿼리 방식의 완전 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JPQL 문자열이 런타임에 SQL로 변환되는 정확한 경로는?
- Criteria API가 타입 안전하지만 가독성이 나쁜 근본적인 이유는?
- QueryDSL이 Q타입을 생성하는 방식과 컴파일 타임 검증이 동작하는 원리는?
- 세 방식이 결국 동일한 Hibernate `Query` 실행 경로로 수렴하는 이유는?
- 동적 쿼리 구성에서 JPQL의 String concatenation이 위험한 이유와 QueryDSL의 대안은?

---

## 🔍 왜 이게 존재하는가

### 문제: 문자열 JPQL은 런타임에야 오류가 발견된다

```java
// ❌ JPQL 문자열 — 런타임에야 오류 발견
@Query("SELECT u FROM Uesr WHERE u.status = :status")
//                  ↑ 오타 — 컴파일 통과, 애플리케이션 시작 시 또는 최초 호출 시 예외
List<User> findByStatus(@Param("status") Status status);

// ❌ 동적 JPQL — String concatenation은 위험
public List<User> search(String name, Status status) {
    String jpql = "SELECT u FROM User u WHERE 1=1";
    if (name != null) jpql += " AND u.name = '" + name + "'";  // SQL Injection!
    if (status != null) jpql += " AND u.status = '" + status + "'";
    return em.createQuery(jpql, User.class).getResultList();
}
```

```
각 방식이 해결하는 문제:

JPQL:     SQL과 유사한 문자열 → 직관적, 빠른 작성
          단점: 런타임 오류, 동적 쿼리 어려움

Criteria API: 자바 코드로 쿼리 구성 → 컴파일 타임 검증
              단점: 장황한 코드, 가독성 저하

QueryDSL: Q타입 기반 → 컴파일 타임 검증 + 가독성
          단점: Q타입 생성을 위한 APT 설정 필요
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: QueryDSL은 JPQL보다 성능이 좋다

```java
// ❌ 잘못된 이해: QueryDSL이 더 빠른 SQL을 생성한다

// ✅ 실제:
// JPQL → HQL → SQL (Hibernate 변환)
// Criteria API → HQL → SQL
// QueryDSL → JPQL → HQL → SQL

// 모두 동일한 Hibernate SQL 생성 경로를 거침
// 성능 차이는 생성되는 SQL에 의존하며, 동일한 조건이면 동일한 SQL 생성
// QueryDSL의 장점은 개발 생산성과 타입 안전성이지 SQL 성능이 아님
```

### Before: Criteria API는 타입 안전하므로 항상 선호해야 한다

```java
// ❌ Criteria API — 간단한 쿼리도 극도로 장황
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
Join<User, Team> teamJoin = root.join("team", JoinType.LEFT);
cq.select(root)
  .where(
      cb.and(
          cb.equal(root.get("status"), Status.ACTIVE),
          cb.like(root.get("name"), "%" + name + "%"),
          cb.greaterThan(root.get("age"), 20)
      )
  )
  .orderBy(cb.desc(root.get("createdAt")));
// 조건 3개인데 코드 10줄 → 유지보수 어려움

// ✅ QueryDSL로 동일 쿼리
List<User> result = queryFactory
    .selectFrom(user)
    .leftJoin(user.team, team)
    .where(
        user.status.eq(Status.ACTIVE),
        user.name.contains(name),
        user.age.gt(20)
    )
    .orderBy(user.createdAt.desc())
    .fetch();
// 훨씬 읽기 쉽고 타입 안전
```

---

## ✨ 올바른 이해와 패턴

### After: 세 방식의 최종 실행 경로 수렴

```
JPQL 문자열
  "SELECT u FROM User u WHERE u.status = :status"
     ↓ HqlParser (ANTLR)
  AST (Abstract Syntax Tree)
     ↓ SemanticQueryBuilder
  SqmSelectStatement (Semantic Query Model)
     ↓ SqlAstTranslator
  SQL: "SELECT u.id, u.name, u.status FROM users u WHERE u.status = ?"
     ↓ JDBC PreparedStatement

Criteria API
  CriteriaQuery → JpaCriteriaQuery
     ↓ 내부적으로 SqmSelectStatement 생성
  → 동일 SqlAstTranslator 경로

QueryDSL
  JPQLQuery (com.querydsl) → toString() → JPQL 문자열 생성
     ↓ em.createQuery(jpqlString)
  → 동일 HqlParser 경로

결론: 세 방식 모두 동일한 SQL 생성 엔진(Hibernate)을 거침
```

---

## 🔬 내부 동작 원리 — 세 방식의 실행 경로 추적

### 1. JPQL → SQL 변환 경로 (Hibernate 6.x)

```java
// em.createQuery("SELECT u FROM User u WHERE u.status = :status", User.class)
// → SharedEntityManagerInvocationHandler → SessionImpl

// SessionImpl.createQuery()
public <T> TypedQuery<T> createQuery(String queryString, Class<T> expectedResultType) {
    // [1] JPQL 파싱 → SQM (Semantic Query Model)
    HqlParsingContext parsingContext = new HqlParsingContext(this);
    SqmSelectStatement<T> sqm = (SqmSelectStatement<T>)
        new HqlParser().parse(queryString, parsingContext);
    // → ANTLR4로 JPQL 문법 파싱
    // → 엔티티/속성 이름을 DB 컬럼/테이블로 매핑
    // → 오류 시: HibernateException ("User" 엔티티 없음, "status" 속성 없음)

    // [2] SQM → SQL AST
    QueryOptions queryOptions = new SimpleQueryOptions(...);
    SqmSelectTranslation<T> sqmTranslation =
        new StandardSqmTranslator<T>(sqm, queryOptions, ...).translate();
    // → SqmFromClause → TableGroupJoin
    // → SqmWhereClause → Predicate
    // → 실제 DB 컬럼명/테이블명으로 변환

    // [3] SQL AST → SQL 문자열 (PreparedStatement)
    JdbcSelect jdbcSelect = new StandardSqlAstTranslator<>(
        sessionFactory, sqmTranslation.getSqlAst()).translate(null, queryOptions);
    // → SELECT u.id, u.name, u.status FROM users u WHERE u.status = ?

    return new QuerySqmImpl<>(sqm, expectedResultType, this);
    // → setParameter("status", ...) 후 getResultList() 호출 시 JDBC 실행
}
```

### 2. Criteria API → SQM 직접 생성

```java
// Criteria API는 JPQL 파싱을 건너뜀 — SQM 직접 생성
CriteriaBuilder cb = em.getCriteriaBuilder();
// → SessionImpl.getCriteriaBuilder() → HibernateCriteriaBuilder

CriteriaQuery<User> cq = cb.createQuery(User.class);
// → SqmSelectStatement<User> 생성 (내부적으로)

Root<User> root = cq.from(User.class);
// → SqmRoot<User> 생성 → FROM 절 등록

cq.where(cb.equal(root.get("status"), Status.ACTIVE));
// → SqmComparisonPredicate(root.get("status"), EQUAL, Status.ACTIVE)
// → WHERE 절에 추가

// 실행: em.createQuery(cq)
// → CriteriaQueryImpl → 내부 SQM에서 직접 SqlAstTranslator로
// → HQL 파싱 단계 없음 (약간 빠를 수 있으나 실측 차이 미미)
```

### 3. QueryDSL 내부 — JPQL 문자열 생성

```java
// QueryDSL JPAQuery 실행 흐름
JPAQuery<User> query = new JPAQuery<>(em);  // 또는 JPAQueryFactory

query.selectFrom(user)
     .where(user.status.eq(Status.ACTIVE))
     .fetch();

// 내부:
// [1] Q타입으로 표현식 트리 구성
//   QUser user = QUser.user;
//   user.status.eq(Status.ACTIVE) → EqOperation(PathExpression("status"), Status.ACTIVE)

// [2] JPQLSerializer — 표현식 트리 → JPQL 문자열 변환
class JPQLSerializer extends SerializerBase<JPQLSerializer> {
    public void serialize(QueryMetadata metadata, boolean forCountRow, String projection) {
        // SELECT 절
        visit(metadata.getProjection(), ...);
        // FROM 절
        for (JoinExpression join : metadata.getJoins()) {
            visit(join, ...);
        }
        // WHERE 절
        visit(metadata.getWhere(), ...);
    }
    // 최종: "SELECT user FROM User user WHERE user.status = ?1"
}

// [3] em.createQuery(jpqlString, User.class) 호출 → Hibernate 파싱 경로
// → JPQL 문자열을 한 번 더 파싱하는 오버헤드 있음 (단, QueryPlanCache로 캐시)
```

### 4. Q타입 생성 — APT(Annotation Processing Tool)

```java
// @Entity User 클래스
@Entity
public class User {
    @Id Long id;
    String name;
    Status status;
    @ManyToOne Team team;
}

// APT(javax.annotation.processing)가 컴파일 시 QUser 자동 생성
// build/generated/sources/annotationProcessor/.../QUser.java
public class QUser extends EntityPathBase<User> {

    public static final QUser user = new QUser("user");

    public final NumberPath<Long> id = createNumber("id", Long.class);
    public final StringPath name = createString("name");
    public final EnumPath<Status> status = createEnum("status", Status.class);
    public final QTeam team;  // 연관관계도 Q타입으로

    public QUser(String variable) {
        super(User.class, variable);
        team = new QTeam(forProperty("team"));
    }
}

// 컴파일 타임 검증:
user.name.eq("test")    // ✅ StringPath.eq() → 타입 일치
user.name.eq(123)       // ❌ 컴파일 오류 — StringPath에 Integer 불가
user.noSuchField        // ❌ 컴파일 오류 — 존재하지 않는 필드
```

### 5. 동적 쿼리 — 세 방식 비교

```java
// [JPQL] 동적 조건 — 위험하고 복잡
public List<User> search(String name, Status status, Integer minAge) {
    StringBuilder jpql = new StringBuilder("SELECT u FROM User u WHERE 1=1");
    Map<String, Object> params = new HashMap<>();

    if (name != null) {
        jpql.append(" AND u.name LIKE :name");
        params.put("name", "%" + name + "%");
    }
    if (status != null) {
        jpql.append(" AND u.status = :status");
        params.put("status", status);
    }
    if (minAge != null) {
        jpql.append(" AND u.age >= :minAge");
        params.put("minAge", minAge);
    }

    TypedQuery<User> query = em.createQuery(jpql.toString(), User.class);
    params.forEach(query::setParameter);
    return query.getResultList();
    // 조건 3개인데 이미 복잡, 조건 10개면 유지보수 불가
}

// [Criteria API] 동적 조건 — 타입 안전하지만 여전히 장황
public List<User> searchCriteria(String name, Status status, Integer minAge) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> cq = cb.createQuery(User.class);
    Root<User> root = cq.from(User.class);

    List<Predicate> predicates = new ArrayList<>();
    if (name != null) predicates.add(cb.like(root.get("name"), "%" + name + "%"));
    if (status != null) predicates.add(cb.equal(root.get("status"), status));
    if (minAge != null) predicates.add(cb.greaterThanOrEqualTo(root.get("age"), minAge));

    cq.where(predicates.toArray(new Predicate[0]));
    return em.createQuery(cq).getResultList();
}

// [QueryDSL] 동적 조건 — 가장 깔끔
public List<User> searchQueryDsl(String name, Status status, Integer minAge) {
    return queryFactory
        .selectFrom(user)
        .where(
            nameContains(name),     // null이면 BooleanExpression null → 자동 무시
            statusEq(status),
            minAgeGoe(minAge)
        )
        .fetch();
}

// BooleanExpression 반환: null이면 where()에서 자동 제외
private BooleanExpression nameContains(String name) {
    return name != null ? user.name.contains(name) : null;
}
private BooleanExpression statusEq(Status status) {
    return status != null ? user.status.eq(status) : null;
}
private BooleanExpression minAgeGoe(Integer minAge) {
    return minAge != null ? user.age.goe(minAge) : null;
}
// → 조건 추가/제거가 메서드 하나 추가/삭제로 해결
// → null 조건 자동 처리 (별도 if문 불필요)
```

---

## 💻 실험으로 확인하기

### 실험 1: 동일 SQL 생성 확인

```yaml
# application.yml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

```java
// 세 방식으로 동일 쿼리 실행
// JPQL
em.createQuery("SELECT u FROM User u WHERE u.status = :s", User.class)
  .setParameter("s", Status.ACTIVE).getResultList();

// Criteria
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> r = cq.from(User.class);
cq.where(cb.equal(r.get("status"), Status.ACTIVE));
em.createQuery(cq).getResultList();

// QueryDSL
queryFactory.selectFrom(user).where(user.status.eq(Status.ACTIVE)).fetch();

// 세 경우 모두 동일 SQL 출력:
// select u1_0.id, u1_0.name, u1_0.status
// from users u1_0
// where u1_0.status=?
```

### 실험 2: QueryPlan 캐시 히트 확인

```java
// 같은 JPQL 두 번 실행
String jpql = "SELECT u FROM User u WHERE u.status = :s";
em.createQuery(jpql, User.class).setParameter("s", Status.ACTIVE).getResultList();
// → 파싱 + 캐시 저장

em.createQuery(jpql, User.class).setParameter("s", Status.INACTIVE).getResultList();
// → 캐시 히트 (파싱 없음)

// Hibernate Statistics로 캐시 히트 확인
SessionFactory sf = emf.unwrap(SessionFactory.class);
Statistics stats = sf.getStatistics();
stats.setStatisticsEnabled(true);
System.out.println("QueryPlan cache hits: " + stats.getQueryPlanCacheHitCount());
```

---

## ⚡ 성능 임팩트

```
파싱/컴파일 비용 비교 (첫 실행, QueryPlanCache 미스 시):

JPQL:         ANTLR4 파싱 → SQM 변환 → SQL 생성 → ~수 ms
Criteria API: SQM 직접 생성 → SQL 생성 → ~수 ms (파싱 생략, 약간 빠름)
QueryDSL:     JPQLSerializer → JPQL 문자열 → ANTLR4 파싱 → ~수 ms (약간 더 걸림)

두 번째 실행 이후 (QueryPlanCache 히트):
  JPQL / Criteria / QueryDSL 모두 → SQL 생성만 → ~수십 μs
  차이 무시 가능

실무 성능 결론:
  세 방식의 SQL 실행 성능은 동일 (같은 SQL)
  파싱 비용은 QueryPlanCache로 상쇄
  선택 기준: 개발 생산성, 타입 안전성, 동적 쿼리 편의성

개발 생산성 (동적 쿼리 조건 10개 기준):
  JPQL:         코드 40줄, 런타임 오류 위험
  Criteria API: 코드 30줄, 컴파일 안전하지만 가독성 나쁨
  QueryDSL:     코드 15줄, 컴파일 안전 + 가독성 우수
```

---

## 🤔 트레이드오프

```
JPQL:
  ✅ 직관적, SQL 경험자에게 친숙
  ✅ 간단한 정적 쿼리에 적합
  ❌ 런타임 오류 (오타, 존재하지 않는 필드)
  ❌ 동적 쿼리 구성 복잡
  → 단순 정적 쿼리, @Query 어노테이션

Criteria API:
  ✅ 컴파일 타임 검증 (타입 안전)
  ✅ 외부 의존성 없음 (JPA 표준)
  ❌ 극도로 장황, 가독성 나쁨
  ❌ 동적 쿼리도 복잡
  → 거의 사용 안 함, QueryDSL로 대체

QueryDSL:
  ✅ 컴파일 타임 검증 + 높은 가독성
  ✅ 동적 쿼리가 자연스럽고 안전
  ✅ IDE 자동완성 완벽 지원
  ❌ APT 설정 필요 (빌드 복잡성)
  ❌ Q타입 재생성 필요 (엔티티 변경 시)
  → 복잡한 동적 쿼리, 대부분의 실무 프로젝트

실무 패턴 (권장):
  단순 CRUD + 정적 조회: Spring Data JPA @Query (JPQL)
  복잡한 동적 쿼리: QueryDSL (Custom Repository)
  Criteria API: 거의 사용 안 함
```

---

## 📌 핵심 정리

```
세 방식의 실행 경로
  JPQL → HqlParser → SQM → SqlAstTranslator → SQL
  Criteria API → SQM 직접 → SqlAstTranslator → SQL
  QueryDSL → JPQLSerializer → JPQL → HqlParser → SQM → SQL
  → 모두 동일한 Hibernate SQL 생성 엔진 수렴

QueryDSL Q타입
  APT가 @Entity 클래스를 분석해 컴파일 시 자동 생성
  Path 타입 계층: StringPath, NumberPath, EnumPath 등
  타입 불일치 → 컴파일 오류

동적 쿼리 QueryDSL 패턴
  BooleanExpression 반환 메서드 분리
  null 반환 시 where()에서 자동 제외
  조건 추가/삭제 = 메서드 추가/삭제

성능 선택 기준
  세 방식 SQL 실행 성능 동일
  개발 생산성, 타입 안전성, 가독성 → QueryDSL 우위
  단순 정적 쿼리 → @Query JPQL
```

---

## 🤔 생각해볼 문제

**Q1.** QueryDSL이 생성하는 JPQL 문자열에 파라미터가 `?1`, `?2` 같은 위치 기반 바인딩으로 포함된다면, QueryPlanCache 히트율에 어떤 영향을 주는가?

**Q2.** Criteria API에서 `root.get("status")`는 문자열로 필드명을 지정한다. 이것이 타입 안전하다고 볼 수 있는가? JPA Metamodel을 사용하면 어떻게 달라지는가?

**Q3.** QueryDSL과 Spring Data JPA를 함께 사용할 때 `QuerydslPredicateExecutor`를 사용하는 방식과 Custom Repository에 `JPAQueryFactory`를 직접 주입하는 방식의 차이점과 각각의 한계는?

> 💡 **해설**
>
> **Q1.** QueryDSL은 파라미터를 JPQL 문자열에 `?1` 형태로 포함하지 않고 `setParameter()`로 바인딩한다. JPQLSerializer가 생성하는 문자열은 파라미터 자리에 `:param0` 같은 이름 기반 바인딩 또는 위치 기반 `?1`을 사용한다. 파라미터 값이 문자열에 포함되지 않으므로 같은 구조의 쿼리는 동일한 캐시 키를 가지고 QueryPlanCache를 재사용한다. 위치 기반이든 이름 기반이든 파라미터 바인딩 방식은 캐시 히트율에 영향을 주지 않는다 — 중요한 것은 파라미터 값이 아닌 쿼리 구조(SQL 패턴)가 캐시 키이기 때문이다.
>
> **Q2.** `root.get("status")`는 문자열로 필드명을 지정하므로 오타가 있어도 컴파일 오류가 발생하지 않는다 — 런타임에 `IllegalArgumentException`이 발생한다. JPA Metamodel을 사용하면 `root.get(User_.status)`처럼 자동 생성된 `User_` 메타모델 클래스의 정적 필드를 사용하여 컴파일 타임 검증이 가능하다. 단 Metamodel 자동 생성도 APT 설정이 필요하며, 가독성은 QueryDSL보다 떨어진다. 결론적으로 순수 Criteria API는 완전한 타입 안전성을 갖추려면 Metamodel이 필수이며, 그 복잡성 때문에 QueryDSL로 대체되는 경향이 있다.
>
> **Q3.** `QuerydslPredicateExecutor`는 `findAll(Predicate)` 형태로 간단히 사용할 수 있지만 Join, Projection, 서브쿼리 등 복잡한 쿼리를 표현할 수 없다는 한계가 있다. 반환 타입도 엔티티로 고정되어 DTO Projection이 불가능하다. `JPAQueryFactory` 직접 주입 방식은 JOIN, 서브쿼리, DTO Projection, 동적 정렬 등 모든 QueryDSL 기능을 사용할 수 있지만 Custom Repository 인터페이스와 구현체를 별도로 작성해야 한다. 실무에서는 단순 필터링은 `QuerydslPredicateExecutor`, 복잡한 조회는 `JPAQueryFactory` Custom Repository로 나누어 사용하는 것이 일반적이다.

---

<div align="center">

**[다음: Fetch Join vs @EntityGraph 선택 기준 ➡️](./02-fetch-join-vs-entity-graph.md)**

</div>
