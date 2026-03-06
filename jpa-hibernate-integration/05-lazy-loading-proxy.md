# Lazy Loading 프록시 생성 과정 — ByteBuddy, LazyInitializationException, OSIV

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Hibernate가 Lazy Loading을 위해 생성하는 프록시의 정확한 구조는?
- `ByteBuddyProxyFactory`는 어떻게 엔티티 서브클래스 프록시를 만드는가?
- `LazyInitializationException`이 발생하는 정확한 조건과 원인은?
- `instanceof` / `getClass()` 비교가 Lazy 프록시와 함께 쓰일 때 주의할 점은?
- OSIV 패턴은 어떤 원리로 트랜잭션 밖에서 Lazy Loading을 가능하게 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 연관 엔티티를 항상 즉시 로드하면 성능이 저하된다

```java
// EAGER 로딩의 문제
@ManyToOne(fetch = FetchType.EAGER)
private Team team;

// User 1명만 필요한 경우에도 Team까지 JOIN 해서 로드
User user = em.find(User.class, 1L);
// SELECT u.*, t.* FROM users u LEFT JOIN teams t ON t.id = u.team_id WHERE u.id=1
// team 정보가 필요 없어도 항상 로드됨

// Lazy Loading의 해결:
@ManyToOne(fetch = FetchType.LAZY)
private Team team;

User user = em.find(User.class, 1L);
// SELECT u.* FROM users WHERE id=1 (team 미로드)

// 실제로 team이 필요할 때만:
String teamName = user.getTeam().getName(); // 이 시점에 SELECT teams WHERE id=?
```

```
Lazy Loading 구현 방법:
  user.getTeam()이 반환하는 것은 실제 Team이 아닌 프록시 객체
  프록시: Team을 상속한 가짜 Team (TeamHibernateProxy...)
  프록시의 필드 접근 시 → 초기화 로직 실행 → DB 조회 → 실제 Team 데이터로 채움
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Lazy 프록시와 실제 엔티티를 구별하지 않고 사용한다

```java
// ❌ 프록시와 실제 엔티티 비교 함정
@Transactional
public void process(Long userId) {
    User user = userRepository.findById(userId).get();
    Team team = user.getTeam(); // Lazy 프록시 반환

    // getClass() 비교 — 프록시와 실제 타입 불일치
    if (team.getClass() == Team.class) { // false! 프록시 클래스
        doSomething();
    }

    // instanceof는 올바르게 동작 (프록시가 Team 상속)
    if (team instanceof Team) { // true
        doSomething(); // ✅
    }

    // Hibernate.isInitialized()로 프록시 상태 확인
    boolean initialized = Hibernate.isInitialized(team); // false (초기화 전)
    // 강제 초기화
    Hibernate.initialize(team); // SELECT teams WHERE id=?
}
```

### Before: @Transactional이 없어도 Lazy Loading이 된다

```java
// ❌ 트랜잭션 없이 Lazy Loading 시도
@Service
public class UserService {

    @Transactional(readOnly = true)
    public User findUser(Long id) {
        return userRepository.findById(id).get();
        // 트랜잭션 종료 → EntityManager 닫힘 → user Detached 상태
    }
}

@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        User user = userService.findUser(id); // Detached 상태
        return new UserResponse(
            user.getName(),
            user.getTeam().getName() // LazyInitializationException!
            // team 프록시 접근 → EntityManager 없음 → 초기화 불가
        );
    }
}
```

---

## ✨ 올바른 이해와 패턴

### After: Lazy 프록시의 정체와 생명주기

```
프록시 구조:
  Team (원본 엔티티 클래스)
    └── Team$HibernateProxy$XYZ (ByteBuddy가 생성한 프록시 서브클래스)
          - teamId: Long (초기화 전에는 ID만 알고 있음)
          - handler: ByteBuddyInterceptor (초기화 로직 담당)
          - target: Team (초기화 후 실제 Team 인스턴스)

