# Projection — Interface vs DTO Projection의 내부 동작 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Interface Projection이 Hibernate 프록시로 구현되는 내부 원리는?
- Closed Projection과 Open Projection의 차이는 무엇이며, 성능 차이가 발생하는 이유는?
- DTO Projection이 Interface Projection보다 성능상 유리한 조건은?
- Class-based Projection(DTO)에서 생성자 파라미터와 쿼리 컬럼이 매핑되는 원리는?
- Dynamic Projection으로 반환 타입을 런타임에 결정하는 방법은?

---

## 🔍 왜 이게 존재하는가

### 문제: 엔티티 전체를 조회하면 불필요한 데이터까지 로드된다

```java
// 문제 상황: 목록 화면에 id, name만 필요한데 전체 엔티티를 조회
@Entity
public class User {
    private Long id;
    private String name;
    private String email;
    private String password;        // 불필요
    private LocalDate birthDate;    // 불필요
    private String address;         // 불필요
    private byte[] profileImage;    // 불필요 (대용량!)
    // ... 20개 필드
}

// 전체 엔티티 조회 — SELECT 20개 컬럼 + profileImage (BLOB) 전송
List<User> users = userRepository.findAll();
// 실제 사용: users.stream().map(u -> new UserListDto(u.getId(), u.getName()))
```

```
문제:
  필요하지 않은 컬럼까지 DB에서 읽어 네트워크로 전송
  특히 BLOB/CLOB 컬럼이 있으면 성능 저하 심각
  Hibernate 1차 캐시에도 전체 엔티티가 올라감

해결:
  필요한 컬럼만 SELECT → Projection
  Interface Projection: 동적 프록시로 게터 메서드 결과 제공
  DTO Projection: 생성자로 직접 매핑 (JPQL new 연산자)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Interface Projection은 항상 SELECT를 최적화한다

```java
// ❌ Open Projection — @Value SpEL 사용 → SELECT 최적화 안 됨
public interface UserSummary {
    Long getId();
    String getName();

    @Value("#{target.firstName + ' ' + target.lastName}")  // SpEL
    String getFullName();  // ← 이 하나 때문에 전체 엔티티 로드!
}

// ✅ Closed Projection — 모든 메서드가 엔티티 프로퍼티에 직접 매핑
public interface UserSummary {
    Long getId();
    String getName();
    // SpEL 없음 → Spring Data가 SELECT id, name 으로 최적화 가능
}
```

### Before: DTO Projection은 JPQL new 연산자와 생성자 이름이 일치해야 한다

```java
// ❌ 잘못된 이해: 패키지명 없이 클래스 이름만 써도 된다
@Query("SELECT new UserDto(u.id, u.name) FROM User u")
List<UserDto> findAllDto();
// → Hibernate는 FQCN(완전한 클래스명) 필요

// ✅ 올바른 사용: 완전한 패키지 경로 포함
@Query("SELECT new com.example.dto.UserDto(u.id, u.name) FROM User u")
List<UserDto> findAllDto();

// ✅ 또는 Spring Data Interface Projection으로 대체 (경로 불필요)
public interface UserSummary {
    Long getId();
    String getName();
}
List<UserSummary> findBy();  // Query Method에서 반환 타입으로 지정
```

---

## ✨ 올바른 이해와 패턴

### After: 상황별 Projection 선택 기준

```
Closed Interface Projection:
  필요한 필드가 명확하고 소수일 때
  Spring Data가 SELECT 컬럼 최적화 적용
  JDK 동적 프록시로 구현 → 인터페이스 정의만으로 사용 가능

Open Interface Projection:
  @Value SpEL로 서버 사이드 계산이 필요할 때
  단, 전체 엔티티를 로드하므로 SELECT 최적화 없음
  SpEL 대신 default 메서드 사용 권장

