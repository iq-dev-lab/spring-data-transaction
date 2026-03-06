# JdbcTemplate 내부 구조 — Template Method 패턴으로 JDBC 보일러플레이트 제거

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JdbcTemplate이 Template Method 패턴으로 JDBC 코드를 추상화하는 구체적인 방식은?
- `execute()`→`query()`→`update()` 메서드 체인에서 Connection 획득과 반환이 어디서 일어나는가?
- `DataSourceUtils.getConnection()`이 트랜잭션 컨텍스트와 연계되는 방법은?
- JdbcTemplate이 SQLException을 Spring `DataAccessException`으로 변환하는 경로는?
- `JdbcTemplate`과 `NamedParameterJdbcTemplate`의 관계는?

---

## 🔍 왜 이게 존재하는가

### 문제: 순수 JDBC는 반복 코드가 극심하다

```java
// ❌ 순수 JDBC — 보일러플레이트 범람
public User findById(Long id) {
    Connection conn = null;
    PreparedStatement ps = null;
    ResultSet rs = null;
    try {
        conn = dataSource.getConnection();           // 1. Connection 획득
        ps = conn.prepareStatement(                  // 2. Statement 생성
            "SELECT id, name, email FROM users WHERE id = ?");
        ps.setLong(1, id);                           // 3. 파라미터 바인딩
        rs = ps.executeQuery();                      // 4. 실행
        if (rs.next()) {
            return new User(                         // 5. 결과 매핑
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email")
            );
        }
        return null;
    } catch (SQLException e) {
        throw new RuntimeException(e);               // 6. 예외 처리
    } finally {
        if (rs != null) try { rs.close(); } catch (SQLException ignored) {}
        if (ps != null) try { ps.close(); } catch (SQLException ignored) {}
        if (conn != null) try { conn.close(); } catch (SQLException ignored) {}
        // 7. 자원 반환 (빠뜨리면 Connection Leak)
    }
}
```

```
JdbcTemplate이 추상화하는 것:
  고정 코드 (JdbcTemplate 담당):
    Connection 획득 / 반환
    PreparedStatement 생성
    ResultSet 닫기
    SQLException → DataAccessException 변환

  가변 코드 (사용자 제공):
    SQL 문자열
    파라미터 바인딩 (PreparedStatementSetter)
    ResultSet 매핑 (RowMapper / ResultSetExtractor)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: JdbcTemplate은 트랜잭션과 무관하게 항상 새 Connection을 사용한다

```java
// ❌ 잘못된 이해
@Transactional
public void process() {
    jdbcTemplate.update("INSERT INTO logs ...");  // 새 Connection?
    jpaRepository.save(entity);                  // JPA Connection?
    // → 서로 다른 Connection → 같은 트랜잭션으로 묶이지 않는다?
}

// ✅ 실제:
// JdbcTemplate.getConnection() → DataSourceUtils.getConnection(dataSource)
// → TransactionSynchronizationManager.getResource(dataSource)
// → 현재 트랜잭션에 바인딩된 ConnectionHolder에서 Connection 반환
// → JPA와 JdbcTemplate이 동일한 Connection 공유!
// (JpaTransactionManager가 DataSource도 ThreadLocal에 등록하기 때문)
```

### Before: JdbcTemplate 사용 시 Connection을 직접 닫아야 한다

```java
// ❌ 절대 하면 안 됨
Connection conn = DataSourceUtils.getConnection(dataSource);
// ... 작업
conn.close();  // 트랜잭션 중이면 Connection을 강제로 닫아버림 → 트랜잭션 깨짐

// ✅ JdbcTemplate이 자동으로 처리
// Connection 반환: DataSourceUtils.releaseConnection(conn, dataSource)
// → 트랜잭션 중이면 실제로 닫지 않고 보관 (트랜잭션 종료 시 반환)
// → 트랜잭션 없으면 실제로 Pool에 반환
```

---

## ✨ 올바른 이해와 패턴

### After: JdbcTemplate 핵심 메서드 계층

```
execute(StatementCallback)         ← 가장 기본, Statement 직접 제어
    ↓ (내부 구현)
