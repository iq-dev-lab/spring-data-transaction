# @DataJpaTest 범위와 제약 — 슬라이스 컨텍스트가 로딩하는 것과 하지 않는 것

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@DataJpaTest`가 로딩하는 빈과 명시적으로 제외하는 빈은 무엇인가?
- 기본 H2 인메모리 DB가 설정되는 내부 경로는?
- `@Service` 빈이 로딩되지 않는 이유와 `@Import`로 추가하는 방법은?
- `@DataJpaTest`와 `@SpringBootTest`를 선택하는 기준은?
- `TypeExcludeFilter`가 슬라이스 범위를 어떻게 결정하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: Repository 테스트에 전체 컨텍스트는 과하다

```java
// ❌ @SpringBootTest — Repository 테스트 목적으로 과도
@SpringBootTest
class UserRepositoryTest {
    @Autowired UserRepository userRepository;
    // Web MVC, Security, Kafka, Redis, 모든 @Service … 전부 로딩
    // 시작 시간: 15~30초
    // 테스트 목적: "JPQL이 올바른가?" 단 하나
}

// ✅ @DataJpaTest — JPA 슬라이스만
@DataJpaTest
class UserRepositoryTest {
    @Autowired UserRepository userRepository;
    // EntityManager, DataSource(H2), JPA Repositories만 로딩
    // 시작 시간: 1~3초
}
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @DataJpaTest는 @Component를 전부 로딩한다

```java
// ❌ 잘못된 이해: 모든 스프링 빈이 등록됨

// ✅ 실제: DataJpaTypeExcludeFilter가 JPA 관련 빈만 통과시킴

// 로딩되는 빈 ✅
// - @Entity 클래스
// - @Repository (JPA Repository 포함)
// - EntityManagerFactory / EntityManager
// - DataSource (H2 인메모리, 기본)
// - JpaTransactionManager
// - TestEntityManager
// - @Converter, @Embeddable

// 로딩되지 않는 빈 ❌
// - @Service, @Component (일반)
// - @Controller, @RestController
// - @ConfigurationProperties (일부)
// - Security, Web MVC 설정
// - Kafka, Redis 등 외부 연동
```

### Before: @Service가 필요하면 @SpringBootTest로 바꿔야 한다

```java
// ❌ @SpringBootTest로 교체 → 무거워짐

// ✅ @Import로 필요한 빈만 추가
@DataJpaTest
@Import({UserDomainService.class, PasswordEncoder.class})
class UserRepositoryWithServiceTest {

    @Autowired UserRepository userRepository;
    @Autowired UserDomainService userDomainService; // 추가됨

    @Test
    void createUserWithEncodedPassword() {
        userDomainService.createUser("홍길동", "rawPassword");
        User found = userRepository.findByEmail("hong@test.com").orElseThrow();
        assertNotEquals("rawPassword", found.getPassword()); // 인코딩됨
    }
}
```

---

## 🔬 내부 동작 원리 — 슬라이스 컨텍스트 구성 소스 추적

### 1. @DataJpaTest 메타 어노테이션 구조

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)       // ← 전체 자동설정 비활성화
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class) // ← JPA 관련만 통과
@Transactional                                     // ← 각 테스트 후 롤백
@AutoConfigureCache
@AutoConfigureDataJpa                              // ← JPA 핵심 설정
@AutoConfigureTestDatabase                         // ← H2 교체 설정
@AutoConfigureTestEntityManager                    // ← TestEntityManager 등록
@ImportAutoConfiguration                           // ← JPA 관련 AutoConfig만
public @interface DataJpaTest {
    // showSql, properties, bootstrapMode 등 설정 가능
}
```

### 2. DataJpaTypeExcludeFilter — 로딩 범위 결정자

```java
// DataJpaTypeExcludeFilter: 어떤 빈을 슬라이스에 포함할지 결정
public class DataJpaTypeExcludeFilter extends StandardAnnotationCustomizableTypeExcludeFilter<DataJpaTest> {

    // 포함 기준 (이 어노테이션이 있으면 로딩)
    private static final Set<Class<? extends Annotation>> INCLUDES = Set.of(
        Repository.class,    // @Repository
        Entity.class,        // @Entity
        Embeddable.class,    // @Embeddable
        MappedSuperclass.class,
        Converter.class
    );

    @Override
    protected Set<Class<? extends Annotation>> getComponentIncludes() {
        return INCLUDES;
    }
}

// @Service에 @Repository가 없으면 → 필터에서 제외 → 로딩 안 됨
// @Component만 있는 클래스 → 제외
// @Entity → 포함
// JpaRepository 구현체 → @Repository 상속 → 포함
```

### 3. @AutoConfigureTestDatabase — H2 교체 경로

```java
// @AutoConfigureTestDatabase → TestDatabaseAutoConfiguration 적용
// → Replace.ANY (기본): 실제 DataSource를 EmbeddedDatabase(H2)로 교체

