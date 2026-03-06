# EntityManager vs Hibernate Session — JPA 표준과 Hibernate 구현의 경계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EntityManager`와 `Hibernate Session`은 어떤 관계인가 — 상속인가 위임인가?
- Spring이 주입하는 `EntityManager`가 실제 `EntityManager`가 아닌 이유는?
- `EntityManager.unwrap(Session.class)`가 필요한 상황은 언제인가?
- `SharedEntityManagerCreator`가 스레드 안전한 `EntityManager` 프록시를 만드는 원리는?
- JPA 표준 API만으로 해결 안 되는 Hibernate 전용 기능은 무엇인가?

---

## 🔍 왜 이게 존재하는가

### 문제: JPA 표준과 Hibernate 구현 사이의 간격

```java
// JPA 표준 API — 모든 JPA 구현체에서 동작
EntityManager em;

em.persist(entity);
em.find(User.class, id);
em.createQuery("SELECT u FROM User u", User.class).getResultList();
em.flush();
em.clear();

// Hibernate 전용 기능 — JPA 표준에 없음
Session session = em.unwrap(Session.class);

session.setDefaultReadOnly(true);          // readOnly 엔티티 최적화
session.enableFilter("activeFilter");      // @Filter 동적 필터
session.createNativeQuery(sql)
       .addScalar("name", StringType.INSTANCE)
       .setResultTransformer(...);         // 결과 변환기
session.doWork(connection -> { ... });     // JDBC Connection 직접 접근
session.getStatistics();                   // Hibernate 통계
```

```
JPA(Jakarta Persistence API): 표준 인터페이스
  → 특정 구현체에 종속되지 않는 코드 작성 가능
  → Hibernate, EclipseLink, OpenJPA 등 교체 가능 (이론적으로)

Hibernate: JPA의 구현체 + 추가 기능
  → Session = EntityManager의 Hibernate 구현
  → JPA 표준에 없는 성능 최적화 기능 다수 포함
  → Spring Boot JPA 환경에서 사실상 표준

실무:
  대부분의 코드 → JPA 표준 API (EntityManager)
  특수 최적화 → Hibernate Session 직접 접근
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @PersistenceContext로 주입받은 EntityManager는 실제 EntityManager다

```java
// ❌ 잘못된 이해
@Repository
public class UserRepository {

    @PersistenceContext
    private EntityManager em;
    // em = 실제 EntityManager? → 아님!
    // em = SharedEntityManagerCreator가 만든 프록시

    public User find(Long id) {
        // em.find() → 프록시의 find() → ThreadLocal에서 실제 EntityManager 조회 → 위임
        return em.find(User.class, id);
    }
}

// 왜 프록시인가?
// EntityManager는 스레드 안전하지 않음 (단일 트랜잭션에서만 사용)
// 여러 스레드가 같은 @Repository 빈을 공유 → 같은 em 필드 공유
// → 실제 EntityManager를 공유하면 위험
// → 프록시가 호출 시마다 현재 스레드의 EntityManager를 가져옴
```

### Before: EntityManager와 Session은 완전히 다른 객체다

```java
// ❌ 잘못된 이해: EntityManager와 Session은 별개의 인스턴스

// ✅ 실제:
// Hibernate 환경에서 EntityManager의 실제 구현이 Session
// EntityManager em = new SessionImpl(...)
// em instanceof Session → true (Hibernate에서)

EntityManager em = entityManagerFactory.createEntityManager();
Session session = em.unwrap(Session.class);
// session == em 내부의 같은 객체 (또는 em 자체가 Session 구현체)
// 동일한 1차 캐시, 동일한 트랜잭션 컨텍스트 공유
```

---

## ✨ 올바른 이해와 패턴

### After: 계층 구조와 Spring 통합

```
JPA 표준 인터페이스 계층:
  EntityManager (jakarta.persistence)
    ↓ 구현
  SessionImpl (org.hibernate.internal) ← 실제 Hibernate 구현체
    implements EntityManager, Session

