# SimpleJdbcInsert로 단순 삽입 — DB 메타데이터 자동 감지와 키 반환

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `SimpleJdbcInsert`가 DB 메타데이터를 조회해 컬럼 목록을 자동으로 감지하는 방식은?
- `usingGeneratedKeyColumns()`로 자동 생성 키를 받는 과정은?
- `SimpleJdbcInsert`가 적합한 상황과 한계는?
- 메타데이터 조회 비용과 캐싱 전략은?
- 테이블명 충돌 시 스키마/카탈로그 지정 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: INSERT SQL 작성은 반복적이고 컬럼 추가 시 수정이 많다

```java
// ❌ 일반 JdbcTemplate INSERT — 컬럼 추가마다 SQL과 파라미터 모두 수정
jdbcTemplate.update(
    "INSERT INTO users (name, email, status, created_at) VALUES (?, ?, ?, ?)",
    user.getName(),
    user.getEmail(),
    user.getStatus().name(),
    LocalDateTime.now()
    // 컬럼 추가 → SQL 변경 + 파라미터 추가 → 순서 실수 위험
);

// ✅ SimpleJdbcInsert — 컬럼 목록을 DB 메타데이터에서 자동 감지
SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
    .withTableName("users")
    .usingGeneratedKeyColumns("id"); // 자동 생성 키 컬럼

Number newId = insert.executeAndReturnKey(
    new BeanPropertySqlParameterSource(user)
);
// → INSERT INTO users (name, email, status) VALUES (?, ?, ?)
//   (id는 자동 생성이므로 제외, created_at은 DB DEFAULT로 처리)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: SimpleJdbcInsert는 매번 메타데이터를 조회한다

```java
// ❌ 잘못된 이해: execute()마다 DB 메타데이터 조회

// ✅ 실제:
// SimpleJdbcInsert는 첫 execute() 시 메타데이터를 조회하고 내부에 캐싱
// 이후 execute()는 캐싱된 정보 사용 → 추가 DB 조회 없음

// 단, SimpleJdbcInsert 인스턴스를 매번 new로 생성하면 캐싱 무효화
// ❌ 잘못된 사용: 매번 new 생성
public void insert(User user) {
    new SimpleJdbcInsert(dataSource)          // ← 매번 새 인스턴스
        .withTableName("users")
        .execute(new BeanPropertySqlParameterSource(user));
    // → 매번 DB 메타데이터 조회 (성능 저하)
}

// ✅ 올바른 사용: 빈으로 등록하거나 필드로 재사용
@Repository
public class UserJdbcRepository {
    private final SimpleJdbcInsert userInsert;

    public UserJdbcRepository(DataSource dataSource) {
        this.userInsert = new SimpleJdbcInsert(dataSource)
            .withTableName("users")
            .usingGeneratedKeyColumns("id");
        // 생성자에서 한 번만 생성 → 이후 재사용
    }

    public Long insert(User user) {
        return userInsert.executeAndReturnKey(
            new BeanPropertySqlParameterSource(user)
        ).longValue();
    }
}
```

### Before: usingColumns()를 지정하지 않으면 모든 컬럼에 삽입한다

```java
// ❌ 잘못된 이해: 파라미터에 없는 컬럼에 null이 삽입된다

// ✅ 실제:
// usingColumns()를 지정하지 않으면 DB 메타데이터로 감지한 컬럼 중
// SqlParameterSource에 제공된 값이 있는 컬럼만 INSERT에 포함

// DB 메타데이터 컬럼: id, name, email, status, created_at, updated_at
// BeanPropertySqlParameterSource(user) → user.getName(), user.getEmail()만 있음
// → INSERT INTO users (name, email) VALUES (?, ?)
// → id (AUTO_INCREMENT), status (DEFAULT), created_at (DEFAULT CURRENT_TIMESTAMP) 제외

// 명시적으로 컬럼 지정 (권장 — 불필요한 메타데이터 조회 방지)
SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
    .withTableName("users")
    .usingColumns("name", "email", "status")   // 삽입할 컬럼 명시
    .usingGeneratedKeyColumns("id");
```

---

## 🔬 내부 동작 원리 — 메타데이터 조회 소스 추적

### 1. 첫 execute() — 메타데이터 초기화

```java
// SimpleJdbcInsert.execute() 첫 호출 시
public int execute(SqlParameterSource parameterSource) {
    // [1] 초기화 여부 확인 (첫 호출만 실행)
    checkCompiled();
    // → compiled == false → compile() 호출

    return doExecute(parameterSource);
}

