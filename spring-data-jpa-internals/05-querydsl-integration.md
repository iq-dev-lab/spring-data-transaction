# QueryDSL 통합 원리 — JPAQueryFactory와 Spring Data 통합 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- QueryDSL이 Criteria API보다 선호되는 기술적 이유는 무엇인가?
- Q타입 클래스는 어떻게 생성되며, 빌드 프로세스에서 어떤 역할을 하는가?
- `QuerydslPredicateExecutor`와 `JPAQueryFactory` 방식의 차이는?
- `JPAQueryFactory`가 내부적으로 `EntityManager`를 어떻게 사용하는가?
- Spring Data Repository에 QueryDSL을 통합할 때 Custom Repository 패턴이 필요한 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: 동적 쿼리를 타입 안전하게 작성하기 어렵다

```java
// 문제 1: JPQL 문자열 — 컴파일 타임 오류 감지 불가
@Query("SELECT u FROM User u WHERE u.name = :name AND u.stauts = :status")
//                                              ↑ 오타! "stauts" — 런타임에 발견
List<User> findByNameAndStatus(@Param("name") String name, @Param("status") Status status);

// 문제 2: Criteria API — 타입 안전하지만 가독성이 극도로 낮음
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
List<Predicate> predicates = new ArrayList<>();

if (name != null) {
    predicates.add(cb.equal(root.get("name"), name));  // ← 문자열! 타입 안전 아님
}
if (status != null) {
    predicates.add(cb.equal(root.get("status"), status));
}
if (minAge != null) {
    predicates.add(cb.greaterThanOrEqualTo(root.get("age"), minAge));
}
cq.where(predicates.toArray(new Predicate[0]));
// 코드량 폭발, 가독성 최악

// 문제 3: Query Method — 동적 조건 표현 불가
// 조건이 없을 수도 있는 경우 처리 불가
```

```
QueryDSL이 해결하는 것:
  타입 안전: Q타입으로 필드 이름을 컴파일 타임에 검증
  가독성: SQL에 가까운 유창한 API (Fluent API)
  동적 쿼리: BooleanBuilder로 조건을 런타임에 조합
  IDE 지원: 자동 완성 가능 (Q타입은 일반 클래스)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: JPAQueryFactory에 EntityManager를 직접 주입하면 스레드 안전하지 않다

```java
// ❌ 잘못된 이해: EntityManager는 스레드 불안전하므로 주입하면 안 된다
@Repository
public class UserQueryRepository {

    // EntityManager는 실제로 프록시 — 스레드마다 다른 인스턴스 사용
    @PersistenceContext
    private EntityManager em;

    // ✅ 안전: @PersistenceContext가 주입하는 EntityManager는
    // 실제 EntityManager가 아닌 SharedEntityManagerCreator가 만든 프록시
    // → 호출 시마다 현재 스레드의 트랜잭션 컨텍스트에서 실제 EntityManager를 가져옴
    private JPAQueryFactory queryFactory;

    @PostConstruct
    public void init() {
        this.queryFactory = new JPAQueryFactory(em);  // ✅ 안전
    }
}
```

### Before: QuerydslPredicateExecutor로 모든 복잡한 쿼리를 처리할 수 있다

```java
// ❌ 과도한 기대:
// QuerydslPredicateExecutor<User> 만 상속하면 모든 QueryDSL 기능 사용 가능

// ✅ 실제 한계:
// QuerydslPredicateExecutor가 지원하는 것:
//   findAll(Predicate), findOne(Predicate), count(Predicate), exists(Predicate)
//   findAll(Predicate, Sort), findAll(Predicate, Pageable)

// 지원하지 않는 것:
//   JOIN을 통한 다른 엔티티 필드 조건 (Fetch Join)
//   Projection (SELECT 컬럼 제한)
//   GROUP BY, HAVING
//   서브쿼리
//   UPDATE, DELETE (벌크)
// → 복잡한 쿼리는 Custom Repository + JPAQueryFactory 필요
```

---

## ✨ 올바른 이해와 패턴

### After: 쿼리 복잡도에 따른 선택 기준

```
단순 동적 조건 (AND 조합) → QuerydslPredicateExecutor
  userRepository.findAll(QUser.user.status.eq(ACTIVE)
                         .and(QUser.user.age.goe(20)));

중간 복잡도 (JOIN, Projection 없이) → QuerydslPredicateExecutor + Predicate 조합
  BooleanBuilder builder = new BooleanBuilder();
  if (name != null) builder.and(QUser.user.name.contains(name));
  if (status != null) builder.and(QUser.user.status.eq(status));
  userRepository.findAll(builder);

