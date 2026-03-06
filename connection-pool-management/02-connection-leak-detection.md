# Connection Leak 탐지와 디버깅 — leakDetectionThreshold와 원인 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Connection Leak이 발생하는 전형적인 코드 패턴은?
- `leakDetectionThreshold`가 Leak를 어떻게 감지하고 경고를 출력하는가?
- HikariCP 경고 로그에서 스택 트레이스를 보고 Leak 원인을 찾는 방법은?
- Spring `@Transactional` 환경에서도 Connection Leak이 발생하는 경우는?
- Leak 방지를 위한 코드 패턴과 리뷰 체크리스트는?

---

## 🔍 왜 이게 존재하는가

### 문제: Connection을 반환하지 않으면 Pool이 고갈된다

```java
// ❌ Connection Leak — finally 블록 없음
public void riskyOperation() throws SQLException {
    Connection conn = dataSource.getConnection(); // Pool에서 획득
    PreparedStatement ps = conn.prepareStatement("UPDATE ...");
    ps.executeUpdate();
    // 예외 발생 시 conn.close() 호출 안 됨!
    conn.close(); // 정상 경로에서만 반환
}
// 예외 발생 → conn이 Pool에 반환되지 않음 → Leaked Connection
// 이런 호출이 반복되면 Pool 고갈 → connectionTimeout 예외

// ❌ Pool 고갈 증상
// HikariPool-1 - Connection is not available, request timed out after 5000ms
// → 모든 Connection이 Leak으로 사라짐
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Transactional을 쓰면 Leak이 발생하지 않는다

```java
// ❌ @Transactional 환경에서도 Leak 발생 케이스

// [케이스 1] 비동기 실행에서 트랜잭션 컨텍스트 누락
@Transactional
public void process() {
    CompletableFuture.runAsync(() -> {
        // 새 스레드 → TransactionSynchronizationManager에 Connection 없음
        // JdbcTemplate이 새 Connection 획득
        jdbcTemplate.update("INSERT ...");
        // 이 Connection은 트랜잭션 없이 auto-commit으로 실행
        // DataSourceUtils.releaseConnection() 없으면 Leak 가능
    });
}

// [케이스 2] try-with-resources 없이 EntityManager 수동 생성
EntityManager em = entityManagerFactory.createEntityManager();
em.getTransaction().begin();
em.persist(entity);
em.getTransaction().commit();
// em.close() 누락! → Connection Leak

// [케이스 3] StatelessSession 미닫힘
SessionFactory sf = emf.unwrap(SessionFactory.class);
StatelessSession ss = sf.openStatelessSession(); // Connection 획득
ss.beginTransaction();
ss.insert(entity);
ss.getTransaction().commit();
// ss.close() 누락! → Connection Leak
```

### Before: Pool 고갈 = 서버 문제, 재시작하면 해결된다

```java
// ❌ 재시작으로 일시적 해결 — 근본 원인 미제거
// → 서비스 재시작 후 시간이 지나면 다시 고갈

// ✅ 올바른 접근:
// leakDetectionThreshold로 Leak 위치 확인
// 스택 트레이스로 원인 코드 식별
// 코드 수정으로 근본 해결
```

---

## ✨ 올바른 이해와 패턴

### After: leakDetectionThreshold 동작 원리

```
leakDetectionThreshold: Connection 대여 후
  설정한 시간(예: 2000ms) 이상 반환되지 않으면
  → "Connection leak detected" 경고 + 스택 트레이스 출력
  → Connection은 계속 사용 가능 (강제 반환 아님)
  → 경고 기반으로 코드 수정

경고 로그 예시:
  WARN HikariPool-1 - Connection leak detection triggered for
  com.zaxxer.hikari.pool.ProxyConnection@1a2b3c4d on thread main,
  stack trace follows
    java.lang.Exception: Apparent connection leak detected
      at com.example.UserService.findUser(UserService.java:42)
      at com.example.UserController.getUser(UserController.java:28)
      ...
  → UserService.java 42번째 줄에서 Connection을 획득한 후 반환 안 함
```

---

## 🔬 내부 동작 원리 — Leak 감지 소스 추적

### 1. leakDetectionThreshold 설정과 동작

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 2000  # 2초 이상 반환 안 되면 경고
      # 개발/스테이징: 2000ms
      # 운영: 5000ms (정상 장기 트랜잭션과 구별)
```