초기화 전:
  team.getId()     → 프록시 내 teamId 반환 (DB 쿼리 없음)
  team.getName()   → handler 개입 → DB 쿼리 → target에 실제 Team 설정 → target.getName()

초기화 후:
  이후 모든 메서드 호출 → target에 위임 (DB 쿼리 없음)

LazyInitializationException 발생 조건:
  EntityManager가 닫힌 상태 (트랜잭션 종료 후)
  프록시 초기화 시도 → EntityManager 없음 → 예외
```

---

## 🔬 내부 동작 원리 — Lazy 프록시 생성 소스 추적

### 1. ByteBuddyProxyFactory — 프록시 클래스 생성

```java
// Hibernate의 프록시 팩토리
public class ByteBuddyProxyFactory implements ProxyFactory {

    @Override
    public HibernateProxy getProxy(
            EntityKey entityKey,
            SessionImplementor session) {

        // ByteBuddy로 엔티티 서브클래스 동적 생성
        // (애플리케이션 시작 시 클래스 생성, 인스턴스는 Lazy 접근 시 생성)

        Class<?> proxyClass = ProxyConfiguration.get(persistentClass);
        // persistentClass = Team.class
        // proxyClass = Team$HibernateProxy$abc123 (미리 생성됨)

        // 프록시 인스턴스 생성
        HibernateProxy proxy = (HibernateProxy) objenesis.newInstance(proxyClass);
        // Objenesis: 생성자 호출 없이 인스턴스 생성 (JVM 레벨)

        // 인터셉터 설정
        ByteBuddyInterceptor interceptor = new ByteBuddyInterceptor(
            entityKey.getIdentifier(),  // team ID만 저장
            session                      // EntityManager 참조
        );
        ((ProxyConfiguration) proxy).$$_hibernate_set_interceptor(interceptor);

        return proxy;
    }
}

// ByteBuddy가 생성하는 프록시 클래스 (개념적)
public class Team$HibernateProxy$XYZ extends Team implements HibernateProxy, ProxyConfiguration {

    private ByteBuddyInterceptor $$_hibernate_interceptor;

    // 모든 메서드 오버라이딩
    @Override
    public String getName() {
        // ByteBuddyInterceptor 개입
        return (String) $$_hibernate_interceptor.intercept(this, "getName", ...);
    }

    // getId()는 오버라이딩 안 함 (또는 interceptor에서 ID는 바로 반환)
}
```

### 2. ByteBuddyInterceptor — 초기화 로직

```java
// ByteBuddyInterceptor.intercept() — 프록시 메서드 호출 시 실행
public class ByteBuddyInterceptor implements MethodInterceptor {

    private final Object id;         // 엔티티 ID (초기화 전부터 알고 있음)
    private SessionImplementor session; // EntityManager 참조
    private Object target;           // 실제 엔티티 (초기화 후 설정)
    private boolean initialized = false;

    @Override
    public Object intercept(Object proxy, Method method, Object[] args, ...) {

        // getId()는 초기화 없이 반환 (ID는 이미 알고 있음)
        if (method.getName().equals("getId")) {
            return this.id;
        }

        // 아직 초기화 안 됨
        if (!initialized) {
            // Session(EntityManager)이 열려있는지 확인
            if (session == null || !session.isOpen()) {
                throw new LazyInitializationException(
                    "could not initialize proxy [" + entityName + "#" + id + "] - no Session");
            }

            // DB에서 실제 엔티티 로드
            target = session.immediateLoad(entityName, id);
            // → SELECT * FROM teams WHERE id=?

            initialized = true;
            session = null; // Session 참조 해제
        }

        // 실제 엔티티에 메서드 위임
        return method.invoke(target, args);
    }
}
```

### 3. LazyInitializationException 발생 상황과 해결

```java
// 상황 1: 트랜잭션 종료 후 Lazy 접근
@Transactional
public User findUser(Long id) {
    return userRepository.findById(id).get();
} // 트랜잭션 종료 → EntityManager 닫힘