execute(PreparedStatementCreator,  ← PreparedStatement 기반
        PreparedStatementCallback)
    ↓ (래핑)
query(String sql, RowMapper)       ← SELECT + 행 단위 매핑
queryForObject(String, RowMapper)  ← 단건 SELECT
update(String sql, Object...)      ← INSERT / UPDATE / DELETE
batchUpdate(String, List<Object[]>)← 배치 처리

모든 메서드는 결국 execute()로 수렴
→ Connection 획득 / 사용 / 반환 로직이 한 곳에 집중
```

---

## 🔬 내부 동작 원리 — JdbcTemplate 소스 추적

### 1. execute() — 모든 작업의 뼈대

```java
// JdbcTemplate.execute(PreparedStatementCreator, PreparedStatementCallback)
// — 핵심 Template Method 구현
@Nullable
public <T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action) {

    // [1] Connection 획득 — 트랜잭션 컨텍스트 연계
    Connection con = DataSourceUtils.getConnection(obtainDataSource());
    PreparedStatement ps = null;
    try {

        // [2] PreparedStatement 생성 (사용자 제공 psc)
        ps = psc.createPreparedStatement(con);

        // [3] Statement 설정 (타임아웃, fetchSize 등)
        applyStatementSettings(ps);

        // [4] 실제 작업 실행 (사용자 제공 action)
        T result = action.doInPreparedStatement(ps);

        // [5] 경고 처리
        handleWarnings(ps);

        return result;

    } catch (SQLException ex) {
        // [6] SQL 오류 → Spring DataAccessException 변환
        // psc가 이름 정보를 갖고 있으면 SQL 포함
        String sql = getSql(psc);
        JdbcUtils.closeStatement(ps);
        ps = null;
        DataSourceUtils.releaseConnection(con, getDataSource());
        con = null;
        throw translateException("PreparedStatementCallback", sql, ex);
        // → SQLExceptionTranslator.translate()
        // → SQLErrorCodeSQLExceptionTranslator (DB 에러 코드 기반)
        // → DuplicateKeyException, DeadlockLoserDataAccessException 등

    } finally {
        JdbcUtils.closeStatement(ps);
        // [7] Connection 반환 — 트랜잭션 중이면 실제로 닫지 않음
        DataSourceUtils.releaseConnection(con, getDataSource());
    }
}
```

### 2. DataSourceUtils.getConnection() — 트랜잭션 연계

```java
// DataSourceUtils.getConnection()
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
    try {
        return doGetConnection(dataSource);
    } catch (SQLException ex) {
        throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection", ex);
    }
}

public static Connection doGetConnection(DataSource dataSource) throws SQLException {

    // 현재 트랜잭션에 바인딩된 Connection 탐색
    ConnectionHolder conHolder =
        (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);

    if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
        conHolder.requested(); // 참조 카운트 증가
        if (!conHolder.hasConnection()) {
            conHolder.setConnection(dataSource.getConnection());
        }
        return conHolder.getConnection();
        // → 트랜잭션에 바인딩된 기존 Connection 반환 (새 Connection 아님!)
    }

    // 트랜잭션 없는 경우 → Pool에서 새 Connection 획득
    Connection con = dataSource.getConnection();

    // 트랜잭션 동기화 활성 상태면 등록
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        ConnectionHolder holderToUse = conHolder;
        if (holderToUse == null) {
            holderToUse = new ConnectionHolder(con);
        } else {
            holderToUse.setConnection(con);
        }
        holderToUse.requested();
        TransactionSynchronizationManager.registerSynchronization(
            new ConnectionSynchronization(holderToUse, dataSource));
        // → 트랜잭션 종료 시 자동 반환 등록
        holderToUse.setSynchronizedWithTransaction(true);
        TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
    }
    return con;
}
```

### 3. SQLException → DataAccessException 변환

```java
// SQLExceptionTranslator 계층
// JdbcTemplate.translateException() 호출
// → SQLErrorCodeSQLExceptionTranslator (기본)
//   → DB 에러 코드 기반 변환 (sql-error-codes.xml)
// → SQLStateSQLExceptionTranslator (폴백)
//   → ANSI SQL State 기반 변환