```java
// HikariCP 내부 — Connection 대여 시 Leak 감지 타이머 시작
// PoolEntry.createProxyConnection()
ProxyConnection proxyConnection = ProxyFactory.getProxyConnection(
    poolEntry, connection, openStatements, metrics, isReadOnly, isAutoCommit);

// leakDetectionThreshold > 0이면 Leak 감지 태스크 스케줄링
if (leakDetectionThreshold > 0) {
    poolEntry.endOfLife = System.currentTimeMillis() + leakDetectionThreshold;
    poolEntry.leakTask = scheduledExecutor.schedule(
        new ProxyLeakTask(poolEntry),     // Leak 감지 Runnable
        leakDetectionThreshold,           // 지연 시간
        MILLISECONDS
    );
}

// ProxyLeakTask.run() — leakDetectionThreshold 후 실행
class ProxyLeakTask implements Runnable {
    private final PoolEntry poolEntry;
    private final Exception exception; // Connection 대여 시점의 스택 트레이스 캡처

    @Override
    public void run() {
        // 아직 반환 안 됐으면 (leakTask가 취소되지 않았으면) 경고
        if (!poolEntry.isMarkedEvicted()) {
            logger.warn("Connection leak detection triggered for {} on thread {}, " +
                "stack trace follows", poolEntry.connection, threadName, exception);
            // exception.printStackTrace() → 대여 시점 스택 트레이스 출력
        }
    }
}

// Connection 반환 시 — Leak 감지 타이머 취소
proxyConnection.close() → poolEntry.recycle()
→ poolEntry.leakTask.cancel(false); // 정상 반환 → 경고 취소
```

### 2. Connection Leak 패턴과 수정

```java
// [패턴 1] 예외 시 미반환 — try-with-resources로 수정
// ❌
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
stmt.execute(sql); // 예외 시 conn.close() 호출 안 됨
conn.close();

// ✅ try-with-resources
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement()) {
    stmt.execute(sql);
} // 자동으로 stmt, conn 닫힘 (예외 발생 시도 포함)

// [패턴 2] DataSourceUtils 미사용 — DataSourceUtils 사용으로 수정
// ❌ 직접 획득, 직접 닫기 → 트랜잭션 무시 + Leak 위험
Connection conn = dataSource.getConnection();
// ... 작업
conn.close(); // 트랜잭션 중이면 잘못된 닫기

// ✅ DataSourceUtils로 획득/반환 → 트랜잭션 연계
Connection conn = DataSourceUtils.getConnection(dataSource);
try {
    // ... 작업
} finally {
    DataSourceUtils.releaseConnection(conn, dataSource);
    // 트랜잭션 중이면 실제 닫지 않음 (트랜잭션 종료 시 반환)
}

// [패턴 3] EntityManager 미닫힘 — try-with-resources
// ❌
EntityManager em = emf.createEntityManager();
em.persist(entity);
// em.close() 없음 → Connection Leak

// ✅
EntityManager em = emf.createEntityManager();
try {
    em.getTransaction().begin();
    em.persist(entity);
    em.getTransaction().commit();
} catch (Exception e) {
    em.getTransaction().rollback();
} finally {
    em.close(); // 반드시 닫기
}

// [패턴 4] StatelessSession — try-with-resources
// ✅
try (StatelessSession ss = sessionFactory.openStatelessSession()) {
    Transaction tx = ss.beginTransaction();
    ss.insert(entity);
    tx.commit();
}
```

### 3. @Transactional 환경에서 안전한 Connection 사용

```java
// @Transactional + JdbcTemplate: 자동으로 안전
@Transactional
public void safeWithJdbcTemplate() {
    // JdbcTemplate 내부에서 DataSourceUtils.getConnection() 사용
    // → 트랜잭션 Connection 재사용
    // → 트랜잭션 종료 시 자동 반환
    jdbcTemplate.update("INSERT ...");
} // 트랜잭션 종료 → Connection 자동 반환

// @Transactional + 직접 Connection 획득: 주의 필요
@Transactional
public void manualConnectionInTransaction() {
    // ✅ DataSourceUtils 사용 → 트랜잭션 Connection 재사용
    Connection conn = DataSourceUtils.getConnection(dataSource);
    // conn은 트랜잭션의 Connection → close() 호출해도 실제 반환 안 됨
    // 트랜잭션 종료 시 자동 반환

    // ❌ dataSource.getConnection() 직접 사용 → 별도 Connection, Leak 위험
    Connection directConn = dataSource.getConnection(); // 새 Connection!
    // close() 없으면 Leak
    directConn.close(); // 명시 필요
}
```

### 4. Leak 디버깅 절차

