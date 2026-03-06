# NamedParameterJdbcTemplate 활용 — 이름 기반 바인딩과 SQL Injection 방지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `:name` 바인딩이 내부적으로 `?`로 치환되는 파싱 과정은?
- `MapSqlParameterSource`와 `BeanPropertySqlParameterSource`의 사용 시점 차이는?
- `JdbcTemplate`의 `?` 바인딩 대비 `NamedParameterJdbcTemplate`이 실수를 줄이는 방법은?
- IN절에 컬렉션을 바인딩할 때 내부적으로 어떤 변환이 일어나는가?
- `NamedParameterJdbcTemplate`이 `JdbcTemplate`을 래핑하는 구조는?

---

## 🔍 왜 이게 존재하는가

### 문제: 위치 기반 `?` 파라미터는 순서 실수가 잦다

```java
// ❌ JdbcTemplate — 위치 기반, 파라미터 순서 실수 위험
jdbcTemplate.update(
    "UPDATE users SET name = ?, email = ?, status = ?, updated_at = ? WHERE id = ?",
    user.getName(),
    user.getStatus(),   // ← email과 status 순서 바뀜!
    user.getEmail(),    // ← 컴파일 오류 없음, 데이터 오염
    LocalDateTime.now(),
    user.getId()
);

// 파라미터가 10개 이상이면 순서 추적이 거의 불가능

// ✅ NamedParameterJdbcTemplate — 이름 기반, 순서 무관
namedJdbc.update(
    "UPDATE users SET name = :name, email = :email, status = :status, " +
    "updated_at = :updatedAt WHERE id = :id",
    new BeanPropertySqlParameterSource(user)
    // user 객체의 필드명과 :파라미터명이 자동 매핑
);
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: NamedParameterJdbcTemplate은 JdbcTemplate과 완전히 별개다

```java
// ❌ 잘못된 이해: 별도 구현체

// ✅ 실제 구조:
// NamedParameterJdbcTemplate은 JdbcTemplate을 내부에 보유 (Wrapper 패턴)
public class NamedParameterJdbcTemplate implements NamedParameterJdbcOperations {

    private final JdbcOperations classicJdbcTemplate; // 내부 JdbcTemplate

    public NamedParameterJdbcTemplate(DataSource dataSource) {
        this.classicJdbcTemplate = new JdbcTemplate(dataSource);
    }

    public int update(String sql, SqlParameterSource paramSource) {
        // [1] :name → ? 변환 + 파라미터 순서 추출
        ParsedSql parsedSql = NamedParameterUtils.parseSqlStatement(sql);
        // [2] 변환된 ? SQL + 순서에 맞는 파라미터 배열 생성
        String sqlToUse = NamedParameterUtils.substituteNamedParameters(parsedSql, paramSource);
        Object[] params = NamedParameterUtils.buildValueArray(parsedSql, paramSource, null);
        // [3] 내부 JdbcTemplate에 위임
        return getJdbcOperations().update(sqlToUse, params);
    }
}

// → 모든 Connection 관리, 예외 변환은 JdbcTemplate에서 처리
// NamedParameterJdbcTemplate은 순수하게 파라미터 파싱/변환만 담당
```

### Before: SQL Injection은 NamedParameterJdbcTemplate을 쓰면 자동으로 방지된다

```java
// ✅ 파라미터 바인딩 방식에서는 SQL Injection 방지됨
// :name → PreparedStatement setParameter() → 이스케이프 처리

// ❌ 그러나 테이블명, 컬럼명은 바인딩 불가 → 직접 문자열 조합 필요
String orderBy = request.getParam("sort"); // "name; DROP TABLE users; --"

// 위험: 동적 ORDER BY는 바인딩이 안 됨
String sql = "SELECT * FROM users ORDER BY " + orderBy; // SQL Injection!

// ✅ 안전한 동적 ORDER BY 처리
private static final Set<String> ALLOWED_SORT_COLUMNS =
    Set.of("id", "name", "email", "created_at");

public List<User> findAll(String sortColumn) {
    if (!ALLOWED_SORT_COLUMNS.contains(sortColumn)) {
        sortColumn = "id"; // 화이트리스트 검증
    }
    return namedJdbc.query(
        "SELECT * FROM users ORDER BY " + sortColumn, // 화이트리스트 통과 후 조합
        Map.of(),
        userRowMapper
    );
}
```

---

## 🔬 내부 동작 원리 — 파싱과 치환 소스 추적

### 1. NamedParameterUtils.parseSqlStatement() — 파싱

```java
// "SELECT * FROM users WHERE name = :name AND status = :status"
// → ParsedSql 생성

