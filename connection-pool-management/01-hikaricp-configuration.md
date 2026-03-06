# HikariCP 설정과 최적화 — Pool 내부 구조와 핵심 파라미터 완전 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HikariCP가 다른 Connection Pool보다 빠른 근본적인 이유는?
- `ConcurrentBag`이 Connection 대여/반환을 Lock 없이 처리하는 방법은?
- `maximumPoolSize`, `minimumIdle`, `connectionTimeout`의 상호작용은?
- `idleTimeout`과 `maxLifetime`이 필요한 이유와 적절한 값은?
- Spring Boot의 HikariCP 자동 설정과 수동 튜닝 포인트는?

---

## 🔍 왜 이게 존재하는가

### 문제: Connection 생성 비용이 크다

```java
// DB Connection 생성 비용:
// TCP 3-way handshake: ~1ms
// DB 인증(auth): ~5ms
// 세션 초기화: ~2ms
// 합계: ~8ms ~ 수십 ms

// 초당 1000 요청 × 8ms = 8초 지연 — Connection 매번 생성 시
// → Connection Pool: 미리 만들어 재사용

// Pool 없는 경우:
// 요청 → Connection 생성(8ms) → 쿼리(5ms) → Connection 닫기
// → 총 ~13ms, DB 서버에 반복 부하

// HikariCP Pool:
// 요청 → Pool에서 Connection 대여(수십 μs) → 쿼리(5ms) → Pool에 반환
// → 총 ~5ms, Connection 재사용
```

```
HikariCP가 빠른 이유:
  1. ConcurrentBag — Lock-free Connection 관리
  2. 바이트코드 최적화 (Javassist → 프록시 오버헤드 최소화)
  3. 객체 재사용 (Statement, Connection 프록시 풀링)
  4. 설계 단순성 (불필요한 추상화 제거)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: maximumPoolSize를 크게 설정하면 성능이 좋아진다

```yaml
# ❌ 잘못된 설정 — Pool을 크게
spring:
  datasource:
    hikari:
      maximum-pool-size: 200  # CPU 8코어 서버에서
```

```
Pool이 크면 오히려 나쁜 이유:
  DB 서버 Connection 수 = 서버 리소스 소비
  Connection 200개 → DB가 200개 스레드/프로세스 유지
  DB 서버 CPU: 쿼리 실행 경쟁 → Context switching 증가
  → 적정 Pool Size가 작은 Pool보다 처리량 높음

HikariCP 권장 공식 (About Pool Sizing):
  connections = (core_count × 2) + effective_spindle_count
  CPU 8코어, SSD(spindle=0):
  → (8 × 2) + 0 = 16~17 (16~20 범위)

현실적 설정:
  OLTP: CPU 코어 × 2 (10~30)
  배치/분석: 상황에 따라 조정
  Read Replica 분리 시: 각 Pool별 설정
```

### Before: minimumIdle=0으로 설정하면 리소스를 절약한다

```yaml
# ❌ 잘못된 이해: 사용 안 할 때 Connection을 모두 닫으면 좋다
spring:
  datasource:
    hikari:
      minimum-idle: 0
```

```
문제: 트래픽 급증 시 Connection 생성 폭발
  minimumIdle=0 → 유휴 시 Connection 0개 유지
  갑자기 100 요청 → Connection 100개 생성 시도
  → 생성 시간(~8ms × 100) + Connection timeout 위험

권장: minimumIdle = maximumPoolSize (고정 Pool)
  항상 Pool을 가득 채워 준비 상태 유지
  HikariCP 공식 문서에서도 고정 Pool 권장
  리소스 약간 더 사용하지만 응답 일관성 확보
