# Specifications를 통한 동적 쿼리 — JpaSpecificationExecutor와 Criteria API 래핑

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Specification<T>`는 내부적으로 JPA Criteria API를 어떻게 래핑하는가?
- `JpaSpecificationExecutor`가 `findAll(Specification)` 호출 시 실제로 어떤 과정을 거치는가?
- `Specification.and()` / `Specification.or()`로 조합할 때 생성되는 Predicate 구조는?
- Specifications와 QueryDSL `Predicate` 방식의 선택 기준은?
- Specification에서 연관 엔티티 조인을 처리할 때 주의해야 할 중복 조인 문제는?

---

## 🔍 왜 이게 존재하는가

### 문제: 동적 조건 조합을 타입 안전하게 재사용하기 어렵다

```java
// 문제: 조건별 Repository 메서드 폭발
List<User> findByStatus(Status status);
List<User> findByStatusAndAgeGreaterThan(Status status, int age);
List<User> findByStatusAndAgeGreaterThanAndNameContaining(Status status, int age, String name);
// 조건 조합 N개 → 메서드 N개

// 또는 모든 조건을 받는 단일 메서드 — null 조건 처리 복잡
@Query("SELECT u FROM User u WHERE " +
       "(:status IS NULL OR u.status = :status) AND " +
       "(:minAge IS NULL OR u.age >= :minAge) AND " +
       "(:name IS NULL OR u.name LIKE %:name%)")
List<User> findWithConditions(...);
// → NULL 처리 JPQL이 DB 최적화 방해 가능 (실행 계획 오염)
```

```
Specification의 해결:
  조건 하나 = Specification 하나 (단일 책임)
  Specification.and() / .or()로 런타임 조합
  재사용 가능한 조건 블록 → UserSpec.hasStatus(), UserSpec.ageGoe()
  null-safe 조합 → null Specification은 조건에서 제외
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Specification으로 연관 엔티티 조인 시 중복 조인이 발생한다

```java
// ❌ 중복 조인 위험: 여러 Specification이 같은 연관 엔티티를 독립적으로 조인
Specification<User> spec1 = (root, query, cb) ->
    cb.equal(root.join("team").get("name"), "개발팀");  // users JOIN teams

Specification<User> spec2 = (root, query, cb) ->
    cb.greaterThan(root.join("team").get("memberCount"), 10);  // 또 users JOIN teams

Specification<User> combined = spec1.and(spec2);
userRepository.findAll(combined);
// 실행되는 SQL: ... FROM users u
//              JOIN teams t1 ON u.team_id = t1.id   ← spec1 조인
//              JOIN teams t2 ON u.team_id = t2.id   ← spec2 조인 (중복!)
// → 잘못된 결과 또는 성능 저하

// ✅ join() 대신 fetch 사용하거나, root.getJoins()로 기존 조인 재사용
Specification<User> spec = (root, query, cb) -> {
    // 이미 조인된 것이 있으면 재사용
    Join<User, Team> team = getOrCreateJoin(root, "team");
    return cb.and(
        cb.equal(team.get("name"), "개발팀"),
        cb.greaterThan(team.get("memberCount"), 10)
    );
};
```

### Before: count 쿼리에도 Fetch Join이 적용된다

```java
// ❌ Pagination + Fetch Join Specification
Specification<User> withTeam = (root, query, cb) -> {
    root.fetch("team");  // Fetch Join — count 쿼리에서 문제 발생!
    return cb.conjunction();
};

Page<User> page = userRepository.findAll(withTeam, pageable);
// count 쿼리: SELECT count(u) FROM User u JOIN FETCH u.team
// → HibernateException 또는 경고 발생
// FETCH JOIN은 count 쿼리에 적합하지 않음

// ✅ count 쿼리와 데이터 쿼리를 구분
Specification<User> withTeam = (root, query, cb) -> {
    // count 쿼리(Long 반환)에서는 fetch 적용 안 함
    if (Long.class != query.getResultType()) {
        root.fetch("team", JoinType.LEFT);
    }
    return cb.conjunction();
};
```

---

## ✨ 올바른 이해와 패턴

### After: Specification을 재사용 가능한 조건 블록으로 설계