복잡한 쿼리 (JOIN Fetch, Projection, 집계) → Custom Repository + JPAQueryFactory
  queryFactory.select(Projections.constructor(UserDto.class, ...))
              .from(QUser.user)
              .join(QUser.user.team, QTeam.team).fetchJoin()
              .where(builder)
              .fetch();
```

---

## 🔬 내부 동작 원리 — QueryDSL 소스 추적

### 1. Q타입 생성 과정 — APT (Annotation Processing Tool)

```java
// @Entity 어노테이션이 있는 클래스
@Entity
public class User {
    @Id private Long id;
    private String name;
    private int age;
    private Status status;

    @ManyToOne
    private Team team;
}

// 빌드 시 APT가 자동 생성하는 Q타입
// build/generated/sources/annotationProcessor/java/main/QUser.java
@Generated("com.querydsl.codegen.DefaultEntitySerializer")
public class QUser extends EntityPathBase<User> {

    public static final QUser user = new QUser("user");

    public final NumberPath<Long> id = createNumber("id", Long.class);
    public final StringPath name = createString("name");
    public final NumberPath<Integer> age = createNumber("age", Integer.class);
    public final EnumPath<Status> status = createEnum("status", Status.class);
    public final QTeam team;  // 연관 엔티티 → QTeam 타입

    public QUser(String variable) {
        super(User.class, forVariable(variable));
        // 연관 엔티티 초기화
        this.team = new QTeam(forProperty("team"));
    }
}
```

```
빌드 설정 (Gradle):
  dependencies {
      implementation 'com.querydsl:querydsl-jpa:5.x:jakarta'
      annotationProcessor 'com.querydsl:querydsl-apt:5.x:jakarta'
      annotationProcessor 'jakarta.persistence:jakarta.persistence-api'
  }

  // Q타입 생성 위치 설정
  sourceSets {
      main.java.srcDir "$buildDir/generated/sources/annotationProcessor/java/main"
  }
```

### 2. JPAQueryFactory 내부 구조

```java
// JPAQueryFactory — QueryDSL의 진입점
public class JPAQueryFactory implements JPQLQueryFactory {

    private final EntityManager entityManager;

    public JPAQueryFactory(EntityManager entityManager) {
        this.entityManager = entityManager;
        // entityManager는 @PersistenceContext 주입 → 실제로는 프록시
    }

    // SELECT 쿼리 시작
    public <T> JPAQuery<T> select(Expression<T> expr) {
        return query().select(expr);
    }

    public JPAQuery<Tuple> select(Expression<?>... exprs) {
        return query().select(exprs);
    }

    private JPAQuery<Void> query() {
        return new JPAQuery<>(entityManager);
        // → 새 JPAQuery 인스턴스 생성 (EntityManager는 공유)
        // → 쿼리 실행 시 entityManager.createQuery() 호출
    }
}

// JPAQuery.fetch() — 실제 쿼리 실행
class JPAQuery<T> extends AbstractJPAQuery<T, JPAQuery<T>> {

    @Override
    public List<T> fetch() {
        try {
            Query query = createQuery();  // JPQL 또는 Criteria로 변환
            return (List<T>) query.getResultList();
        } finally {
            reset();  // 쿼리 상태 초기화
        }
    }

    private Query createQuery() {
        // QueryDSL AST → JPQL 문자열 또는 Criteria API 변환
        JPQLSerializer serializer = new JPQLSerializer(templates, entityManager);
        serializer.serialize(queryMixin.getMetadata(), false, null);
        // → "SELECT u FROM User u WHERE u.status = ?1 AND u.age >= ?2"
        return entityManager.createQuery(serializer.toString(), ...);
    }
}
```

### 3. QuerydslPredicateExecutor 통합 — Spring Data 내부

```java
// Spring Data JPA의 QuerydslJpaPredicateExecutor
public class QuerydslJpaPredicateExecutor<T> implements QuerydslPredicateExecutor<T> {

    private final EntityPath<T> path;
    private final Querydsl querydsl;
    private final EntityManager entityManager;

    @Override
    public List<T> findAll(Predicate predicate) {
        return createQuery(predicate).select(path).fetch();
    }