```
Step 1: leakDetectionThreshold 설정 (개발 환경)
  spring.datasource.hikari.leak-detection-threshold=2000

Step 2: 경고 로그 확인
  "Connection leak detection triggered ... stack trace follows"
  → 스택 트레이스에서 Connection 획득 위치 확인

Step 3: 해당 코드 분석
  Connection.close()가 모든 경로에서 호출되는지 확인
  try-finally 또는 try-with-resources 적용

Step 4: 검증
  동일 시나리오 반복 실행 후 경고 없는지 확인
  Pool 모니터링: getActiveConnections() 증가 없는지 확인

Step 5: 운영 환경 leakDetectionThreshold 조정
  정상 장기 트랜잭션(배치 등) 고려해 값 설정
  너무 짧으면 오탐, 너무 길면 탐지 늦음
```

### 5. 코드 리뷰 체크리스트

```java
// Connection Leak 방지 리뷰 포인트

// ✅ 1. dataSource.getConnection() 사용 시 try-with-resources
try (Connection conn = dataSource.getConnection()) { ... }

// ✅ 2. EntityManager 수동 생성 시 finally에서 close()
EntityManager em = emf.createEntityManager();
try { ... } finally { em.close(); }

// ✅ 3. StatelessSession try-with-resources
try (StatelessSession ss = sf.openStatelessSession()) { ... }

// ✅ 4. 비동기 코드에서 Connection 획득 시 반환 보장
CompletableFuture.runAsync(() -> {
    // 새 트랜잭션 명시 또는 DataSourceUtils 사용
    transactionTemplate.execute(status -> {
        jdbcTemplate.update("...");
        return null;
    }); // 트랜잭션 종료 시 Connection 반환
});

// ✅ 5. @Transactional 없는 곳에서 JdbcTemplate: 자동 반환 (안전)
// JdbcTemplate 내부 execute()에서 finally로 반환

// ❌ 위험 패턴:
// - finally 없는 Connection.close()
// - @Transactional 밖의 emf.createEntityManager()
// - 비동기 스레드의 DataSource 직접 접근
```

---

## 💻 실험으로 확인하기

### 실험 1: Leak 재현 + 경고 로그 확인

```java
@Test
void leakDetectionDemo() throws Exception {
    // leakDetectionThreshold=2000 설정 후 실행
    Connection conn = dataSource.getConnection(); // 획득
    // 의도적으로 close() 안 함

    // 2초 후 경고 로그 출력:
    // WARN HikariPool-1 - Connection leak detection triggered...
    Thread.sleep(3000);

    // 경고 후에도 Connection은 사용 가능 (강제 종료 아님)
    conn.close(); // 이제 반환 → leakTask 취소
}
```

### 실험 2: Pool 고갈 시뮬레이션

```java
@Test
void poolExhaustionSimulation() {
    // Pool 전체 소진
    List<Connection> leaked = new ArrayList<>();
    for (int i = 0; i < 20; i++) { // maximumPoolSize=20
        try { leaked.add(dataSource.getConnection()); }
        catch (Exception e) { break; }
    }

    // Pool 고갈 상태에서 추가 요청
    long start = System.currentTimeMillis();
    assertThrows(Exception.class, () -> dataSource.getConnection());
    System.out.println("대기 후 실패: " + (System.currentTimeMillis() - start) + "ms");
    // connectionTimeout 후 예외 (기본 30초, 설정 5초)

    leaked.forEach(c -> { try { c.close(); } catch (Exception ignored) {} });
}
```

---

## ⚡ 성능 임팩트

```
leakDetectionThreshold 오버헤드:
  스케줄링 태스크 등록: ~수 μs (Connection 대여 시)
  정상 반환 시 취소: ~수 μs
  → 무시 가능

스택 트레이스 캡처 비용:
  new Exception() (스택 트레이스 포함): ~수십 μs
  → Connection 대여 시 한 번만 발생
  → 고빈도 대여 환경에서 약간의 오버헤드
  → 개발/스테이징에서 활성화, 운영에서는 선택적

Connection Leak의 실제 성능 영향:
  Leak 1개: Pool 크기 1개 감소 → 미미함
  Leak 누적: Pool 고갈 → connectionTimeout 폭발
  → 서비스 중단 수준의 영향
```

---

## 🤔 트레이드오프

```
leakDetectionThreshold 짧게 (예: 500ms):
  ✅ 빠른 Leak 감지
  ❌ 정상 장기 쿼리(배치, 분석)에서 오탐
  → 개발 환경, 모든 트랜잭션이 짧은 경우

leakDetectionThreshold 길게 (예: 60000ms):
  ✅ 오탐 없음
  ❌ 실제 Leak 감지 늦음
  → 운영 환경, 장기 배치 트랜잭션 있는 경우

leakDetectionThreshold=0 (비활성):
  ✅ 오버헤드 없음
  ❌ Leak 발생해도 Pool 고갈 전까지 감지 불가
  → 코드 품질에 자신 있는 경우만

권장 전략:
  개발: 2000ms (빠른 탐지, 오탐 허용)
  스테이징: 5000ms
  운영: 10000~30000ms (정상 트랜잭션 최대 수행 시간 고려)
```

