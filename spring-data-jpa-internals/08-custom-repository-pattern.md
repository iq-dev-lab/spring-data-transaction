# Custom Repository 구현 패턴 — Impl 네이밍과 EntityManager 직접 사용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `UserRepositoryImpl`이라는 이름만으로 Spring Data가 자동으로 인식하는 원리는?
- Custom Repository가 `JpaRepository`와 합쳐지는 내부 합성(Composition) 과정은?
- `EntityManager`를 Custom Repository에서 직접 사용해야 하는 상황은?
- `repositoryImplementationPostfix` 설정으로 `Impl` 접미사를 변경하는 방법과 그 이유는?
- Custom Repository와 `@Service` 레이어 중 쿼리 로직을 어디에 위치시킬지 판단하는 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: Spring Data가 제공하지 않는 쿼리 기능이 필요하다

```java
// Query Method, @Query, Specification, QueryDSL로도 어려운 상황들

// 1. 동적으로 테이블명이 바뀌는 쿼리 (샤딩, 파티셔닝)
em.createNativeQuery("SELECT * FROM orders_" + shardKey + " WHERE ...");

// 2. 복잡한 벌크 연산 + 영속성 컨텍스트 조작
em.createQuery("UPDATE ...").executeUpdate();
em.clear();  // 캐시 명시적 비움

// 3. StoredProcedure 호출
em.createStoredProcedureQuery("proc_monthly_report")
  .registerStoredProcedureParameter(...)
  .execute();

// 4. EntityManager 레벨 힌트 / 락 제어
em.find(User.class, id, LockModeType.PESSIMISTIC_WRITE,
        Map.of("javax.persistence.lock.timeout", 3000));

// 5. Flush 타이밍 수동 제어
em.flush();
// 이런 로직들은 Repository 인터페이스 메서드로 표현 불가
// → Custom Repository에서 EntityManager 직접 사용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Custom Repository 구현체에 @Repository를 붙여야 한다

```java
// ❌ 불필요한 @Repository
@Repository  // ← 없어도 동작, 오히려 혼란
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;
    // ...
}

// ✅ @Repository 없어도 Spring Data가 자동 감지
// Spring Data가 "Impl" 접미사로 찾아 합성
// 단, @Repository를 붙이면 PersistenceExceptionTranslation 적용
//   → DataAccessException 변환 (일관성 측면에서 붙이는 팀도 있음)
```

### Before: 인터페이스 이름 + "Impl"이 아닌 Repository 이름 + "Impl"을 써야 한다

```java
// 혼동 포인트:
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom { }
public interface UserRepositoryCustom { List<User> findSpecial(); }

// ❌ 잘못된 이해: UserRepositoryImpl로 구현해야 한다
public class UserRepositoryImpl implements UserRepositoryCustom { ... }
// → 동작하긴 함 (Repository 이름 + Impl도 탐색)

// ✅ 권장: 커스텀 인터페이스 이름 + Impl
public class UserRepositoryCustomImpl implements UserRepositoryCustom { ... }
// → 명확성 측면에서 권장

// Spring Data 탐색 순서:
//   1. UserRepositoryCustomImpl (커스텀 인터페이스명 + Impl)
//   2. UserRepositoryImpl (Repository 인터페이스명 + Impl)
// 두 클래스가 모두 있으면 1번 우선
```

### Before: Custom Repository의 메서드도 자동으로 트랜잭션이 적용된다

```java
// ❌ 잘못된 이해: Spring Data Repository이므로 트랜잭션 자동 적용

// ✅ 실제:
// Custom Repository 구현체의 메서드는 @Transactional 없으면 트랜잭션 없음
// SimpleJpaRepository의 @Transactional은 CRUD 메서드에만 적용
// Custom 메서드에는 별도 선언 필요

public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    // ❌ 트랜잭션 없음 — 쓰기 작업 실패 가능
    public void bulkUpdateStatus(Status status) {
        em.createQuery("UPDATE User u SET u.status = :status")
          .setParameter("status", status)
          .executeUpdate();
    }

    // ✅ 트랜잭션 명시
    @Transactional
    public void bulkUpdateStatus(Status status) {
        em.createQuery("UPDATE User u SET u.status = :status")
          .setParameter("status", status)
          .executeUpdate();
    }
}
```

---

## ✨ 올바른 이해와 패턴

### After: Custom Repository 설계 원칙

```
Custom Repository가 적합한 경우:
  EntityManager 직접 사용 (힌트, 락, StoredProcedure)
  복잡한 네이티브 쿼리 + 영속성 컨텍스트 조작
  QueryDSL 기반 복잡한 조회 (JPAQueryFactory 사용)
  배치 처리 (flush/clear 수동 제어)