```

---

## ✨ 올바른 이해와 패턴

### After: HikariCP 핵심 파라미터 상호관계

```
┌─────────────────────────────────────────────────────┐
│                   Connection Pool                   │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              ConcurrentBag                   │   │
│  │  [Conn1:IDLE] [Conn2:INUSE] [Conn3:IDLE] ... │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  maximumPoolSize: Pool 최대 Connection 수             │
│  minimumIdle:     최소 유휴 Connection 수              │
│  connectionTimeout: Pool에서 대기 최대 시간             │
│  idleTimeout:     유휴 Connection 최대 보유 시간        │
│  maxLifetime:     Connection 최대 수명                │
│  keepaliveTime:   유휴 Connection 생존 확인 주기        │
└─────────────────────────────────────────────────────┘

대여 흐름:
  요청 → ConcurrentBag에서 IDLE Connection 탐색
  → 없으면 connectionTimeout 동안 대기
  → 대기 초과 → SQLTimeoutException

반환 흐름:
  Connection.close() (HikariCP 프록시)
  → Pool에 반환 (실제 닫기 아님)
  → maxLifetime 초과 시 실제 닫기 + 새 Connection 생성
```

---

## 🔬 내부 동작 원리 — ConcurrentBag 소스 추적

### 1. ConcurrentBag — Lock-free Connection 관리

```java
// HikariCP ConcurrentBag — Connection 대여/반환 핵심 자료구조
public class ConcurrentBag<T extends IConcurrentBagEntry> {

    // 전체 Connection 목록 (CopyOnWriteArrayList — 읽기 Lock-free)
    private final CopyOnWriteArrayList<T> sharedList;

    // 스레드 로컬 캐시 — 자신이 마지막에 사용한 Connection 보관
    private final ThreadLocal<List<Object>> threadList;

    // 대기 스레드 수 (AtomicInteger)
    private final AtomicInteger waiters;

    // [대여] borrow()
    public T borrow(long timeout, TimeUnit timeUnit) throws InterruptedException {

        // [1] 스레드 로컬 캐시 먼저 확인 (가장 빠름)
        final List<Object> list = threadList.get();
        for (int i = list.size() - 1; i >= 0; i--) {
            final T bagEntry = (T) list.remove(i);
            if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                return bagEntry; // CAS 성공 → Lock-free 대여
            }
        }

        // [2] sharedList 전체 순회
        final int waiting = waiters.incrementAndGet();
        try {
            for (T bagEntry : sharedList) {
                if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                    // CAS로 상태 변경 성공 → Lock 없이 대여
                    if (waiting > 1) {
                        listener.addBagItem(waiting - 1); // 필요시 Connection 추가 생성 신호
                    }
                    return bagEntry;
                }
            }

            // [3] 모두 사용 중 → 대기
            listener.addBagItem(waiting); // Pool 확장 요청
            timeout = timeUnit.toNanos(timeout);
            do {
                final long start = currentTime();
                final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
                // → 반환된 Connection을 handoffQueue에서 받음
                if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
                    return bagEntry; // null이면 timeout
                }
                timeout -= elapsedNanos(start);
            } while (timeout > 10_000);

            return null; // timeout → connectionTimeout 예외
        } finally {
            waiters.decrementAndGet();
        }
    }

    // [반환] requite()
    public void requite(final T bagEntry) {
        bagEntry.setState(STATE_NOT_IN_USE);

        // 대기 스레드에게 직접 전달 (handoffQueue)
        for (int i = 0; waiters.get() > 0; i++) {
            if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
                return;
            }
            // backoff
        }

        // 대기 스레드 없으면 스레드 로컬 캐시에 저장
        final List<Object> threadLocalList = threadList.get();
        if (threadLocalList.size() < 50) {
            threadLocalList.add(weakReference ? new WeakReference<>(bagEntry) : bagEntry);
        }
    }
}
```

### 2. Connection 프록시 — close()의 실제 동작

```java
// HikariCP가 반환하는 Connection은 프록시 (HikariProxyConnection)
// → close() 호출 시 실제 닫지 않고 Pool에 반환

public final class HikariProxyConnection extends ProxyConnection {
    // Javassist로 바이트코드 생성 (리플렉션 없음 → 오버헤드 최소)