    @Override
    public Page<T> findAll(Predicate predicate, Pageable pageable) {
        JPQLQuery<T> countQuery = createCountQuery(predicate);
        JPQLQuery<T> query = querydsl.applyPagination(
            pageable, createQuery(predicate).select(path));

        return PageableExecutionUtils.getPage(
            query.fetch(),
            pageable,
            countQuery::fetchCount  // count 쿼리 지연 실행
        );
    }

    private JPQLQuery<?> createQuery(Predicate... predicates) {
        AbstractJPAQuery<?, ?> query = doCreateQuery(predicates);
        // EntityGraph, QueryHints 등 적용
        return query;
    }
}
```

### 4. Custom Repository 통합 패턴 — 전체 구조

```java
// [1] Custom Repository 인터페이스 정의
public interface UserRepositoryCustom {
    List<UserDto> findUsersWithCondition(UserSearchCondition condition);
    Page<UserDto> findUsersWithPaging(UserSearchCondition condition, Pageable pageable);
}

// [2] 검색 조건 DTO
@Data
public class UserSearchCondition {
    private String nameLike;
    private Integer minAge;
    private Integer maxAge;
    private Status status;
    private String teamName;
}

// [3] Custom Repository 구현체 (네이밍: 인터페이스명 + "Impl")
@Repository
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    private final JPAQueryFactory queryFactory;

    @Override
    public List<UserDto> findUsersWithCondition(UserSearchCondition cond) {
        return queryFactory
            .select(Projections.constructor(UserDto.class,
                QUser.user.id,
                QUser.user.name,
                QTeam.team.name.as("teamName")
            ))
            .from(QUser.user)
            .leftJoin(QUser.user.team, QTeam.team)
            .where(
                nameLike(cond.getNameLike()),
                ageBetween(cond.getMinAge(), cond.getMaxAge()),
                statusEq(cond.getStatus()),
                teamNameEq(cond.getTeamName())
            )
            .fetch();
    }

    // 동적 조건 메서드 — null이면 조건 제외 (null-safe)
    private BooleanExpression nameLike(String nameLike) {
        return hasText(nameLike) ? QUser.user.name.contains(nameLike) : null;
        // BooleanBuilder에 null을 추가하면 무시됨
    }

    private BooleanExpression ageBetween(Integer minAge, Integer maxAge) {
        if (minAge == null && maxAge == null) return null;
        if (minAge == null) return QUser.user.age.loe(maxAge);
        if (maxAge == null) return QUser.user.age.goe(minAge);
        return QUser.user.age.between(minAge, maxAge);
    }

    private BooleanExpression statusEq(Status status) {
        return status != null ? QUser.user.status.eq(status) : null;
    }

    private BooleanExpression teamNameEq(String teamName) {
        return hasText(teamName) ? QTeam.team.name.eq(teamName) : null;
    }
}

// [4] Repository 인터페이스에서 Custom Repository 상속
public interface UserRepository
        extends JpaRepository<User, Long>,
                QuerydslPredicateExecutor<User>,
                UserRepositoryCustom {  // ← Custom 통합
}
```

### 5. Projections — QueryDSL 결과 매핑 방식

```java
// 방식 1: Projections.constructor — 생성자 기반
queryFactory
    .select(Projections.constructor(UserDto.class,
        QUser.user.id,
        QUser.user.name,
        QTeam.team.name
    ))
    .from(QUser.user)
    .join(QUser.user.team, QTeam.team)
    .fetch();
// → UserDto(Long id, String name, String teamName) 생성자 직접 호출

// 방식 2: Projections.fields — 필드명 기반 (setter 없어도 됨)
queryFactory
    .select(Projections.fields(UserDto.class,
        QUser.user.id,
        QUser.user.name.as("userName"),  // 필드명과 다르면 as() 사용
        QTeam.team.name.as("teamName")
    ))
    .from(QUser.user)
    .join(QUser.user.team, QTeam.team)
    .fetch();

// 방식 3: @QueryProjection — 가장 타입 안전
// DTO 생성자에 @QueryProjection 추가 → QUserDto 자동 생성
public class UserDto {
    @QueryProjection
    public UserDto(Long id, String name, String teamName) { ... }
}

queryFactory
    .select(new QUserDto(QUser.user.id, QUser.user.name, QTeam.team.name))
    .from(QUser.user)
    .join(QUser.user.team, QTeam.team)
    .fetch();