Spring 통합 계층:
  @PersistenceContext EntityManager em
    → SharedEntityManagerCreator.SharedEntityManagerInvocationHandler (프록시)
    → 메서드 호출 시 TransactionSynchronizationManager에서 실제 EntityManager 조회
    → 없으면 새 EntityManager 생성 (트랜잭션 없는 경우)

접근 방식:
  일반 JPA 작업       → EntityManager 직접 사용
  Hibernate 전용 기능 → em.unwrap(Session.class)
  JDBC 직접 접근      → em.unwrap(Connection.class) 또는 session.doWork()
```

---

## 🔬 내부 동작 원리 — EntityManager 프록시 소스 추적

### 1. SharedEntityManagerCreator — 프록시 생성

```java
// SharedEntityManagerCreator — EntityManager 프록시 생성의 핵심
public class SharedEntityManagerCreator {

    public static EntityManager createSharedEntityManager(
            EntityManagerFactory emf, @Nullable Map<?, ?> properties, boolean synchronizedWithTransaction) {

        // JDK 동적 프록시 생성
        // EntityManager 인터페이스를 구현하는 프록시
        return (EntityManager) Proxy.newProxyInstance(
            SharedEntityManagerCreator.class.getClassLoader(),
            new Class<?>[] { EntityManager.class, EntityManagerProxy.class },
            new SharedEntityManagerInvocationHandler(emf, properties, synchronizedWithTransaction)
        );
    }
}

// SharedEntityManagerInvocationHandler — 실제 EntityManager 조회 로직
class SharedEntityManagerInvocationHandler implements InvocationHandler {

    private final EntityManagerFactory targetFactory;

    @Override
    @Nullable
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        // EntityManager 프록시 자체에 대한 메서드 (getEntityManagerFactory 등) 처리
        if (method.getName().equals("getEntityManagerFactory")) {
            return this.targetFactory;
        }

        // 현재 스레드의 실제 EntityManager 조회
        EntityManager target = EntityManagerFactoryUtils.doGetTransactionalEntityManager(
            this.targetFactory, this.properties, this.synchronizedWithTransaction);
        // → TransactionSynchronizationManager.getResource(emf)
        // → 트랜잭션 활성 시: 해당 트랜잭션의 EntityManager 반환
        // → 트랜잭션 없을 시: 임시 EntityManager 생성

        if (target == null) {
            // 트랜잭션 없는 상태에서 쓰기 작업 → 예외 또는 임시 EM 생성
            if (method.getName().equals("getTransaction")) {
                throw new IllegalStateException(
                    "Cannot get transaction for shared EntityManager");
            }
            // 읽기 전용 작업은 임시 EntityManager로 처리
            target = this.targetFactory.createEntityManager(this.properties);
        }

        // 실제 EntityManager에 메서드 위임
        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }
}
```

### 2. EntityManager.unwrap() — Hibernate Session 접근

```java
// EntityManager.unwrap() — JPA 표준 메서드
// 구현체의 실제 타입으로 캐스팅

// Hibernate의 SessionImpl.unwrap() 구현
@Override
public <T> T unwrap(Class<T> type) {
    if (type.isAssignableFrom(Session.class)) {
        return type.cast(this);  // this = SessionImpl (Session 구현체)
    }
    if (type.isAssignableFrom(SessionImplementor.class)) {
        return type.cast(this);
    }
    if (type.isAssignableFrom(SharedSessionContractImplementor.class)) {
        return type.cast(this);
    }
    if (type.isAssignableFrom(Connection.class)) {
        return type.cast(connection());  // JDBC Connection 반환
    }
    // ...
    throw new PersistenceException("Hibernate cannot unwrap " + type);
}

