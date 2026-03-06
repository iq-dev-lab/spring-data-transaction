# Batch 처리 최적화 — batchUpdate, Chunk 전략, JPA saveAll() 비교

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JdbcTemplate.batchUpdate()`가 `PreparedStatement.addBatch()`/`executeBatch()`를 감싸는 구조는?
- MySQL `rewriteBatchedStatements=true`가 성능에 미치는 실질적인 효과는?
- `JPA saveAll()` vs `JdbcTemplate batchUpdate()` — 언제 어떤 방식이 빠른가?
- Chunk 단위 처리가 필요한 이유와 적절한 Chunk 크기는?
- 배치 실행 중 일부 실패 시 오류 처리 전략은?

---

## 🔍 왜 이게 존재하는가

### 문제: 단건 INSERT를 반복하면 극도로 느리다

```java
// ❌ 단건 반복 INSERT — 1만 건에 ~30초
for (Product product : products) { // 10,000건
    jdbcTemplate.update(
        "INSERT INTO products (name, price) VALUES (?, ?)",
        product.getName(), product.getPrice()
    );
    // 매번: Connection 조회 → PreparedStatement 생성 → execute → 네트워크 왕복
}
// 10,000 × (PS 생성 + 네트워크 왕복) = 극히 느림

// ✅ batchUpdate — 1만 건에 ~0.5초
jdbcTemplate.batchUpdate(
    "INSERT INTO products (name, price) VALUES (?, ?)",
    products,
    1000,  // Chunk 크기
    (ps, product) -> {
        ps.setString(1, product.getName());
        ps.setBigDecimal(2, product.getPrice());
    }
);
// 1000건씩 묶어 executeBatch() → 네트워크 왕복 10번
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: batchUpdate를 쓰면 DB에 한 번만 전송된다

```java
// ❌ 잘못된 이해: 10,000건이 SQL 1번으로 처리된다

// ✅ 실제 (rewriteBatchedStatements=false 기본값):
// batchUpdate → addBatch() × 1000 → executeBatch()
// executeBatch(): DB에 SQL 1000번 전송 (but 하나의 라운드트립)
// → PreparedStatement는 재사용, 네트워크 왕복은 chunk 수만큼

// ✅ rewriteBatchedStatements=true (MySQL):
// executeBatch(): "INSERT INTO t VALUES (?,?),(?,?),(?,?),..."
// → SQL 1개로 재작성 → 진정한 단일 전송

// JDBC URL 설정:
// jdbc:mysql://localhost/db?rewriteBatchedStatements=true
```

### Before: JPA saveAll()은 배치 처리를 한다

```java
// ❌ 잘못된 이해: saveAll()은 batchUpdate()와 동일하게 동작

// ✅ 실제 — 두 가지 함정:

// 함정 1: IDENTITY 전략에서 배치 불가
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // AUTO_INCREMENT
    private Long id;
}

// saveAll(products):
// → SimpleJpaRepository.save() 반복 호출
// → em.persist() 호출 시 즉시 INSERT 실행 (ID 필요, IDENTITY 전략)
// → 배치 비활성화 → 1만 건 = INSERT 1만 번
// hibernate.jdbc.batch_size=1000 설정이 있어도 무효!

// 함정 2: 영속성 컨텍스트 메모리
// saveAll(10000개) → 10000개 엔티티 + 스냅샷이 1차 캐시에 쌓임
// → GC 압력, OutOfMemoryError 위험

// ✅ 배치가 동작하는 조건
@GeneratedValue(strategy = GenerationType.SEQUENCE) // SEQUENCE 전략
// + hibernate.jdbc.batch_size=1000
// + hibernate.order_inserts=true
```

---

## ✨ 올바른 이해와 패턴

### After: 배치 방식 선택 기준