// → 컴파일 타임에 파라미터 타입까지 검증
```

---

## 💻 실험으로 확인하기

### 실험 1: 동적 쿼리 — 조건 유무에 따른 SQL 변화

```java
@Test
void dynamicQueryTest() {
    UserSearchCondition allConditions = UserSearchCondition.builder()
        .nameLike("길동").minAge(20).maxAge(40).status(ACTIVE).build();

    // 모든 조건: WHERE name LIKE '%길동%' AND age BETWEEN 20 AND 40 AND status = ?
    List<UserDto> result1 = userRepository.findUsersWithCondition(allConditions);

    UserSearchCondition noCondition = new UserSearchCondition();
    // 조건 없음: WHERE 절 없음 (전체 조회)
    List<UserDto> result2 = userRepository.findUsersWithCondition(noCondition);

    UserSearchCondition nameOnly = UserSearchCondition.builder()
        .nameLike("홍").build();
    // name만: WHERE name LIKE '%홍%'
    List<UserDto> result3 = userRepository.findUsersWithCondition(nameOnly);
}
```

### 실험 2: QuerydslPredicateExecutor 사용

```java
@Test
void predicateExecutorTest() {
    QUser user = QUser.user;

    // 단순 조건
    Predicate predicate = user.status.eq(Status.ACTIVE)
                              .and(user.age.goe(20));
    List<User> result = userRepository.findAll(predicate);

    // 정렬 + 페이징
    Page<User> page = userRepository.findAll(
        predicate,
        PageRequest.of(0, 10, Sort.by("name").ascending())
    );
    System.out.println("전체 수: " + page.getTotalElements());
}
```

### 실험 3: @QueryProjection vs Projections.constructor 비교

```java
@Test
void projectionComparison() {
    QUser u = QUser.user;
    QTeam t = QTeam.team;

    // Projections.constructor — 런타임에 생성자 파라미터 순서 검증
    List<UserDto> byConstructor = queryFactory
        .select(Projections.constructor(UserDto.class, u.id, u.name, t.name))
        .from(u).join(u.team, t).fetch();

    // @QueryProjection — 컴파일 타임에 타입 검증
    List<UserDto> byAnnotation = queryFactory
        .select(new QUserDto(u.id, u.name, t.name))
        .from(u).join(u.team, t).fetch();
    // u.id를 String이 필요한 위치에 넣으면 컴파일 오류
}
```

---

## ⚡ 성능 임팩트

```
Q타입 생성 비용:
  빌드 시 APT 처리 → 빌드 시간 증가 (엔티티 수에 비례)
  엔티티 50개: 빌드 시간 약 1~3초 증가
  런타임 비용 없음 (생성된 클래스는 일반 Java 클래스)

JPAQueryFactory 쿼리 실행 비용:
  QueryDSL AST → JPQL 직렬화 → em.createQuery()
  Hibernate Query Plan Cache 적용 (JPQL 캐시)
  → 첫 실행 후 캐시: Criteria API 직접 사용과 동일한 성능

동적 조건 성능:
  null 조건 제외 → 실제 WHERE 절 컬럼 수 최소화
  정적 쿼리보다 실행 계획 다양 → DB 실행 계획 캐시 미스 가능
  → 파라미터 수가 크게 다른 동적 쿼리 → bindvariable 방식 선택 중요

Projections.constructor vs @QueryProjection:
  런타임 성능은 동일 (둘 다 생성자 직접 호출)
  @QueryProjection: 컴파일 타임 안전성 추가 비용 → 빌드 시간 약간 증가
```

---

## 🤔 트레이드오프

```
QuerydslPredicateExecutor:
  사용 편의성    높음 (Repository 인터페이스 상속만으로 사용)
  기능 제한      JOIN Fetch, Projection, 집계 불가
  적합한 경우   단순 동적 조건 검색

Custom Repository + JPAQueryFactory:
  사용 편의성    낮음 (구현 클래스 직접 작성)
  기능 제한      없음 (Hibernate가 지원하는 모든 쿼리)
  적합한 경우   복잡한 JOIN, Projection, 집계, 벌크 연산

@QueryProjection vs Projections.constructor:
  @QueryProjection: 타입 안전, DTO가 QueryDSL에 의존 (계층 오염 논란)
  Projections.constructor: 타입 안전 낮음, DTO는 순수 Java
  → 아키텍처 원칙에 따라 선택

QueryDSL 도입 비용:
  빌드 설정 추가, Q타입 관리, 버전 호환성 관리
  Spring Boot 3.x + QueryDSL 5.x Jakarta 버전 확인 필수
  대안: Blaze-Persistence, Spring Data Specifications
