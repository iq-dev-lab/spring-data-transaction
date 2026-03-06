# Connection Health Check 전략 — isValid(), keepalive, 장애 복구 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HikariCP가 Connection 유효성을 검증하는 세 가지 시점은?
- `isValid()`와 `connectionTestQuery`의 차이와 선택 기준은?
- DB 재시작 후 Pool의 stale Connection이 자동으로 교체되는 메커니즘은?
- `initializationFailTimeout`이 애플리케이션 시작에 미치는 영향은?
- Spring Actuator의 DB Health Indicator가 Connection Pool과 연계되는 방식은?

---

## 🔍 왜 이게 존재하는가

### 문제: Pool 안의 Connection이 DB 재시작 후 무효화된다

```java
// 시나리오: DB 재시작 또는 방화벽에 의한 강제 연결 종료
// Pool에는 Connection 20개가 있음

// DB 재시작 이전에 맺어진 Connection들:
// conn1 ~ conn20: 모두 DB와 연결된 상태
// → DB 재시작 → DB 측에서 강제 종료
// → Pool은 모름 (TCP 레벨에서 소켓이 끊겼으나 Pool은 IDLE 상태로 인식)

// DB 재시작 후 첫 요청:
jdbcTemplate.query("SELECT * FROM users", rowMapper);
// → Pool에서 conn1 대여 (IDLE로 표시)
// → conn1으로 SQL 실행 시도
// → IOException: Connection reset by peer
// → HikariCP: Connection 무효 → 제거 + 새 Connection 생성 시도
// → 새 Connection 생성 성공 → 재시도 가능

// 문제: 첫 요청이 예외를 받음 (1회 실패)
// HikariCP는 Connection 획득 시 유효성을 확인해 이 문제를 줄일 수 있음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: connectionTestQuery만 설정하면 항상 안전하다

```yaml
# ❌ 잘못된 이해: connectionTestQuery = 만능 검증
spring:
  datasource:
    hikari:
      connection-test-query: SELECT 1
```

```
connectionTestQuery의 한계:
  Connection 대여 시마다 SELECT 1 실행 → 매 요청 추가 쿼리 1번
  → 1000 TPS × 1회 = 초당 1000번 추가 쿼리 → DB 부하
  → 레이턴시 증가

더 좋은 방법:
  1. JDBC 4 isValid(): 드라이버가 직접 ping → 더 빠름
  2. keepaliveTime: 유휴 Connection만 주기적 확인
  → connectionTestQuery는 JDBC 3 이하 레거시 드라이버용
```

### Before: DB 재시작 후 Pool이 자동으로 복구되지 않는다

```java
// ❌ 잘못된 이해: DB 재시작 → 수동으로 Pool 재시작 필요

// ✅ 실제:
// HikariCP는 Connection 사용 시 예외 발생 → 해당 Connection 제거
// → addBagItem() 신호 → Pool이 자동으로 새 Connection 생성 시도
// → DB가 살아있으면 새 Connection 생성 성공 → 자동 복구

// 단, 복구 시간 동안 일부 요청 실패 가능
// maxLifetime 기반 자동 교체로 예방 가능
```

---

## ✨ 올바른 이해와 패턴

### After: HikariCP Health Check 세 가지 시점

```
시점 1: Connection 생성 시 (initializationFailTimeout)
  Pool 시작 시 최초 Connection 생성
  → 실패 시 시작 실패 여부 결정
  → initializationFailTimeout=0: 실패해도 시작 허용 (나중에 재시도)
  → initializationFailTimeout=-1: 즉시 실패 없이 백그라운드 재시도
  → initializationFailTimeout>0: 해당 시간 동안 재시도, 이후 예외

시점 2: Connection 대여 시 (isValid / connectionTestQuery)
  Pool에서 Connection 꺼낼 때 유효성 확인
  → JDBC 4+: isValid(validationTimeout) — 드라이버 자체 ping
  → JDBC 3 이하: connectionTestQuery 실행 (SELECT 1)
  → HikariCP 기본: connectionTestQuery 미설정 시 isValid() 사용

시점 3: 유휴 상태 주기적 확인 (keepaliveTime)
  유휴 Connection에 keepaliveTime마다 isValid() 실행
  → 끊긴 Connection 사전 감지 + 제거
  → DB 재시작 또는 방화벽 타임아웃 대응