// sql-error-codes.xml (Spring 내장)
// MySQL:
//   23000 → DuplicateKeyException
//   1213  → DeadlockLoserDataAccessException
//   1205  → CannotAcquireLockException

// PostgreSQL:
//   23505 → DuplicateKeyException
//   40P01 → DeadlockLoserDataAccessException

// 변환 결과 예시
try {
    jdbcTemplate.update("INSERT INTO users(email) VALUES(?)", "dup@test.com");
} catch (DuplicateKeyException e) {
    // SQLException(23000) → DuplicateKeyException (RuntimeException)
    // → @Transactional이 자동 롤백
    log.warn("중복 이메일: {}", e.getMessage());
}

// DataAccessException 계층
// DataAccessException (RuntimeException)
// ├── DataIntegrityViolationException
// │   └── DuplicateKeyException (UK 위반)
// ├── CannotAcquireLockException (락 획득 실패)
// ├── DeadlockLoserDataAccessException (데드락)
// ├── QueryTimeoutException (쿼리 타임아웃)
// └── EmptyResultDataAccessException (단건 조회 결과 없음)
```

### 4. query() — SELECT 실행 흐름

```java
// JdbcTemplate.query(String sql, RowMapper rowMapper)
public <T> List<T> query(String sql, RowMapper<T> rowMapper) {
    return result(query(sql, new RowMapperResultSetExtractor<>(rowMapper)));
    // RowMapper → RowMapperResultSetExtractor로 래핑
}

// execute()로 위임
public <T> T query(String sql, ResultSetExtractor<T> rse) {
    return query(new SimplePreparedStatementCreator(sql), rse);
}

// PreparedStatementCallback 내부
T result = action.doInPreparedStatement(ps);
// action = (ps) -> {
//     ResultSet rs = ps.executeQuery();
//     try {
//         return rse.extractData(rs);  // ResultSetExtractor 호출
//     } finally {
//         JdbcUtils.closeResultSet(rs);
//     }
// }

// RowMapperResultSetExtractor.extractData()
public List<T> extractData(ResultSet rs) throws SQLException {
    List<T> results = new ArrayList<>();
    int rowNum = 0;
    while (rs.next()) {
        results.add(this.rowMapper.mapRow(rs, rowNum++));
        // RowMapper: 한 행씩 처리
    }
    return results;
}
```

### 5. update() — INSERT/UPDATE/DELETE 실행 흐름

```java
// JdbcTemplate.update(String sql, Object... args)
public int update(String sql, @Nullable Object... args) {
    return update(sql, newArgPreparedStatementSetter(args));
}

public int update(String sql, @Nullable PreparedStatementSetter pss) {
    return update(new SimplePreparedStatementCreator(sql), pss);
}