```
JPA saveAll():
  소규모 (1,000건 미만): 충분히 빠름
  SEQUENCE 전략 + batch_size 설정 시 배치 동작
  IDENTITY 전략: 배치 불가 → JdbcTemplate 사용

JdbcTemplate.batchUpdate():
  대용량 (1만 건 이상): 권장
  IDENTITY 전략 테이블도 배치 가능
  영속성 컨텍스트 오버헤드 없음
  Chunk 처리로 메모리 제어 가능

StatelessSession (Hibernate):
  영속성 컨텍스트 완전 제거
  중간 규모 배치 (1만~100만 건)
  JPA 매핑 유지하면서 배치 성능 확보
```

---

## 🔬 내부 동작 원리 — batchUpdate 소스 추적

### 1. JdbcTemplate.batchUpdate() — 핵심 구현

```java
// JdbcTemplate.batchUpdate(String sql, Collection<T> batchArgs,
//                          int batchSize, ParameterizedPreparedStatementSetter<T> pss)
public <T> int[][] batchUpdate(String sql, Collection<T> batchArgs, int batchSize,
                                ParameterizedPreparedStatementSetter<T> pss) {

    return execute(sql, (PreparedStatementCallback<int[][]>) ps -> {
        // 결과 저장 (청크별 결과 배열)
        List<int[]> rowsAffected = new ArrayList<>();
        boolean batchSupported = JdbcUtils.supportsBatchUpdates(ps.getConnection());
        // → 드라이버가 배치를 지원하는지 확인

        int n = 0;
        for (T obj : batchArgs) {
            pss.setValues(ps, obj); // 파라미터 바인딩
            n++;

            if (batchSupported) {
                ps.addBatch();       // 배치에 추가
                if (n % batchSize == 0 || n == batchArgs.size()) {
                    // chunk 크기 도달 또는 마지막 → 실행
                    int[] batchResult = ps.executeBatch();
                    rowsAffected.add(batchResult);
                }
            } else {
                // 배치 미지원 드라이버 → 개별 실행
                int i = ps.executeUpdate();
                rowsAffected.add(new int[]{i});
            }
        }
        return rowsAffected.toArray(new int[0][]);
    });
}
```

### 2. rewriteBatchedStatements — MySQL 최적화

```java
// rewriteBatchedStatements=false (기본):
// executeBatch() → MySQL로 전송:
//   COM_STMT_EXECUTE (id=1, params=[A, 1000])
//   COM_STMT_EXECUTE (id=1, params=[B, 2000])
//   COM_STMT_EXECUTE (id=1, params=[C, 3000])
// → 3개 별도 요청 (단, 하나의 TCP 패킷에 묶일 수 있음)

// rewriteBatchedStatements=true:
// executeBatch() → MySQL로 전송:
//   INSERT INTO products (name, price) VALUES ('A',1000),('B',2000),('C',3000)
// → 단일 INSERT 문 → DB 파싱 1번, 트랜잭션 로그 최소화

// 성능 차이 (1만 건 INSERT):
// rewriteBatch=false: ~5초 (executeBatch 기준)
// rewriteBatch=true:  ~0.3초 (멀티 VALUES INSERT)
// → 최대 10~20배 차이

// JDBC URL 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db?rewriteBatchedStatements=true&useSSL=false
```

### 3. Chunk 단위 처리 — 메모리 제어

```java
// 대용량 배치 Chunk 패턴
@Transactional
public void insertLargeDataset(List<Product> products) {
    int chunkSize = 1000;

    // 전체를 Chunk로 분할하여 처리
    for (int i = 0; i < products.size(); i += chunkSize) {
        int end = Math.min(i + chunkSize, products.size());
        List<Product> chunk = products.subList(i, end);

        jdbcTemplate.batchUpdate(
            "INSERT INTO products (name, price, category_id) VALUES (?, ?, ?)",
            chunk,
            chunk.size(),
            (ps, p) -> {
                ps.setString(1, p.getName());
                ps.setBigDecimal(2, p.getPrice());
                ps.setLong(3, p.getCategoryId());
            }
        );
    }
}

// 파일 또는 스트림 소스에서 처리 (메모리 절약)
@Transactional
public void insertFromStream(Stream<ProductDto> productStream) {
    List<ProductDto> buffer = new ArrayList<>(1000);

    productStream.forEach(dto -> {
        buffer.add(dto);
        if (buffer.size() >= 1000) {
            flushBatch(buffer);
            buffer.clear(); // 버퍼 비우기 → GC 가능
        }
    });

    if (!buffer.isEmpty()) {
        flushBatch(buffer); // 잔여분 처리
    }
}

private void flushBatch(List<ProductDto> batch) {
    jdbcTemplate.batchUpdate(
        "INSERT INTO products (name, price) VALUES (?, ?)",
        batch, batch.size(),
        (ps, dto) -> {
            ps.setString(1, dto.name());
            ps.setBigDecimal(2, dto.price());
        }
    );
}
```

