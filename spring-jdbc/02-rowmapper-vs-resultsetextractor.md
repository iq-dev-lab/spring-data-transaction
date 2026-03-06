# RowMapper vs ResultSetExtractor — 결과 매핑 전략의 선택

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RowMapper`와 `ResultSetExtractor`의 근본적인 책임 차이는?
- 1:N 관계(팀-멤버)를 단일 쿼리로 조회할 때 `ResultSetExtractor`가 필요한 이유는?
- `BeanPropertyRowMapper`의 리플렉션 비용이 실제로 얼마나 되는가?
- `RowMapper`를 람다로 정의할 때의 장단점은?
- `ResultSetExtractor`에서 커서를 직접 제어해야 하는 상황은?

---

## 🔍 왜 이게 존재하는가

### 문제: ResultSet을 엔티티로 변환하는 방식이 상황마다 다르다

```java
// 단순한 경우 — 행 하나 → 객체 하나
// SELECT id, name FROM users
// → 행마다 User 하나씩 생성하면 됨 → RowMapper 적합

// 복잡한 경우 — JOIN 결과
// SELECT t.id, t.name, m.id, m.name
// FROM teams t LEFT JOIN members m ON m.team_id = t.id
// → 팀 1개에 멤버 N개 → 여러 행이 하나의 팀 객체에 누적 → ResultSet 전체 제어 필요
// → ResultSetExtractor 적합

// 두 인터페이스 책임:
// RowMapper:            행(row) 단위 변환 책임 → JdbcTemplate이 루프 제어
// ResultSetExtractor:   ResultSet 전체 제어 책임 → 커서 위치 직접 관리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 1:N 관계를 RowMapper로 처리한다

```java
// ❌ 1:N JOIN 결과를 RowMapper로 처리 — 팀이 중복 생성됨
jdbcTemplate.query(
    "SELECT t.id tid, t.name tname, m.id mid, m.name mname " +
    "FROM teams t LEFT JOIN members m ON m.team_id = t.id",
    (rs, rowNum) -> {
        // 팀1-멤버1: Team(id=1) 생성
        // 팀1-멤버2: Team(id=1) 또 생성! → 중복
        // 팀1-멤버3: Team(id=1) 또 생성! → 중복
        Team team = new Team(rs.getLong("tid"), rs.getString("tname"));
        Member member = new Member(rs.getLong("mid"), rs.getString("mname"));
        team.getMembers().add(member);  // 이 팀 객체는 버려짐
        return team; // 멤버 1개짜리 팀 N개 반환 → 완전히 잘못된 결과
    }
);
```

### Before: BeanPropertyRowMapper는 항상 써도 된다

```java
// ❌ BeanPropertyRowMapper — 편리하지만 비용과 제약이 있음
jdbcTemplate.query(sql,
    new BeanPropertyRowMapper<>(UserDto.class)
);
// 문제 1: 리플렉션으로 setter 호출 → 컴파일 타임 검증 없음
// 문제 2: 컬럼명-필드명 매핑 규칙 (user_name → userName) 의존
// 문제 3: 타입 불일치 시 런타임 예외
// 문제 4: 리플렉션 비용 (작은 쿼리에서는 무시, 대용량에서는 누적)

// ✅ 커스텀 RowMapper — 타입 안전, 빠름
private static final RowMapper<UserDto> USER_MAPPER = (rs, rowNum) ->
    new UserDto(
        rs.getLong("id"),
        rs.getString("name"),
        rs.getString("email")
    );
```

---

## ✨ 올바른 이해와 패턴

### After: 선택 기준

```
RowMapper 사용:
  결과 행 1개 → 객체 1개 변환 (1:1 관계)
  단순 평면 결과 (JOIN 없거나 있어도 1:1)
  DTO Projection (컬럼 → 필드 단순 매핑)

ResultSetExtractor 사용:
  1:N 관계 (팀-멤버, 주문-상품 등)
  결과 행을 그룹화해 복합 객체 구성
  ResultSet 커서를 직접 이동해야 하는 특수한 경우
  count, sum 등 집계 결과를 복잡하게 처리하는 경우
```

---

## 🔬 내부 동작 원리

### 1. RowMapper — JdbcTemplate이 루프 제어