```

---

## 🔬 내부 동작 원리 — Health Check 소스 추적

### 1. isValid() vs connectionTestQuery

```java
// HikariCP — Connection 유효성 검사
// PoolBase.isConnectionAlive()
boolean isConnectionAlive(final Connection connection) {
    try {
        // validationTimeout: 유효성 확인 최대 대기 시간
        setNetworkTimeout(connection, validationTimeout);

        final int validationSeconds = (int) Math.max(1000L, validationTimeout) / 1000;

        if (isUseJdbc4Validation) {
            // JDBC 4+ → isValid() 사용 (드라이버가 직접 ping)
            return connection.isValid(validationSeconds);
            // MySQL: "/* ping */ SELECT 1" 내부 실행
            // PostgreSQL: 드라이버 자체 소켓 체크
        }

        // JDBC 3 이하 → connectionTestQuery 실행
        try (Statement statement = connection.createStatement()) {
            if (isNetworkTimeoutSupported != TRUE) {
                setQueryTimeout(statement, validationSeconds);
            }
            statement.execute(config.getConnectionTestQuery()); // "SELECT 1"
        }
        return true;

    } catch (Exception e) {
        lastConnectionFailure.set(e);
        return false; // 유효하지 않음 → Connection 제거
    } finally {
        setNetworkTimeout(connection, networkTimeout);
    }
}
```

### 2. keepaliveTime — 유휴 Connection 주기적 확인

```java
// HikariPool — keepalive 하우스키퍼
class KeepaliveTask implements Runnable {
    @Override
    public void run() {
        // Pool의 모든 IDLE Connection 순회
        connectionBag.values(STATE_NOT_IN_USE).forEach(poolEntry -> {
            // 마지막 사용 후 keepaliveTime 경과한 Connection만 대상
            if (poolEntry.isMarkedEvicted()) return;

            long lastUsed = poolEntry.lastAccessed;
            if (elapsedMillis(lastUsed) >= keepaliveTime) {
                // 상태를 IN_USE로 변경 (다른 스레드가 대여 못하게)
                if (connectionBag.reserve(poolEntry)) {
                    // 유효성 확인
                    if (!isConnectionAlive(poolEntry.connection)) {
                        // 유효하지 않음 → 제거
                        closeConnection(poolEntry, "(keepalive validation failed)");
                        // → Pool이 자동으로 새 Connection 생성
                    } else {
                        // 유효함 → Pool에 반환
                        connectionBag.unreserve(poolEntry);
                        // lastAccessed 갱신
                    }
                }
            }
        });
    }
}
```

### 3. initializationFailTimeout — 시작 전략

```yaml
# 설정 옵션별 동작

# [옵션 1] 기본값 (1ms): 1ms 내에 Connection 획득 실패 시 시작 실패
spring.datasource.hikari.initialization-fail-timeout=1

# [옵션 2] 0: DB 연결 실패해도 애플리케이션 시작 허용
spring.datasource.hikari.initialization-fail-timeout=0
# → 마이크로서비스에서 DB보다 앱이 먼저 뜨는 경우 적합
# → Kubernetes: Pod 재시작 시 DB가 준비되기 전에 앱 시작 가능

# [옵션 3] -1: 백그라운드에서 무한 재시도
spring.datasource.hikari.initialization-fail-timeout=-1
# → 절대 시작 실패하지 않음
# → DB 준비되면 자동으로 Connection 생성
```

```java
// initializationFailTimeout 내부 동작
// HikariPool 생성자
public HikariPool(final HikariConfig config) {
    // ...
    if (config.getInitializationFailTimeout() > 0) {
        // 지정된 시간 동안 Connection 생성 시도
        final long startTime = currentTime();
        while (elapsedMillis(startTime) < config.getInitializationFailTimeout()) {
            if (checkFailFast()) break; // Connection 생성 성공
            quietlySleep(SECONDS.toMillis(1));
        }
        if (poolState == POOL_SUSPENDED) {
            throw new PoolInitializationException(
                "HikariPool-1 - Failed to initialize pool (timeout waiting for initial connections)");
        }
    }
    // initializationFailTimeout=0 이하: 위 블록 건너뜀 → 즉시 완료
}
```

### 4. DB 재시작 후 자동 복구 흐름

```
DB 재시작 시나리오:
  1. DB 재시작 → 기존 TCP 연결 모두 종료
  2. Pool 내 Connection: IDLE 상태이나 실제로 끊긴 상태

  [요청 도착]
  3. Pool에서 IDLE Connection 대여
  4. keepaliveTime 설정 시: 유효하지 않은 Connection 이미 제거됨
                           → 새 Connection으로 대여
     keepaliveTime 미설정: 끊긴 Connection 대여
  5. SQL 실행 → IOException (Connection reset)
  6. HikariCP: 예외 → Connection 무효 → 제거
  7. Pool: 빈 슬롯 감지 → 새 Connection 생성 시도
  8. DB 재시작 완료 → 새 Connection 생성 성공
  9. 재시도 또는 다음 요청부터 정상 처리