private void compile() {
    if (this.tableMetaDataContext.isTableColumnMetaDataUsed()) {
        // [2] DB 메타데이터 조회
        // DataSource.getConnection() → DatabaseMetaData
        this.tableMetaDataContext.processMetaData(
            obtainDataSource(), Collections.emptyList(), new String[0]);
        // → connection.getMetaData().getColumns(catalog, schema, tableName, null)
        // → ResultSet으로 컬럼 목록, 타입, nullable 정보 수집
    }

    // [3] INSERT SQL 구성
    this.insertString = MetaDataTableMetaDataContext.createInsertString(
        this.generatedKeyNames, this.tableMetaDataContext, this.declaredColumns);
    // "INSERT INTO users (name, email, status) VALUES (?, ?, ?)"

    // [4] 컴파일 완료 마킹
    this.compiled = true;
}
```

### 2. DatabaseMetaData.getColumns() — 실제 DB 조회

```java
// 내부적으로 실행되는 메타데이터 쿼리 (DB마다 다름)
// MySQL의 경우 (JDBC 드라이버 내부):
// SELECT * FROM information_schema.COLUMNS
// WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users'

// 조회 결과로 수집하는 정보:
// COLUMN_NAME: id, name, email, status, created_at
// DATA_TYPE: BIGINT, VARCHAR, VARCHAR, VARCHAR, TIMESTAMP
// IS_NULLABLE: NO, NO, NO, YES, NO
// COLUMN_DEF: null, null, null, 'ACTIVE', CURRENT_TIMESTAMP
// IS_AUTOINCREMENT: YES (id), NO, NO, NO, NO

// SimpleJdbcInsert는 이 정보로:
// 1. usingGeneratedKeyColumns("id") → id 제외
// 2. COLUMN_DEF가 있고 파라미터에 없는 컬럼 → 제외 (DB DEFAULT 사용)
// 3. 파라미터에 있는 컬럼만 INSERT 포함
```

### 3. executeAndReturnKey() — 자동 생성 키 반환

```java
// Statement.RETURN_GENERATED_KEYS 사용
public Number executeAndReturnKey(SqlParameterSource parameterSource) {
    return getKeyHolder(doExecute(parameterSource)).getKey();
    // → KeyHolder에 생성된 키 저장
}

// 내부 JdbcTemplate 호출
// PreparedStatement ps = conn.prepareStatement(insertSql, new String[]{"id"});
// → RETURN_GENERATED_KEYS 설정
// ps.executeUpdate();
// ResultSet keys = ps.getGeneratedKeys();
// keys.next();
// keyHolder.key = keys.getLong(1); // 생성된 id

// 복합 키 반환 (다중 자동생성 컬럼)
KeyHolder keyHolder = new GeneratedKeyHolder();
userInsert.execute(paramSource, keyHolder);
Long id = Objects.requireNonNull(keyHolder.getKey()).longValue();
// 또는 특정 컬럼
Long id2 = ((Number) keyHolder.getKeys().get("id")).longValue();
```

### 4. 스키마/카탈로그 지정

```java
// 동일 테이블명이 여러 스키마에 있는 경우
SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
    .withSchemaName("app_schema")       // PostgreSQL 스키마
    .withTableName("users")
    .usingGeneratedKeyColumns("id");

// MySQL (스키마 = 데이터베이스)
SimpleJdbcInsert mysqlInsert = new SimpleJdbcInsert(dataSource)
    .withCatalogName("mydb")            // MySQL 데이터베이스명
    .withTableName("users")
    .usingGeneratedKeyColumns("id");

// usingColumns 명시 (메타데이터 조회 생략)
SimpleJdbcInsert fastInsert = new SimpleJdbcInsert(dataSource)
    .withTableName("users")
    .usingColumns("name", "email", "status")  // 메타데이터 조회 건너뜀
    .usingGeneratedKeyColumns("id");
// → 첫 execute()에도 메타데이터 조회 없음
// → usingColumns가 명시되면 메타데이터 자동 감지 비활성화
```

### 5. 실제 활용 패턴

```java
@Repository
public class ProductJdbcRepository {

    private final SimpleJdbcInsert productInsert;
    private final SimpleJdbcInsert productImageInsert;
    private final NamedParameterJdbcTemplate namedJdbc;