```java
// UserSpec — 재사용 가능한 Specification 정의
public class UserSpec {

    // null 안전: 값이 없으면 항상 참(null을 반환하면 Spring Data가 무시)
    public static Specification<User> hasStatus(Status status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<User> ageGoe(Integer minAge) {
        return (root, query, cb) ->
            minAge == null ? null : cb.greaterThanOrEqualTo(root.get("age"), minAge);
    }

    public static Specification<User> nameLike(String name) {
        return (root, query, cb) ->
            !StringUtils.hasText(name) ? null :
            cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%");
    }

    public static Specification<User> inTeam(String teamName) {
        return (root, query, cb) -> {
            if (!StringUtils.hasText(teamName)) return null;
            Join<User, Team> team = root.join("team", JoinType.LEFT);
            return cb.equal(team.get("name"), teamName);
        };
    }
}

// 사용: 조건 조합이 자유롭고 가독성 높음
Specification<User> spec = Specification
    .where(UserSpec.hasStatus(condition.getStatus()))
    .and(UserSpec.ageGoe(condition.getMinAge()))
    .and(UserSpec.nameLike(condition.getName()))
    .and(UserSpec.inTeam(condition.getTeamName()));

List<User> users = userRepository.findAll(spec);
```

---

## 🔬 내부 동작 원리 — JpaSpecificationExecutor 소스 추적

### 1. Specification 인터페이스 — 함수형 인터페이스

```java
// Specification<T> — JPA Criteria API의 Predicate를 생성하는 함수형 인터페이스
@FunctionalInterface
public interface Specification<T> extends Serializable {

    // 핵심 메서드: CriteriaBuilder로 Predicate 생성
    @Nullable
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);

    // 정적 팩토리 메서드
    static <T> Specification<T> not(Specification<T> spec) {
        return SpecificationComposition.negated(spec);
    }

    static <T> Specification<T> where(Specification<T> spec) {
        return spec == null ? (r, q, cb) -> null : spec;
    }

    // 조합 메서드 (default)
    default Specification<T> and(Specification<T> other) {
        return SpecificationComposition.composed(this, other, CriteriaBuilder::and);
    }

    default Specification<T> or(Specification<T> other) {
        return SpecificationComposition.composed(this, other, CriteriaBuilder::or);
    }
}
```

### 2. SpecificationComposition — and/or 조합 내부

```java
// SpecificationComposition — Specification 조합 처리
abstract class SpecificationComposition {

    static <T> Specification<T> composed(
            Specification<T> lhs, Specification<T> rhs,
            BiFunction<CriteriaBuilder, Predicate[], Predicate> combiner) {

        return (root, query, builder) -> {

            // 각 Specification의 toPredicate() 호출
            Predicate leftPredicate  = toPredicate(lhs, root, query, builder);
            Predicate rightPredicate = toPredicate(rhs, root, query, builder);

            // null 처리 — null이면 해당 조건 제외
            if (leftPredicate == null) return rightPredicate;
            if (rightPredicate == null) return leftPredicate;

            // cb.and(left, right) 또는 cb.or(left, right) 호출
            return combiner.apply(builder, new Predicate[]{leftPredicate, rightPredicate});
        };
    }

    private static <T> Predicate toPredicate(
            Specification<T> specification, Root<T> root,
            CriteriaQuery<?> query, CriteriaBuilder builder) {
        return specification == null ? null : specification.toPredicate(root, query, builder);
    }
}
```

### 3. JpaSpecificationExecutor — findAll 실행 과정