// 내부 동작:
@Configuration
class EmbeddedDatabaseConfiguration {

    @Bean
    @Primary // 실제 DataSource보다 우선 적용
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
        // → 인메모리 H2, 테스트마다 스키마 자동 생성
        // → spring.jpa.hibernate.ddl-auto=create-drop 적용
    }
}

// application.yml의 실제 DataSource 설정은 무시됨 (Replace.ANY)
// → H2 의존성만 있으면 별도 DB 서버 없이 테스트 가능

// 실제 DB 사용하려면:
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// → H2 교체 비활성화 → application.yml DataSource 사용
```

### 4. TestEntityManager — JPA 직접 제어

```java
// TestEntityManager: EntityManager의 테스트 전용 래퍼
// flush(), clear(), persist(), find() 명시적 호출 가능

@DataJpaTest
class UserRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired UserRepository userRepository;

    @Test
    void findByEmail_hitsDatabase() {
        // 데이터 준비: persist + flush (DB에 SQL 전송) + clear (1차 캐시 초기화)
        User saved = em.persistAndFlush(new User("홍길동", "hong@test.com"));
        em.clear(); // 1차 캐시 비움 → 이후 조회는 반드시 DB 왕복

        // 실제 SELECT 실행 확인
        Optional<User> found = userRepository.findByEmail("hong@test.com");

        assertTrue(found.isPresent());
        assertEquals("홍길동", found.get().getName());
        // saved와 found는 다른 인스턴스 (em.clear() 후 새로 로딩)
        assertNotSame(saved, found.get());
    }

    @Test
    void persistFlushFind_returnsFromDb() {
        // persistFlushFind = persist → flush → clear → find 한 번에
        User user = em.persistFlushFind(new User("이순신", "lee@test.com"));
        // → DB에 저장 후 다시 SELECT로 가져온 인스턴스 반환
        assertNotNull(user.getId());
        assertEquals("이순신", user.getName());
    }
}
```

### 5. 슬라이스 컨텍스트 캐싱

```java
// Spring은 동일한 컨텍스트 설정이면 ApplicationContext를 캐시해 재사용
// @DataJpaTest만 있는 테스트 클래스들 → 같은 컨텍스트 공유

// ✅ 컨텍스트 캐시 활용 (빠름)
@DataJpaTest
class UserRepositoryTest { ... }

@DataJpaTest
class TeamRepositoryTest { ... }
// → 두 클래스가 같은 ApplicationContext 사용 (1회 초기화)

// ❌ 컨텍스트 캐시 무효화 (느려짐)
@DataJpaTest
@MockBean(SomeService.class) // 다른 설정 → 새 컨텍스트
class UserRepositoryWithMockTest { ... }

// @MockBean마다 새 컨텍스트 생성 → 많으면 전체 테스트 느려짐
// 해결: 공통 @MockBean을 추상 베이스 클래스로 묶기
@DataJpaTest
abstract class JpaTestBase {
    @MockBean SomeService someService; // 공통 Mock → 캐시 재사용
}
class UserRepositoryTest extends JpaTestBase { ... }
class TeamRepositoryTest extends JpaTestBase { ... }
```

---

## 💻 실험으로 확인하기

### 실험 1: 로딩된 빈 목록 출력

```java
@DataJpaTest
class ContextInspectionTest {

    @Autowired ApplicationContext context;

    @Test
    void printLoadedBeans() {
        String[] beans = context.getBeanDefinitionNames();
        Arrays.stream(beans)
            .filter(name -> !name.startsWith("org.springframework"))
            .sorted()
            .forEach(System.out::println);

        // 출력 예:
        // entityManagerFactory
        // jpaTransactionManager
        // testEntityManager
        // userRepository
        // teamRepository
        // user (QClass, QueryDSL 설정 시)

        // 없는 것: userService, orderService, emailService ...
    }
}
```

### 실험 2: @Import로 추가된 빈 동작 확인

```java
@DataJpaTest
@Import(UserValidator.class)
class UserValidatorWithRepoTest {

    @Autowired UserValidator userValidator; // Import로 등록됨
    @Autowired UserRepository userRepository;
    @Autowired TestEntityManager em;

    @Test
    void validatorUsesRepository() {
        em.persistAndFlush(new User("홍길동", "hong@test.com"));
        em.clear();

        // UserValidator 내부에서 UserRepository 사용
        boolean exists = userValidator.isEmailTaken("hong@test.com");
        assertTrue(exists);
    }
}
```

---

## ⚡ 성능 임팩트

```
ApplicationContext 로딩 시간 비교:

@SpringBootTest (전체):        15~30초 (빈 300~600개)
@DataJpaTest (JPA 슬라이스):    1~3초  (빈 20~60개)
→ 10배 이상 차이

