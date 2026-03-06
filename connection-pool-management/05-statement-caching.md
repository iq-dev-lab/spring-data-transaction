# Statement Caching 효과 — PreparedStatement 재사용과 파싱 비용 절감

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DB 서버의 PreparedStatement 파싱 비용이 실제로 얼마나 되는가?
- HikariCP의 `Statement Cache`와 DB 서버 자체 PreparedStatement Cache의 차이는?
- `cachePrepStmts`, `prepStmtCacheSize`, `prepStmtCacheSqlLimit` 설정의 역할은?
- JDBC 드라이버 레벨 캐싱 vs 애플리케이션 레벨 캐싱(Hibernate QueryPlanCache)의 관계는?
- Statement 캐싱이 오히려 문제를 일으키는 경우는?

---

## 🔍 왜 이게 존재하는가

### 문제: PreparedStatement 생성마다 DB 파싱 비용이 발생한다

```java
// PreparedStatement 생성 시 DB 서버 내부 처리:
// 1. SQL 문자열 파싱 (ANTLR 또는 DB 자체 파서)
// 2. 문법 검증 (테이블명, 컬럼명 존재 확인)
// 3. 실행 계획 수립 (통계 기반 옵티마이저)
// 4. 실행 계획 캐싱 (Prepared Statement 핸들 반환)
// → 합계: ~수십 μs ~ 수 ms

// 동일 SQL을 1000번 실행하면:
// 캐싱 없음: 파싱 1000번 × 수 ms = 수 초 낭비
// 캐싱 있음: 파싱 1번, 이후 캐시 히트 → 수십 μs × 999번

// 특히 문제가 되는 패턴:
for (Product p : products) {
    jdbcTemplate.update(
        "INSERT INTO products (name, price) VALUES (?, ?)", // 매번 동일 SQL
        p.getName(), p.getPrice()
    );
    // PreparedStatement 매번 새로 생성 → 파싱 매번 → 낭비
}
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: PreparedStatement는 한 번 만들면 자동으로 재사용된다

```java
// ❌ 잘못된 이해: JdbcTemplate이 자동으로 캐싱한다

// 실제 JdbcTemplate:
public int update(String sql, Object... args) {
    return execute(new SimplePreparedStatementCreator(sql), ...);
    // → execute() 내부: conn.prepareStatement(sql)
    // → 매번 새 PreparedStatement 생성
    // → execute() 종료 시 ps.close()
    // → JdbcTemplate은 Statement 캐싱 없음!
}

// ✅ Statement 캐싱은 JDBC 드라이버 레벨에서 처리
// MySQL Connector/J: cachePrepStmts=true → 드라이버가 캐싱
// PostgreSQL pgjdbc: prepareThreshold 설정 → 서버 사이드 캐싱
```

### Before: Statement 캐싱은 항상 활성화해야 한다

```java
// ❌ 항상 좋은 것은 아님

// 문제 1: 동적 SQL (매번 다른 SQL 구조)
for (int id : ids) {
    // SQL 구조가 달라지면 캐시 효과 없음 + 캐시 오염
    jdbcTemplate.queryForObject(
        "SELECT * FROM products WHERE id = " + id, // 값을 직접 삽입!
        rowMapper
    );
    // "WHERE id = 1", "WHERE id = 2", ... → 각각 다른 SQL → 캐시 miss
    // → 파라미터 바인딩으로 수정 필요
}

// 문제 2: 동적 IN절 (컬렉션 크기마다 다른 SQL)
List<Long> ids = /* 크기가 매번 다름 */;
namedJdbc.query(
    "SELECT * FROM products WHERE id IN (:ids)",
    Map.of("ids", ids), rowMapper
);
// IN (?,?,?) vs IN (?,?,?,?) → 다른 SQL → 캐시 miss
// → 캐시 슬롯 낭비

// 문제 3: 메모리 제한
// prepStmtCacheSize=250 → Connection당 250개 → Pool 20개 = 5000개 Statement 캐시
// 메모리 사용 증가 (Statement마다 수 KB ~ 수십 KB)
```

---

## ✨ 올바른 이해와 패턴

### After: 캐싱 레이어 전체 구조

```
[SQL 실행 요청]
     │
     ▼