// Spring 프록시 환경에서의 unwrap
// em (프록시) → em.unwrap(Session.class)
// → SharedEntityManagerInvocationHandler.invoke() 호출
// → 실제 EntityManager(SessionImpl) 획득
// → SessionImpl.unwrap(Session.class) 호출 → SessionImpl 반환
```

### 3. EntityManagerFactory vs SessionFactory

```java
// EntityManagerFactory — JPA 표준
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myUnit");
EntityManager em = emf.createEntityManager();

// SessionFactory — Hibernate 전용 (JPA 사용 시 내부에서 사용)
SessionFactory sf = emf.unwrap(SessionFactory.class);  // EMF에서 SF 획득
Session session = sf.openSession();
// 또는:
Session session = sf.getCurrentSession();  // 현재 트랜잭션의 Session

// Spring Boot에서의 관계:
// EntityManagerFactory (Spring 관리 빈)
//   = LocalContainerEntityManagerFactoryBean이 생성
//   = 내부적으로 Hibernate SessionFactory를 래핑
// SessionFactory = emf.unwrap(SessionFactory.class) 로 접근 가능
```

### 4. Hibernate 전용 기능 — Session 직접 사용 사례

```java
@Repository
@RequiredArgsConstructor
public class UserHibernateRepository {

    private final EntityManager em;

    // 사례 1: @Filter 동적 필터 적용
    public List<User> findActiveUsers() {
        Session session = em.unwrap(Session.class);
        session.enableFilter("statusFilter").setParameter("status", "ACTIVE");
        try {
            return em.createQuery("SELECT u FROM User u", User.class).getResultList();
        } finally {
            session.disableFilter("statusFilter");
        }
    }

    // 사례 2: readOnly 엔티티 최적화 (대량 조회 시)
    public List<User> findAllReadOnly() {
        Session session = em.unwrap(Session.class);
        session.setDefaultReadOnly(true);  // 로드되는 엔티티를 readOnly로 마킹
        try {
            return em.createQuery("SELECT u FROM User u", User.class).getResultList();
            // loadedState(스냅샷) 생성 안 함 → 메모리 절감
        } finally {
            session.setDefaultReadOnly(false);
        }
    }

    // 사례 3: Hibernate 통계 조회
    public void printStatistics() {
        Session session = em.unwrap(Session.class);
        Statistics stats = session.getSessionFactory().getStatistics();
        System.out.println("쿼리 수: " + stats.getQueryExecutionCount());
        System.out.println("2차 캐시 히트율: " + stats.getSecondLevelCacheHitCount());
    }

    // 사례 4: JDBC Connection 직접 접근 (StoredProcedure 등)
    public void executeNativeWork() {
        Session session = em.unwrap(Session.class);
        session.doWork(connection -> {
            CallableStatement cs = connection.prepareCall("{CALL my_procedure(?)}");
            cs.setLong(1, 42L);
            cs.execute();
        });
        // doWork: Connection 관리를 Hibernate에 위임 (트랜잭션 공유)
    }

    // 사례 5: Scroll (대용량 커서 기반 처리)
    public void processLargeDataset() {
        Session session = em.unwrap(Session.class);
        ScrollableResults<User> results = session.createQuery("FROM User", User.class)
            .setReadOnly(true)
            .setFetchSize(100)  // JDBC fetchSize (한 번에 가져오는 행 수)
            .scroll(ScrollMode.FORWARD_ONLY);

        int count = 0;
        while (results.next()) {
            User user = results.get();
            process(user);
            if (++count % 100 == 0) {
                session.flush();
                session.clear();  // 1차 캐시 비움 → 메모리 관리
            }
        }
        results.close();
    }
}
```

### 5. 트랜잭션 없는 환경에서의 EntityManager 동작

```java
// 트랜잭션 없이 EntityManager 메서드 호출
@Service
public class UserService {

    @PersistenceContext
    private EntityManager em;