```java
// SimpleJpaRepository — JpaSpecificationExecutor 구현
@Override
public List<T> findAll(Specification<T> spec) {
    return getQuery(spec, Sort.unsorted()).getResultList();
}

@Override
public Page<T> findAll(Specification<T> spec, Pageable pageable) {
    TypedQuery<T> query = getQuery(spec, pageable);

    return isUnpaged(pageable)
        ? new PageImpl<>(query.getResultList())
        : readPage(query, getDomainClass(), pageable, spec);
}

// 핵심: Specification → CriteriaQuery 변환
protected TypedQuery<T> getQuery(@Nullable Specification<T> spec, Sort sort) {

    CriteriaBuilder builder = em.getCriteriaBuilder();
    CriteriaQuery<T> query = builder.createQuery(getDomainClass());

    Root<T> root = applySpecificationToCriteria(spec, getDomainClass(), query);
    // ↑ Specification.toPredicate()를 호출하여 WHERE 절 구성

    query.select(root);

    if (sort.isSorted()) {
        query.orderBy(toOrders(sort, root, builder));
    }

    return applyRepositoryMethodMetadata(em.createQuery(query));
}

// Specification → Predicate 적용
private <S, U extends T> Root<U> applySpecificationToCriteria(
        @Nullable Specification<U> spec,
        Class<U> domainClass,
        CriteriaQuery<S> query) {

    Root<U> root = query.from(domainClass);

    if (spec == null) return root;

    CriteriaBuilder builder = em.getCriteriaBuilder();
    Predicate predicate = spec.toPredicate(root, query, builder);
    // ← 여기서 Specification 체인 실행
    // spec1.and(spec2).and(spec3).toPredicate() 재귀 호출

    if (predicate != null) {
        query.where(predicate);
    }

    return root;
}
```

### 4. count 쿼리 처리 — Long 타입 분기

```java
// Pagination 시 count 쿼리 생성
protected TypedQuery<Long> getCountQuery(@Nullable Specification<T> spec, Class<T> domainClass) {

    CriteriaBuilder builder = em.getCriteriaBuilder();
    CriteriaQuery<Long> query = builder.createQuery(Long.class);

    Root<T> root = applySpecificationToCriteria(spec, domainClass, query);
    // ← 동일한 Specification이지만 CriteriaQuery<Long>으로 실행

    if (query.isDistinct()) {
        query.select(builder.countDistinct(root));
    } else {
        query.select(builder.count(root));
    }
    query.orderBy(Collections.emptyList());  // count에 ORDER BY 불필요

    return em.createQuery(query);
}

// Specification 내에서 count vs 데이터 쿼리 구분
Specification<User> withTeamFetch = (root, query, cb) -> {
    // query.getResultType()으로 count 쿼리 여부 판별
    if (Long.class != query.getResultType()) {
        // 데이터 쿼리: Fetch Join 적용
        root.fetch("team", JoinType.LEFT);
    }
    // count 쿼리: Fetch Join 미적용
    return cb.conjunction();  // 항상 참 (다른 Specification과 조합 목적)
};
```

### 5. 중복 조인 방지 — getJoins() 재사용

```java
// 안전한 Join 처리 — 이미 조인된 것 재사용
public class UserSpec {

    public static Specification<User> hasTeamName(String teamName) {
        return (root, query, cb) -> {
            if (!StringUtils.hasText(teamName)) return null;
            // 이미 "team"으로 join된 것이 있으면 재사용
            Join<User, Team> teamJoin = getOrCreateTeamJoin(root);
            return cb.equal(teamJoin.get("name"), teamName);
        };
    }

    public static Specification<User> hasTeamMemberCountGoe(int count) {
        return (root, query, cb) -> {
            Join<User, Team> teamJoin = getOrCreateTeamJoin(root);
            return cb.greaterThanOrEqualTo(teamJoin.get("memberCount"), count);
        };
    }

    @SuppressWarnings("unchecked")
    private static Join<User, Team> getOrCreateTeamJoin(Root<User> root) {
        // 기존 조인 탐색
        for (Join<User, ?> join : root.getJoins()) {
            if (join.getAttribute().getName().equals("team")) {
                return (Join<User, Team>) join;  // 재사용
            }
        }
        // 없으면 새로 생성
        return root.join("team", JoinType.LEFT);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Specification 조합 SQL 확인

```java
@Test
void specificationCombination() {
    // 모든 조건
    Specification<User> spec = Specification
        .where(UserSpec.hasStatus(Status.ACTIVE))
        .and(UserSpec.ageGoe(20))
        .and(UserSpec.nameLike("길동"));

    List<User> result = userRepository.findAll(spec);
    // SQL: SELECT ... FROM users u
    //      WHERE u.status = ? AND u.age >= ? AND lower(u.name) LIKE ?

    // null 조건 제외 확인
    Specification<User> partialSpec = Specification
        .where(UserSpec.hasStatus(Status.ACTIVE))
        .and(UserSpec.ageGoe(null))        // null → 조건 제외
        .and(UserSpec.nameLike(""));       // 빈 문자열 → 조건 제외

    List<User> partialResult = userRepository.findAll(partialSpec);
    // SQL: SELECT ... FROM users u WHERE u.status = ?
    // ageGoe, nameLike 조건 없음
}
```

### 실험 2: Pagination과 Specification

```java
@Test
void paginationWithSpec() {
    Specification<User> spec = UserSpec.hasStatus(Status.ACTIVE);
    Pageable pageable = PageRequest.of(0, 10, Sort.by("name"));

    Page<User> page = userRepository.findAll(spec, pageable);
    // 실행 쿼리 2개:
    // 1. SELECT ... FROM users u WHERE u.status=? ORDER BY u.name LIMIT 10
    // 2. SELECT count(u.id) FROM users u WHERE u.status=?
}
```

### 실험 3: Specification vs QueryDSL 동적 쿼리 코드 비교

```java
// Specification 방식
Specification<User> spec = Specification
    .where(condition.getStatus() != null ? UserSpec.hasStatus(condition.getStatus()) : null)
    .and(UserSpec.ageGoe(condition.getMinAge()))
    .and(UserSpec.nameLike(condition.getName()));