DTO (Class-based) Projection:
  JSON 직렬화 등 구체적인 타입이 필요할 때
  생성자 직접 호출 → 프록시 오버헤드 없음
  JPQL new 연산자 사용 → 완전한 패키지명 필수

Dynamic Projection:
  같은 쿼리로 다양한 반환 타입을 지원해야 할 때
  제네릭 메서드로 런타임에 타입 결정
```

---

## 🔬 내부 동작 원리 — Projection 처리 소스 추적

### 1. Interface Projection — JDK 동적 프록시 생성

```java
// ProxyProjectionFactory — Interface Projection의 핵심
public class ProxyProjectionFactory implements ProjectionFactory {

    @Override
    public <T> T createProjection(Class<T> projectionType, Object source) {

        if (!projectionType.isInterface()) {
            throw new IllegalArgumentException("...");
        }

        // JDK 동적 프록시 생성
        // InvocationHandler: ProjectingMethodInterceptor
        return (T) Proxy.newProxyInstance(
            projectionType.getClassLoader(),
            new Class<?>[] { projectionType },
            new ProjectingMethodInterceptor(projectionInformation, source)
        );
    }
}

// ProjectingMethodInterceptor — 게터 메서드 호출 처리
class ProjectingMethodInterceptor implements InvocationHandler {

    private final Object source;  // 원본 데이터 (엔티티 또는 Object[])

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {

        // getId() → "id" → source에서 "id" 프로퍼티 반환
        String propertyName = getPropertyName(method);  // "getId" → "id"

        // source가 엔티티인 경우: 리플렉션으로 필드 접근
        // source가 Object[]인 경우: 인덱스로 접근
        return getPropertyValue(source, propertyName);
    }
}
```

### 2. Closed Projection — SELECT 최적화 과정

```java
// Spring Data가 Closed Projection을 감지하고 SELECT 컬럼 최적화
// ProjectionInformation.isClosed() → true 이면 최적화 적용

// UserSummary 인터페이스 분석
public interface UserSummary {
    Long getId();     // → 프로퍼티: "id"
    String getName(); // → 프로퍼티: "name"
    // @Value 없음 → Closed Projection
}

// 생성되는 JPQL (최적화됨)
// findBy() → SELECT u.id, u.name FROM User u
// 전체 SELECT * 대신 필요한 컬럼만 선택
```

```
Closed Projection 판별 기준:
  모든 게터 메서드가 @Value(SpEL) 없이
  엔티티 프로퍼티에 직접 매핑되면 → Closed
  → Spring Data가 필요한 컬럼만 SELECT

Open Projection 판별 기준:
  하나라도 @Value SpEL 또는 default 메서드가 있으면 → Open
  → 전체 엔티티 로드 후 SpEL 평가
  → SELECT 최적화 불가
```

### 3. Open Projection — SpEL 평가 과정

```java
// Open Projection 예시
public interface UserDetail {
    Long getId();
    String getFirstName();
    String getLastName();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();  // ← SpEL 사용 → Open Projection
}

// 내부 처리:
// 1. 전체 User 엔티티를 로드 (SELECT * FROM users)
// 2. getFullName() 호출 시 SpEL 평가
//    target = User 엔티티 인스턴스
//    "#{target.firstName + ' ' + target.lastName}" 평가
// 3. 결과 반환

// ✅ default 메서드로 SpEL 없이 구현 (Open이지만 명시적)
public interface UserDetail {
    Long getId();
    String getFirstName();
    String getLastName();

    default String getFullName() {
        return getFirstName() + " " + getLastName();
    }
    // default 메서드는 프록시에서 직접 호출됨
    // 단, Closed Projection이 되지는 않음 (전체 엔티티 로드)
}
```

### 4. DTO (Class-based) Projection — JPQL new 연산자

```java
// DTO 클래스 정의
public class UserSummaryDto {
    private final Long id;
    private final String name;

