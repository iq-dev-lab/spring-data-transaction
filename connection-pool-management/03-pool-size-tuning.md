# Pool Size 튜닝 공식 — Connections = Cores × 2 + Spindle Count의 근거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HikariCP 공식 "Connections = Cores × 2 + Spindle Count"의 이론적 근거는?
- Pool Size를 늘리면 처리량이 오히려 감소하는 이유는?
- OLTP vs 배치 vs 혼합 환경에서 Pool Size를 어떻게 달리 설정하는가?
- 멀티 서비스(MSA)에서 DB Connection 총 수를 어떻게 계산하는가?
- Pool Size 결정을 위한 실측 방법과 지표는?

---

## 🔍 왜 이게 존재하는가

### 문제: Pool Size 설정 근거 없이 임의로 잡는다

```
흔한 잘못된 설정 패턴:
  "동시 사용자 1000명이니까 Pool 1000개"
  "안전하게 넉넉히 100개"
  "기본값 10개 그냥 사용"

실제로 일어나는 일:
  Pool 1000개: DB 서버 CPU Context switching 폭발, 처리량 감소
  Pool 100개: DB 서버 Connection 유지 비용 낭비
  Pool 10개: 피크 트래픽 시 connectionTimeout 발생

올바른 접근:
  DB 서버 CPU 코어 수와 스토리지 특성으로 시작
  → 실측으로 보정
  → 모니터링으로 지속 관리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Thread 수 = Pool Size로 설정하면 된다

```
❌ 잘못된 이해:
  톰캣 스레드 200개 → Pool 200개

실제 문제:
  SQL 실행은 DB CPU에서 일어남
  DB CPU 코어 8개 → 동시 실행 가능 쿼리 8개 (단순화)
  Pool 200개 → 200개 스레드가 DB에 쿼리 전송
  → DB CPU: 8코어에서 200개 쿼리 처리 → Context switching 지옥
  → 200개 동시 쿼리가 8코어를 두고 경쟁
  → 실제 처리량: Pool 8~16개일 때보다 낮음

올바른 이해:
  Pool은 "DB가 효율적으로 처리 가능한 수"로 맞춤
  애플리케이션 스레드가 Pool을 기다리는 것이 맞음 (DB 쪽이 병목이면 안 됨)
```

### Before: SSD를 쓰면 Spindle Count는 0이니까 Pool을 더 줄여야 한다

```
❌ 잘못된 이해: SSD → spindle=0 → Connections = Cores × 2 + 0

실제 공식의 의미:
  spindle_count: HDD 스핀들 수 (디스크 I/O 대기 중 다른 요청 처리 가능)
  SSD: I/O 지연이 극히 짧아 spindle 추가 효과 없음 → 0
  → SSD 환경: Connections = Cores × 2
  → 이것이 기준값 (최솟값이 아님)

실제로는 네트워크 지연, 잠금 대기 등 추가 I/O 유사 상황이 있어
  기준값보다 약간 높게 설정하는 것이 일반적
  (Cores × 2 + 여유분 3~5개)
```

---

## 🔬 내부 동작 원리 — 공식의 이론적 근거

### 1. Little's Law와 Pool Size

```
Little's Law (큐잉 이론):
  L = λ × W
  L: 시스템 내 평균 요청 수
  λ: 평균 도착률 (요청/초)
  W: 평균 처리 시간 (초)

DB Connection Pool 적용:
  평균 쿼리 시간: 10ms (= 0.01초)
  목표 처리량: 1000 쿼리/초
  필요 Connection: L = 1000 × 0.01 = 10개

  → 1000 QPS를 쿼리당 10ms로 처리하려면 Connection 10개로 충분!
  Connection을 100개로 늘려도 DB CPU가 병목이면 1000 QPS 초과 불가

현실적 고려:
  쿼리 시간은 일정하지 않음 (10~100ms 범위)
  피크 트래픽에서 λ가 순간 증가
  → 기본 계산값 + 여유분 설정