    @Override
    public final void close() throws SQLException {
        // 실제 Connection 닫기 대신 Pool에 반환
        poolEntry.recycle(lastAccess);
        // → ConcurrentBag.requite() 호출
        // → Pool에서 대여 가능 상태로 전환
    }
}

// 사용자 코드:
Connection conn = dataSource.getConnection(); // HikariProxyConnection 반환
conn.close(); // Pool 반환 (실제 닫기 아님!)
```

### 3. maxLifetime — Connection 수명 관리

```java
// maxLifetime: Connection 최대 수명 (기본 30분)
// 이유:
// - DB 서버의 wait_timeout: 일정 시간 유휴 Connection 강제 종료
// - 방화벽: 오래된 Connection 끊기
// - maxLifetime < DB wait_timeout 으로 설정해야 함

// HikariCP 내부:
// Connection 생성 시 maxLifetime ± 2.5% 랜덤 지터 추가
// → 모든 Connection이 동시에 만료되는 것 방지 (Thunder Herd 방지)

// 만료 처리:
// maxLifetime 도달 → 해당 Connection이 Pool에 반환될 때 실제 close()
// → 새 Connection 생성으로 대체
// → 진행 중인 쿼리는 영향 없음

// 설정 권장:
spring:
  datasource:
    hikari:
      max-lifetime: 1800000         # 30분 (ms)
      # DB wait_timeout이 8시간(MySQL 기본)이면 30분으로 충분
      # DB wait_timeout이 1시간이면 50분 이하로 설정
```

### 4. keepaliveTime — 유휴 Connection 생존 확인

```java
// keepaliveTime: 유휴 Connection에 주기적으로 SELECT 1 실행
// 목적: 방화벽/로드밸런서가 오래된 유휴 Connection을 끊는 문제 방지

spring:
  datasource:
    hikari:
      keepalive-time: 60000  # 60초마다 ping (기본: 0, 비활성)

// 내부 동작:
// 60초 경과한 유휴 Connection → connectionTestQuery 실행
// 기본: "SELECT 1" (isValid() 지원 드라이버는 드라이버 자체 ping 사용)
// 실패 시: Connection 제거 + 새 Connection 생성

// connectionTestQuery vs isValid():
// isValid(): JDBC 4 표준, 드라이버가 직접 ping → 더 효율적
// connectionTestQuery: 구형 드라이버용 ("SELECT 1")
// HikariCP: isValid() 우선 사용 (drvier 지원 시)
```

### 5. Spring Boot 자동 설정과 수동 튜닝

```yaml
# application.yml — 핵심 HikariCP 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?serverTimezone=Asia/Seoul&characterEncoding=UTF-8
    username: app_user
    password: secret
    hikari:
      # Pool 크기
      maximum-pool-size: 20          # CPU 8코어 기준: 코어×2+여유
      minimum-idle: 20               # 고정 Pool (maximum과 동일)

      # 타임아웃
      connection-timeout: 5000       # Pool 대기 최대 5초 (기본 30초)
      idle-timeout: 600000           # 유휴 Connection 10분 후 제거
      max-lifetime: 1800000          # Connection 최대 수명 30분

      # 안정성
      keepalive-time: 60000          # 60초마다 ping
      validation-timeout: 3000       # Connection 유효성 확인 타임아웃

      # 식별
      pool-name: MainPool            # 로그/JMX에서 Pool 이름

      # 성능
      auto-commit: false             # Spring이 트랜잭션 관리 → false 권장
```

```java
// 프로그래밍 방식 설정 (멀티 DataSource 환경)
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource masterDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://master:3306/db");
        config.setUsername("app");
        config.setPassword("secret");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(20);
        config.setConnectionTimeout(5_000);
        config.setMaxLifetime(1_800_000);
        config.setKeepaliveTime(60_000);
        config.setPoolName("MasterPool");
        config.setAutoCommit(false);
        return new HikariDataSource(config);
    }

    @Bean
    public DataSource replicaDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://replica:3306/db");
        config.setMaximumPoolSize(30); // 읽기 전용 → 더 큰 Pool
        config.setMinimumIdle(30);
        config.setReadOnly(true);      // 읽기 전용 Connection
        config.setPoolName("ReplicaPool");
        return new HikariDataSource(config);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Pool 상태 모니터링