Service 레이어가 적합한 경우:
  여러 Repository 호출을 조합하는 비즈니스 로직
  트랜잭션 경계가 Service 메서드 수준인 경우
  Repository에는 쿼리 로직만, 조합 로직은 Service로

판단 기준:
  "이 메서드가 DB 접근 로직인가?" → Custom Repository
  "이 메서드가 비즈니스 규칙인가?" → Service
```

---

## 🔬 내부 동작 원리 — Custom Repository 합성 소스 추적

### 1. Impl 탐색 과정 — RepositoryFactorySupport

```java
// RepositoryConfigurationExtensionSupport — Impl 클래스 탐색
class RepositoryConfigurationExtensionSupport {

    protected Iterable<String> getImplementationBasePackages() {
        // Repository 인터페이스와 같은 패키지 탐색
        return Collections.singleton(repositoryInterface.getPackage().getName());
    }
}

// RepositoryFactorySupport — Impl 클래스 로드 및 인스턴스 생성
public <T> T getRepository(Class<T> repositoryInterface, RepositoryFragments fragments) {

    // Custom 구현체 탐색
    // 기본 접미사: "Impl" (@EnableJpaRepositories의 repositoryImplementationPostfix)
    Class<?> implementationClass = findImplementationClass(repositoryInterface);
    // 탐색 순서:
    //   1. UserRepositoryCustomImpl
    //   2. UserRepositoryImpl

    Object customImplementation = implementationClass != null
        ? instantiateClass(implementationClass)  // 인스턴스 생성
        : null;

    // RepositoryFragments 조합
    RepositoryFragments resultFragments = RepositoryFragments
        .just(customImplementation)  // Custom 구현체
        .append(fragments);           // 기본 fragments (SimpleJpaRepository 포함)

    // 프록시 생성 시 두 구현체를 합성
    return (T) createRepositoryProxy(repositoryInterface, resultFragments);
}
```

### 2. RepositoryFragments — 조각 합성 구조

```java
// RepositoryFragment — 단일 조각 (인터페이스 + 구현체 쌍)
public interface RepositoryFragment<T> {
    Optional<Class<?>> getInterfaceClass();
    T getImplementation();
}

// RepositoryFragments — 여러 Fragment의 컬렉션
// 최종 프록시가 메서드 호출 시 아래 순서로 Fragment 탐색:
//
// UserRepository 인터페이스의 메서드 호출 시:
//   1. UserRepositoryCustom 메서드 → UserRepositoryCustomImpl 에서 처리
//   2. JpaRepository 메서드 (findById, save 등) → SimpleJpaRepository 에서 처리
//   3. QuerydslPredicateExecutor 메서드 → QuerydslJpaPredicateExecutor 에서 처리

// RepositoryMethodInvocationListener가 메서드→구현체 매핑을 캐시
```

### 3. 전체 Custom Repository 구성 예시

```java
// [1] Custom 인터페이스 정의
public interface UserRepositoryCustom {

    // EntityManager 직접 사용이 필요한 경우들
    List<User> findWithPessimisticLock(List<Long> ids);
    int bulkUpdateLastLoginAt(List<Long> ids, LocalDateTime loginAt);
    void flushAndClear();
    List<UserStatDto> findUserStatistics(YearMonth yearMonth);
}