```

### 2. DB 서버 CPU 관점 — 왜 Cores × 2인가

```
DB CPU 처리 과정:
  1. SQL 파싱 (CPU)
  2. 실행 계획 수립 (CPU)
  3. I/O 대기 (디스크/네트워크) ← CPU 유휴
  4. 결과 조합 (CPU)

I/O 대기 중 CPU 유휴 활용:
  코어 8개:
  - 순수 CPU 작업: 8개 동시 처리
  - I/O 대기 고려: 다른 요청으로 교체 가능
  → × 2 배수: I/O 대기 동안 다른 쿼리 1개 처리
  → Cores × 2 = 16개가 CPU 낭비 없이 효율적 처리 가능한 상한

spindle_count 추가:
  HDD 각 스핀들: 별도 I/O 채널
  스핀들 8개: 8개 병렬 I/O → CPU 추가 유휴 활용 여지
  SSD: 단일 I/O 채널 (가상적), 지연 극히 짧음 → 추가 없음
```

### 3. 환경별 Pool Size 가이드

```java
// [OLTP - 기본 웹 API 서버]
// DB: MySQL, 16코어 CPU, SSD
// 공식: 16 × 2 + 0 = 32 → 실제: 25~35 범위에서 실측

spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.minimum-idle=30

// [읽기 전용 Replica]
// 쓰기보다 읽기 쿼리가 많고 빠름 → 약간 더 높게
spring.datasource.hikari.maximum-pool-size=40

// [배치 처리 서버]
// 쿼리 1개가 길고 무거움 → Pool 적게 (DB CPU 독점 방지)
// 배치 전용 DataSource
HikariConfig batchConfig = new HikariConfig();
batchConfig.setMaximumPoolSize(5); // 배치는 소수 Connection으로 처리
batchConfig.setConnectionTimeout(300_000); // 5분 (배치 쿼리 대기)
batchConfig.setMaxLifetime(7_200_000);    // 2시간 (배치 장기 실행)

// [MSA 환경 — DB Connection 총량 계산]
// DB 서버: 16코어 SSD → 최대 권장 Connection: 35개 (여유 포함)
// 서비스 Pod: 3개 × Pool 10개 = 30개 ← 35개 이내
// 신규 서비스 추가 시: 35 - 30 = 여유 5개 → Pool 5개 한도

// DB max_connections 확인 (MySQL):
// SHOW VARIABLES LIKE 'max_connections'; -- 기본 151
// 운영 서비스 전용: max_connections의 80% 이하로 Pool 합계 유지
```

### 4. 실측으로 최적값 찾기

```
측정 방법 1: 처리량-Pool Size 그래프

Pool Size:  5    10   15   20   25   30   40   50
TPS:       450  820  950  980  985  982  930  870
                      ↑ 최적 구간 (15~25)
                  처리량이 더 이상 늘지 않는 지점 = Pool 포화점

Pool 포화점 이후 증가분은 낭비
→ 포화점 직전 값이 최적 Pool Size

측정 방법 2: HikariCP 메트릭 모니터링
  hikaricp.connections.active   : 사용 중 Connection 수
  hikaricp.connections.pending  : 대기 스레드 수
  hikaricp.connections.idle     : 유휴 Connection 수

정상 상태:
  active: Pool의 50~80%
  pending: 0 (대기 스레드 없음)
  idle: 20~50%

Pool 부족 신호:
  pending > 0 (지속적으로)
  active ≈ maximumPoolSize (Pool 포화)
  → maximumPoolSize 증가 고려

Pool 과잉 신호:
  active < 30% (지속적으로)
  idle > 70%
  → maximumPoolSize 감소 고려
```

### 5. Spring Actuator + Micrometer로 실시간 모니터링

```yaml
# Micrometer + Prometheus + Grafana 설정
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