### 4. JPA saveAll() vs JdbcTemplate 실측 비교

```java
// 테스트: 상품 10,000건 INSERT

// [방법 1] JPA saveAll() — IDENTITY 전략
List<Product> products = createProducts(10000);
long start = System.currentTimeMillis();
productRepository.saveAll(products); // IDENTITY → 배치 비활성화
System.out.println("JPA saveAll(IDENTITY): " +
    (System.currentTimeMillis() - start) + "ms");
// 결과: ~8,000ms (8초) — 10,000번 개별 INSERT

// [방법 2] JPA saveAll() — SEQUENCE 전략 + batch_size=1000
// hibernate.jdbc.batch_size=1000
// hibernate.order_inserts=true
start = System.currentTimeMillis();
productRepository.saveAll(products); // SEQUENCE → 배치 동작
System.out.println("JPA saveAll(SEQUENCE): " +
    (System.currentTimeMillis() - start) + "ms");
// 결과: ~2,000ms (2초) — 1000건씩 배치

// [방법 3] JdbcTemplate batchUpdate() — rewriteBatch=false
start = System.currentTimeMillis();
jdbcTemplate.batchUpdate(insertSql, products, 1000, setter);
System.out.println("JdbcTemplate batch: " +
    (System.currentTimeMillis() - start) + "ms");
// 결과: ~500ms (0.5초)

// [방법 4] JdbcTemplate batchUpdate() — rewriteBatch=true (MySQL)
// JDBC URL: ?rewriteBatchedStatements=true
start = System.currentTimeMillis();
jdbcTemplate.batchUpdate(insertSql, products, 1000, setter);
System.out.println("JdbcTemplate rewrite batch: " +
    (System.currentTimeMillis() - start) + "ms");
// 결과: ~150ms (0.15초) — 최고 성능
```

### 5. 배치 오류 처리 전략

```java
// batchUpdate() 실패 시 처리
try {
    jdbcTemplate.batchUpdate(insertSql, batch, batchSize, setter);
} catch (DataAccessException e) {
    if (e.getCause() instanceof BatchUpdateException bue) {
        int[] updateCounts = bue.getUpdateCounts();
        // updateCounts[i] = 성공: 영향 받은 행 수 (≥0)
        //                   실패: Statement.EXECUTE_FAILED (-3)
        for (int i = 0; i < updateCounts.length; i++) {
            if (updateCounts[i] == Statement.EXECUTE_FAILED) {
                log.error("{}번째 항목 실패: {}", i, batch.get(i));
            }
        }
    }
    throw e; // @Transactional → 롤백
}

// 실패 허용 배치 (일부 실패해도 계속 처리)
// → @Transactional 없이 청크별 트랜잭션 처리
public BatchResult insertWithPartialFailure(List<Product> products) {
    int success = 0, failed = 0;
    List<Product> failedItems = new ArrayList<>();

    for (List<Product> chunk : Lists.partition(products, 100)) {
        try {
            // 청크별 별도 트랜잭션
            transactionTemplate.execute(status -> {
                jdbcTemplate.batchUpdate(sql, chunk, chunk.size(), setter);
                return null;
            });
            success += chunk.size();
        } catch (DataAccessException e) {
            failed += chunk.size();
            failedItems.addAll(chunk);
            log.warn("청크 실패: {}건", chunk.size());
        }
    }
    return new BatchResult(success, failed, failedItems);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 배치 지원 여부 확인

```java
@Test
void batchSupportCheck() throws SQLException {
    try (Connection conn = dataSource.getConnection()) {
        DatabaseMetaData meta = conn.getMetaData();
        System.out.println("Batch supported: " + meta.supportsBatchUpdates());
        // MySQL, PostgreSQL, H2: true
        // 일부 레거시 DB: false → 개별 실행 폴백
    }
}
```

### 실험 2: Chunk 크기별 성능

```java
@Test
void chunkSizePerformance() {
    List<Product> products = createProducts(10000);

    for (int chunkSize : new int[]{100, 500, 1000, 5000}) {
        long start = System.currentTimeMillis();
        jdbcTemplate.batchUpdate(sql, products, chunkSize, setter);
        System.out.printf("Chunk %d: %dms%n",
            chunkSize, System.currentTimeMillis() - start);
    }
    // Chunk 100:  ~800ms (100번 executeBatch)
    // Chunk 500:  ~350ms
    // Chunk 1000: ~250ms ← 보통 최적
    // Chunk 5000: ~230ms (메모리 더 사용, 미미한 추가 개선)
}
```

---

## ⚡ 성능 임팩트

```
10,000건 INSERT 성능 비교 (MySQL, 로컬 환경):