```java
@Autowired
DataSource dataSource;

@Test
void monitorPoolStats() {
    HikariPoolMXBean poolMXBean = ((HikariDataSource) dataSource).getHikariPoolMXBean();

    System.out.println("전체 Connection: " + poolMXBean.getTotalConnections());
    System.out.println("사용 중: " + poolMXBean.getActiveConnections());
    System.out.println("유휴: " + poolMXBean.getIdleConnections());
    System.out.println("대기 스레드: " + poolMXBean.getThreadsAwaitingConnection());
}

// Actuator로 노출 (운영 환경)
// management.endpoints.web.exposure.include=health,metrics
// 접근: GET /actuator/metrics/hikaricp.connections.active
```

### 실험 2: connectionTimeout 테스트

```java
@Test
void connectionTimeoutTest() throws Exception {
    // Pool 전체 소진
    List<Connection> heldConnections = new ArrayList<>();
    for (int i = 0; i < 20; i++) { // maximumPoolSize=20 모두 획득
        heldConnections.add(dataSource.getConnection());
    }

    // Pool이 비었을 때 추가 요청 → connectionTimeout 후 예외
    long start = System.currentTimeMillis();
    assertThrows(SQLTimeoutException.class, () -> {
        dataSource.getConnection(); // 5초 대기 후 예외 (connectionTimeout=5000)
    });
    long elapsed = System.currentTimeMillis() - start;
    assertTrue(elapsed >= 4900 && elapsed <= 6000);

    heldConnections.forEach(c -> {
        try { c.close(); } catch (Exception ignored) {}
    });
}
```

---

## ⚡ 성능 임팩트

```
Pool 대여 성능 (ConcurrentBag):
  스레드 로컬 캐시 히트: ~200ns
  sharedList CAS 성공: ~500ns
  handoffQueue 대기 (경합 시): ~수 μs~수십 μs
  → DB 쿼리 실행 대비 무시 가능

Pool Size와 처리량 관계:
  CPU 8코어, DB 기준:
  Pool 5:   처리량 낮음 (Connection 부족으로 대기)
  Pool 16:  최적 (CPU core × 2)
  Pool 50:  처리량 감소 (DB Context switching 증가)
  Pool 200: 처리량 더 낮음

auto-commit=false 효과:
  Spring @Transactional이 commit/rollback 제어
  auto-commit=true: 모든 SQL이 개별 auto-commit
  → Spring이 Connection 획득 시 setAutoCommit(false) 호출 비용 절감
  → auto-commit=false로 Pool 생성하면 이 비용 없음
```

---

## 🤔 트레이드오프

```
maximumPoolSize 크게:
  ✅ 동시 요청 많을 때 대기 없음
  ❌ DB 서버 부하 증가, 메모리 사용 증가
  ❌ 적정값 이상은 오히려 처리량 감소

maximumPoolSize 작게:
  ✅ DB 서버 부하 최소
  ❌ 피크 트래픽 시 connectionTimeout 발생

connectionTimeout 짧게 (예: 1초):
  ✅ 장애 빠른 감지, 요청 빠른 실패
  ❌ 일시적 DB 부하 시 정상 요청도 실패

connectionTimeout 길게 (예: 30초):
  ✅ 일시적 부하 흡수
  ❌ 장애 시 스레드 누적 → 서버 다운

maxLifetime 짧게:
  ✅ 오래된 Connection 빠른 교체
  ❌ Connection 생성 빈도 증가

권장: connectionTimeout 5~10초, maxLifetime 30분
```

---

## 📌 핵심 정리