keepaliveTime 설정 효과:
  DB 재시작 완료 후 keepaliveTime 이내에 유효성 검사 실행
  → 무효 Connection 사전 제거
  → 요청 시 바로 새 Connection 사용 (1회 실패 없음)
```

### 5. Spring Actuator DB Health 연계

```yaml
management:
  health:
    db:
      enabled: true
  endpoint:
    health:
      show-details: always
```

```java
// Spring Boot DataSourceHealthIndicator
// → GET /actuator/health 응답에 DB 상태 포함
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    }
  }
}

// DB Health Check 동작:
// DataSourceHealthIndicator.doHealthCheck()
// → dataSource.getConnection() — Pool에서 Connection 획득
// → connection.isValid(validationTimeout)
// → 성공: UP, 실패: DOWN
// → Connection 반환

// 주의: Actuator health check도 Pool Connection 사용
// → health-check 주기가 짧으면 Pool 소비
// management.health.db.enabled=false (운영 환경에서 필요 없으면 비활성)
```

---

## 💻 실험으로 확인하기

### 실험 1: DB 재시작 시뮬레이션

```java
@Test
void connectionRecoveryAfterDbRestart() throws Exception {
    // 정상 동작 확인
    jdbcTemplate.queryForObject("SELECT 1", Integer.class);

    // DB 연결 강제 종료 시뮬레이션 (테스트 환경)
    // Testcontainers: container.stop() → container.start()
    // 또는 HikariPool 직접 조작
    HikariDataSource hds = (HikariDataSource) dataSource;
    hds.getHikariPoolMXBean().softEvictConnections(); // 모든 Connection 무효화

    // 잠시 대기 (새 Connection 생성 시간)
    Thread.sleep(2000);

    // 자동 복구 확인
    Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
    assertEquals(1, result); // 새 Connection으로 성공
}
```

### 실험 2: keepaliveTime 효과 확인

```java
@Test
void keepalivePreventsStaleConnection() throws Exception {
    // keepaliveTime=5000 설정

    // Connection 유휴 상태로 10초 대기
    Thread.sleep(10_000);
    // → keepaliveTime=5000 → 5초마다 isValid() 실행
    // → 로그: "HikariPool-1 - Reset (keepalive) - ..." 확인 가능

    // 정상 조회 (stale Connection 없음)
    Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
    assertEquals(1, result);
}
```

### 실험 3: Actuator Health 응답 확인

```bash
# GET /actuator/health/db
curl http://localhost:8080/actuator/health/db

# 응답:
{
  "status": "UP",
  "details": {
    "database": "MySQL",
    "validationQuery": "isValid()"
  }
}

# DB 중단 시:
{
  "status": "DOWN",
  "details": {
    "error": "...: Connection refused"
  }
}
```

---

## ⚡ 성능 임팩트

```
isValid() 비용:
  JDBC 4+ 드라이버: 소켓 상태 확인 → ~수십 μs
  connectionTestQuery (SELECT 1): DB 왕복 → ~1ms
  → isValid()가 10~50배 빠름

keepaliveTime 오버헤드:
  60초마다 유휴 Connection 수만큼 isValid() 실행
  Pool 20개 × 60초마다 1번 = 초당 0.33번
  → 무시 가능

initializationFailTimeout=0 효과:
  Kubernetes, 컨테이너 환경에서 시작 실패 방지
  DB 준비 전 앱 시작 → 요청 도달 전 Connection 자동 생성
  → 다운타임 없는 롤링 배포 가능

connectionTestQuery 사용 시:
  1000 TPS → 초당 1000번 SELECT 1 추가
  → DB 부하 +10~20% (불필요)
  → JDBC 4 드라이버 사용 시 connectionTestQuery 제거 권장
```

---

## 🤔 트레이드오프

```
connectionTestQuery 설정:
  ✅ 레거시 JDBC 3 드라이버 호환
  ❌ 매 요청 추가 DB 왕복 → 성능 저하
  → JDBC 4+ 환경에서는 미설정 권장

keepaliveTime 짧게 (예: 10초):
  ✅ 끊긴 Connection 빠른 감지
  ❌ 잦은 ping → DB 부하 증가
  → 방화벽 타임아웃이 매우 짧은 환경

keepaliveTime 길게 (예: 300초):
  ✅ ping 부하 최소
  ❌ 끊긴 Connection 늦게 감지 → 일부 요청 실패
  → 방화벽 타임아웃이 충분히 긴 환경