```

---

## 📌 핵심 정리

```
Q타입
  APT가 @Entity 클래스 분석 → QUser 등 자동 생성 (빌드 시)
  타입 안전한 경로 표현: QUser.user.name, QUser.user.age
  연관 엔티티도 타입 안전: QUser.user.team.name

JPAQueryFactory 내부
  @PersistenceContext EntityManager (프록시) → 스레드 안전
  쿼리 실행 시 QueryDSL AST → JPQL 직렬화 → em.createQuery()
  Hibernate Query Plan Cache 적용

두 가지 통합 방식
  QuerydslPredicateExecutor: 간단한 동적 조건 (AND 조합)
  Custom Repository + JPAQueryFactory: 복잡한 쿼리 (JOIN, Projection, 집계)

동적 조건 패턴
  BooleanExpression 반환 메서드 → null 반환 시 조건 제외
  BooleanBuilder: 여러 조건을 순차적으로 조합
  where(cond1, cond2, cond3): null 자동 무시

Projection 방식
  Projections.constructor: 생성자 기반, 파라미터 순서 주의
  Projections.fields: 필드명 기반, as()로 별칭 지정
  @QueryProjection: 컴파일 타임 타입 검증, DTO 의존 논란
```

---

## 🤔 생각해볼 문제

**Q1.** `QuerydslPredicateExecutor`를 사용해 `Page<User> findAll(Predicate, Pageable)`을 호출할 때, count 쿼리가 실제로 실행되는 시점은 언제인가? `PageableExecutionUtils.getPage()`가 count 쿼리를 최적화하는 방법은?

**Q2.** `@QueryProjection`을 사용하면 DTO 계층이 QueryDSL에 의존하게 된다. 이를 피하면서도 컴파일 타임 안전성을 확보하는 방법이 있는가?

**Q3.** `JPAQueryFactory`로 실행한 쿼리는 Spring Data의 `@Transactional` 없이도 동작하는가? 트랜잭션이 없는 상황에서 `queryFactory.select(...).fetch()`를 호출하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `PageableExecutionUtils.getPage()`는 count 쿼리를 `Supplier<Long>`(람다)으로 받아 **지연 실행**한다. count 쿼리가 실제로 실행되는 조건은: (1) 현재 페이지가 첫 번째 페이지이고 가져온 데이터가 페이지 크기보다 작을 때, (2) 마지막 페이지일 때 — 이 두 경우에는 count 쿼리 없이 총 개수를 계산할 수 있으므로 count 쿼리를 생략한다. 예를 들어 페이지 크기가 10인데 결과가 7개라면 이 페이지가 마지막이므로 `offset + 7`이 총 개수가 된다. 전체 데이터가 많은 서비스에서 마지막 페이지 count 쿼리를 생략하는 것은 성능상 의미 있는 최적화다.
>
> **Q2.** 몇 가지 방법이 있다. 첫째, `Projections.constructor()`를 사용하되 `ExpressionUtils.as()`와 함께 명시적으로 파라미터를 지정하면 생성자 순서 오류를 테스트로 커버할 수 있다. 둘째, `@QueryProjection`은 유지하되 DTO를 별도의 `query` 하위 패키지에 위치시켜 의존 방향을 명확히 한다. 셋째, Interface Projection + QueryDSL `bean()` 방식을 결합하면 인터페이스는 QueryDSL에 의존하지 않는다. 아키텍처 원칙보다 실용성을 중시한다면 `@QueryProjection`이 가장 안전하다.
>
> **Q3.** 동작은 하지만 트랜잭션 없이 단일 쿼리마다 Connection을 가져오고 반환하는 auto-commit 모드로 실행된다. `EntityManager`는 트랜잭션이 없어도 `em.createQuery().getResultList()`를 실행할 수 있다. 하지만 Lazy Loading이 포함된 엔티티를 반환하면 나중에 `LazyInitializationException`이 발생할 수 있다. 또한 쓰기 쿼리(`update`, `delete`)는 `@Transactional` 없이는 `javax.persistence.TransactionRequiredException`이 발생한다. 읽기 전용 QueryDSL 쿼리에는 `@Transactional(readOnly = true)`를 명시하는 것이 Connection 관리와 성능 면에서 권장된다.

---

<div align="center">

**[⬅️ 이전: Projection — Interface vs DTO](./04-projection-interface-dto.md)** | **[다음: Specifications를 통한 동적 쿼리 ➡️](./06-specifications-dynamic-query.md)**

</div>