[Hibernate QueryPlanCache]  ← JPQL → SQL 변환 결과 캐싱
     │                         (SQM, SqlAst 캐싱)
     │ 변환된 SQL
     ▼
[JDBC 드라이버 Statement Cache]  ← PreparedStatement 핸들 캐싱
     │  (MySQL: cachePrepStmts)    (SQL 파싱 + 실행계획 서버 캐시 재사용)
     │
     ▼
[DB 서버 Statement Cache]  ← DB 내부 실행계획 캐시
     │  (MySQL: query_cache,       (파싱된 Statement 재사용)
     │   PostgreSQL: query plan cache)
     │
     ▼
[실제 SQL 실행]

각 레이어의 캐싱 범위:
  Hibernate QueryPlanCache: 프로세스 전체 공유 (JPQL → SQL)
  JDBC 드라이버 Cache:      Connection별 (PreparedStatement 핸들)
  DB 서버 Cache:            DB 서버 전체 공유 (실행 계획)
```

---

## 🔬 내부 동작 원리 — Statement 캐싱 소스 추적

### 1. MySQL Connector/J 캐싱 설정

```yaml
# application.yml — MySQL Statement Cache 설정
spring:
  datasource:
    url: >
      jdbc:mysql://localhost:3306/mydb
      ?cachePrepStmts=true
      &prepStmtCacheSize=250
      &prepStmtCacheSqlLimit=2048
      &useServerPrepStmts=true
```

```java
// 파라미터 의미:
// cachePrepStmts=true:
//   JDBC 드라이버가 PreparedStatement를 Connection별로 캐싱
//   동일 SQL 재실행 시 새 prepareStatement() 호출 없이 캐시에서 반환

// prepStmtCacheSize=250:
//   Connection당 최대 250개 Statement 캐싱
//   LRU 방식으로 초과 시 오래된 항목 제거

// prepStmtCacheSqlLimit=2048:
//   2048바이트 이하 SQL만 캐싱 (긴 SQL은 캐싱 안 함)
//   매우 긴 IN절 SQL은 캐싱에서 제외됨

// useServerPrepStmts=true:
//   서버 사이드 PreparedStatement 사용
//   → DB 서버에 SQL 파싱 + 실행계획을 Statement 핸들로 저장
//   → 클라이언트는 핸들만 전송 → 네트워크 전송량 감소 + 파싱 생략

// MySQL Connector/J 내부 캐싱 구조:
// Connection → Map<String, ServerPreparedStatement>
// key: SQL 문자열, value: 서버 PreparedStatement 핸들
```

### 2. PostgreSQL pgjdbc — 서버 사이드 캐싱

```java
// PostgreSQL JDBC 설정
spring.datasource.url=jdbc:postgresql://localhost/db?prepareThreshold=5&preparedStatementCacheQueries=256

// prepareThreshold=5:
//   동일 SQL이 5번 실행되면 서버 사이드 Prepared Statement로 전환
//   처음 4번: 텍스트 쿼리 (simple query protocol)
//   5번째부터: PREPARE ... AS + EXECUTE ... (extended query protocol)
//   → 실행계획 서버에 저장, 이후 EXECUTE만으로 재실행

// preparedStatementCacheQueries=256:
//   Connection당 최대 256개 서버 사이드 PreparedStatement 유지

// PostgreSQL 서버 사이드 PreparedStatement:
// PREPARE myplan AS SELECT * FROM users WHERE id = $1
// EXECUTE myplan(42)
// DEALLOCATE myplan (Connection 닫힐 때)
```

### 3. HikariCP와 Statement 캐싱의 관계

```java
// HikariCP 자체는 Statement 캐싱 없음
// → 캐싱은 JDBC 드라이버가 담당

// HikariCP의 Statement 관련 설정:
// connectionInitSql: Connection 생성 시 실행할 SQL
// → Statement 캐싱과 무관