public int update(PreparedStatementCreator psc, @Nullable PreparedStatementSetter pss) {
    return execute(psc, ps -> {
        if (pss != null) {
            pss.setValues(ps); // 파라미터 바인딩
        }
        int rows = ps.executeUpdate(); // SQL 실행
        if (logger.isTraceEnabled()) {
            logger.trace("SQL update affected " + rows + " rows");
        }
        return rows;
    });
    // execute()로 수렴 → Connection 관리 + 예외 변환 자동
}
```

---

## 💻 실험으로 확인하기

### 실험 1: JdbcTemplate과 JPA가 같은 Connection 사용 확인

```java
@Transactional
@Test
void sharedConnectionWithJpa() {
    // JPA로 저장
    User user = userRepository.save(new User("테스트"));

    // JdbcTemplate으로 조회 (flush 전이지만 같은 Connection이면 보임)
    String name = jdbcTemplate.queryForObject(
        "SELECT name FROM users WHERE id = ?",
        String.class, user.getId()
    );
    // JPA가 아직 flush 안 했으면 조회 안 됨
    // em.flush() 후에는 같은 Connection 내 미커밋 데이터 조회 가능

    em.flush(); // 강제 flush
    name = jdbcTemplate.queryForObject(
        "SELECT name FROM users WHERE id = ?",
        String.class, user.getId()
    );
    assertEquals("테스트", name); // ✅ 같은 Connection, 같은 트랜잭션
}
```

### 실험 2: 예외 변환 확인

```java
@Test
void sqlExceptionTranslation() {
    jdbcTemplate.update("INSERT INTO users(id, email) VALUES(1, 'a@b.com')");

    // 동일 PK로 다시 삽입 → SQLException(23000/duplicate)
    assertThrows(DuplicateKeyException.class, () ->
        jdbcTemplate.update("INSERT INTO users(id, email) VALUES(1, 'a@b.com')")
    );
    // DuplicateKeyException: RuntimeException → @Transactional 자동 롤백
}
```

### 실험 3: queryForObject 결과 없을 때

```java
@Test
void queryForObjectEmpty() {
    // 결과가 없으면 EmptyResultDataAccessException
    assertThrows(EmptyResultDataAccessException.class, () ->
        jdbcTemplate.queryForObject(
            "SELECT id FROM users WHERE id = ?",
            Long.class, 999999L
        )
    );
    // → Optional로 감싸거나 query() + 결과 체크로 처리
}

// 안전한 단건 조회 패턴
public Optional<User> findById(Long id) {
    List<User> results = jdbcTemplate.query(
        "SELECT id, name FROM users WHERE id = ?",
        userRowMapper, id
    );
    return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
}
```

---

## ⚡ 성능 임팩트

```
JdbcTemplate 오버헤드:
  execute() 프레임워크 코드: ~수 μs
  DataSourceUtils.getConnection(): ThreadLocal 조회 ~수백 ns
  SQLExceptionTranslator (에러 시만): ~수 μs
  → 실제 SQL 실행 대비 무시 가능 (< 1%)

순수 JDBC vs JdbcTemplate 처리량:
  1만 건 단건 INSERT:
    순수 JDBC: 기준값
    JdbcTemplate: 기준값 ±1% (오버헤드 거의 없음)
  → 성능 차이 없음, 코드 품질만 개선

applyStatementSettings() 설정 활용:
  jdbcTemplate.setFetchSize(100);     // 대량 조회 시 네트워크 왕복 감소
  jdbcTemplate.setQueryTimeout(5);    // 5초 타임아웃
  jdbcTemplate.setMaxRows(1000);      // 최대 행 수 제한
  → 이 설정들이 실제 성능에 영향
```

---

## 🤔 트레이드오프

```
JdbcTemplate vs JPA:
  JdbcTemplate:
    ✅ SQL 완전 제어 (복잡한 쿼리, DB 전용 기능)
    ✅ 영속성 컨텍스트 오버헤드 없음 (대량 조회)
    ✅ 네이티브 SQL 그대로 사용
    ❌ 결과 매핑 수동 (RowMapper 작성)
    ❌ 엔티티 상태 관리 없음 (dirty checking 없음)
    → 복잡한 리포트 쿼리, 대량 배치, JPA로 표현 어려운 SQL

  JPA (EntityManager):
    ✅ 엔티티 상태 관리 (dirty checking, cascade)
    ✅ 1차 캐시, 2차 캐시 활용
    ❌ 복잡한 쿼리 표현 한계
    → 도메인 객체 CRUD, 연관관계 탐색

  혼합 사용 (실무 권장):
    도메인 로직: JPA
    복잡한 조회/리포트: JdbcTemplate 또는 QueryDSL
    대량 배치: JdbcTemplate batchUpdate()