// Controller에서:
User user = userService.findUser(id);
user.getTeam().getName(); // LazyInitializationException

// 해결 1: 서비스에서 DTO로 변환
@Transactional(readOnly = true)
public UserDto findUserDto(Long id) {
    User user = userRepository.findById(id).get();
    return new UserDto(user.getName(), user.getTeam().getName()); // 트랜잭션 내에서 접근
}

// 해결 2: Fetch Join으로 즉시 로드
@Transactional(readOnly = true)
public User findUserWithTeam(Long id) {
    return userRepository.findByIdWithTeam(id); // JOIN FETCH team
}

// 해결 3: OSIV 활성화 (권장하지 않음)
// spring.jpa.open-in-view=true → 요청 범위 EntityManager
```

### 4. OSIV(Open Session In View) — 원리

```java
// OpenEntityManagerInViewInterceptor — HTTP 요청 범위 EntityManager
public class OpenEntityManagerInViewInterceptor implements AsyncWebRequestInterceptor {

    @Override
    public void preHandle(WebRequest request) {
        // HTTP 요청 시작 시 EntityManager 생성
        EntityManagerFactory emf = getEntityManagerFactory();
        EntityManager em = createEntityManager(emf);
        // → ThreadLocal에 EntityManager 바인딩 (트랜잭션 없이)
        EntityManagerHolder emHolder = new EntityManagerHolder(em);
        TransactionSynchronizationManager.bindResource(emf, emHolder);
    }

    @Override
    public void afterCompletion(WebRequest request, Exception ex) {
        // HTTP 요청 종료 시 EntityManager 닫힘
        EntityManagerHolder emHolder =
            (EntityManagerHolder) TransactionSynchronizationManager.unbindResource(emf);
        emHolder.getEntityManager().close();
    }
}

// OSIV 동작:
// HTTP 요청 → EntityManager 오픈 (트랜잭션 없음)
// @Transactional 서비스 → 같은 EntityManager에서 트랜잭션 시작/종료
// 서비스 트랜잭션 종료 후에도 EntityManager 열려있음
// Controller에서 Lazy 접근 가능 (EntityManager가 아직 열려있으므로)
// HTTP 응답 완료 후 EntityManager 닫힘

// OSIV의 문제:
// 트랜잭션 없이 DB Connection 점유
// 뷰 렌더링 중 DB 쿼리 발생 → Connection Pool 고갈 위험
// 긴 요청 처리 시 Connection 오래 점유
```

### 5. 프록시 관련 실용 유틸리티

```java
// Hibernate 프록시 유틸리티
import org.hibernate.Hibernate;

// 초기화 여부 확인
boolean isLoaded = Hibernate.isInitialized(user.getTeam()); // false면 프록시

// 강제 초기화 (트랜잭션 내에서 사용)
Hibernate.initialize(user.getTeam()); // SELECT teams WHERE id=?

// 프록시에서 실제 엔티티 추출
Team realTeam = Hibernate.unproxy(user.getTeam()); // Team 인스턴스 반환
// 또는:
Team realTeam = (Team) ((HibernateProxy) user.getTeam())
    .getHibernateLazyInitializer().getImplementation();