// HikariCP가 Statement에 영향을 주는 방식:
// Connection 반환 시 Statement 정리
// ProxyConnection.close() → 열린 Statement 닫기 (실제 Connection은 유지)

// JDBC 드라이버 캐싱과의 시너지:
// HikariCP: Connection 재사용 (새 Connection 생성 방지)
// JDBC 드라이버: 동일 Connection에서 동일 SQL → 캐시 히트
// → 두 캐싱이 함께 동작해야 효과적
```

### 4. 캐싱 효과가 극대화되는 패턴

```java
// ✅ 파라미터 바인딩 사용 → SQL 구조 동일 → 캐시 히트

// 패턴 1: JdbcTemplate (파라미터 바인딩)
for (User user : users) {
    jdbcTemplate.update(
        "INSERT INTO users (name, email) VALUES (?, ?)", // SQL 구조 동일
        user.getName(), user.getEmail()                  // 파라미터만 변경
    );
    // → 드라이버 캐시: 동일 SQL → 캐시 히트
    // → DB 서버: 동일 실행계획 재사용
}

// 패턴 2: JPQL (Hibernate QueryPlanCache + Statement Cache)
List<User> users = em.createQuery(
    "SELECT u FROM User u WHERE u.status = :s", User.class)
    .setParameter("s", Status.ACTIVE)
    .getResultList();
// → Hibernate QueryPlanCache: JPQL → SQL 변환 결과 캐시
// → JDBC 드라이버: 동일 SQL → Statement 캐시 히트

// 패턴 3: Spring Data JPA @Query
@Query("SELECT u FROM User u WHERE u.status = :status")
List<User> findByStatus(@Param("status") Status status);
// → Spring 시작 시 쿼리 파싱 → QueryPlanCache 저장
// → JDBC 드라이버 Statement Cache와 함께 두 단계 캐싱
```

### 5. Statement 캐싱 모니터링

```java
// MySQL — Statement Cache 히트율 확인
// SHOW STATUS LIKE 'Qcache%';  // Query Cache 통계 (MySQL 5.x)
// SHOW STATUS LIKE 'Com_stmt%'; // Prepared Statement 통계

// MySQL Connector/J 메트릭 (5.1.3+)
// MBean: com.mysql.jdbc:type=Driver
// → preparedStatementCacheHitCount
// → preparedStatementCacheMissCount

// PostgreSQL — 서버 사이드 Prepared Statement 확인
// SELECT * FROM pg_prepared_statements;
// → 현재 세션의 서버 사이드 Prepared Statement 목록