단건 반복 (JdbcTemplate.update()):    ~30,000ms (30초)
JPA saveAll() IDENTITY:                ~8,000ms  (8초)
JPA saveAll() SEQUENCE+batch:          ~2,000ms  (2초)
JdbcTemplate.batchUpdate():            ~500ms    (0.5초)
batchUpdate() + rewriteBatch=true:     ~150ms    (0.15초)
batchUpdate() + LOAD DATA INFILE:      ~30ms     (0.03초, 파일 기반)

권장 Chunk 크기:
  일반적: 500~1000
  메모리 여유 충분: 2000~5000
  메모리 제한: 100~500
  → DB 종류, 행 크기, 네트워크 지연에 따라 다름

rewriteBatchedStatements 적용 조건:
  MySQL + Connector/J JDBC 드라이버
  PostgreSQL: 자체 배치 최적화 있음 (rewrite 옵션 없음)
  → DB별 최적화 방법 다름
```

---

## 🤔 트레이드오프

```
JPA saveAll():
  ✅ 도메인 모델 유지 (엔티티 매핑, 생명주기 이벤트)
  ✅ 영속성 컨텍스트 관리 (dirty checking, cascade)
  ❌ IDENTITY 전략에서 배치 불가
  ❌ 대용량 시 메모리 문제 (1차 캐시 누적)
  → 소규모 (1,000건 미만), 도메인 이벤트 필요 시

JdbcTemplate.batchUpdate():
  ✅ DB 독립적 배치 (IDENTITY도 가능)
  ✅ 영속성 컨텍스트 없음 → 메모리 효율
  ✅ rewriteBatchedStatements로 최고 성능
  ❌ SQL 직접 작성, RowMapper 필요
  ❌ 도메인 이벤트(@PrePersist 등) 미실행
  → 대용량 (1만 건 이상), 성능 우선

StatelessSession:
  ✅ JPA 매핑 유지, 영속성 컨텍스트 없음
  ✅ 도메인 모델과 배치 성능 절충
  ❌ cascade, dirty checking 없음
  → 중간 규모, JPA 매핑 재사용 원할 때
```

---

## 📌 핵심 정리

```
batchUpdate() 동작
  addBatch() × chunkSize → executeBatch()
  → chunk 수만큼 DB 라운드트립
  rewriteBatchedStatements=true (MySQL)
  → executeBatch() → INSERT ... VALUES (...),(...),...
  → 라운드트립 최소화, 최대 성능

JPA saveAll() 배치 조건
  IDENTITY 전략: 배치 불가 (즉시 INSERT로 ID 필요)
  SEQUENCE 전략: 배치 가능
  + hibernate.jdbc.batch_size
  + hibernate.order_inserts=true

