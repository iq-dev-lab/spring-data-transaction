# Connection Timeout vs Idle Timeout — HikariCP 타임아웃 파라미터 완전 정리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `connectionTimeout`, `idleTimeout`, `maxLifetime`, `validationTimeout`의 정확한 의미 차이는?
- `connectionTimeout` 초과 시 발생하는 예외 타입과 상위 트랜잭션에 미치는 영향은?
- `idleTimeout`이 `minimumIdle` 설정과 상호작용하는 방식은?
- DB 서버의 `wait_timeout`과 HikariCP `maxLifetime`의 올바른 관계는?
- 각 타임아웃을 너무 길거나 너무 짧게 설정했을 때 나타나는 증상은?

---

## 🔍 왜 이게 존재하는가

### 문제: 타임아웃 설정이 없으면 장애가 무한 전파된다

```java
// 타임아웃 미설정 시 장애 시나리오:

// 1. DB 서버 과부하 → 응답 지연
// 2. Pool에서 Connection 대기 → connectionTimeout=무한대
//    → 스레드가 무한 대기
// 3. 톰캣 스레드 200개 모두 DB 대기
// 4. 새 요청 → 스레드 없음 → 서비스 중단

// 타임아웃 설정 시:
// connectionTimeout=5000ms → 5초 후 예외
// → 상위 레이어가 빠른 실패(fail-fast) 처리 가능
// → 일부 요청 실패, 서비스 전체 중단 방지
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 타임아웃을 길게 잡으면 안전하다

```
❌ 잘못된 이해: connectionTimeout=60000ms (1분)

결과:
  DB 과부하 → 스레드 60초씩 대기
  톰캣 200 스레드 × 60초 = 사실상 무한 대기와 동일
  60초 동안 새 요청 처리 불가 → 서비스 실질적 중단

올바른 접근:
  connectionTimeout: 빠른 실패 → 5~10초
  → 5초 대기 후 실패 → Circuit Breaker 동작
  → 일부 요청 503, 나머지 요청 정상 처리 유지
```

### Before: idleTimeout=0으로 설정하면 Connection을 영구 유지한다

```yaml
# ❌ idleTimeout=0의 실제 의미
spring:
  datasource:
    hikari:
      idle-timeout: 0

# idleTimeout=0: 유휴 Connection을 제거하지 않음 (무한 유지)
# BUT: maxLifetime은 여전히 동작!
# maxLifetime 도달 시 Connection 교체
# → idleTimeout=0이어도 maxLifetime 기반 교체는 일어남

# 진짜 영구 유지 = maxLifetime=0 + idleTimeout=0
# → 권장하지 않음 (DB wait_timeout 초과 시 강제 종료)
```

---

## ✨ 올바른 이해와 패턴

### After: 타임아웃 파라미터 완전 정리

```
HikariCP 타임아웃 파라미터 7개:

connectionTimeout   Pool에서 Connection 대여 최대 대기 시간
                    → 초과 시: SQLTimeoutException
                    기본: 30,000ms (30초) → 권장: 5,000ms

validationTimeout   Connection 유효성 검증(isValid) 최대 시간
                    → connectionTimeout보다 짧아야 함
                    기본: 5,000ms → 대부분 그대로 사용

idleTimeout         Pool에서 유휴 Connection 최대 보유 시간
                    → minimumIdle < maximumPoolSize일 때만 의미 있음
                    → minimumIdle에 도달하면 더 이상 제거 안 함
                    기본: 600,000ms (10분)

maxLifetime         Connection 최대 수명 (생성 후 경과 시간)
                    → 초과 시 다음 반환 시점에 실제 close()
                    → DB wait_timeout보다 짧아야 함
                    기본: 1,800,000ms (30분)

keepaliveTime       유휴 Connection에 ping 실행 주기
                    기본: 0 (비활성) → 권장: 60,000ms

initializationFailTimeout Pool 초기화 시 연결 실패 허용 시간
                    기본: 1ms → 컨테이너 환경: 0

leakDetectionThreshold  Leak 감지 임계값
                    기본: 0 (비활성) → 개발: 2,000ms
```

---

## 🔬 내부 동작 원리 — 타임아웃 상호작용 추적

### 1. connectionTimeout — Pool 대기와 예외

```java
// connectionTimeout 동작 경로
// HikariPool.getConnection()
public Connection getConnection(final long hardTimeout) throws SQLException {
    suspendResumeLock.acquire();
    final long startTime = currentTime();

    try {
        long timeout = hardTimeout;
        do {
            PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);

            if (poolEntry == null) {
                break; // timeout 초과
            }

            // Connection 유효성 확인
            if (!isConnectionAlive(poolEntry.connection)) {
                closeConnection(poolEntry, DEAD_CONNECTION_MESSAGE);
                timeout = hardTimeout - elapsedMillis(startTime);
                continue; // 다른 Connection 시도
            }

            // 유효한 Connection 반환
            return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry));

        } while (timeout > 0L);

        // timeout 초과 → 예외
        throw new SQLTimeoutException(
            "HikariPool-1 - Connection is not available, request timed out after " +
            elapsedMillis(startTime) + "ms"
        );

    } finally {
        suspendResumeLock.release();
    }
}