// 애플리케이션 레벨 모니터링 (P6Spy, datasource-proxy)
@Bean
public DataSource dataSource() {
    return ProxyDataSourceBuilder.create(hikariDataSource)
        .logQueryBySlf4j(INFO)
        .countQuery()        // 쿼리별 실행 횟수
        .build();
}
```

---

## 💻 실험으로 확인하기

### 실험 1: Statement Cache 히트율 측정

```java
@Test
void statementCacheEffect() {
    String sql = "SELECT id, name FROM users WHERE status = ?";
    int iterations = 1000;

    // cachePrepStmts=false (캐싱 없음)
    long start = System.nanoTime();
    for (int i = 0; i < iterations; i++) {
        jdbcTemplate.query(sql, rowMapper, "ACTIVE");
    }
    long noCacheTime = System.nanoTime() - start;

    // cachePrepStmts=true (캐싱 있음)
    // → JDBC URL에 ?cachePrepStmts=true 추가 후 재실행
    start = System.nanoTime();
    for (int i = 0; i < iterations; i++) {
        jdbcTemplate.query(sql, rowMapper, "ACTIVE");
    }
    long cacheTime = System.nanoTime() - start;

    System.out.printf("캐싱 없음: %dms%n", noCacheTime / 1_000_000);
    System.out.printf("캐싱 있음: %dms%n", cacheTime / 1_000_000);
    // 결과 (로컬 MySQL):
    // 캐싱 없음: ~320ms
    // 캐싱 있음: ~190ms (~40% 개선)
}
```

### 실험 2: 동적 SQL vs 파라미터 바인딩 캐시 효과 비교

```java
@Test
void dynamicSqlVsParameterBinding() {
    int iterations = 500;

    // ❌ 동적 SQL — 캐시 미스 반복
    long start = System.nanoTime();
    for (int i = 0; i < iterations; i++) {
        jdbcTemplate.queryForObject(
            "SELECT name FROM users WHERE id = " + i, String.class);
        // SQL 구조 매번 다름 → 캐시 미스
    }
    long dynamicTime = System.nanoTime() - start;

    // ✅ 파라미터 바인딩 — 캐시 히트
    start = System.nanoTime();
    for (int i = 0; i < iterations; i++) {
        jdbcTemplate.queryForObject(
            "SELECT name FROM users WHERE id = ?", String.class, (long) i);
        // SQL 구조 동일 → 캐시 히트
    }
    long bindingTime = System.nanoTime() - start;

    System.out.printf("동적 SQL: %dms%n", dynamicTime / 1_000_000);
    System.out.printf("파라미터 바인딩: %dms%n", bindingTime / 1_000_000);
    // 동적 SQL: ~450ms, 바인딩: ~200ms → 2배 이상 차이
}
```

---

## ⚡ 성능 임팩트

```
Statement 캐싱 효과 (MySQL, 파싱 비용 기준):

SQL 파싱 비용:
  단순 SELECT: ~0.1ms
  복잡한 JOIN/서브쿼리: ~1~5ms

1000 TPS, 파싱 비용 0.5ms:
  캐싱 없음: 1000 × 0.5ms = 500ms 순수 파싱 오버헤드/초
  캐싱 있음: 초기 파싱 후 캐시 히트 → ~수 μs/회
  → 캐싱으로 파싱 비용 ~99% 절감

실제 애플리케이션 영향:
  단순 쿼리 중심: ~5~20% 응답 시간 개선
  복잡한 쿼리 중심: ~20~40% 개선
  배치 처리 (동일 SQL 반복): ~40~60% 개선

메모리 비용:
  prepStmtCacheSize=250, Pool 20개:
  → 최대 5000개 Statement 캐시
  → Statement당 수 KB → 총 ~수십 MB
  → 현대 서버에서 허용 가능한 수준
```

---

## 🤔 트레이드오프

```
Statement 캐싱 활성화 (cachePrepStmts=true):
  ✅ 반복 SQL 파싱 비용 절감
  ✅ DB 서버 실행계획 재사용
  ✅ 네트워크 전송량 감소 (useServerPrepStmts=true)
  ❌ Connection당 메모리 증가
  ❌ 동적 SQL에는 효과 없음 (캐시 오염)
  → 정적 SQL 중심 애플리케이션에 권장

useServerPrepStmts=true:
  ✅ DB 서버에 실행계획 저장 → 클라이언트 전송량 최소
  ✅ DB 서버 파싱 완전 생략
  ❌ Connection당 DB 서버 리소스 사용 증가
  ❌ 짧은 수명 Connection에서는 오버헤드 (생성 비용)
  → 장기 Connection, 반복 쿼리 많은 환경

prepStmtCacheSize 크게:
  ✅ 더 많은 SQL 캐싱
  ❌ 메모리 증가
  → SQL 종류 많으면 크게, 적으면 작게

SQL Injection 방지와의 관계:
  파라미터 바인딩 = SQL Injection 방지 + Statement 캐싱 효과
  → 보안 + 성능을 동시에 얻는 올바른 패턴
```

---

## 📌 핵심 정리

```
캐싱 레이어
  Hibernate QueryPlanCache: JPQL → SQL 변환 결과 (프로세스 공유)
  JDBC 드라이버 Cache:      PreparedStatement 핸들 (Connection별)
  DB 서버 Cache:            실행계획 (DB 서버 공유)