// QueryDSL 방식 (동등한 코드)
BooleanBuilder builder = new BooleanBuilder();
if (condition.getStatus() != null) builder.and(QUser.user.status.eq(condition.getStatus()));
if (condition.getMinAge() != null) builder.and(QUser.user.age.goe(condition.getMinAge()));
if (hasText(condition.getName())) builder.and(QUser.user.name.contains(condition.getName()));

// 코드량은 유사하지만 Specification은 재사용성, QueryDSL은 타입 안전성이 강점
```

---

## ⚡ 성능 임팩트

```
Specification 실행 비용:
  toPredicate() 호출: ~수 μs (재귀 조합 포함)
  CriteriaQuery 생성: ~0.1~0.5ms
  JPQL 직렬화 + Query Plan Cache: 첫 실행 후 캐시
  → 실질적 오버헤드는 무시 가능

동적 조건에 따른 실행 계획 변화:
  조건 조합마다 다른 WHERE 절 → DB 실행 계획 캐시 미스 가능
  PreparedStatement 파라미터화: 파라미터 수가 같으면 캐시 공유
  → 조건 유무에 따라 다른 실행 계획 → DB 옵티마이저에 의존

중복 조인 성능 영향:
  같은 테이블을 2번 JOIN → DB가 중복 제거 못하면 성능 저하
  특히 대용량 테이블에서 치명적
  → getOrCreateJoin() 패턴으로 반드시 방지

Fetch Join + Pagination:
  count 쿼리에 Fetch Join 포함 시 쿼리 오류 또는 성능 저하
  → query.getResultType() 분기 패턴 필수
```

---

## 🤔 트레이드오프

```
Specification 장점:
  외부 라이브러리 없음 (Spring Data JPA 기본 제공)
  재사용 가능한 조건 블록
  JPA Criteria API 완전 지원
  @DataJpaTest 등 테스트 친화적

Specification 단점:
  Criteria API 기반 → 장황한 코드 (root.get("name") 문자열)
  조인이 복잡해지면 가독성 저하
  Projection (SELECT 컬럼 제한) 지원 제한적
  GROUP BY, HAVING, 서브쿼리 표현 복잡

QueryDSL과 비교:
  Specification: 외부 라이브러리 없음, 타입 안전성 낮음 (문자열 필드명)
  QueryDSL: 빌드 설정 필요, 타입 안전성 높음 (Q타입)
  → 팀 역량, 프로젝트 복잡도에 따라 선택

선택 가이드:
  단순~중간 복잡도 동적 쿼리, 외부 라이브러리 추가 꺼릴 때 → Specification
  복잡한 쿼리, JOIN Fetch, Projection 빈번 → QueryDSL
  두 가지를 혼용 가능: Specification으로 단순 조건, QueryDSL로 복잡 쿼리