H2 스키마 생성:
  ddl-auto=create-drop → 테스트 시작 시 테이블 생성, 종료 시 삭제
  @Entity 50개: 테이블 50개 생성 ~수백 ms
  → 첫 컨텍스트 로딩에 포함, 이후 테스트는 캐시로 재사용

컨텍스트 캐시 효과:
  같은 @DataJpaTest 설정 100개 테스트 클래스
  → ApplicationContext 1회만 초기화
  → 이후 99개는 캐시 재사용 (수 ms 수준)
```

---

## 🤔 트레이드오프

```
@DataJpaTest:
  ✅ 빠름 (슬라이스만 로딩)
  ✅ JPA 레이어 격리 테스트
  ✅ H2 인메모리 → 설치 불필요
  ❌ @Service 직접 주입 불가 (@Import 필요)
  ❌ H2 ↔ 실제 DB 방언 차이
  → Repository JPQL, 매핑, 연관관계 검증

@SpringBootTest:
  ✅ 전체 스택 통합 테스트
  ✅ 실제 서비스 흐름 검증
  ❌ 느림, 외부 의존성 필요
  → 서비스 + Repository 통합, HTTP API 테스트

@DataJpaTest + @Import:
  ✅ JPA 성능 + 특정 서비스 추가
  ✅ 필요한 것만 정확히 로딩
  ❌ @Import 대상이 많으면 관리 부담
  → Repository + 도메인 서비스 단위 테스트
```

---

## 📌 핵심 정리

```
@DataJpaTest 로딩 범위
  포함: @Entity, @Repository, EntityManager, DataSource(H2)
  제외: @Service, @Controller, 외부 연동 빈
  결정자: DataJpaTypeExcludeFilter

H2 자동 교체
  @AutoConfigureTestDatabase(replace=ANY) → H2 EmbeddedDatabase
  application.yml DataSource는 무시됨
  실제 DB 사용: replace=NONE + 실제 DataSource 설정

@Service 추가 방법
  @Import(UserService.class) → 특정 빈만 추가
  여러 빈: @Import({A.class, B.class})

TestEntityManager
  flush(): SQL을 DB에 전송 (트랜잭션 내)
  clear(): 1차 캐시 비움 → 이후 조회는 DB 왕복
  persistFlushFind(): persist+flush+clear+find 일괄

컨텍스트 캐시
  같은 @DataJpaTest 설정 → 컨텍스트 공유
  @MockBean 추가 → 새 컨텍스트 → 느려짐
  공통 베이스 클래스로 설정 통합 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `@DataJpaTest`에서 `@SpringBootApplication`이 붙은 메인 클래스는 컴포넌트 스캔 대상에 포함되는가?

**Q2.** `@DataJpaTest`에 `properties = {"spring.jpa.show-sql=true"}`를 설정하면 어떤 우선순위로 적용되는가?

**Q3.** `@DataJpaTest`로 `@Embeddable`이 포함된 엔티티를 테스트할 때, `@Embeddable` 클래스는 별도 `@Import` 없이 자동으로 로딩되는가?

> 💡 **해설**
>
> **Q1.** 포함되지 않는다. `@DataJpaTest`는 `@OverrideAutoConfiguration(enabled=false)`로 전체 자동 설정을 비활성화하고, `DataJpaTypeExcludeFilter`로 JPA 관련 빈만 통과시킨다. 메인 클래스의 `@SpringBootApplication`(= `@ComponentScan` + `@EnableAutoConfiguration`)이 동작하지 않으므로, 메인 클래스를 통한 전체 컴포넌트 스캔은 발생하지 않는다. 단, `@Entity`나 `@Repository`가 붙은 클래스는 TypeExcludeFilter를 통과해 스캔 대상에 포함된다.
>
> **Q2.** `properties`에 직접 설정한 값이 가장 높은 우선순위를 가진다. 적용 순서: `@DataJpaTest(properties=...)` → `@TestPropertySource` → `application-test.yml` → `application.yml`. 즉 어노테이션에 직접 지정한 프로퍼티가 파일 기반 설정을 덮어쓴다. 이를 활용해 테스트별로 특정 JPA 설정만 변경할 수 있다 (예: `spring.jpa.properties.hibernate.format_sql=true`).
>
> **Q3.** 자동으로 로딩된다. `DataJpaTypeExcludeFilter`의 포함 기준에 `@Embeddable`이 명시되어 있기 때문이다. `@Embeddable` 클래스는 `@Entity`의 일부로 동작하므로 JPA 레이어에 속하는 것으로 분류된다. 별도 `@Import` 없이 컴포넌트 스캔 경로 내의 `@Embeddable`은 자동으로 로딩된다. `@MappedSuperclass`도 동일하게 자동 포함된다.

---

<div align="center">

**[다음: @Transactional in Test의 함정 ➡️](./02-transactional-test-pitfall.md)**

</div>