// [2] 구현체 — Impl 접미사 필수
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    private final EntityManager em;
    // @PersistenceContext 대신 생성자 주입도 가능 (Spring 5+)

    // 비관적 락 + 힌트
    @Override
    @Transactional
    public List<User> findWithPessimisticLock(List<Long> ids) {
        return em.createQuery(
                "SELECT u FROM User u WHERE u.id IN :ids", User.class)
            .setParameter("ids", ids)
            .setLockMode(LockModeType.PESSIMISTIC_WRITE)
            .setHint("javax.persistence.lock.timeout", 5000)  // 5초 타임아웃
            .getResultList();
    }

    // 벌크 업데이트 + 영속성 컨텍스트 무효화
    @Override
    @Transactional
    public int bulkUpdateLastLoginAt(List<Long> ids, LocalDateTime loginAt) {
        int updated = em.createQuery(
                "UPDATE User u SET u.lastLoginAt = :loginAt WHERE u.id IN :ids")
            .setParameter("loginAt", loginAt)
            .setParameter("ids", ids)
            .executeUpdate();

        // 벌크 연산은 영속성 컨텍스트를 거치지 않으므로 캐시 무효화
        em.clear();
        return updated;
    }

    // 수동 flush/clear (배치 처리용)
    @Override
    public void flushAndClear() {
        em.flush();
        em.clear();
    }

    // 복잡한 네이티브 집계 쿼리
    @Override
    @Transactional(readOnly = true)
    public List<UserStatDto> findUserStatistics(YearMonth yearMonth) {
        String sql = """
            SELECT DATE_FORMAT(u.created_at, '%Y-%m') AS month,
                   u.status,
                   COUNT(u.id) AS user_count,
                   AVG(u.age) AS avg_age
            FROM users u
            WHERE DATE_FORMAT(u.created_at, '%Y-%m') = :yearMonth
            GROUP BY DATE_FORMAT(u.created_at, '%Y-%m'), u.status
            ORDER BY u.status
            """;

        return em.createNativeQuery(sql)
            .setParameter("yearMonth", yearMonth.toString())
            .unwrap(NativeQuery.class)
            .addScalar("month", StandardBasicTypes.STRING)
            .addScalar("status", StandardBasicTypes.STRING)
            .addScalar("user_count", StandardBasicTypes.LONG)
            .addScalar("avg_age", StandardBasicTypes.DOUBLE)
            .setResultTransformer(Transformers.aliasToBean(UserStatDto.class))
            .getResultList();
    }
}

// [3] Repository 통합
public interface UserRepository
        extends JpaRepository<User, Long>,
                UserRepositoryCustom {
    // JpaRepository: findById, save, delete, ...
    // UserRepositoryCustom: findWithPessimisticLock, bulkUpdateLastLoginAt, ...
    // Query Method와 @Query도 여기에 추가
    Optional<User> findByEmail(String email);
}
```

### 4. 접미사 변경 — repositoryImplementationPostfix

```java
// 기본값 "Impl" → 커스터마이징
@EnableJpaRepositories(
    basePackages = "com.example.repository",
    repositoryImplementationPostfix = "Impl"  // 기본값, 명시적으로 변경 가능
)

// "Impl" 대신 다른 접미사 사용 예
@EnableJpaRepositories(repositoryImplementationPostfix = "Dao")
// → UserRepositoryCustomDao 로 탐색

// 실무에서 변경하는 경우:
// - 회사 코딩 컨벤션이 "Impl" 대신 다른 접미사를 요구
// - 레거시 코드와의 통합 (기존 Dao 패턴 유지)
```

### 5. QueryDSL과 Custom Repository 조합 — 완성형 패턴

```java
// JPAQueryFactory 기반 Custom Repository
@RequiredArgsConstructor
public class UserRepositoryCustomImpl implements UserRepositoryCustom {

    private final JPAQueryFactory queryFactory;
    // JPAQueryFactory 빈 등록 필요:
    // @Bean JPAQueryFactory queryFactory(EntityManager em) { return new JPAQueryFactory(em); }

    private final QUser user = QUser.user;
    private final QTeam team = QTeam.team;

    @Override
    public Page<UserSearchResult> searchUsers(UserSearchCondition cond, Pageable pageable) {

        // 데이터 쿼리
        List<UserSearchResult> content = queryFactory
            .select(Projections.constructor(UserSearchResult.class,
                user.id, user.name, user.email, team.name.as("teamName")))
            .from(user)
            .leftJoin(user.team, team)
            .where(buildPredicate(cond))
            .orderBy(toOrderSpecifiers(pageable.getSort()))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

        // count 쿼리 (join 최소화)
        JPAQuery<Long> countQuery = queryFactory
            .select(user.count())
            .from(user)
            .where(buildPredicate(cond));

        return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
    }

    private BooleanBuilder buildPredicate(UserSearchCondition cond) {
        BooleanBuilder builder = new BooleanBuilder();
        if (hasText(cond.getName()))    builder.and(user.name.contains(cond.getName()));
        if (cond.getStatus() != null)   builder.and(user.status.eq(cond.getStatus()));
        if (cond.getMinAge() != null)   builder.and(user.age.goe(cond.getMinAge()));
        if (hasText(cond.getTeamName())) builder.and(team.name.eq(cond.getTeamName()));
        return builder;
    }