---

## 📌 핵심 정리

```
Leak 감지 동작
  Connection 대여 시 leakTask 스케줄링 (스택 트레이스 캡처)
  leakDetectionThreshold 경과 후 반환 안 됨 → WARN 로그 + 스택 트레이스
  정상 반환 시 leakTask.cancel() → 경고 없음

Leak 발생 전형 패턴
  finally 없는 Connection.close()
  EntityManager 수동 생성 + close() 누락
  StatelessSession close() 누락
  비동기 스레드의 DataSource 직접 접근

안전한 패턴
  try-with-resources: Connection, EntityManager, StatelessSession
  JdbcTemplate/NamedParameterJdbcTemplate: 내부적으로 안전
  @Transactional + Spring Data JPA: 자동 관리

디버깅 흐름
  leakDetectionThreshold 설정 → WARN 로그 확인
  → 스택 트레이스에서 획득 위치 파악
  → try-with-resources 적용 → 검증
```

---

## 🤔 생각해볼 문제

**Q1.** `leakDetectionThreshold=2000`으로 설정한 환경에서 정상적인 배치 처리 트랜잭션이 5초 걸린다면 어떤 일이 발생하는가? 이를 해결하는 방법은?

**Q2.** HikariCP Leak 경고 스택 트레이스에서 `com.zaxxer.hikari.pool.ProxyConnection`이 아닌 실제 코드 위치를 찾는 방법은? 스택 트레이스 어디를 봐야 하는가?

**Q3.** `@Transactional` 메서드 내에서 `new Thread(() -> jdbcTemplate.update("...")).start()`를 실행했을 때 발생하는 문제와 Connection Leak 가능성은?

> 💡 **해설**
>
> **Q1.** 5초 트랜잭션에서 `leakDetectionThreshold=2000`이면 2초 후 "Connection leak detected" 경고가 출력된다. 실제 Leak이 아님에도 오탐이 발생하는 것이다. 경고가 출력된 후 트랜잭션이 정상 완료되면 Connection은 Pool에 반환된다 — Leak이 아닌 정상 동작이다. 해결 방법은 세 가지다: (1) `leakDetectionThreshold`를 배치 최대 수행 시간보다 길게 설정한다 (예: 10000ms). (2) 배치 전용 DataSource를 별도 설정하고 해당 Pool에는 `leakDetectionThreshold=0`으로 비활성화한다. (3) 배치 트랜잭션을 더 작은 Chunk로 분할해 각 트랜잭션이 threshold 이내에 완료되도록 한다.
>
> **Q2.** Leak 경고의 스택 트레이스는 Connection을 **대여한 시점**의 콜 스택이다. 맨 위는 HikariCP 내부 코드(`ProxyFactory`, `HikariPool`)가 나오고, 아래로 내려가면 실제 비즈니스 코드가 나온다. `com.example` 또는 자신의 패키지명이 시작되는 줄을 찾으면 된다. 예를 들어 `at com.example.UserService.findUser(UserService.java:42)`처럼 나오면 `UserService.java` 42번째 줄에서 `dataSource.getConnection()`이나 `JdbcTemplate.query()` 등을 호출한 곳이 Connection 획득 위치다. 이 위치에서 `close()`가 호출되지 않는 경로를 찾으면 된다.
>
> **Q3.** 새 스레드는 `TransactionSynchronizationManager`의 ThreadLocal을 공유하지 않는다. 따라서 `jdbcTemplate.update()`가 새 스레드에서 실행될 때 현재 트랜잭션에 바인딩된 Connection이 없고, `DataSourceUtils.getConnection()`이 Pool에서 새 Connection을 획득한다. 이 Connection은 `@Transactional` 트랜잭션과 무관하게 auto-commit 모드로 즉시 커밋된다. 또한 새 스레드에서 `JdbcTemplate.execute()`는 finally로 `DataSourceUtils.releaseConnection()`을 호출하므로 Connection은 정상 반환된다 — Leak은 없다. 그러나 주 트랜잭션이 롤백되어도 새 스레드의 INSERT는 롤백되지 않아 데이터 불일치가 발생한다. Leak보다 트랜잭션 일관성 문제가 더 심각하다.

---

<div align="center">

**[⬅️ 이전: HikariCP 설정과 최적화](./01-hikaricp-configuration.md)** | **[다음: Pool Size 튜닝 공식 ➡️](./03-pool-size-tuning.md)**

</div>