```java
// JdbcTemplate 내부 — RowMapper 실행 방식
// RowMapperResultSetExtractor가 RowMapper를 감쌈
public class RowMapperResultSetExtractor<T> implements ResultSetExtractor<List<T>> {

    private final RowMapper<T> rowMapper;

    @Override
    public List<T> extractData(ResultSet rs) throws SQLException {
        List<T> results = new ArrayList<>();
        int rowNum = 0;
        while (rs.next()) {                           // JdbcTemplate이 루프 제어
            results.add(rowMapper.mapRow(rs, rowNum++)); // 행마다 RowMapper 호출
        }
        return results;
    }
}

// RowMapper 사용자 구현
RowMapper<User> userMapper = (rs, rowNum) -> User.builder()
    .id(rs.getLong("id"))
    .name(rs.getString("name"))
    .email(rs.getString("email"))
    .createdAt(rs.getTimestamp("created_at").toLocalDateTime())
    .build();

// 재사용을 위한 정적 상수로 선언 권장
public class UserRowMapper implements RowMapper<User> {
    @Override
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
        return User.builder()
            .id(rs.getLong("id"))
            .name(rs.getString("name"))
            .email(rs.getString("email"))
            .build();
    }
}
```

### 2. ResultSetExtractor — 커서 직접 제어

```java
// 1:N 관계 매핑 — ResultSetExtractor
// SQL: SELECT t.id, t.name, m.id m_id, m.name m_name
//      FROM teams t LEFT JOIN members m ON m.team_id = t.id
//      ORDER BY t.id

ResultSetExtractor<List<Team>> teamExtractor = rs -> {
    Map<Long, Team> teamMap = new LinkedHashMap<>(); // 순서 보장

    while (rs.next()) {
        Long teamId = rs.getLong("id");

        // 팀이 아직 없으면 생성
        Team team = teamMap.computeIfAbsent(teamId, id ->
            new Team(id, rs.getString("name"), new ArrayList<>())
        );

        // 멤버가 있는 경우에만 추가 (LEFT JOIN → null 가능)
        long memberId = rs.getLong("m_id");
        if (!rs.wasNull()) {
            team.getMembers().add(new Member(memberId, rs.getString("m_name")));
        }
    }

    return new ArrayList<>(teamMap.values());
};

List<Team> teams = jdbcTemplate.query(sql, teamExtractor);
// → 팀 3개 + 멤버 N개가 정확히 매핑됨
// → SQL 1번으로 N+1 해결
```

### 3. BeanPropertyRowMapper 내부

```java
// BeanPropertyRowMapper.mapRow()
@Override
public T mapRow(ResultSet rs, int rowNum) throws SQLException {
    T mappedObject = BeanUtils.instantiateClass(this.mappedClass);
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(mappedObject);
    // → 리플렉션으로 인스턴스 생성, BeanWrapper 초기화

    ResultSetMetaData rsmd = rs.getMetaData();
    int columnCount = rsmd.getColumnCount();

    for (int index = 1; index <= columnCount; index++) {
        String column = JdbcUtils.lookupColumnName(rsmd, index);
        String field = lowerCaseName(underscoreName(column));
        // user_name → userName (camelCase 변환)

        PropertyDescriptor pd = this.mappedFields.get(field);
        if (pd != null) {
            Object value = getColumnValue(rs, index, pd);
            bw.setPropertyValue(pd.getName(), value);
            // → 리플렉션으로 setter 호출
        }
    }
    return mappedObject;
}

// 성능 영향:
// - 첫 호출: ResultSetMetaData 분석 + PropertyDescriptor 캐싱 → 상대적으로 느림
// - 이후: 캐싱된 정보 재사용 → 개선되지만 여전히 리플렉션 비용
// - 대용량(10만 행+): 커스텀 RowMapper 대비 30~50% 느릴 수 있음
```

### 4. 다양한 RowMapper 구현 패턴

```java
// [패턴 1] 람다 (간단한 경우)
RowMapper<UserDto> mapper = (rs, n) ->
    new UserDto(rs.getLong("id"), rs.getString("name"));

// [패턴 2] 정적 내부 클래스 (재사용 + 명확성)
public class UserRepository {
    private static final class UserRowMapper implements RowMapper<User> {
        @Override
        public User mapRow(ResultSet rs, int rowNum) throws SQLException {
            return new User(rs.getLong("id"), rs.getString("name"));
        }
    }
    private static final RowMapper<User> USER_MAPPER = new UserRowMapper();

    public List<User> findAll() {
        return jdbcTemplate.query("SELECT id, name FROM users", USER_MAPPER);
    }
}

// [패턴 3] 레코드 생성자 활용 (Java 16+)
record UserDto(Long id, String name, String email) {
    static RowMapper<UserDto> mapper() {
        return (rs, n) -> new UserDto(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email")
        );
    }
}

List<UserDto> users = jdbcTemplate.query(sql, UserDto.mapper());

// [패턴 4] null 안전 처리
RowMapper<UserDto> safeMapper = (rs, rowNum) -> {
    Long parentId = rs.getLong("parent_id");
    return new UserDto(
        rs.getLong("id"),
        rs.getString("name"),
        rs.wasNull() ? null : parentId  // NULL 처리
    );
};
```