Chunk 처리 원칙
  Chunk 크기: 500~1000 권장
  Chunk별 트랜잭션: 메모리 + 롤백 범위 제어
  스트림 소스: 버퍼 + flush + clear 패턴

오류 처리
  전체 실패 허용: @Transactional + 예외 전파
  부분 실패 허용: Chunk별 트랜잭션 + 실패 로깅
```

---

## 🤔 생각해볼 문제

**Q1.** `JdbcTemplate.batchUpdate(String sql, List<Object[]> batchArgs)` 오버로드와 `batchUpdate(String sql, Collection<T> batchArgs, int batchSize, ParameterizedPreparedStatementSetter<T> pss)` 오버로드의 동작 차이는? 전자에서 batchSize를 제어하는 방법은?

**Q2.** `rewriteBatchedStatements=true`를 설정한 MySQL 환경에서 `@Transactional` 없이 `batchUpdate()`를 실행하면 어떻게 되는가? 중간에 실패가 발생하면?

**Q3.** PostgreSQL에서 `JdbcTemplate.batchUpdate()`의 성능을 MySQL의 `rewriteBatchedStatements`와 유사한 수준으로 끌어올리는 방법은?

> 💡 **해설**
>
> **Q1.** `batchUpdate(String sql, List<Object[]> batchArgs)` 오버로드는 전체 목록을 하나의 배치로 처리한다 — Chunk 분할 없이 `addBatch()` × 전체 크기 → `executeBatch()` 1번. 데이터가 10만 건이면 10만 개를 한 번에 `addBatch()`에 쌓은 후 `executeBatch()` 1번으로 실행한다. 메모리에 전체 데이터가 올라오고 PreparedStatement 내부 버퍼도 커진다. Chunk를 제어하려면 `Lists.partition(batchArgs, 1000).forEach(chunk -> jdbcTemplate.batchUpdate(sql, chunk))`처럼 직접 분할하거나, `ParameterizedPreparedStatementSetter` 오버로드를 사용해 `batchSize` 파라미터를 명시한다.
>
> **Q2.** `@Transactional` 없으면 각 `addBatch()` / `executeBatch()`가 auto-commit 모드에서 실행된다. `rewriteBatchedStatements=true`로 재작성된 단일 `INSERT ... VALUES (...), (...), ...`는 DB에서 원자적으로 실행된다. 즉 하나의 멀티-VALUES INSERT가 성공하면 전체 chunk가 커밋된다. 중간 실패는 chunk 내에서 발생하는데, 하나의 SQL 문이므로 해당 chunk 전체가 실패한다. 다른 이미 실행된 chunk는 auto-commit으로 이미 커밋되어 롤백 불가다. 대용량 배치에서 오류 허용 범위를 chunk 단위로 제어하려면 `TransactionTemplate`으로 chunk별 트랜잭션을 명시적으로 관리해야 한다.
>
> **Q3.** PostgreSQL에서는 `COPY` 명령을 사용하는 것이 가장 빠르다. PostgreSQL JDBC 드라이버의 `CopyManager`를 통해 CSV 형식으로 데이터를 스트리밍 전송할 수 있다: `new CopyManager(pgConn).copyIn("COPY products (name, price) FROM STDIN WITH CSV", reader)`. 이 방식은 MySQL `rewriteBatchedStatements`와 유사하거나 더 빠른 성능을 낸다. `JdbcTemplate`으로 직접 지원하지 않으므로 `jdbcTemplate.execute((ConnectionCallback<Long>) conn -> new CopyManager(conn.unwrap(BaseConnection.class)).copyIn(...))`처럼 ConnectionCallback을 통해 접근한다. 또한 PostgreSQL은 `INSERT ... ON CONFLICT` (Upsert) 지원과 함께 배치 INSERT 최적화가 잘 되어있어, 일반 `batchUpdate()`도 `rewrite` 없이 충분히 빠른 경우가 많다.

---

<div align="center">

**[⬅️ 이전: SimpleJdbcInsert로 단순 삽입](./04-simple-jdbc-insert.md)** | **[홈으로 🏠](../README.md)**

</div>