MySQL 권장 설정
  cachePrepStmts=true
  prepStmtCacheSize=250
  prepStmtCacheSqlLimit=2048
  useServerPrepStmts=true
  → JDBC URL에 파라미터로 추가

효과 극대화 조건
  동일 SQL 구조 반복 (파라미터만 변경)
  장기 Connection 재사용 (HikariCP)
  정적 SQL (동적 IN절, 문자열 연결 없음)

캐싱 효과 없는 경우
  SQL 구조가 매번 다른 동적 쿼리
  SQL 값을 직접 삽입 (파라미터 바인딩 미사용)
  매우 짧은 Connection 수명
```

---

## 🤔 생각해볼 문제

**Q1.** `useServerPrepStmts=true` 설정 시 Connection을 Pool에 반환할 때 서버 사이드 PreparedStatement는 어떻게 되는가? HikariCP가 Connection을 재사용하면 이전 PreparedStatement가 유지되는가?

**Q2.** Hibernate `hibernate.jdbc.batch_size=1000`으로 배치 INSERT를 수행할 때 Statement Cache가 동작하는가? JDBC 드라이버 캐시와 Hibernate 배치의 관계는?

**Q3.** `prepStmtCacheSize=250`이고 애플리케이션에서 사용하는 고유 SQL이 300개라면 어떤 일이 발생하는가? 이 상황을 어떻게 모니터링하고 대응하는가?

> 💡 **해설**
>
> **Q1.** HikariCP가 Connection을 Pool에 반환할 때(`ProxyConnection.close()`) 서버 사이드 PreparedStatement는 닫히지 않는다. 실제 Connection은 Pool에서 계속 유지되므로 서버 사이드 PreparedStatement도 그대로 존재한다. 다음 요청이 같은 Connection을 대여받으면 JDBC 드라이버의 캐시에서 해당 PreparedStatement를 찾아 재사용한다. 즉 HikariCP의 Connection 재사용이 JDBC 드라이버 Statement 캐싱의 전제 조건이다 — Connection이 닫히면 서버 사이드 PreparedStatement도 `DEALLOCATE`된다. 따라서 Connection Pool을 쓰지 않고 매번 새 Connection을 생성하면 Statement Cache 효과가 없다.
>
> **Q2.** 동작한다. Hibernate 배치는 동일 SQL 구조(`INSERT INTO products (name, price) VALUES (?, ?)`)를 반복 실행하므로 JDBC 드라이버 Statement Cache가 효과적으로 적용된다. Hibernate가 배치 모드에서 `addBatch()`를 호출할 때 내부적으로 동일 PreparedStatement 객체를 재사용한다 — 파라미터만 교체하고 `addBatch()`로 쌓은 뒤 `executeBatch()`로 한 번에 전송한다. JDBC 드라이버 캐시는 PreparedStatement 생성 시점에 적용되고, 배치는 PreparedStatement를 재사용하므로 두 최적화가 자연스럽게 연계된다.
>
> **Q3.** 캐시 크기(250)보다 고유 SQL(300)이 많으면 LRU 정책으로 오래된 항목이 제거된다. 접근 빈도가 낮은 50개 SQL은 캐시에서 밀려나고, 해당 SQL 실행 시 파싱이 다시 발생한다. 모니터링: MySQL `SHOW STATUS LIKE 'Com_stmt_prepare'` — Prepared Statement 생성 횟수가 지속적으로 증가하면 캐시 미스가 많다는 의미다. 대응: (1) `prepStmtCacheSize`를 고유 SQL 수 이상으로 늘린다(예: 350). (2) 고유 SQL을 분석해 동적 SQL을 파라미터 바인딩으로 통합해 SQL 종류를 줄인다. (3) 접근 빈도 낮은 SQL은 캐싱 효과가 작으므로 캐시 크기 증가보다 SQL 통합이 더 효과적이다.

---

<div align="center">

**[⬅️ 이전: Connection Health Check 전략](./04-connection-health-check.md)** | **[다음: Connection Timeout vs Idle Timeout ➡️](./06-connection-timeout-vs-idle-timeout.md)**

</div>