// 실제 클래스 확인 (프록시 우회)
Class<?> realClass = Hibernate.getClass(user.getTeam()); // Team.class (프록시 클래스 아님)
```

---

## 💻 실험으로 확인하기

### 실험 1: 프록시 클래스 확인

```java
@Transactional
@Test
void proxyClassInspection() {
    User user = userRepository.findById(1L).get(); // team은 LAZY

    Team team = user.getTeam(); // 프록시 반환

    System.out.println(team.getClass().getName());
    // com.example.Team$HibernateProxy$xyz123

    System.out.println(team instanceof Team); // true (상속)
    System.out.println(Hibernate.isInitialized(team)); // false

    // ID 접근 → 초기화 없음
    Long id = team.getId(); // SELECT 없음

    // 다른 필드 접근 → 초기화
    String name = team.getName(); // SELECT teams WHERE id=?
    System.out.println(Hibernate.isInitialized(team)); // true
}
```

### 실험 2: LazyInitializationException 재현

```java
@Test  // @Transactional 없음
void lazyInitializationException() {
    User user = transactionTemplate.execute(status ->
        userRepository.findById(1L).get()
    ); // 트랜잭션 종료 → Detached

    // 프록시는 아직 초기화 안 됨
    assertThrows(LazyInitializationException.class, () -> {
        user.getTeam().getName(); // 예외 발생
    });
}
```

### 실험 3: Hibernate.initialize()로 명시적 초기화

```java
@Transactional
@Test
void explicitInitialization() {
    User user = userRepository.findById(1L).get();

    // 트랜잭션 내에서 명시적 초기화
    Hibernate.initialize(user.getTeam()); // SELECT teams WHERE id=?

    // 이제 트랜잭션 종료 후에도 사용 가능
    // (team이 이미 초기화됐으므로 프록시가 실제 엔티티로 채워짐)
}
```

---

## ⚡ 성능 임팩트

```
Lazy vs Eager 로딩:

Eager (@ManyToOne 기본):
  항상 JOIN → 필요 없어도 로드 → 불필요한 DB 쿼리 + 메모리
  JPQL에서 N+1 발생

Lazy:
  필요할 때만 로드 → 최적화 가능
  단, 초기화 타이밍 관리 필요

프록시 생성 비용:
  클래스 생성: 애플리케이션 시작 시 1회 (ByteBuddy)
  인스턴스 생성: Objenesis 사용 → 생성자 호출 없음 → ~수 μs
  intercept() 오버헤드: 초기화 여부 확인 → ~수백 ns

Hibernate.initialize() vs Fetch Join:
  initialize(): 별도 SELECT 1번 (N+1의 1회 버전)
  Fetch Join: JOIN으로 한 번에 → 더 효율적
  → 여러 엔티티에 대해 필요 → Fetch Join 또는 @BatchSize 사용

OSIV 성능 영향:
  Connection 점유 시간: DB 쿼리 시간 → HTTP 응답 완료 시간으로 늘어남
  예: 서비스(50ms) + 뷰 렌더링 Lazy 쿼리(10ms) = 60ms 동안 Connection 점유
  고트래픽: Connection Pool 고갈 → 대기 → 레이턴시 폭발
```

---

## 🤔 트레이드오프

```
Lazy Loading + OSIV (편의):
  ✅ 어디서나 Lazy Loading 가능 (Controller, View)
  ✅ N+1 걱정 없이 개발
  ❌ Connection Pool 고갈 위험
  ❌ Controller에서 DB 접근 (레이어 경계 모호)
  ❌ 성능 예측 어려움
  → 소규모, 트래픽 낮은 서비스

Lazy Loading + OSIV 비활성화 (엄격):
  ✅ 레이어 경계 명확 (Service만 DB 접근)
  ✅ Connection 사용 예측 가능
  ✅ 고트래픽 환경 적합
  ❌ 모든 Lazy Loading을 Service에서 처리 필요
  ❌ Fetch Join, @BatchSize 설계 필요
  → 프로덕션 권장

Eager Loading:
  ✅ LazyInitializationException 없음
  ❌ 불필요한 데이터 항상 로드
  ❌ JPQL에서 N+1 더 심각
  → 소수의 항상 필요한 연관에만 (일반적으로 비권장)
```

---

## 📌 핵심 정리

```
Lazy 프록시 구조
  Team$HibernateProxy$XYZ extends Team implements HibernateProxy
  ByteBuddyInterceptor: ID 저장 + Session 참조 + 초기화 로직
  getId(): 초기화 없이 반환 (ID는 미리 알고 있음)
  기타 메서드: 초기화 후 target에 위임