    // JPQL new 연산자가 이 생성자를 호출
    public UserSummaryDto(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    // getter, equals, hashCode ...
}

// JPQL new 연산자 사용
@Query("SELECT new com.example.dto.UserSummaryDto(u.id, u.name) FROM User u")
List<UserSummaryDto> findAllAsDtos();

// 내부 동작:
// 1. Hibernate가 SELECT u.id, u.name FROM users 실행
// 2. ResultSet 각 행을 UserSummaryDto(id, name) 생성자로 직접 매핑
// 3. 엔티티 생성 없음, 영속성 컨텍스트에 등록 없음
// → 조회 전용 데이터는 DTO Projection이 가장 효율적
```

### 5. Spring Data의 자동 DTO Projection (JPQL 없이)

```java
// @Query 없이 Query Method에서 반환 타입으로 Projection 지정
public interface UserRepository extends JpaRepository<User, Long> {

    // Interface Projection — Query Method
    List<UserSummary> findByStatus(Status status);
    // → Spring Data가 자동으로 SELECT id, name FROM users WHERE status=? 생성

    // DTO Projection — Query Method (Spring Data 자동 처리)
    // 단, DTO는 인터페이스가 아니므로 자동 최적화 적용 안 됨
    // @Query 또는 Interface Projection 권장
}
```

### 6. Dynamic Projection — 런타임 타입 결정

```java
// 제네릭 메서드로 다양한 Projection 지원
public interface UserRepository extends JpaRepository<User, Long> {

    <T> List<T> findByStatus(Status status, Class<T> type);
    // type에 따라 반환 타입 결정:
    // findByStatus(ACTIVE, UserSummary.class) → Interface Projection
    // findByStatus(ACTIVE, UserDto.class) → DTO Projection
    // findByStatus(ACTIVE, User.class) → 전체 엔티티