public static ParsedSql parseSqlStatement(String sql) {
    ParsedSql parsedSql = new ParsedSql(sql);
    // 문자열 스캔: ':name' 패턴 탐색
    // ':' 다음에 알파벳/숫자/밑줄이 오면 파라미터로 인식
    // 문자열 리터럴 (':' in '...')은 건너뜀

    int i = 0;
    while (i < sql.length()) {
        char c = sql.charAt(i);
        if (c == ':' && i + 1 < sql.length()) {
            // 다음 문자가 알파벳이면 파라미터 시작
            int j = i + 1;
            while (j < sql.length() && isParameterSeparator(sql.charAt(j))) {
                j++;
            }
            // sql[i+1 .. j] = 파라미터명 (예: "name")
            parsedSql.addNamedParameter("name", i, j);
        }
        i++;
    }
    return parsedSql;
    // parsedSql.parameterNames = ["name", "status"]
    // parsedSql.parameterIndexes = [위치 정보]
}
```

### 2. substituteNamedParameters() — ? 치환

```java
// "SELECT * FROM users WHERE name = :name AND status = :status"
// → "SELECT * FROM users WHERE name = ? AND status = ?"

public static String substituteNamedParameters(ParsedSql parsedSql,
                                                SqlParameterSource paramSource) {
    String originalSql = parsedSql.getOriginalSql();
    StringBuilder actualSql = new StringBuilder(originalSql.length());

    // parsedSql의 파라미터 위치 정보로 :name을 ?로 치환
    for (ParameterHolder holder : parsedSql.getParameterList()) {
        // :name의 시작~끝 위치를 ?로 교체
        actualSql.append(originalSql, lastIndex, holder.getStartIndex());

        // IN절의 컬렉션 바인딩: ?를 여러 개로 확장
        if (paramSource.hasValue(holder.getParameterName())) {
            Object value = paramSource.getValue(holder.getParameterName());
            if (value instanceof Collection<?> coll) {
                // (:ids) → (?, ?, ?) — 컬렉션 크기만큼 ? 생성
                actualSql.append("(");
                for (int i = 0; i < coll.size(); i++) {
                    if (i > 0) actualSql.append(", ");
                    actualSql.append("?");
                }
                actualSql.append(")");
            } else {
                actualSql.append("?");
            }
        }
        lastIndex = holder.getEndIndex();
    }
    return actualSql.toString();
}
```

### 3. MapSqlParameterSource vs BeanPropertySqlParameterSource

```java
// MapSqlParameterSource — Map 기반, 명시적 바인딩
MapSqlParameterSource params = new MapSqlParameterSource()
    .addValue("name", "홍길동")
    .addValue("status", Status.ACTIVE)
    .addValue("minAge", 20);

namedJdbc.query(
    "SELECT * FROM users WHERE name = :name AND status = :status AND age >= :minAge",
    params, userRowMapper
);

// 또는 Map으로 간단히
namedJdbc.query(sql,
    Map.of("name", "홍길동", "status", Status.ACTIVE, "minAge", 20),
    userRowMapper
);

// BeanPropertySqlParameterSource — 객체 필드 자동 매핑
public class UserSearchDto {
    private String name;
    private Status status;
    private Integer minAge;
    // getter 필수 (BeanPropertySqlParameterSource가 리플렉션으로 접근)
}

UserSearchDto searchDto = new UserSearchDto("홍길동", Status.ACTIVE, 20);
namedJdbc.query(
    "SELECT * FROM users WHERE name = :name AND status = :status AND age >= :minAge",
    new BeanPropertySqlParameterSource(searchDto),
    userRowMapper
);
// → searchDto.getName() → :name
// → searchDto.getStatus() → :status
// → searchDto.getMinAge() → :minAge (자동 매핑)
```

### 4. IN절 컬렉션 바인딩

```java
// ✅ IN절에 컬렉션 바인딩 — NamedParameterJdbcTemplate 전용 기능
List<Long> ids = List.of(1L, 2L, 3L, 100L);

List<User> users = namedJdbc.query(
    "SELECT * FROM users WHERE id IN (:ids)",
    Map.of("ids", ids),
    userRowMapper
);
// 내부 변환:
// "WHERE id IN (:ids)" + ids.size()=4
// → "WHERE id IN (?, ?, ?, ?)"
// → setLong(1, 1L), setLong(2, 2L), setLong(3, 3L), setLong(4, 100L)