initializationFailTimeout=0 vs 양수:
  0: 컨테이너 환경, DB 준비 타이밍 불확실
  양수: 전통적 배포, DB가 앱보다 먼저 준비 보장되는 경우
```

---

## 📌 핵심 정리

```
Health Check 세 시점
  생성: initializationFailTimeout 내 재시도
  대여: isValid() (JDBC 4+) 또는 connectionTestQuery
  유휴: keepaliveTime 주기로 isValid() 실행

isValid() vs connectionTestQuery
  isValid(): JDBC 4 표준, 드라이버 자체 ping, 빠름 (~수십 μs)
  connectionTestQuery: DB 왕복 필요 (~1ms), 레거시 드라이버용
  → JDBC 4+ 환경에서 connectionTestQuery 제거

DB 재시작 자동 복구
  keepaliveTime 설정: 유휴 중 무효 Connection 사전 제거
  미설정: 첫 요청에서 예외 → HikariCP 자동 교체 (1회 실패)
  maxLifetime: Connection 수명 제한 → DB 재시작 후 자연스러운 교체

initializationFailTimeout
  0: 시작 시 DB 연결 실패해도 앱 시작 (컨테이너 환경 권장)
  -1: 백그라운드 무한 재시도
  양수(ms): 해당 시간 동안 재시도, 초과 시 시작 실패
```

---

## 🤔 생각해볼 문제

**Q1.** `keepaliveTime=60000`이고 DB 방화벽 idle timeout이 30초로 설정된 환경에서 어떤 일이 발생하는가? 해결 방법은?

**Q2.** `connectionTestQuery=SELECT 1`과 `keepaliveTime=60000`을 동시에 설정하면 어떻게 동작하는가? 중복인가?

**Q3.** Spring Actuator `/actuator/health` 엔드포인트를 Kubernetes liveness probe로 사용할 때, DB Health Check가 Pool Connection을 소비한다. 이것이 문제가 될 수 있는 상황과 해결 방법은?

> 💡 **해설**
>
> **Q1.** 방화벽이 30초 유휴 Connection을 끊는데 keepaliveTime이 60초이면, 30초 후 방화벽이 연결을 끊고 그 다음 60초 ping 시점에 Connection이 이미 끊긴 상태가 된다. 따라서 ping이 실행되기 전인 30~60초 구간에서 요청이 들어오면 끊긴 Connection을 대여받아 1회 실패가 발생한다. 해결: `keepaliveTime`을 방화벽 idle timeout보다 짧게 설정한다 (예: 25000ms). 이렇게 하면 방화벽이 끊기 전에 keepalive ping이 실행되어 Connection이 살아있음을 확인하고 타임아웃 카운터가 리셋된다.
>
> **Q2.** 중복이 아니며 역할이 다르다. `connectionTestQuery`는 **Connection 대여 시** 매번 실행되는 검증 쿼리로, JDBC 4 드라이버가 있으면 `isValid()`로 대체된다. `keepaliveTime`은 **유휴 Connection에 주기적으로** 실행되는 별도 메커니즘이다. 두 설정이 공존하면 JDBC 4 드라이버 환경에서: 대여 시 `isValid()` 사용(connectionTestQuery 무시), keepalive 시 `isValid()` 주기 실행. JDBC 3 환경: 대여 시 connectionTestQuery 실행, keepalive 시 `isValid()` 또는 connectionTestQuery 실행. 결론적으로 JDBC 4+에서 `connectionTestQuery`는 설정해도 무시되므로 불필요하다.
>
> **Q3.** Kubernetes liveness probe가 `/actuator/health`를 30초마다 호출하면, 매 호출마다 `DataSourceHealthIndicator`가 Pool에서 Connection을 대여해 `isValid()`를 실행하고 반환한다. Pool Size가 작고(예: 5개) 동시 요청이 많은 상황에서 liveness probe 타이밍에 추가 Connection 소비가 발생하면 일시적인 연결 경합이 생길 수 있다. 해결 방법: (1) liveness probe를 `/actuator/health/liveness`(Spring Boot 2.3+)로 변경 — 이 엔드포인트는 DB 체크 없이 앱 상태만 확인한다. (2) `management.health.db.enabled=false`로 DB Health Check를 비활성화하고 readiness probe에서만 DB 상태 확인. (3) Pool Size를 liveness probe 트래픽까지 고려해 여유있게 설정.

---

<div align="center">

**[⬅️ 이전: Pool Size 튜닝 공식](./03-pool-size-tuning.md)** | **[다음: Statement Caching 효과 ➡️](./05-statement-caching.md)**

</div>