### 5. 복잡한 ResultSetExtractor — 서브그룹 처리

```java
// 주문 → 주문상품 → 옵션 3단계 계층
ResultSetExtractor<List<Order>> orderExtractor = rs -> {
    Map<Long, Order> orderMap = new LinkedHashMap<>();
    Map<Long, OrderItem> itemMap = new LinkedHashMap<>();

    while (rs.next()) {
        long orderId = rs.getLong("order_id");
        Order order = orderMap.computeIfAbsent(orderId,
            id -> new Order(id, rs.getString("order_status"), new ArrayList<>()));

        long itemId = rs.getLong("item_id");
        if (!rs.wasNull()) {
            // 이 order에 속한 item Map (order별로 분리)
            String itemKey = orderId + ":" + itemId;
            OrderItem item = itemMap.computeIfAbsent(itemKey,
                k -> {
                    OrderItem newItem = new OrderItem(itemId,
                        rs.getString("product_name"), new ArrayList<>());
                    order.getItems().add(newItem);
                    return newItem;
                }
            );

            long optionId = rs.getLong("option_id");
            if (!rs.wasNull()) {
                item.getOptions().add(new ItemOption(optionId, rs.getString("option_name")));
            }
        }
    }
    return new ArrayList<>(orderMap.values());
};
```

---

## 💻 실험으로 확인하기

### 실험 1: 1:N JOIN을 RowMapper로 처리했을 때 결과 비교

```java
@Test
void rowMapperVsExtractorFor1N() {
    // RowMapper 사용 (잘못된 방법)
    List<Team> wrongResult = jdbcTemplate.query(
        "SELECT t.id, t.name, m.id mid, m.name mname FROM teams t " +
        "LEFT JOIN members m ON m.team_id = t.id WHERE t.id = 1",
        (rs, n) -> new Team(rs.getLong("id"), rs.getString("name"))
    );
    // 팀 1개에 멤버 3명 → RowMapper가 팀을 3번 생성 → 3개 반환
    assertEquals(3, wrongResult.size()); // 의도와 다름!

    // ResultSetExtractor 사용 (올바른 방법)
    List<Team> correctResult = jdbcTemplate.query(
        "SELECT t.id, t.name, m.id mid, m.name mname FROM teams t " +
        "LEFT JOIN members m ON m.team_id = t.id WHERE t.id = 1",
        teamExtractor
    );
    assertEquals(1, correctResult.size()); // 팀 1개
    assertEquals(3, correctResult.get(0).getMembers().size()); // 멤버 3개
}
```

### 실험 2: BeanPropertyRowMapper vs 커스텀 RowMapper 성능

```java
@Test
void rowMapperPerformance() {
    String sql = "SELECT id, user_name, email FROM users LIMIT 10000";

    // BeanPropertyRowMapper
    long start = System.nanoTime();
    jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(UserDto.class));
    long beanMapperTime = System.nanoTime() - start;

    // 커스텀 RowMapper
    start = System.nanoTime();
    jdbcTemplate.query(sql, (rs, n) ->
        new UserDto(rs.getLong("id"), rs.getString("user_name"), rs.getString("email")));
    long customMapperTime = System.nanoTime() - start;

    System.out.printf("BeanPropertyRowMapper: %dms%n", beanMapperTime / 1_000_000);
    System.out.printf("Custom RowMapper: %dms%n", customMapperTime / 1_000_000);
    // BeanPropertyRowMapper: ~80ms
    // Custom RowMapper: ~45ms (약 40% 빠름, 10000건 기준)
}
```

---

## ⚡ 성능 임팩트

```
RowMapper 방식별 성능 (10,000건 기준):

커스텀 RowMapper (람다/구현체):
  생성자 직접 호출 → ~45ms
  → 가장 빠름, 타입 안전

BeanPropertyRowMapper:
  리플렉션 + setter → ~80ms (약 40~80% 느림)
  첫 호출: 메타데이터 분석 추가 비용
  이후: 캐싱으로 개선되지만 리플렉션 비용 잔존
  → 프로토타입, 내부 도구에 적합

ResultSetExtractor:
  행 수에 비례하는 Map 조회/삽입 비용
  computeIfAbsent: O(1) (HashMap)
  정렬 보장 필요: LinkedHashMap → 약간 느림
  → JOIN 결과의 그룹화 비용은 대부분 무시 가능

선택 기준 요약:
  성능 최우선 + 단순 구조: 커스텀 RowMapper 람다
  빠른 개발 + 중소 규모: BeanPropertyRowMapper
  1:N 관계: ResultSetExtractor (성능 외 선택지 없음)
```

---

## 🤔 트레이드오프