// 빈 컬렉션 주의!
List<Long> emptyIds = Collections.emptyList();
namedJdbc.query(
    "SELECT * FROM users WHERE id IN (:ids)",
    Map.of("ids", emptyIds), // IN () → SQL 오류!
    userRowMapper
);
// → SQLException: syntax error near ")"

// 안전한 빈 컬렉션 처리
public List<User> findByIds(List<Long> ids) {
    if (ids == null || ids.isEmpty()) return Collections.emptyList();
    return namedJdbc.query(
        "SELECT * FROM users WHERE id IN (:ids)",
        Map.of("ids", ids), userRowMapper
    );
}
```

### 5. SqlParameterSource 타입 등록

```java
// MapSqlParameterSource로 타입 명시 (자동 변환이 안 될 때)
MapSqlParameterSource params = new MapSqlParameterSource();
params.addValue("status", Status.ACTIVE.name(), Types.VARCHAR);
// Enum을 String으로 명시 저장

params.addValue("createdAt", LocalDateTime.now(), Types.TIMESTAMP);
// LocalDateTime → TIMESTAMP 명시

// SqlTypeValue로 커스텀 타입
params.addValue("data", new AbstractSqlTypeValue() {
    @Override
    protected Object createTypeValue(Connection conn, int sqlType, String typeName) {
        // 복잡한 타입 변환
        return ...; // ARRAY, STRUCT 등
    }
}, Types.ARRAY);
```

---

## 💻 실험으로 확인하기

### 실험 1: 파싱된 SQL 확인

```java
@Test
void parsedSqlVerification() {
    String sql = "SELECT * FROM users WHERE name = :name AND id IN (:ids)";
    ParsedSql parsed = NamedParameterUtils.parseSqlStatement(sql);

    System.out.println("파라미터 목록: " + parsed.getParameterNames());
    // [name, ids]

    String substituted = NamedParameterUtils.substituteNamedParameters(
        parsed,
        new MapSqlParameterSource()
            .addValue("name", "test")
            .addValue("ids", List.of(1L, 2L, 3L))
    );
    System.out.println("변환된 SQL: " + substituted);
    // SELECT * FROM users WHERE name = ? AND id IN (?, ?, ?)
}
```

### 실험 2: BeanPropertySqlParameterSource 매핑 확인

```java
@Test
void beanPropertyMapping() {
    record SearchCriteria(String name, Status status, List<Long> ids) {}

    SearchCriteria criteria = new SearchCriteria("홍", Status.ACTIVE, List.of(1L, 2L));
    BeanPropertySqlParameterSource source = new BeanPropertySqlParameterSource(criteria);

    // 파라미터 이름 확인
    assertTrue(source.hasValue("name"));
    assertTrue(source.hasValue("status"));
    assertTrue(source.hasValue("ids"));
    assertEquals("홍", source.getValue("name"));
}
```

---

## ⚡ 성능 임팩트

```
파싱 비용:
  NamedParameterUtils.parseSqlStatement(): ~수십 μs (첫 호출)
  캐시: ParsedSqlCache(256개 LRU) → 두 번째부터 캐시 히트
  → 실제 SQL 실행 대비 무시 가능

BeanPropertySqlParameterSource 비용:
  리플렉션으로 getter 호출 → ~수백 ns/필드
  10개 파라미터 → ~수 μs → 무시 가능
  vs MapSqlParameterSource: Map 조회 → ~수십 ns
  → 차이 실질적으로 없음

IN절 컬렉션 크기와 성능:
  IN (?, ?, ...) 개수 = 컬렉션 크기
  MySQL: IN절 1000개까지 최적화, 그 이상은 성능 저하
  PostgreSQL: 상대적으로 너그러움
  → 컬렉션 크기 > 1000이면 분할 처리 권장

  List<Long> ids = /* 10000개 */;
  // 1000개씩 분할
  Lists.partition(ids, 1000).stream()
       .flatMap(chunk -> namedJdbc.query(sql, Map.of("ids", chunk), mapper).stream())
       .toList();
```

---

## 🤔 트레이드오프

```
JdbcTemplate (?):
  ✅ 단순, 추가 파싱 없음
  ❌ 파라미터 순서 실수 위험 (3개 이상부터)
  → 파라미터 1~2개 단순 쿼리

NamedParameterJdbcTemplate (:name):
  ✅ 파라미터 순서 무관, 가독성 높음
  ✅ IN절 컬렉션 바인딩 지원
  ✅ BeanPropertySqlParameterSource로 객체 자동 매핑
  ❌ 파싱 비용 (캐시로 상쇄)
  → 파라미터 3개 이상, IN절, 복잡한 UPDATE