// 예외 계층:
// SQLTimeoutException (java.sql)
//   → Spring: CannotAcquireLockException? 아님
//   → QueryTimeoutException 아님
//   → DataAccessResourceFailureException으로 변환될 수 있음
// @Transactional: RuntimeException 하위 → 자동 롤백
```

### 2. idleTimeout과 minimumIdle 상호작용

```java
// HikariPool의 하우스키퍼 — 유휴 Connection 정리
class HouseKeeper implements Runnable {
    @Override
    public void run() {
        // 현재 유휴 Connection 수
        final int idleCount = connectionBag.getCount(STATE_NOT_IN_USE);
        // 제거 가능한 수 = 유휴 수 - minimumIdle
        final int idleToRemove = idleCount - config.getMinimumIdle();

        if (idleToRemove <= 0) return; // minimumIdle 유지 → 제거 안 함

        // idleTimeout 초과한 Connection만 제거
        int removed = 0;
        for (PoolEntry entry : connectionBag.values(STATE_NOT_IN_USE)) {
            if (removed >= idleToRemove) break;

            if (elapsedMillis(entry.lastAccessed) > idleTimeout
                    && connectionBag.reserve(entry)) {
                closeConnection(entry, "(connection has passed idleTimeout)");
                removed++;
            }
        }
    }
}

// minimumIdle = maximumPoolSize (고정 Pool):
//   idleToRemove = maximumPoolSize - maximumPoolSize = 0
//   → idleTimeout 무관, Connection 제거 안 함
//   → 고정 Pool에서 idleTimeout은 의미 없음

// minimumIdle < maximumPoolSize (동적 Pool):
//   유휴 Connection이 minimumIdle 초과분만 idleTimeout 후 제거
//   → 저트래픽 시 Pool 크기 축소
```

### 3. maxLifetime — DB wait_timeout 관계

```
DB wait_timeout 설정 (MySQL 기본: 8시간):
  DB가 일정 시간 유휴 Connection을 강제 종료하는 시간

maxLifetime 설정:
  HikariCP가 Connection을 자발적으로 교체하는 수명

올바른 관계:
  maxLifetime < DB wait_timeout
  예: DB wait_timeout = 8시간 → maxLifetime = 30분 (충분한 여유)
  예: DB wait_timeout = 1시간 → maxLifetime = 50분 (90% 이내)

잘못된 설정:
  maxLifetime > DB wait_timeout
  → DB가 먼저 Connection 종료
  → Pool은 모름 → 끊긴 Connection 대여 → 첫 요청 실패

maxLifetime 랜덤 지터:
  설정값 ± 2.5% 범위에서 랜덤하게 만료 시간 분산
  Pool 20개 → 동시에 만료 방지 (Thunder Herd 방지)
  maxLifetime=1800000ms → 실제 만료: 1755000~1845000ms 범위
```

### 4. 타임아웃별 장애 시나리오와 증상

```
[connectionTimeout 너무 김 (예: 60초)]
증상:
  DB 과부하 → 스레드 60초 블로킹
  톰캣 스레드 풀 소진 → 서비스 응답 없음
  연쇄 장애: 상위 서비스도 대기 → 전체 시스템 마비

[connectionTimeout 너무 짧음 (예: 100ms)]
증상:
  일시적 부하 스파이크에서 정상 요청도 실패
  SQLTimeoutException 빈번 → 불필요한 에러율 증가

[maxLifetime > DB wait_timeout]
증상:
  간헐적 Connection 오류 (특히 야간 저트래픽 후 첫 요청)
  "Communication link failure", "Connection reset"

[idleTimeout < keepaliveTime]
증상:
  유휴 Connection이 keepalive ping 전에 제거
  트래픽 증가 시 새 Connection 생성 지연
  → idleTimeout > keepaliveTime 유지 필요

[validationTimeout > connectionTimeout]
증상:
  유효성 검사 중 connectionTimeout 먼저 도달
  → SQLTimeoutException (유효하지 않은 상태로 간주될 수 있음)
  → validationTimeout < connectionTimeout 필수
```

### 5. 환경별 권장 설정

```yaml
# [일반 웹 서비스 — OLTP]
spring:
  datasource:
    hikari:
      connection-timeout: 5000        # 5초 빠른 실패
      validation-timeout: 3000        # 3초 (connectionTimeout보다 짧게)
      idle-timeout: 600000            # 10분 (고정 Pool에서는 의미 없음)
      max-lifetime: 1800000           # 30분 (DB wait_timeout보다 짧게)
      keepalive-time: 60000           # 60초 ping
      maximum-pool-size: 20
      minimum-idle: 20                # 고정 Pool