```java
// Pool 메트릭 직접 조회 (개발/디버깅용)
@Autowired
MeterRegistry meterRegistry;

@Scheduled(fixedDelay = 10000) // 10초마다
public void logPoolMetrics() {
    double active = meterRegistry.gauge("hikaricp.connections.active",
        Tags.of("pool", "MainPool"), 0.0);
    double pending = meterRegistry.gauge("hikaricp.connections.pending",
        Tags.of("pool", "MainPool"), 0.0);
    double idle = meterRegistry.gauge("hikaricp.connections.idle",
        Tags.of("pool", "MainPool"), 0.0);

    log.info("Pool - active: {}, pending: {}, idle: {}", active, pending, idle);

    // 경보 로직
    if (pending > 5) {
        log.warn("Pool 대기 스레드 {}개 — Pool 크기 확인 필요", pending);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Pool Size별 처리량 측정

```java
// JMH 또는 간단한 부하 테스트
@Test
void poolSizeVsThroughput() throws InterruptedException {
    int[] poolSizes = {5, 10, 15, 20, 30, 50};
    int threadCount = 100; // 동시 요청 스레드

    for (int poolSize : poolSizes) {
        // Pool 재설정
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setMaximumPoolSize(poolSize);
        config.setMinimumIdle(poolSize);
        config.setConnectionTimeout(10_000);
        DataSource ds = new HikariDataSource(config);

        // 부하 테스트
        CountDownLatch latch = new CountDownLatch(threadCount);
        long start = System.currentTimeMillis();
        AtomicInteger success = new AtomicInteger();

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    new JdbcTemplate(ds).queryForObject(
                        "SELECT COUNT(*) FROM users", Integer.class);
                    success.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(30, TimeUnit.SECONDS);
        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("Pool %d: %d req in %dms (TPS: %.0f)%n",
            poolSize, success.get(), elapsed, success.get() * 1000.0 / elapsed);

        ((HikariDataSource) ds).close();
    }
}
```

---

## ⚡ 성능 임팩트

```
Pool Size 최적화 효과 (DB: 8코어 SSD, OLTP):

Pool 5:   TPS ~400  (Connection 부족, 대기 발생)
Pool 10:  TPS ~750
Pool 16:  TPS ~980  ← 공식 기준값 (8코어 × 2)
Pool 20:  TPS ~1000 ← 최적 근방
Pool 30:  TPS ~990  (약간 감소 시작)
Pool 50:  TPS ~900  (DB Context switching 영향)
Pool 100: TPS ~780  (명확한 성능 저하)
Pool 200: TPS ~600

결론: 최적 Pool은 최대 TPS의 90% 이상을 내는 가장 작은 값
→ 적게 쓰면서 충분한 성능 확보

MSA DB 총 Connection 수 관리:
  DB max_connections = 500 (설정값)
  운영 서비스: 400개 이내 (80%)
  관리 도구(모니터링, DBA): 50개
  여유: 50개
  → 서비스별 Pool 합계 400개 이내로 설계
```

---

## 🤔 트레이드오프

```
Pool 크게:
  ✅ 트래픽 스파이크 흡수 (대기 줄어듦)
  ❌ DB 서버 부하, Context switching
  ❌ DB max_connections 초과 위험
  ❌ Connection 유지 비용 (메모리, 파일 디스크립터)

Pool 작게:
  ✅ DB 서버 부하 최소
  ✅ 리소스 절약
  ❌ 피크 시 connectionTimeout 발생 위험

고정 Pool (minimum=maximum):
  ✅ 미리 Connection 준비 → 요청 즉시 처리
  ✅ Pool 확장/축소 오버헤드 없음
  ❌ 저트래픽 시에도 Connection 유지

동적 Pool (minimum < maximum):
  ✅ 저트래픽 시 리소스 절약
  ❌ 트래픽 급증 시 Connection 생성 지연
  → Cloud 환경, 트래픽 변동 심한 경우 고려