MapSqlParameterSource vs BeanPropertySqlParameterSource:
  Map: 명시적, 타입 지정 가능, 객체 없을 때
  Bean: 기존 DTO/도메인 객체 재사용, 코드 감소
       리플렉션 기반, getter 필수
  → 기존 객체가 있으면 Bean, 없으면 Map
```

---

## 📌 핵심 정리

```
NamedParameterJdbcTemplate 구조
  내부에 JdbcTemplate 보유 (Wrapper 패턴)
  자신은 :name → ? 변환만 담당
  Connection 관리/예외 변환은 JdbcTemplate에 위임

파싱 과정
  parseSqlStatement(): :name 위치 파악
  substituteNamedParameters(): :name → ? 치환
  buildValueArray(): 파라미터 이름 순서로 값 배열 구성
  → ParsedSqlCache(256 LRU)로 캐싱

IN절 컬렉션
  List<Long> → IN (?, ?, ?) 자동 확장
  빈 컬렉션 → SQL 오류 → 사전 체크 필수
  1000개 이상 → 분할 처리 권장

SQL Injection
  파라미터 바인딩 → PreparedStatement → 안전
  테이블명/컬럼명 동적 조합 → 화이트리스트 필수
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 파라미터를 여러 위치에서 사용하는 경우 (`WHERE id = :id OR parent_id = :id`) `NamedParameterJdbcTemplate`은 어떻게 처리하는가?

**Q2.** `BeanPropertySqlParameterSource`에 Java Record를 사용할 때 주의점은?

**Q3.** `NamedParameterJdbcTemplate`의 `ParsedSqlCache`가 256개 제한인데, 동적으로 SQL 문자열이 매번 다르게 생성되는 경우(예: 조건에 따라 다른 WHERE절) 캐시 효과가 없다. 이를 어떻게 해결하는가?

> 💡 **해설**
>
> **Q1.** `parseSqlStatement()`는 `:id`가 나타날 때마다 파라미터 목록에 추가한다. 같은 이름이 두 번 등장하면 파라미터 목록에 `["id", "id"]`가 된다. `buildValueArray()`는 파라미터 목록 순서대로 값을 구성하므로 `params.getValue("id")`를 두 번 호출해 배열에 `[idValue, idValue]`를 넣는다. 최종 SQL은 `WHERE id = ? OR parent_id = ?`이며 PreparedStatement에 같은 값이 두 번 바인딩된다. 즉 이름이 같아도 값이 자동으로 재사용되므로 별도 설정 없이 정상 동작한다.
>
> **Q2.** Java Record는 접근자 메서드 이름이 일반 클래스와 다르다. 일반 클래스는 `getName()`이지만 Record는 `name()`이다. `BeanPropertySqlParameterSource`는 내부적으로 `BeanWrapper`를 사용하며 JavaBeans 규약(getXxx 형식)을 따른다. Spring 5.3.x 이전에는 Record의 접근자를 인식하지 못해 파라미터 매핑에 실패할 수 있다. Spring 6.x부터는 Record를 올바르게 지원한다. 구버전 환경이라면 `MapSqlParameterSource`를 사용하거나, 일반 클래스(getter 보유)로 변환하는 것이 안전하다.
>
> **Q3.** 동적 SQL을 매번 새 문자열로 생성하면 `ParsedSqlCache` 미스가 계속 발생해 캐시 의미가 없다. 해결 방법은 두 가지다. 첫째, 동적 조건을 파라미터로 처리하고 SQL 구조 자체는 고정한다. 예를 들어 조건이 있을 때와 없을 때를 별도 SQL 상수로 정의해 두 개의 SQL 중 선택하는 방식으로 SQL 종류를 최소화한다. 둘째, 동적 쿼리가 복잡하다면 `NamedParameterJdbcTemplate` 대신 QueryDSL이나 Criteria API를 사용한다 — 이 경우 Hibernate QueryPlanCache가 동적 조건 조합별로 별도 플랜을 캐싱한다. `NamedParameterJdbcTemplate`의 ParsedSqlCache는 주로 정적 SQL에 효과적이다.

---

<div align="center">

**[⬅️ 이전: RowMapper vs ResultSetExtractor](./02-rowmapper-vs-resultsetextractor.md)** | **[다음: SimpleJdbcInsert로 단순 삽입 ➡️](./04-simple-jdbc-insert.md)**

</div>