# [배치 처리 서버]
spring:
  datasource:
    hikari:
      connection-timeout: 300000      # 5분 (배치 쿼리 대기)
      idle-timeout: 60000             # 1분 (배치 간 유휴 빠른 제거)
      max-lifetime: 7200000           # 2시간 (배치 장기 실행)
      maximum-pool-size: 5
      minimum-idle: 1                 # 동적 Pool (배치 없을 때 최소)

# [Kubernetes / 컨테이너 환경]
spring:
  datasource:
    hikari:
      connection-timeout: 5000
      max-lifetime: 1800000
      initialization-fail-timeout: 0  # DB 준비 전 시작 허용
      keepalive-time: 30000           # 30초 (클라우드 방화벽 대응)
```

---

## 💻 실험으로 확인하기

### 실험 1: connectionTimeout 동작 확인

```java
@Test
void connectionTimeoutBehavior() {
    // Pool 전체 소진
    List<Connection> held = new ArrayList<>();
    for (int i = 0; i < 20; i++) {
        try { held.add(dataSource.getConnection()); } catch (Exception e) { break; }
    }

    long start = System.currentTimeMillis();
    try {
        dataSource.getConnection();
        fail("예외 발생해야 함");
    } catch (SQLException e) {
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("예외 타입: " + e.getClass().getSimpleName());
        // SQLTimeoutException
        System.out.println("대기 시간: " + elapsed + "ms");
        // connectionTimeout 설정값과 일치
        assertTrue(e.getMessage().contains("Connection is not available"));
    } finally {
        held.forEach(c -> { try { c.close(); } catch (Exception ignored) {} });
    }
}
```

### 실험 2: maxLifetime 교체 확인

```java
@Test
void maxLifetimeReplacement() throws Exception {
    // maxLifetime=5000ms (테스트용 짧은 설정)

    Connection conn1 = dataSource.getConnection();
    String connId1 = conn1.toString();
    conn1.close(); // Pool 반환

    Thread.sleep(6000); // maxLifetime 초과

    Connection conn2 = dataSource.getConnection();
    String connId2 = conn2.toString();
    conn2.close();

    // conn1 != conn2 (새 Connection 생성됨)
    assertNotEquals(connId1, connId2);
}
```

---

## ⚡ 성능 임팩트

```
connectionTimeout의 응답 시간 영향:
  설정값 = 최악의 경우 추가 대기 시간
  connectionTimeout=5000ms:
    DB 정상: 추가 대기 없음 (~수십 μs)
    Pool 고갈: 최대 5초 대기 후 실패
  → 빠른 실패 설계 가능

idleTimeout의 리소스 절감:
  동적 Pool (minimumIdle < maximumPoolSize):
    저트래픽 시 유휴 Connection 제거 → DB 서버 리소스 절감
    Connection 제거/생성 오버헤드 주기적 발생
  고정 Pool (minimumIdle = maximumPoolSize):
    idleTimeout 무관 → Connection 항상 준비 → 대기 없음

maxLifetime의 성능 영향:
  Connection 교체 시: close() + getConnection() ~수 ms
  교체는 Pool에 반환 시 조용히 진행 (사용 중 교체 없음)
  → 실제 요청 처리에 영향 없음 (백그라운드 교체)
```

---

## 🤔 트레이드오프

```
connectionTimeout 짧게 vs 길게:
  짧게(5초): 빠른 실패, Circuit Breaker 연계, 스레드 보호
  길게(60초): 일시적 DB 과부하 흡수, 스레드 고갈 위험
  → 짧게 설정 + Circuit Breaker 조합 권장

idleTimeout 있음 vs 없음:
  있음: 동적 Pool 축소, DB 리소스 절감, 재확장 지연
  없음: 항상 Pool 최대 유지, 안정적, DB 리소스 사용
  → 고정 Pool + idleTimeout 무의미 vs 동적 Pool + idleTimeout 유의미

maxLifetime 짧게 vs 길게:
  짧게(5분): 잦은 Connection 교체, DB wait_timeout 문제 없음
  길게(1시간): 교체 빈도 낮음, DB wait_timeout 설정에 종속
  → 30분이 대부분 환경에서 적절 (DB wait_timeout 1시간 이상 가정)

타임아웃 설계 원칙:
  validationTimeout < connectionTimeout
  maxLifetime < DB wait_timeout
  keepaliveTime < 방화벽 idle timeout
  idleTimeout > keepaliveTime (동적 Pool 사용 시)