```
ConcurrentBag 대여 순서
  1. 스레드 로컬 캐시 (가장 빠름, ~200ns)
  2. sharedList CAS 순회
  3. handoffQueue 대기 (connectionTimeout까지)
  → Lock-free로 고성능 달성

핵심 파라미터
  maximumPoolSize: CPU × 2 기준 (OLTP: 10~30)
  minimumIdle:     maximumPoolSize와 동일 권장 (고정 Pool)
  connectionTimeout: 5~10초 (기본 30초는 너무 김)
  maxLifetime:     30분 (DB wait_timeout보다 짧게)
  keepaliveTime:   60초 (방화벽 연결 유지)
  autoCommit:      false (Spring 트랜잭션 관리 시)

Connection.close() 동작
  HikariProxyConnection.close()
  → Pool 반환 (실제 닫기 아님)
  → maxLifetime 초과 시에만 실제 close() + 새 Connection 생성
```

---

## 🤔 생각해볼 문제

**Q1.** HikariCP `maximumPoolSize=10`인 환경에서 `@Transactional(propagation=REQUIRES_NEW)` 메서드를 10단계 깊이로 중첩 호출하면 어떤 일이 발생하는가?

**Q2.** `autoCommit=false`로 설정한 HikariCP Pool에서 Connection을 Pool에 반환할 때, 미완료 트랜잭션이 있으면 어떻게 처리되는가?

**Q3.** HikariCP `keepaliveTime=60000`을 설정했을 때 `SELECT 1` ping이 실행되는 정확한 조건은? 모든 Connection에 매 60초마다 실행되는가?

> 💡 **해설**
>
> **Q1.** 데드락이 발생한다. `REQUIRES_NEW`는 매 호출마다 새 Connection을 Pool에서 획득한다. 1단계 호출 시 Connection 1개, 2단계에서 또 1개, ... 10단계에서 10번째 Connection을 요청한다. `maximumPoolSize=10`이면 Pool이 꽉 차서 11번째 요청(10단계 REQUIRES_NEW)이 `connectionTimeout`까지 대기한다. 그런데 1~9단계의 트랜잭션이 종료되어 Connection을 반환하지 않으므로(10단계가 완료되어야 돌아올 수 있음) 영원히 대기한다 → `SQLTimeoutException`(connectionTimeout 후) 또는 무한 대기. 이것이 REQUIRES_NEW와 Pool 크기 관리가 중요한 이유다. 해결: `maximumPoolSize` 증가, 또는 REQUIRES_NEW 중첩 깊이 제한.
>
> **Q2.** HikariCP는 Connection 반환(`requite()`) 시 `Connection.rollback()`을 호출해 미완료 트랜잭션을 정리한다. 이후 `setAutoCommit(false)`가 Pool 설정과 다르면 복원한다. Spring의 `@Transactional`이 정상적으로 commit/rollback을 처리하면 Connection 반환 시 이미 정리된 상태이지만, 예외적으로 트랜잭션이 열린 채 Connection이 반환되는 경우(비정상 코드)에도 HikariCP가 자동 rollback으로 다음 사용자가 clean한 Connection을 받도록 보장한다.
>
> **Q3.** `keepaliveTime`은 모든 Connection에 매 60초마다 실행되지 않는다. 유휴(`IDLE`) 상태이고 마지막 사용 또는 ping 이후 `keepaliveTime`이 경과한 Connection에만 실행된다. 사용 중(`IN_USE`)인 Connection은 ping 대상에서 제외된다. HikariCP 내부에서 `housekeeper` 스레드가 주기적으로 Pool을 스캔해 조건을 만족한 유휴 Connection에만 `isValid()` 또는 `SELECT 1`을 실행한다. 즉 활발히 사용되는 Connection은 keepalive ping이 거의 실행되지 않고, 오래 유휴 상태인 Connection만 대상이 된다.

---

<div align="center">

**[다음: Connection Leak 탐지와 디버깅 ➡️](./02-connection-leak-detection.md)** | **[홈으로 🏠](../README.md)**

</div>