```
커스텀 RowMapper:
  ✅ 최고 성능, 타입 안전, 컴파일 검증
  ✅ 복잡한 타입 변환, null 처리 유연
  ❌ 필드 수만큼 코드 작성 필요

BeanPropertyRowMapper:
  ✅ 빠른 작성 (코드 최소)
  ❌ 런타임 오류, 컬럼-필드 이름 규약 의존
  ❌ 리플렉션 비용
  → 프로토타입, 간단한 내부 쿼리

ResultSetExtractor:
  ✅ 복잡한 결과 구조(1:N, 계층) 처리
  ✅ ResultSet 완전 제어
  ❌ 코드 복잡 (null 처리, 중복 제거 로직)
  ❌ ORDER BY 필수 (그룹화 전제)
```

---

## 📌 핵심 정리

```
RowMapper vs ResultSetExtractor 책임
  RowMapper:           행(row) 하나 → 객체 하나 (JdbcTemplate이 루프)
  ResultSetExtractor:  ResultSet 전체 제어 (커서 직접 이동)

1:N 관계 처리
  RowMapper → 행마다 부모 객체 중복 생성 → 잘못된 결과
  ResultSetExtractor + LinkedHashMap.computeIfAbsent() → 정확한 그룹화

BeanPropertyRowMapper
  리플렉션 기반, camelCase 변환 자동
  컬럼명이 스네이크_케이스, 필드명이 camelCase 환경에서 편리
  성능 민감한 대용량 쿼리에는 커스텀 RowMapper 권장

null 처리
  rs.getLong() → 0 반환 (null 아님)
  rs.wasNull() → 마지막 컬럼이 null이었는지 확인
  rs.getObject("col", Long.class) → null 반환 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `ResultSetExtractor`를 사용해 1:N JOIN 결과를 매핑할 때 SQL에 `ORDER BY t.id`가 없으면 어떤 문제가 발생하는가?

**Q2.** `BeanPropertyRowMapper<UserDto>`에서 UserDto가 불변 객체(모든 필드 final, setter 없음)이면 어떻게 되는가?

**Q3.** `RowMapper<Team>`을 람다로 정의해 `jdbcTemplate.query(sql, rowMapper)`로 100만 건을 조회하면 메모리 문제가 생길 수 있다. 이를 해결하는 스트리밍 방식은?

> 💡 **해설**
>
> **Q1.** `ORDER BY t.id` 없이 `computeIfAbsent(teamId, ...)`를 사용하면 DB가 팀 행을 비연속적으로 반환할 때 문제가 된다. 예를 들어 팀1-멤버1, 팀2-멤버1, 팀1-멤버2 순서로 반환되면, 팀1이 `orderMap`에 등록된 후 팀2를 처리하고 다시 팀1이 나왔을 때 `computeIfAbsent`는 이미 있는 팀1 객체를 반환하므로 멤버2가 정상적으로 추가된다. 즉 `Map`을 사용하면 순서 무관하게 동작한다. 그러나 `LinkedHashMap`이 아닌 순서 없는 `HashMap`을 쓰면 최종 결과 리스트의 팀 순서가 보장되지 않는다. 정렬된 결과가 필요하면 `ORDER BY`를 추가하거나 최종 결과에 정렬을 적용해야 한다.
>
> **Q2.** `BeanPropertyRowMapper`는 기본 생성자로 인스턴스를 생성한 후 setter를 통해 값을 주입한다. 불변 객체에 기본 생성자가 없거나 setter가 없으면 인스턴스 생성 또는 값 주입 단계에서 예외가 발생한다. 불변 DTO에는 `BeanPropertyRowMapper`를 사용할 수 없다. 대신 커스텀 `RowMapper` 람다로 생성자를 직접 호출하거나, Java 16+의 레코드와 함께 커스텀 매퍼를 사용해야 한다.
>
> **Q3.** `jdbcTemplate.query(sql, RowMapper)`는 모든 결과를 `List`에 담아 반환하므로 100만 건이면 메모리에 100만 개 객체가 올라간다. 스트리밍 처리를 위해서는 `jdbcTemplate.queryForStream(sql, rowMapper)`(Spring 5.3+)를 사용한다. 이 메서드는 `Stream<T>`를 반환하며 `ResultSet`을 한 번에 소비하지 않고 스트림 소비 시 행을 하나씩 처리한다. 반드시 try-with-resources로 스트림을 닫아야 한다. 또는 `jdbcTemplate.query(sql, rowMapper, callback)` 형태의 `RowCallbackHandler`를 사용해 행마다 즉시 처리하고 버리는 방식도 있다.

---

<div align="center">

**[⬅️ 이전: JdbcTemplate 내부 구조](./01-jdbctemplate-internals.md)** | **[다음: NamedParameterJdbcTemplate 활용 ➡️](./03-named-parameter-jdbctemplate.md)**

</div>