```

---

## 📌 핵심 정리

```
Specification<T> 구조
  @FunctionalInterface: toPredicate(Root, CriteriaQuery, CriteriaBuilder) → Predicate
  and() / or(): SpecificationComposition으로 Predicate 조합
  null 반환 → 해당 조건 무시 (null-safe 동적 쿼리 핵심)

내부 동작 흐름
  userRepository.findAll(spec)
    → SimpleJpaRepository.getQuery(spec, sort)
    → applySpecificationToCriteria() → spec.toPredicate() 호출
    → CriteriaQuery.where(predicate)
    → em.createQuery(criteriaQuery).getResultList()

count 쿼리 분기
  query.getResultType() == Long.class → count 쿼리
  Fetch Join은 count 쿼리에서 제외 (필수 패턴)

중복 조인 방지
  root.getJoins()로 기존 조인 탐색 후 재사용
  여러 Specification이 같은 연관 엔티티 조인 시 필수

Specification vs QueryDSL
  Specification: 기본 제공, 재사용 편리, 문자열 필드명
  QueryDSL: 추가 설정, 타입 안전, 복잡한 쿼리 적합
```

---

## 🤔 생각해볼 문제

**Q1.** `Specification.where(null).and(spec2)`를 호출하면 어떤 결과가 나오는가? `Specification.where(null)`의 반환값은 무엇인가?

**Q2.** Specification을 사용해 `findAll(spec, pageable)`을 호출할 때, 정렬 조건이 연관 엔티티 필드(`team.name`)인 경우 어떤 문제가 발생하고 어떻게 해결하는가?

**Q3.** 동일한 Specification 객체를 여러 스레드에서 동시에 사용해도 안전한가? `toPredicate()` 내부에서 상태를 공유하는 경우 어떤 문제가 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** `Specification.where(null)`은 `(root, query, cb) -> null`을 반환하는 항상-null Specification을 반환한다. `where(null).and(spec2)`를 호출하면 `SpecificationComposition.composed()`에서 `leftPredicate = null`이 되고, null이면 `rightPredicate`를 그대로 반환하는 로직에 따라 `spec2`의 Predicate만 적용된다. 즉 `Specification.where(null).and(spec2)`는 `spec2`만 적용된 것과 동일하다. 이 동작을 이용해 조건 배열을 순회하며 안전하게 체인을 구성할 수 있다.
>
> **Q2.** 연관 엔티티 필드(`team.name`)로 정렬하려면 해당 테이블이 JOIN되어 있어야 한다. `Sort.by("team.name")`을 사용하면 Spring Data가 `ORDER BY t.name`을 생성하려 하지만 `team` 테이블이 JOIN되지 않으면 오류가 발생한다. 해결책은 Specification 내에서 `root.join("team")`을 수행하여 JOIN을 보장하거나, `Sort.by("team.name")` 대신 Specification과 `JpaSort.unsafe("t.name")`을 결합하거나, QueryDSL로 전환하는 것이다. 가장 간단한 방법은 팀 이름으로 정렬이 필요할 때 반드시 JOIN을 포함하는 별도의 Specification을 `and()`로 추가하는 것이다.
>
> **Q3.** Specification 자체(`@FunctionalInterface`)는 상태가 없는 순수 함수이면 스레드 안전하다. 문제는 Specification 구현체가 외부 가변 상태를 캡처할 때 발생한다. 예를 들어 람다 외부에서 선언한 `List<Predicate> conditions = new ArrayList<>()`를 람다 내에서 수정하면 여러 스레드에서 동시에 `toPredicate()`를 호출할 때 race condition이 발생한다. 안전한 패턴은 `toPredicate()` 내에서 모든 변수를 로컬로 선언하는 것이다. Spring Data가 제공하는 `SpecificationComposition`도 내부 상태 없이 순수 함수형으로 구현되어 있어 스레드 안전하다.

---

<div align="center">

**[⬅️ 이전: QueryDSL 통합 원리](./05-querydsl-integration.md)** | **[다음: Auditing 동작 원리 ➡️](./07-auditing-internals.md)**

</div>