    public ProductJdbcRepository(DataSource dataSource, NamedParameterJdbcTemplate namedJdbc) {
        this.namedJdbc = namedJdbc;

        // 자동 생성 키 반환
        this.productInsert = new SimpleJdbcInsert(dataSource)
            .withTableName("products")
            .usingColumns("name", "price", "category_id", "status")
            .usingGeneratedKeyColumns("id");

        // 키 반환 없는 단순 삽입
        this.productImageInsert = new SimpleJdbcInsert(dataSource)
            .withTableName("product_images")
            .usingColumns("product_id", "url", "sort_order");
    }

    @Transactional
    public Long insertProduct(Product product) {
        // 상품 삽입 + 생성된 ID 반환
        Long productId = productInsert.executeAndReturnKey(
            new MapSqlParameterSource()
                .addValue("name", product.getName())
                .addValue("price", product.getPrice())
                .addValue("category_id", product.getCategoryId())
                .addValue("status", product.getStatus().name())
        ).longValue();

        // 이미지 일괄 삽입 (반환 키 없음)
        List<Map<String, Object>> imageBatch = product.getImages().stream()
            .map(img -> Map.<String, Object>of(
                "product_id", productId,
                "url", img.getUrl(),
                "sort_order", img.getSortOrder()
            ))
            .toList();

        productImageInsert.executeBatch(
            imageBatch.toArray(new Map[0])
        );

        return productId;
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 메타데이터 조회 횟수 확인

```java
@Test
void metadataFetchedOnce() {
    // DataSource 프록시로 쿼리 로그 확인
    SimpleJdbcInsert insert = new SimpleJdbcInsert(dataSource)
        .withTableName("users")
        .usingGeneratedKeyColumns("id");

    // 첫 실행 — 메타데이터 조회 발생
    insert.executeAndReturnKey(Map.of("name", "user1", "email", "a@a.com"));

    // 두 번째 실행 — 메타데이터 조회 없음
    insert.executeAndReturnKey(Map.of("name", "user2", "email", "b@b.com"));

    // 메타데이터 조회는 information_schema 또는 JDBC 메타데이터 API 1회만 발생
    assertEquals(2, userRepository.count());
}
```

### 실험 2: 생성된 키 타입 확인

```java
@Test
void generatedKeyType() {
    Number key = userInsert.executeAndReturnKey(
        Map.of("name", "홍길동", "email", "hong@test.com")
    );
    // DB에 따라 반환 타입 다름
    // MySQL: Long
    // PostgreSQL: Long
    // H2: Long

    Long id = key.longValue(); // Number.longValue()로 안전하게 변환
    assertNotNull(id);
    assertTrue(id > 0);
}
```

---

## ⚡ 성능 임팩트

```
메타데이터 조회 비용:
  첫 execute(): DB 메타데이터 조회 → ~수 ms (네트워크 왕복)
  이후 execute(): 캐싱 → 추가 비용 없음
  → SimpleJdbcInsert 인스턴스 재사용 필수

usingColumns() 명시 vs 자동 감지:
  usingColumns() 명시: 첫 호출도 메타데이터 조회 없음 → 약간 빠름
  자동 감지: 첫 호출 메타데이터 조회 → 이후 동일
  → 성능 차이보다 명시성(어떤 컬럼에 삽입하는지 코드에서 보임) 장점

SimpleJdbcInsert vs JdbcTemplate.update():
  단건 INSERT: 거의 동일
  배치 INSERT: executeBatch() → JdbcTemplate batchUpdate()와 동일 성능
  → 성능보다 코드 가독성/유지보수성이 선택 기준

적합한 사용 케이스:
  컬럼이 많은 테이블 (10개 이상)
  컬럼이 자주 추가/변경되는 경우 (SQL 수정 최소화)
  자동 생성 키를 받아야 하는 경우
```

---

## 🤔 트레이드오프

```
SimpleJdbcInsert:
  ✅ 컬럼 목록 자동 감지 → 테이블 변경 시 코드 수정 최소
  ✅ 자동 생성 키 반환 편리
  ✅ 배치 삽입 지원
  ❌ INSERT만 지원 (UPDATE, DELETE 없음)
  ❌ 복잡한 INSERT (ON DUPLICATE KEY, RETURNING 절 등) 불가
  ❌ 인스턴스 재사용 필수 (잘못 쓰면 성능 저하)
  → 단순 INSERT + 키 반환이 필요한 경우

JdbcTemplate.update():
  ✅ 모든 DML 지원
  ✅ 복잡한 SQL 표현 가능
  ❌ SQL 직접 작성 → 컬럼 추가 시 SQL 수정
  → 복잡한 INSERT, UPDATE, DELETE

NamedParameterJdbcTemplate.update():
  ✅ 이름 기반 바인딩 → 순서 실수 없음
  ✅ 모든 DML 지원
  → 파라미터 많은 복잡한 INSERT/UPDATE
```

---

## 📌 핵심 정리

```
SimpleJdbcInsert 초기화 흐름
  첫 execute() → checkCompiled() → compile()
  → DatabaseMetaData.getColumns() → 컬럼 목록 캐싱
  → INSERT SQL 구성 → compiled = true
  이후: 캐싱된 SQL 재사용

자동 생성 키 반환
  usingGeneratedKeyColumns("id")
  → PreparedStatement.RETURN_GENERATED_KEYS 설정
  → executeAndReturnKey() → GeneratedKeyHolder.getKey()

메타데이터 조회 생략
  usingColumns("name", "email") 명시 → 자동 감지 비활성화
  → 첫 호출부터 메타데이터 조회 없음

인스턴스 재사용
  SimpleJdbcInsert는 반드시 필드/빈으로 재사용
  매번 new → 메타데이터 매번 조회 → 성능 저하
```

---

## 🤔 생각해볼 문제

**Q1.** `SimpleJdbcInsert`를 사용해 `@Transactional` 내에서 INSERT한 후, 같은 트랜잭션 내 `JdbcTemplate.queryForObject()`로 방금 삽입한 행을 조회하면 볼 수 있는가?

**Q2.** `executeBatch(Map<String, ?>[] batch)`로 1000건을 삽입할 때, 내부적으로 PreparedStatement 배치가 사용되는가? 아니면 개별 INSERT가 1000번 실행되는가?

**Q3.** `usingGeneratedKeyColumns("id")`를 설정했을 때 PostgreSQL의 `SERIAL` 타입과 MySQL의 `AUTO_INCREMENT`가 모두 동작하는가? 내부적으로 어떤 차이가 있는가?

> 💡 **해설**
>
> **Q1.** 볼 수 있다. `SimpleJdbcInsert`는 내부적으로 `JdbcTemplate`을 사용하고, `JdbcTemplate`은 `DataSourceUtils.getConnection()`으로 현재 트랜잭션의 Connection을 가져온다. 따라서 `SimpleJdbcInsert`의 INSERT와 `JdbcTemplate.queryForObject()`의 SELECT는 같은 Connection, 같은 트랜잭션에서 실행된다. 같은 트랜잭션 내에서는 커밋 전이라도 자신의 변경사항을 읽을 수 있다(RDBMS의 기본 동작). 단, `em.flush()`가 먼저 실행되지 않은 JPA 엔티티는 DB에 반영되지 않을 수 있으므로, JPA와 혼용 시 flush 순서를 주의해야 한다.
>
> **Q2.** `SimpleJdbcInsert.executeBatch()`는 내부적으로 `JdbcTemplate.batchUpdate()`를 사용한다. `batchUpdate()`는 `PreparedStatement.addBatch()` + `executeBatch()`를 사용하므로 JDBC 수준의 배치 처리가 된다. 즉 1000건이 개별 `executeUpdate()` 1000번이 아니라 하나의 `executeBatch()`로 묶여 DB로 전송된다. 단, DB와 JDBC 드라이버가 배치를 지원해야 하며(대부분 지원), `rewriteBatchedStatements=true`(MySQL) 같은 추가 설정으로 성능을 더 높일 수 있다.
>
> **Q3.** 둘 다 동작한다. JDBC 표준의 `Statement.RETURN_GENERATED_KEYS` 또는 `connection.prepareStatement(sql, new String[]{"id"})`는 DB에 무관하게 동작하도록 설계되어 있다. MySQL `AUTO_INCREMENT`는 INSERT 후 `LAST_INSERT_ID()`를 통해 키를 반환하고, PostgreSQL `SERIAL`(내부적으로 Sequence)은 `RETURNING id` 또는 `getGeneratedKeys()`를 통해 반환한다. JDBC 드라이버가 각 DB에 맞게 구현을 담당한다. 단 PostgreSQL에서 `RETURNING`을 명시적으로 쓰려면 네이티브 쿼리를 사용해야 하며, `SimpleJdbcInsert`는 `getGeneratedKeys()` 방식을 사용한다.

---

<div align="center">

**[⬅️ 이전: NamedParameterJdbcTemplate 활용](./03-named-parameter-jdbctemplate.md)** | **[다음: Batch 처리 최적화 ➡️](./05-batch-processing.md)**

</div>