```

---

## 📌 핵심 정리

```
Template Method 패턴 구조
  고정: Connection 획득/반환, 예외 변환, 자원 정리
  가변: SQL, 파라미터 바인딩, 결과 매핑 (사용자 제공)
  → execute() 하나로 모든 메서드 수렴

트랜잭션 연계
  DataSourceUtils.getConnection()
  → TransactionSynchronizationManager.getResource(dataSource)
  → 현재 트랜잭션의 ConnectionHolder에서 Connection 반환
  → JPA + JdbcTemplate이 같은 Connection 공유

예외 변환
  SQLException → SQLExceptionTranslator
  → DB 에러 코드 기반 → 구체적 DataAccessException
  DuplicateKeyException, CannotAcquireLockException 등

Connection 반환
  DataSourceUtils.releaseConnection()
  → 트랜잭션 중: 실제 반환 안 함 (트랜잭션 종료 시 반환)
  → 트랜잭션 없음: Pool에 즉시 반환
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 없는 서비스 메서드에서 `JdbcTemplate.update()`를 두 번 호출하면 각각 별도의 Connection을 사용하는가? auto-commit은 어떻게 동작하는가?

**Q2.** `JdbcTemplate`의 `queryForObject(String, Class, Object...)`는 결과가 2개 이상일 때 어떤 예외를 던지는가? 이 예외의 계층 구조는?

**Q3.** `JdbcTemplate`을 사용하면서 `DataSource`를 직접 `@Autowired`로 주입받아 `dataSource.getConnection()`으로 Connection을 획득하면 어떤 문제가 생기는가?

> 💡 **해설**
>
> **Q1.** 트랜잭션이 없으면 `DataSourceUtils.getConnection()`이 매번 Pool에서 새 Connection을 획득한다. `isSynchronizationActive()`가 false이므로 ThreadLocal 바인딩도 없다. 따라서 첫 번째 `update()`와 두 번째 `update()`는 서로 다른 Connection을 사용할 수 있다. auto-commit은 HikariCP가 Pool에서 Connection을 꺼낼 때의 상태에 따르는데, 기본적으로 `autoCommit=true`이므로 각 `update()`가 독립적으로 즉시 커밋된다. 첫 번째 성공하고 두 번째가 실패해도 첫 번째는 롤백되지 않는다.
>
> **Q2.** `IncorrectResultSizeDataAccessException`을 던진다. 이 예외는 `DataAccessException → IncorrectResultSizeDataAccessException`으로, "예상한 결과 크기와 실제 결과 크기가 다름"을 나타낸다. 내부적으로 `queryForObject()`는 `RowMapperResultSetExtractor`로 모든 행을 수집 후 결과가 1개가 아니면 이 예외를 발생시킨다. 결과가 0개이면 `EmptyResultDataAccessException`(IncorrectResultSizeDataAccessException의 하위)이 발생한다.
>
> **Q3.** `dataSource.getConnection()`은 HikariCP Pool에서 완전히 새 Connection을 획득하며 `TransactionSynchronizationManager`와 무관하다. 따라서 현재 트랜잭션에 바인딩된 Connection과 다른 별도 Connection이 된다. 이 Connection으로 실행한 SQL은 현재 트랜잭션에 포함되지 않는다 — 트랜잭션이 롤백돼도 이 Connection의 작업은 롤백되지 않는다. 또한 이 Connection을 명시적으로 `close()`하지 않으면 Pool에 반환되지 않아 Connection Leak이 발생한다. 항상 `DataSourceUtils.getConnection(dataSource)`를 사용하고 `DataSourceUtils.releaseConnection(con, dataSource)`으로 반환해야 한다.

---

<div align="center">

**[다음: RowMapper vs ResultSetExtractor ➡️](./02-rowmapper-vs-resultsetextractor.md)**

</div>