권장: 고정 Pool (minimum=maximum), 공식 기반 크기, 실측 보정
```

---

## 📌 핵심 정리

```
HikariCP 공식
  Connections = (Core Count × 2) + Effective Spindle Count
  SSD: + 0, HDD 8스핀들: + 8
  OLTP 기준: Core × 2 ~ Core × 2 + 5 범위에서 실측

과도한 Pool이 느린 이유
  DB CPU 코어보다 많은 동시 쿼리 → Context switching
  → 처리량 오히려 감소 (공식 범위의 3배 이상은 역효과)

MSA 설계
  DB max_connections의 80% 이하로 전체 Pool 합계 유지
  서비스별 Pool = 전체 한도 / 서비스 Pod 수

실측 방법
  Pool Size 변화에 따른 TPS 그래프
  TPS 증가 정체 지점 = 포화점 → 직전값이 최적
  Actuator 메트릭: active, pending, idle 모니터링
```

---

## 🤔 생각해볼 문제

**Q1.** DB 서버가 16코어 SSD이고 평균 쿼리 시간이 50ms, 목표 TPS가 500이면 Little's Law로 필요한 Pool Size를 계산하라. HikariCP 공식과 비교하면?

**Q2.** 애플리케이션 서버 Pod가 10개이고 각 Pod의 Pool Size가 30이면 DB 총 Connection은 300개다. DB `max_connections=200`으로 설정되어 있으면 어떤 일이 발생하는가?

**Q3.** `hikaricp.connections.pending` 메트릭이 피크 시간대에 지속적으로 5~10 이상을 기록한다. 이것이 Pool Size 부족을 의미하는가? 다른 원인 가능성은?

> 💡 **해설**
>
> **Q1.** Little's Law: L = λ × W = 500 TPS × 0.05초 = 25개. 25개로 500 TPS를 처리할 수 있다. HikariCP 공식: 16코어 × 2 = 32개. Little's Law(25)와 HikariCP 공식(32)이 비슷한 범위를 가리킨다. 실제로는 쿼리 시간이 50ms로 고정이 아니라 분산되고, 피크 트래픽에서 순간적으로 500 TPS를 초과할 수 있으므로 여유분을 더해 30~35 범위가 현실적인 시작점이다. 두 공식이 상호 검증 역할을 한다 — 크게 다르면 가정을 재점검해야 한다.
>
> **Q2.** Pod 10개가 각각 Pool 30개로 DB에 연결을 시도하면 총 300개 Connection이 필요하다. DB `max_connections=200`이면 200개 초과 연결 시도는 DB에서 거부된다. 일부 Pod는 Connection 획득에 실패하고 `connectionTimeout` 예외가 발생한다. 어떤 Pod가 먼저 연결에 성공하느냐에 따라 일부 Pod는 정상 동작하고 나머지는 완전히 서비스 불가 상태가 된다. 해결: `max_connections` 증가, 또는 Pod별 Pool을 `200 / 10 = 20`개로 줄이거나 Pod 수를 줄여야 한다.
>
> **Q3.** `pending`이 높다고 무조건 Pool Size 부족은 아니다. 다른 원인으로는 첫째, 느린 쿼리가 Connection을 오래 점유하는 경우다 — 쿼리 최적화(인덱스, EXPLAIN)가 우선이다. 둘째, DB 서버 자체가 과부하 상태인 경우 — Pool을 늘려도 DB가 처리 못하면 pending이 줄지 않는다. 셋째, 락 경합(LOCK WAIT, 데드락)으로 Connection이 해제되지 않는 경우다. 진단 순서: DB 서버 CPU/메모리 → 느린 쿼리 로그 → 락 상태 (`SHOW PROCESSLIST`, `pg_stat_activity`) → Connection 사용 시간 분포 → 이후에 Pool Size 조정을 고려한다.

---

<div align="center">

**[⬅️ 이전: Connection Leak 탐지와 디버깅](./02-connection-leak-detection.md)** | **[다음: Connection Health Check 전략 ➡️](./04-connection-health-check.md)**

</div>