    <T> Optional<T> findById(Long id, Class<T> type);
}

// 사용 예시
List<UserSummary> summaries = userRepository.findByStatus(ACTIVE, UserSummary.class);
List<UserDto> dtos = userRepository.findByStatus(ACTIVE, UserDto.class);
```

### 7. Native Query + Interface Projection

```java
// Native Query에서 Interface Projection 사용
public interface UserOrderSummary {
    Long getUserId();
    String getUserName();
    Integer getOrderCount();
    // 컬럼 별칭: user_id → getUserId(), user_name → getUserName(), order_count → getOrderCount()
}

@Query(value = """
    SELECT u.id AS userId, u.name AS userName, COUNT(o.id) AS orderCount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.name
    """, nativeQuery = true)
List<UserOrderSummary> findUsersWithOrderCount();
// 별칭(camelCase) ↔ 메서드명 자동 매핑
```

---

## 💻 실험으로 확인하기

### 실험 1: Closed vs Open Projection SQL 비교

```java
// Closed Projection
public interface ClosedProjection {
    Long getId();
    String getName();
}

// Open Projection
public interface OpenProjection {
    Long getId();
    String getName();
    @Value("#{target.name.toUpperCase()}")
    String getUpperName();
}

@Test
void projectionSqlComparison() {
    // Closed: SELECT u1_0.id, u1_0.name FROM users u1_0
    List<ClosedProjection> closed = userRepository.findBy(ClosedProjection.class);

    // Open: SELECT u1_0.id, u1_0.name, u1_0.email, u1_0.password, ...
    //        FROM users u1_0  ← 전체 컬럼 로드!
    List<OpenProjection> open = userRepository.findBy(OpenProjection.class);
}
```

### 실험 2: Projection 타입 확인

```java
@Test
void projectionIsProxy() {
    List<UserSummary> result = userRepository.findBy(UserSummary.class);

    UserSummary first = result.get(0);
    System.out.println(first.getClass().getName());
    // com.sun.proxy.$Proxy123 (Interface Projection = JDK Proxy)

    // 접근하면 내부에서 프로퍼티 조회
    System.out.println(first.getId());   // 리플렉션으로 User.id 접근
    System.out.println(first.getName()); // 리플렉션으로 User.name 접근
}
```

### 실험 3: DTO vs Interface Projection 성능 비교

```java
@Test
void projectionPerformance() {
    int COUNT = 10_000;

    // Interface Projection: JDK Proxy 10,000개 생성
    long start = System.nanoTime();
    List<UserSummary> interfaces = userRepository.findAllInterface();
    long interfaceTime = System.nanoTime() - start;

    // DTO Projection: UserDto 생성자 직접 호출 10,000개
    start = System.nanoTime();
    List<UserDto> dtos = userRepository.findAllDto();
    long dtoTime = System.nanoTime() - start;

    System.out.printf("Interface: %dms%n", interfaceTime / 1_000_000);
    System.out.printf("DTO:       %dms%n", dtoTime / 1_000_000);
    // DTO가 약 10~30% 빠름 (Proxy 생성 오버헤드 없음)
    // 단, DB I/O가 지배적이면 차이는 미미
}
```

---

## ⚡ 성능 임팩트

```
SELECT 컬럼 수 비교 (User 엔티티 20개 컬럼 가정):

엔티티 전체 조회:
  SELECT 20개 컬럼 (profileImage BLOB 포함)
  네트워크 전송량: 20개 컬럼 × 행 수

Closed Interface Projection (id, name):
  SELECT 2개 컬럼
  네트워크 전송량: ~1/10 수준

DTO Projection (id, name):
  SELECT 2개 컬럼
  네트워크 전송량: ~1/10 수준
  + Proxy 생성 오버헤드 없음

Open Interface Projection:
  SELECT 20개 컬럼 (최적화 없음)
  + Proxy 생성 + SpEL 평가 오버헤드
  → 가장 비효율

조회 건수 10,000건, profileImage 평균 1KB 가정:
  엔티티 전체: ~10MB 전송
  Closed Projection: ~100KB 전송
  → 100배 차이 (특히 BLOB 포함 엔티티)
```

---

## 🤔 트레이드오프

```
Interface Projection 장점:
  정의 간단 (인터페이스만 선언)
  Spring Data가 SELECT 최적화 자동 적용 (Closed)
  Query Method와 자연스럽게 결합

Interface Projection 단점:
  JDK Proxy → 구체적인 타입 조작 불가 (JSON 직렬화 어색)
  Proxy 생성 비용 (대량 조회 시 누적)
  Open Projection은 SELECT 최적화 없음

DTO Projection 장점:
  구체 타입 → JSON 직렬화, 로직 추가 자유
  생성자 직접 호출 → Proxy 오버헤드 없음
  불변 객체 설계 용이 (final 필드)

DTO Projection 단점:
  JPQL에 완전한 패키지명 포함 → 리팩토링 시 문자열 수정
  Query Method와 결합 어색 (@Query 필수)
  Record 타입 사용으로 단점 일부 해소 가능

권장 패턴:
  목록 조회 API → Closed Interface Projection (간결, 최적화)
  복잡한 집계 결과 → DTO Projection (@Query + new 연산자)
  단건 상세 조회 → 엔티티 전체 (변경 감지 필요 시)
  Native Query 결과 → Interface Projection (별칭 매핑 편리)
```

---

## 📌 핵심 정리

```
Interface Projection 내부 구조
  JDK 동적 프록시 (ProjectingMethodInterceptor)
  게터 호출 → 프로퍼티명 추출 → 원본 데이터에서 값 조회

Closed vs Open 판별
  Closed: @Value SpEL 없음, 모든 메서드가 프로퍼티 직접 매핑
          → Spring Data가 SELECT 컬럼 최적화
  Open:   @Value SpEL 또는 default 메서드 포함
          → 전체 엔티티 로드, SELECT 최적화 없음

DTO Projection
  JPQL new 연산자 + 완전한 패키지명(FQCN) 필수
  생성자 직접 호출 → 프록시 없음, 영속성 컨텍스트 미등록
  조회 전용 대용량 처리에 최적

Dynamic Projection
  <T> List<T> findByXxx(Class<T> type) 패턴
  런타임에 반환 타입 결정 → 코드 재사용성 향상

선택 기준
  목록 조회: Closed Interface Projection
  집계/계산 결과: DTO Projection
  Native Query: Interface Projection (별칭 매핑)
  전체 엔티티: 변경이 필요한 경우에만
```

---

## 🤔 생각해볼 문제

**Q1.** `List<UserSummary> findByStatus(Status status)`에서 `UserSummary`가 Closed Projection이어도, 연관 엔티티(`@ManyToOne User.team`)의 필드를 `getTeam().getName()`으로 접근하면 어떤 일이 발생하는가?

**Q2.** Interface Projection으로 반환된 결과를 Jackson으로 JSON 직렬화할 때, 프록시 타입 정보가 JSON에 포함되지 않도록 하려면 어떻게 해야 하는가?

**Q3.** `<T> List<T> findByStatus(Status status, Class<T> type)` Dynamic Projection 메서드에서 `type`이 `User.class`(전체 엔티티)일 때와 `UserSummary.class`(Interface Projection)일 때 내부적으로 생성되는 JPQL은 어떻게 다른가?

> 💡 **해설**
>
> **Q1.** N+1 문제가 발생한다. Closed Projection은 명시된 프로퍼티(`id`, `name`)에 대해서만 SELECT를 최적화한다. `getTeam()`을 호출하면 `User.team`은 기본적으로 `@ManyToOne(fetch = EAGER)` 또는 Lazy이며, Projection 결과에서 `team`이 로드되지 않은 상태이다. Lazy라면 접근 시 `LazyInitializationException`이 발생할 수 있고, 루프에서 호출하면 N+1이 발생한다. 해결책은 Interface Projection에 `interface TeamInfo { String getName(); }` 중첩 인터페이스를 정의하고 `TeamInfo getTeam()`으로 선언하면 Spring Data가 JOIN을 포함한 최적화된 쿼리를 생성한다.
>
> **Q2.** Interface Projection의 프록시 타입은 런타임에 생성된 `com.sun.proxy.$Proxy123` 형태이다. Jackson은 기본적으로 인터페이스 타입을 직렬화할 수 있으므로 별도 설정 없이도 게터 메서드 기반으로 JSON이 생성된다(`{"id": 1, "name": "홍길동"}`). 프록시 타입 정보가 JSON에 포함되려면 `@JsonTypeInfo`가 있어야 하는데 Projection에는 없으므로 타입 정보는 자동으로 제외된다. 단, `@JsonIgnoreProperties(ignoreUnknown = true)` 또는 `@JsonSerialize(as = UserSummary.class)` 설정으로 인터페이스 기준으로만 직렬화되도록 명시하는 것이 안전하다.
>
> **Q3.** `User.class` 전체 엔티티: `SELECT u FROM User u WHERE u.status = ?1` (모든 필드 로드). `UserSummary.class` Closed Projection: `SELECT u.id, u.name FROM User u WHERE u.status = ?1` (선언된 프로퍼티만 선택). Spring Data는 `Class<T> type` 파라미터를 분석하여 인터페이스면 `ProjectionInformation`을 통해 Closed 여부를 판단하고, Closed이면 필요한 컬럼만 선택하는 JPQL을 동적으로 생성한다. 이 분석 결과도 캐시되므로 반복 호출 시 추가 비용은 없다.

---

<div align="center">

**[⬅️ 이전: @Query JPQL vs Native Query 처리](./03-query-annotation-jpql-native.md)** | **[다음: QueryDSL 통합 원리 ➡️](./05-querydsl-integration.md)**

</div>