LazyInitializationException 원인
  EntityManager가 닫힌 상태 (트랜잭션 종료 후)
  프록시 초기화 시도 → Session 없음 → 예외

해결 방법
  Service에서 DTO 변환 (가장 권장)
  Fetch Join / @EntityGraph (즉시 로드)
  Hibernate.initialize() (트랜잭션 내 명시적 초기화)
  OSIV 활성화 (편의, 성능 위험)

프록시 유틸리티
  Hibernate.isInitialized(obj): 초기화 여부
  Hibernate.initialize(obj): 강제 초기화
  Hibernate.unproxy(obj): 실제 엔티티 추출
  instanceof: 올바르게 동작 (getClass()는 주의)
```

---

## 🤔 생각해볼 문제

**Q1.** `user.getTeam().getId()`를 호출할 때 SQL이 발생하는가? Lazy 프록시가 ID를 미리 알고 있다면 어떻게 알고 있는가?

**Q2.** OSIV 활성화 상태에서 `@Transactional` 서비스가 데이터를 수정하고 트랜잭션을 커밋했다. 이후 Controller에서 동일 엔티티의 Lazy 연관에 접근한다면, 수정된 데이터가 반영된 연관을 볼 수 있는가?

**Q3.** `equals()` 메서드를 엔티티에 ID 기반으로 구현했을 때, Lazy 프록시와 비교하면 올바르게 동작하는가?

> 💡 **해설**
>
> **Q1.** SQL이 발생하지 않는다. 프록시는 생성 시점에 ID 값을 `ByteBuddyInterceptor`의 `id` 필드에 저장한다. `user.getTeam()`이 프록시를 반환할 때 이미 `team_id`(외래키) 값을 알고 있기 때문이다. `getId()` 메서드는 인터셉터에서 DB 조회 없이 저장된 ID를 직접 반환한다. Hibernate는 이를 위해 `@Id` 필드에 대한 getter는 초기화 없이 반환하도록 프록시를 설계한다. 이것이 `user.getTeam().getId()`는 N+1을 유발하지 않는 이유다.
>
> **Q2.** 수정된 데이터가 반영된 연관을 볼 수 있다. OSIV에서 같은 EntityManager(같은 Persistence Context)가 요청 전체를 커버한다. 서비스에서 트랜잭션을 커밋하면 DB에 변경이 반영된다. Controller에서 Lazy 연관에 접근할 때 프록시가 초기화되며 DB에서 최신 데이터를 가져온다(1차 캐시에 없다면). 단, 이미 1차 캐시에 있는 엔티티라면 캐시에서 반환되므로 주의가 필요하다. `@Modifying` JPQL UPDATE 후에는 1차 캐시가 자동으로 무효화되지 않으므로 `clearAutomatically=true` 또는 `em.refresh()`가 필요하다.
>
> **Q3.** `equals()`가 ID만으로 비교한다면 올바르게 동작한다. 프록시는 Team을 상속하므로 Team의 `equals()`를 상속받는다. 프록시의 `getId()`는 초기화 없이 ID를 반환하므로, ID 기반 `equals()`는 DB 쿼리 없이 올바르게 비교된다. 단, `instanceof` 대신 `getClass()`를 사용하거나 `obj.id`에 직접 필드 접근한다면 문제가 생긴다. Hibernate 공식 권장: `equals()`/`hashCode()`를 비즈니스 키(ID 또는 자연키)로 구현하되, `getClass()` 대신 `instanceof`를 사용하고, 프록시 초기화가 필요한 필드(ID 외)에는 접근하지 않는다. `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` + `@EqualsAndHashCode.Include`로 ID 필드만 지정하는 Lombok 패턴이 실용적이다.

---

<div align="center">

**[⬅️ 이전: N+1 문제 완전 해결](./04-n-plus-one-complete-solution.md)** | **[다음: Cascade 타입과 Orphan Removal ➡️](./06-cascade-orphan-removal.md)**

</div>