    // 트랜잭션 없이 조회 — 임시 EntityManager 생성 후 즉시 닫힘
    public User findUser(Long id) {
        return em.find(User.class, id);
        // → 프록시: 실제 EM 없음 → 임시 EM 생성
        // → find() 실행 후 임시 EM 닫힘
        // → Detached 상태의 엔티티 반환 (Lazy Loading 불가!)
    }

    // 트랜잭션 없이 쓰기 — TransactionRequiredException
    public void saveWithoutTx(User user) {
        em.persist(user);  // → IllegalStateException 또는 TransactionRequiredException
        // 프록시가 "Cannot get transaction for shared EntityManager" 예외
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 주입된 EntityManager가 프록시인지 확인

```java
@Autowired
EntityManager em;

@Test
void entityManagerIsProxy() {
    System.out.println(em.getClass().getName());
    // com.sun.proxy.$Proxy61 (JDK 동적 프록시)

    System.out.println(em instanceof Proxy);  // true

    // 실제 EntityManager 가져오기
    EntityManager real = em.unwrap(EntityManager.class);
    // 또는:
    Session session = em.unwrap(Session.class);
    System.out.println(session.getClass().getName());
    // org.hibernate.internal.SessionImpl
}
```

### 실험 2: 동일 트랜잭션에서 EntityManager 동일성 확인

```java
@Transactional
@Test
void sameEntityManagerInTransaction() {
    Session session1 = em.unwrap(Session.class);
    Session session2 = em.unwrap(Session.class);
    // 동일 트랜잭션 → 동일 Session 인스턴스
    assertSame(session1, session2);  // ✅ 같은 객체
}
```

### 실험 3: Hibernate Filter 적용

```java
// @Filter 정의
@Entity
@Filter(name = "statusFilter", condition = "status = :status")
public class User {
    @Id private Long id;
    private String status;
}

@Test
@Transactional
void hibernateFilter() {
    Session session = em.unwrap(Session.class);
    session.enableFilter("statusFilter").setParameter("status", "ACTIVE");

    List<User> users = em.createQuery("FROM User", User.class).getResultList();
    // SQL: SELECT * FROM users WHERE status = 'ACTIVE'
    // @Filter가 자동으로 WHERE 절 추가

    session.disableFilter("statusFilter");
}
```

---

## ⚡ 성능 임팩트

```
EntityManager 프록시 오버헤드:
  메서드 호출당 프록시 인터셉션: ~수백 ns
  ThreadLocal에서 실제 EM 조회: ~수백 ns
  합계: ~1μs 미만 → 무시 가능

Session.setDefaultReadOnly(true) 효과:
  로드 엔티티당 loadedState(스냅샷) 생성 생략
  1000건 × 10 fields × 8 bytes = ~80KB 절감
  flush 시 dirty checking 비용 제거

session.setFetchSize() 효과:
  기본값: DB 드라이버 기본 (MySQL: 전체 결과를 메모리에 로드)
  setFetchSize(100): 100건씩 스트리밍 → 메모리 폭발 방지
  대용량 조회 (100만 건): 기본 vs fetchSize(100) → 메모리 차이 수백 MB
```

---

## 🤔 트레이드오프

```
JPA 표준 EntityManager만 사용:
  ✅ 구현체 독립적 코드 (이론적)
  ✅ 테스트 용이 (Mock 가능)
  ❌ Hibernate 최적화 기능 사용 불가
  ❌ 실무에서 Hibernate를 교체하는 경우는 거의 없음

Hibernate Session 직접 사용:
  ✅ readOnly 최적화, Filter, Scroll, doWork 등 강력한 기능
  ❌ Hibernate 의존성 고착
  ❌ 코드 가독성 저하 (unwrap 코드)
  → 최적화가 필요한 특정 지점에만 제한적으로 사용

실무 권장:
  일반 CRUD → JPA 표준 (EntityManager / Spring Data Repository)
  대량 조회 최적화 → session.setDefaultReadOnly(true), setFetchSize
  동적 필터 → session.enableFilter()
  JDBC 직접 접근 → session.doWork() 또는 JdbcTemplate
```

---

## 📌 핵심 정리

```
EntityManager와 Session의 관계
  JPA: EntityManager (인터페이스)
  Hibernate: SessionImpl implements EntityManager, Session
  → em.unwrap(Session.class) → 같은 객체(SessionImpl) 반환

Spring의 EntityManager 프록시
  @PersistenceContext EntityManager → SharedEntityManagerCreator 프록시
  호출 시마다 TransactionSynchronizationManager에서 실제 EM 조회
  → 스레드 안전한 EntityManager 단일 빈 공유 가능

unwrap 사용 시점
  session.setDefaultReadOnly(true): 대량 조회 스냅샷 비용 절감
  session.enableFilter(): 동적 필터 (@Filter)
  session.doWork(): JDBC Connection 직접 접근
  session.scroll(): 대용량 커서 기반 처리
  session.setFetchSize(): 스트리밍 처리

트랜잭션 없는 EntityManager
  조회: 임시 EM 생성 → 즉시 닫힘 → Detached 엔티티 반환
  쓰기: TransactionRequiredException
```

---

## 🤔 생각해볼 문제

**Q1.** `em.unwrap(Session.class)`를 호출하면 Spring의 프록시가 해제되어 스레드 안전성이 깨지는가?

**Q2.** `session.doWork(connection -> { ... })`에서 직접 Connection을 사용해 INSERT를 실행했을 때, 이 변경이 현재 트랜잭션에 포함되는가?

**Q3.** `session.scroll()`로 대용량 데이터를 처리하면서 중간에 `session.flush(); session.clear()`를 호출하는 패턴에서, `setDefaultReadOnly(true)`를 함께 사용하면 어떤 효과가 있는가?

> 💡 **해설**
>
> **Q1.** 스레드 안전성이 깨지지 않는다. `em.unwrap(Session.class)`는 현재 스레드의 트랜잭션에 바인딩된 실제 `SessionImpl`을 반환한다. 이 `SessionImpl`은 현재 스레드의 트랜잭션 컨텍스트에 속하므로, 다른 스레드와 공유되지 않는다. 프록시를 우회하는 것이지, 다른 스레드의 Session을 가져오는 것이 아니다. 단, 반환된 `Session`을 다른 스레드로 전달하면 스레드 안전성이 깨질 수 있다.
>
> **Q2.** 현재 트랜잭션에 포함된다. `session.doWork()`는 현재 트랜잭션의 JDBC Connection을 제공한다. 이 Connection은 `auto-commit=false`이고 현재 트랜잭션과 동일한 Connection이므로, `doWork` 내부의 INSERT는 현재 트랜잭션의 일부다. 트랜잭션이 롤백되면 `doWork` 내의 변경도 롤백된다. 단, `doWork` 내부에서 `connection.commit()`을 직접 호출하면 현재 트랜잭션 범위를 벗어나게 된다.
>
> **Q3.** `setDefaultReadOnly(true)`와 `flush/clear` 패턴을 함께 사용하면 메모리 효율이 크게 향상된다. `readOnly=true`이면 로드되는 엔티티의 `loadedState`(스냅샷)를 생성하지 않으므로 메모리 사용량이 절반 이하로 줄어든다. 또한 `flush()` 호출 시 dirty checking이 스킵되어(`readOnly` 엔티티는 변경 감지 대상 제외) `flush` 비용이 거의 없어진다. 결과적으로 100만 건 처리 시: `flush/clear`만 사용 시 청크마다 dirty checking 발생, `readOnly + flush/clear`는 flush 비용 제거 + 스냅샷 메모리 제거로 처리 속도가 30~50% 향상될 수 있다.

---

<div align="center">

**[다음: Persistence Context — 1차 캐시 동작 원리 ➡️](./02-persistence-context-first-cache.md)**

</div>