    private OrderSpecifier<?>[] toOrderSpecifiers(Sort sort) {
        return sort.stream()
            .map(order -> {
                PathBuilder<User> path = new PathBuilder<>(User.class, "user");
                return new OrderSpecifier<>(
                    order.isAscending() ? Order.ASC : Order.DESC,
                    path.getString(order.getProperty())
                );
            })
            .toArray(OrderSpecifier[]::new);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Custom Repository 메서드 호출 경로 확인

```java
@Test
void customRepositoryResolution() {
    // UserRepository 프록시에서 custom 메서드 호출
    List<User> result = userRepository.findWithPessimisticLock(List.of(1L, 2L, 3L));

    // 실행 SQL:
    // SELECT u.* FROM users u WHERE u.id IN (?, ?, ?)
    // FOR UPDATE  ← 비관적 락
    // lock_timeout 힌트 적용
}
```

### 실험 2: 합성 구조 확인

```java
@Test
void repositoryComposition() {
    // 동일 프록시에서 JPA 메서드 + Custom 메서드 모두 호출 가능
    Optional<User> byId = userRepository.findById(1L);     // SimpleJpaRepository
    List<User> locked = userRepository.findWithPessimisticLock(List.of(1L)); // Custom

    // 프록시 내부에 두 구현체가 Fragment로 합성됨
    // 메서드 시그니처로 어느 Fragment를 호출할지 결정
}
```

### 실험 3: 배치 처리 패턴 — flush/clear 수동 제어

```java
@Transactional
public void batchProcess(List<Long> allIds) {
    int chunkSize = 100;

    for (int i = 0; i < allIds.size(); i += chunkSize) {
        List<Long> chunk = allIds.subList(i, Math.min(i + chunkSize, allIds.size()));

        // 청크 처리
        int updated = userRepository.bulkUpdateLastLoginAt(chunk, LocalDateTime.now());
        // bulkUpdateLastLoginAt 내부에서 em.clear() 호출
        // → 각 청크 후 영속성 컨텍스트 비워 메모리 절약

        log.info("처리 완료: {}/{}", Math.min(i + chunkSize, allIds.size()), allIds.size());
    }
}
```

---

## ⚡ 성능 임팩트

```
Custom Repository 합성 비용:
  Impl 클래스 탐색: 애플리케이션 시작 시 1회 (classpath 스캔)
  Fragment 조합: 프록시 생성 시 1회
  메서드 → Fragment 매핑: 첫 호출 후 캐시
  런타임 오버헤드: 무시 가능 (~수 μs)

EntityManager 직접 사용 시 주의:
  em.createNativeQuery() → Query Plan Cache 적용 안 됨
  JPQL em.createQuery() → Hibernate가 캐시 적용
  벌크 연산 후 em.clear() → 이후 같은 트랜잭션에서 재조회 시 DB 접근

비관적 락 (PESSIMISTIC_WRITE) 성능 영향:
  SELECT ... FOR UPDATE → 해당 행 Lock
  lock.timeout 미설정 → 무한 대기 가능
  고트래픽 환경: 락 경합 → Deadlock 위험
  → Lock 범위 최소화, 타임아웃 필수 설정

배치 처리 청크 사이즈:
  너무 작음 → flush/clear 오버헤드 증가
  너무 큼 → 메모리 부족 (영속성 컨텍스트에 모든 엔티티 적재)
  실무 권장: 500~1000건 per chunk
```

---

## 🤔 트레이드오프

```
Custom Repository 장점:
  EntityManager 전체 기능 사용 가능
  힌트, 락, 네이티브 쿼리, StoredProcedure
  Repository 인터페이스에서 메서드 통합 제공
  Spring Data의 트랜잭션, 예외 변환 인프라 재사용

Custom Repository 단점:
  인터페이스 + 구현체 + Impl 네이밍 규칙 → 파일 3개
  `@Transactional` 수동 관리 필요
  테스트 시 구현체 직접 Mock 어려움 (인터페이스로 Mock)

Service vs Custom Repository 경계:
  쿼리 로직 (어떻게 가져오나) → Custom Repository
  비즈니스 로직 (무엇을 하나) → Service
  여러 Repository 조합 → Service
  트랜잭션 경계 설계가 핵심 판단 기준

별도 QueryRepository 패턴:
  JpaRepository 상속 없이 순수 조회 전용 클래스
  @Repository + JPAQueryFactory만 사용
  DDD의 Repository 개념과 더 일치
  → 팀 컨벤션에 따라 선택
```

---

## 📌 핵심 정리

```
Impl 탐색 순서
  1. {커스텀인터페이스명}Impl (UserRepositoryCustomImpl)
  2. {Repository인터페이스명}Impl (UserRepositoryImpl)
  → 두 개 모두 있으면 1번 우선

합성 구조
  RepositoryFragments = [UserRepositoryCustomImpl, SimpleJpaRepository, ...]
  프록시가 메서드 호출 시 Fragment 목록에서 구현체 탐색
  → Custom 메서드 → CustomImpl, CRUD → SimpleJpaRepository

Custom Repository 필수 주의사항
  @Transactional 별도 선언 (SimpleJpaRepository 트랜잭션 미적용)
  벌크 연산 후 em.clear() (1차 캐시 무효화)
  PESSIMISTIC_LOCK 시 lock.timeout 설정

접미사 변경
  @EnableJpaRepositories(repositoryImplementationPostfix = "Dao")
  팀 컨벤션에 맞게 조정 가능

설계 기준
  DB 접근 + 쿼리 로직 → Custom Repository
  비즈니스 규칙 + Repository 조합 → Service
```

---

## 🤔 생각해볼 문제

**Q1.** `UserRepository`가 `UserRepositoryCustom`을 상속하고, `UserRepositoryCustomImpl`과 `UserRepositoryImpl`이 모두 존재하면서 둘 다 `UserRepositoryCustom`을 구현한다면 어떻게 동작하는가?

**Q2.** Custom Repository 구현체에서 `@PersistenceContext EntityManager em`을 사용하는 것과, `JPAQueryFactory`를 생성자로 주입받는 것의 차이는? 어느 상황에서 무엇을 선택해야 하는가?

**Q3.** Custom Repository 메서드가 `@Transactional(readOnly = true)`인 Service 메서드 내에서 호출되고, Custom Repository 메서드에는 `@Transactional`이 없다. 이 상황에서 Custom Repository 내부의 `em.createQuery().executeUpdate()`(DML)는 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** Spring Data는 탐색 순서에 따라 `UserRepositoryCustomImpl`을 우선으로 선택한다. `UserRepositoryImpl`은 무시된다. 단, 두 클래스가 각각 `UserRepositoryCustom`의 서로 다른 메서드를 구현하도록 설계할 수는 없다 — Spring Data는 Fragment 당 하나의 구현체만 사용한다. 만약 두 클래스를 모두 사용하고 싶다면, `UserRepositoryCustom`을 두 개의 인터페이스로 분리하고 각각 별도의 Custom Repository로 구성한 뒤 `UserRepository`에서 두 인터페이스를 모두 상속해야 한다.
>
> **Q2.** `@PersistenceContext EntityManager`는 JPA 표준 어노테이션으로, Spring이 주입하는 것은 실제 `EntityManager`가 아닌 `SharedEntityManagerCreator` 프록시다. 이 프록시는 호출 시마다 현재 트랜잭션 컨텍스트에서 실제 `EntityManager`를 가져온다. `JPAQueryFactory`는 `EntityManager`를 생성자에서 받아 내부에 저장하는데, 이 역시 프록시이므로 스레드 안전하다. 선택 기준: JPQL/Native/StoredProcedure/Hint 등 JPA 저수준 기능이 필요하면 `EntityManager` 직접 사용, 타입 안전 동적 쿼리가 필요하면 `JPAQueryFactory`. 두 가지를 모두 주입받아 상황에 따라 사용해도 된다.
>
> **Q3.** `TransactionRequiredException`이 발생한다. Service의 `@Transactional(readOnly = true)`는 현재 트랜잭션을 readOnly로 시작한다. Custom Repository 메서드에 `@Transactional`이 없으면 기본 Propagation인 `REQUIRED`가 적용되어 Service의 readOnly 트랜잭션에 참여한다. readOnly 트랜잭션에서 `executeUpdate()`(DML)를 실행하면 Hibernate가 `FlushMode.MANUAL`로 동작하거나, 일부 드라이버/DB에서 읽기 전용 연결에 쓰기를 시도해 예외가 발생한다. 해결: Custom Repository의 DML 메서드에 `@Transactional(propagation = REQUIRES_NEW)`를 선언하여 별도의 readOnly가 아닌 트랜잭션으로 실행하거나, Service 메서드를 readOnly가 아닌 트랜잭션으로 변경해야 한다.

---

<div align="center">

**[⬅️ 이전: Auditing 동작 원리](./07-auditing-internals.md)** | **[홈으로 🏠](../README.md)**

</div>