```

---

## 📌 핵심 정리

```
타임아웃 7개 역할
  connectionTimeout:   Pool 대여 대기 최대 시간 (빠른 실패)
  validationTimeout:   isValid() 최대 시간 (< connectionTimeout)
  idleTimeout:         유휴 Connection 제거 기준 (minimumIdle 유지)
  maxLifetime:         Connection 최대 수명 (< DB wait_timeout)
  keepaliveTime:       유휴 Connection ping 주기 (방화벽 대응)
  initializationFail:  Pool 초기화 연결 재시도 시간
  leakDetection:       Leak 감지 임계값

필수 설계 규칙
  maxLifetime < DB wait_timeout (가장 중요)
  validationTimeout < connectionTimeout
  고정 Pool: minimumIdle = maximumPoolSize, idleTimeout 무의미
  빠른 실패: connectionTimeout = 5~10초

장애 증상-원인 매핑
  야간 후 첫 요청 실패: maxLifetime > DB wait_timeout
  피크 시 timeout 폭발: connectionTimeout 너무 짧거나 Pool 부족
  간헐적 503: connectionTimeout 너무 김 → 스레드 고갈
```

---

## 🤔 생각해볼 문제

**Q1.** `idleTimeout=600000`(10분)과 `keepaliveTime=60000`(1분)을 동시에 설정했을 때, 유휴 Connection은 keepalive ping을 10번 받은 후 제거되는가? 아니면 다른 방식으로 동작하는가?

**Q2.** `connectionTimeout=5000`인 환경에서 `@Transactional` 메서드 내부에서 `SQLTimeoutException`이 발생하면 해당 트랜잭션은 어떻게 처리되는가? 롤백이 발생하는가?

**Q3.** MSA 환경에서 서비스 A가 서비스 B의 DB를 공유한다(동일 DB, 다른 Pool). 서비스 B가 배포로 재시작되면서 서비스 A의 Pool Connection이 영향을 받는 상황은 어떤 경우인가?

> 💡 **해설**
>
> **Q1.** 두 설정은 독립적으로 동작하며 keepalive ping 횟수가 idleTimeout과 직접 연관되지 않는다. `keepaliveTime=60000`은 60초마다 유휴 Connection에 `isValid()`를 실행해 생존을 확인하고, 성공 시 `lastAccessed`를 갱신한다. `idleTimeout=600000`은 마지막으로 Connection이 **사용된 시간**(`lastAccessed`)으로부터 10분이 지나면 제거하는 기준이다. keepalive ping이 `lastAccessed`를 갱신하면 idleTimeout 카운터가 리셋된다. 따라서 keepalive가 동작하는 한 실제로 idleTimeout이 발동하지 않을 수 있다. HikariCP 구현을 확인하면 keepalive 성공 시 `lastAccessed`를 갱신하지 않는 버전도 있으므로 버전별 차이를 주의해야 한다. 결론적으로 두 설정이 함께 있으면 keepalive < idleTimeout을 유지해야 Connection이 의도치 않게 제거되지 않는다.
>
> **Q2.** `SQLTimeoutException`은 `SQLException`의 하위이고, Spring은 이를 `DataAccessException`(RuntimeException 하위)으로 변환한다. `@Transactional`은 `RuntimeException` 발생 시 롤백하므로 자동 롤백이 발생한다. 단, `connectionTimeout` 초과 시 Connection을 아직 획득하지 못한 상태이므로 DB에 아무 작업도 수행되지 않은 시점이다. 트랜잭션이 시작되지 않았거나 이미 시작됐다면 롤백 처리가 된다. 상위 레이어(Controller)는 `SQLTimeoutException` → `DataAccessResourceFailureException`을 받아 적절한 에러 응답을 반환해야 한다.
>
> **Q3.** 서비스 A와 B가 동일 DB 서버를 사용하더라도 각자의 Connection Pool은 독립적이다. 서비스 B의 재시작은 서비스 A의 Pool Connection에 직접 영향을 주지 않는다. 그러나 간접적으로 영향을 주는 경우가 있다. 첫째, DB 서버가 재시작되는 경우(서비스 B 배포 중 DB 재시작 포함) — 이때는 서비스 A의 Connection도 끊긴다. 둘째, 서비스 B가 배포 중 대량 쿼리나 마이그레이션으로 DB에 과부하를 유발하면 서비스 A의 쿼리 응답 시간도 늘어난다. 셋째, DB `max_connections` 한도 근처에서 서비스 B가 재시작 중 새 Connection을 많이 열면 서비스 A의 Pool 확장이 실패할 수 있다. 따라서 MSA에서도 DB 리소스 공유 영향을 항상 고려해야 한다.

---

<div align="center">

**[⬅️ 이전: Statement Caching 효과](./05-statement-caching.md)** | **[홈으로 🏠](../README.md)**

</div>
