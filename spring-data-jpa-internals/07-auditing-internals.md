# Auditing 동작 원리 — AuditingEntityListener와 @CreatedDate의 비밀

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@CreatedDate` / `@LastModifiedDate`는 어느 시점에, 어떤 메커니즘으로 값이 채워지는가?
- `AuditingEntityListener`와 JPA `@EntityListeners`의 관계는?
- `@EnableJpaAuditing` 하나로 전체 감사가 활성화되는 설정 체인은?
- `@CreatedBy` / `@LastModifiedBy`에서 현재 사용자를 가져오는 `AuditorAware`는 어떻게 작동하는가?
- `@CreatedDate`가 이미 값이 있을 때 덮어쓰지 않는 원리는?

---

## 🔍 왜 이게 존재하는가

### 문제: 생성/수정 시각과 작성자를 매번 수동으로 채워야 한다

```java
// Auditing 이전 — 모든 저장 로직에서 수동 처리
@Service
public class UserService {

    public User createUser(UserCreateRequest request) {
        User user = new User(request.getName());
        user.setCreatedAt(LocalDateTime.now());    // 매번 직접 설정
        user.setCreatedBy(getCurrentUserId());     // 현재 사용자 매번 조회
        user.setUpdatedAt(LocalDateTime.now());
        user.setUpdatedBy(getCurrentUserId());
        return userRepository.save(user);
    }

    public User updateUser(Long id, UserUpdateRequest request) {
        User user = userRepository.findById(id).orElseThrow();
        user.update(request.getName());
        user.setUpdatedAt(LocalDateTime.now());    // 업데이트마다 직접 설정
        user.setUpdatedBy(getCurrentUserId());
        return userRepository.save(user);
    }
    // Order, Product, Comment ... 모든 엔티티에서 반복
}
```

```
반복 코드의 문제:
  모든 서비스 메서드에서 createdAt, updatedAt 설정 반복
  현재 사용자 조회 로직이 여러 곳에 산재
  누락 시 null → 데이터 무결성 손상
  테스트 시 시각 고정 어려움

Spring Data Auditing이 해결하는 것:
  @CreatedDate, @LastModifiedDate → JPA 이벤트 훅에서 자동 설정
  @CreatedBy, @LastModifiedBy → AuditorAware 빈에서 현재 사용자 자동 조회
  횡단 관심사를 한 곳에 집중
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @EnableJpaAuditing 없이 @CreatedDate만 붙이면 동작한다

```java
// ❌ 동작 안 됨 — @EnableJpaAuditing 누락
@Entity
public class User {
    @CreatedDate
    private LocalDateTime createdAt;  // 항상 null
}

// Spring Boot에서도 자동 활성화되지 않음
// @EnableJpaAuditing은 개발자가 명시적으로 추가해야 함

// ✅ 필수 설정
@Configuration
@EnableJpaAuditing  // ← 이 한 줄이 전체 Auditing 체인 활성화
public class JpaConfig { }

// + 엔티티에 @EntityListeners 추가
@Entity
@EntityListeners(AuditingEntityListener.class)  // ← Listener 등록
public class User {
    @CreatedDate
    private LocalDateTime createdAt;
}
```

### Before: @CreatedDate는 update 시에도 덮어씌워진다

```java
// ❌ 잘못된 이해: save()마다 createdAt이 현재 시각으로 갱신된다

// ✅ 실제 동작:
// @CreatedDate는 영속성 상태가 NEW(최초 저장)일 때만 설정
// 이미 값이 있으면 덮어쓰지 않음

// 내부 로직:
// AuditingHandler.markCreated() 호출 시:
//   → 필드에 이미 값이 있으면 → 건너뜀
//   → 값이 없으면 → 현재 시각 설정
```

### Before: @CreatedBy에 AuditorAware 없이 SecurityContext에서 직접 가져온다

```java
// ❌ AuditorAware 없이 @CreatedBy 사용 — 값이 채워지지 않음
@Entity
@EntityListeners(AuditingEntityListener.class)
public class User {
    @CreatedBy
    private String createdBy;  // AuditorAware 없으면 null
}

// ✅ AuditorAware 빈 등록 필수
@Bean
public AuditorAware<String> auditorAware() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                         .map(SecurityContext::getAuthentication)
                         .filter(Authentication::isAuthenticated)
                         .map(Authentication::getName);
}
```

---

## ✨ 올바른 이해와 패턴

### After: Auditing 설정 전체 구조

```java
// [1] 설정 클래스
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<Long> auditorProvider() {
        // Spring Security 연동
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(auth -> auth != null && auth.isAuthenticated()
                            && !(auth instanceof AnonymousAuthenticationToken))
            .map(auth -> ((UserPrincipal) auth.getPrincipal()).getId());
    }
}

// [2] 공통 Auditing 엔티티 (BaseEntity)
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)  // DB 레벨에서도 수정 방지
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @LastModifiedBy
    private Long updatedBy;
}

// [3] 엔티티 상속
@Entity
public class User extends BaseEntity {
    @Id @GeneratedValue
    private Long id;
    private String name;
    // createdAt, updatedAt, createdBy, updatedBy는 BaseEntity에서 상속
}
```

---

## 🔬 내부 동작 원리 — Spring Data Auditing 소스 추적

### 1. @EnableJpaAuditing → 설정 체인 활성화

```java
// @EnableJpaAuditing 어노테이션
@Import(JpaAuditingRegistrar.class)  // ← 핵심
public @interface EnableJpaAuditing {
    String auditorAwareRef() default "";
    String dateTimeProviderRef() default "";
    boolean modifyOnCreate() default true;  // 생성 시 @LastModifiedDate도 설정?
    boolean setDates() default true;
}

// JpaAuditingRegistrar — BeanDefinition 등록
class JpaAuditingRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
                                        BeanDefinitionRegistry registry) {
        // 핵심 빈 등록:
        // 1. AuditingEntityListener — JPA 이벤트 훅 처리
        // 2. AuditingHandler — 실제 감사 필드 설정 로직
        // 3. IsNewAwareAuditingHandler — 신규 엔티티 판별 포함

        registerAuditListenerBeanDefinition(
            factory.getAuditHandlerBeanDefinitionBuilder(configuration).getBeanDefinition(),
            registry
        );
    }
}
```

### 2. AuditingEntityListener — JPA 이벤트 훅

```java
// AuditingEntityListener — @EntityListeners에 등록되는 핵심 리스너
public class AuditingEntityListener {

    // AuditingHandler는 Spring이 주입 (Lazy 로딩)
    @Autowired
    private ObjectFactory<AuditingHandler> handler;

    // JPA @PrePersist 이벤트 — persist() 직전에 호출
    @PrePersist
    public void touchForCreate(Object target) {
        handler.getObject().markCreated(target);
        // → @CreatedDate, @LastModifiedDate, @CreatedBy, @LastModifiedBy 설정
    }

    // JPA @PreUpdate 이벤트 — update 직전에 호출
    @PreUpdate
    public void touchForUpdate(Object target) {
        handler.getObject().markModified(target);
        // → @LastModifiedDate, @LastModifiedBy만 설정 (@CreatedDate는 건너뜀)
    }
}
```

```
JPA 이벤트 훅 종류:
  @PrePersist   — em.persist() 호출 직전
  @PostPersist  — persist 완료 후
  @PreUpdate    — dirty checking 후 UPDATE 직전
  @PostUpdate   — UPDATE 완료 후
  @PreRemove    — em.remove() 직전
  @PostRemove   — remove 완료 후
  @PostLoad     — 엔티티가 DB에서 로드된 직후

AuditingEntityListener 사용 훅:
  @PrePersist → touchForCreate() → @CreatedDate + @LastModifiedDate
  @PreUpdate  → touchForUpdate() → @LastModifiedDate만
```

### 3. AuditingHandler — 감사 필드 설정 로직

```java
// AuditingHandler — 실제 필드 값 설정
public class AuditingHandler extends AuditingHandlerSupport {

    private final PersistentEntities entities;
    private Optional<AuditorAware<?>> auditorAware;

    public void markCreated(Object source) {
        mark(source, true);  // isNew = true
    }

    public void markModified(Object source) {
        mark(source, false); // isNew = false
    }

    private void mark(Object source, boolean isNew) {
        // 1. 엔티티의 Auditing 메타데이터 로드
        AuditableBeanWrapper<?> wrapper = factory.getBeanWrapperFor(source);
        // wrapper: 엔티티의 @CreatedDate, @LastModifiedDate 필드 정보 포함

        // 2. 날짜 설정
        if (touchDates) {
            wrapper.setCreatedDate(now());    // @CreatedDate — isNew일 때만
            wrapper.setLastModifiedDate(now()); // @LastModifiedDate — 항상
        }

        // 3. 사용자 설정
        auditorAware.ifPresent(aware -> {
            Optional<?> auditor = aware.getCurrentAuditor();
            if (isNew) {
                wrapper.setCreatedBy(auditor); // @CreatedBy — isNew일 때만
            }
            wrapper.setLastModifiedBy(auditor); // @LastModifiedBy — 항상
        });
    }
}
```

### 4. @CreatedDate 덮어쓰기 방지 — isNew 판별

```java
// ReflectionAuditingBeanWrapper — 필드 접근 및 값 설정
class ReflectionAuditingBeanWrapper<T> extends AuditableBeanWrapper<T> {

    @Override
    public void setCreatedDate(TemporalAccessor value) {
        // 핵심: 이미 값이 있으면 설정하지 않음
        Object currentValue = getField(createdDateField, target);

        if (currentValue != null) {
            return;  // ← 덮어쓰기 방지!
        }

        setField(createdDateField, target, value);
    }

    @Override
    public void setLastModifiedDate(TemporalAccessor value) {
        // LastModifiedDate는 항상 덮어씀 (조건 없음)
        setField(lastModifiedDateField, target, value);
    }
}

// 신규 엔티티 판별: IsNewAwareAuditingHandler
// EntityInformation.isNew(entity) 기준:
//   @Id 필드가 null → 신규
//   @Version 필드가 null → 신규
//   Persistable.isNew() 직접 구현 가능
```

### 5. AuditorAware — 현재 사용자 조회 타이밍

```java
// AuditorAware<T> — 현재 감사 주체를 반환하는 SPI
@FunctionalInterface
public interface AuditorAware<T> {
    Optional<T> getCurrentAuditor();
    // → @PrePersist / @PreUpdate 시점에 호출
    // → Spring Security: SecurityContextHolder에서 인증 정보 조회
    // → 비동기 처리: SecurityContext 전파 필요 (@Async + DelegatingSecurityContextAsyncTaskExecutor)
}

// 실용적인 구현 예시 — SecurityContext 연동
@Component("auditorProvider")
public class SecurityAuditorAware implements AuditorAware<Long> {

    @Override
    public Optional<Long> getCurrentAuditor() {
        Authentication authentication =
            SecurityContextHolder.getContext().getAuthentication();

        if (authentication == null
                || !authentication.isAuthenticated()
                || authentication instanceof AnonymousAuthenticationToken) {
            return Optional.empty();
            // → @CreatedBy, @LastModifiedBy 에 null 설정
        }

        return Optional.of(((UserPrincipal) authentication.getPrincipal()).getId());
    }
}
```

### 6. DateTimeProvider — 시각 제공 방법 커스터마이징

```java
// 기본: AuditingHandler가 LocalDateTime.now() 사용
// 커스터마이징: DateTimeProvider 빈 등록

// 테스트에서 시각 고정 — DateTimeProvider 활용
@TestConfiguration
public class TestAuditingConfig {

    // 고정된 시각을 반환하는 DateTimeProvider
    @Bean
    @Primary
    public DateTimeProvider fixedDateTimeProvider() {
        return () -> Optional.of(
            LocalDateTime.of(2024, 1, 1, 12, 0, 0)
        );
    }
}

// 또는 @EnableJpaAuditing에서 dateTimeProviderRef 지정
@EnableJpaAuditing(dateTimeProviderRef = "fixedDateTimeProvider")
public class JpaConfig { }
```

### 7. @MappedSuperclass 패턴 — 실무 설계

```java
// BaseTimeEntity — 시각만 필요한 경우
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;
}

// BaseEntity — 시각 + 작성자 필요한 경우
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity extends BaseTimeEntity {

    @CreatedBy
    @Column(nullable = false, updatable = false)
    private Long createdBy;

    @LastModifiedBy
    @Column(nullable = false)
    private Long updatedBy;
}

// 용도에 따라 선택
@Entity
public class Post extends BaseEntity { }      // 작성자 필요

@Entity
public class SystemLog extends BaseTimeEntity { } // 시각만 필요
```

---

## 💻 실험으로 확인하기

### 실험 1: @CreatedDate 자동 설정 확인

```java
@SpringBootTest
@Transactional
class AuditingTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void createdDateIsSetAutomatically() {
        User user = new User("홍길동");
        assertNull(user.getCreatedAt());

        User saved = userRepository.save(user);

        // @PrePersist 후 자동 설정
        assertNotNull(saved.getCreatedAt());
        assertTrue(saved.getCreatedAt().isBefore(LocalDateTime.now().plusSeconds(1)));
        System.out.println("createdAt: " + saved.getCreatedAt());
    }

    @Test
    void createdDateIsNotOverwritten() {
        User user = userRepository.save(new User("홍길동"));
        LocalDateTime originalCreatedAt = user.getCreatedAt();

        // 수정 후 저장
        user.setName("김길동");
        userRepository.save(user);

        // createdAt 변경 없음
        assertEquals(originalCreatedAt, user.getCreatedAt());
        // updatedAt은 갱신됨
        assertTrue(user.getUpdatedAt().isAfter(originalCreatedAt)
                   || user.getUpdatedAt().isEqual(originalCreatedAt));
    }
}
```

### 실험 2: AuditorAware 테스트 — MockSecurityContext

```java
@SpringBootTest
class AuditingWithSecurityTest {

    @Autowired
    UserRepository userRepository;

    @Test
    @WithMockUser(username = "testuser", roles = "USER")  // Spring Security Test
    @Transactional
    void createdByIsSetFromSecurityContext() {
        User user = userRepository.save(new User("테스트"));

        // SecurityContext의 "testuser"가 @CreatedBy에 설정됨
        assertEquals("testuser", user.getCreatedBy());
    }

    @Test
    @Transactional
    void createdByIsNullWhenNotAuthenticated() {
        // 인증 없는 상황
        User user = userRepository.save(new User("익명"));
        assertNull(user.getCreatedBy());  // AuditorAware가 empty 반환 → null
    }
}
```

### 실험 3: DateTimeProvider로 시각 고정

```java
@SpringBootTest
class AuditingWithFixedTimeTest {

    @Autowired
    UserRepository userRepository;

    @MockBean
    DateTimeProvider dateTimeProvider;

    @BeforeEach
    void setUp() {
        LocalDateTime fixedTime = LocalDateTime.of(2024, 6, 1, 12, 0, 0);
        when(dateTimeProvider.getNow()).thenReturn(Optional.of(fixedTime));
    }

    @Test
    @Transactional
    void createdAtIsFixed() {
        User user = userRepository.save(new User("테스트"));
        assertEquals(
            LocalDateTime.of(2024, 6, 1, 12, 0, 0),
            user.getCreatedAt()
        );
    }
}
```

---

## ⚡ 성능 임팩트

```
AuditingEntityListener 실행 비용:

@PrePersist / @PreUpdate 당:
  AuditingHandler.mark() 호출: ~수 μs
  AuditorAware.getCurrentAuditor() 호출: SecurityContextHolder 조회 ~수백 ns
  리플렉션으로 필드 설정: ~수 μs
  합계: ~10~50μs per persist/update
  → 무시 가능 (DB 작업이 수십 ms)

주의가 필요한 상황:
  배치 처리 (수만 건 insert) — @PrePersist가 매 건마다 호출
  AuditorAware 구현이 DB 조회를 포함하면 → N번 DB 조회 발생
  → AuditorAware 구현은 반드시 캐시(ThreadLocal, SecurityContext) 기반으로

@Column(updatable = false) 미설정 시:
  @CreatedDate 필드를 updatable=false로 선언하지 않으면
  merge() 호출 시 불필요한 UPDATE 포함 가능
  → createdAt 컬럼도 UPDATE 절에 포함 (값은 동일하지만 쿼리 포함)
```

---

## 🤔 트레이드오프

```
Spring Data Auditing 장점:
  보일러플레이트 완전 제거 (서비스 레이어 코드 단순화)
  AuditorAware 한 곳에서 사용자 조회 로직 집중
  @MappedSuperclass로 전체 엔티티 일관 적용
  DateTimeProvider로 테스트 시 시각 제어 가능

Spring Data Auditing 단점:
  @EntityListeners 누락 시 동작 안 함 (누락 오류 런타임에 발견)
  @EnableJpaAuditing과 @SpringBootTest 충돌 가능
  → @WebMvcTest 등 슬라이스 테스트에서 JPA 빈 없어도 AuditorAware 요구
  → @MockBean AuditorAware로 해결 또는 @EnableJpaAuditing 분리

@SpringBootTest + @EnableJpaAuditing 충돌 해결:
  @MockBean(JpaMetamodelMappingContext.class) → 자주 사용하는 패턴
  또는 @EnableJpaAuditing을 @Configuration으로 분리 (테스트에서 로딩 제외)

비동기 처리 주의:
  @Async 메서드에서 엔티티 저장 시 → SecurityContext 미전파
  → AuditorAware가 empty 반환 → @CreatedBy null
  → DelegatingSecurityContextAsyncTaskExecutor 설정 필요
```

---

## 📌 핵심 정리

```
Auditing 활성화 체인
  @EnableJpaAuditing
    → JpaAuditingRegistrar (ImportBeanDefinitionRegistrar)
    → AuditingEntityListener 빈 등록
    → AuditingHandler 빈 등록 (AuditorAware 주입)

이벤트 훅
  @PrePersist → touchForCreate() → @CreatedDate + @LastModifiedDate + @CreatedBy + @LastModifiedBy
  @PreUpdate  → touchForUpdate() → @LastModifiedDate + @LastModifiedBy만

@CreatedDate 덮어쓰기 방지
  ReflectionAuditingBeanWrapper.setCreatedDate()
  → 기존 값이 있으면 즉시 반환 (skip)

AuditorAware
  @PrePersist / @PreUpdate 시점에 getCurrentAuditor() 호출
  구현 시 DB 조회 금지 → SecurityContext / ThreadLocal 기반
  비동기 환경: SecurityContext 전파 설정 필요

실무 패턴
  BaseTimeEntity (@CreatedDate, @LastModifiedDate)
  BaseEntity extends BaseTimeEntity (@CreatedBy, @LastModifiedBy)
  @Column(updatable = false) → createdAt, createdBy에 필수
```

---

## 🤔 생각해볼 문제

**Q1.** `@EnableJpaAuditing(modifyOnCreate = false)`로 설정하면 엔티티 최초 저장 시 `@LastModifiedDate`는 어떻게 되는가?

**Q2.** `@Version`(낙관적 락)과 Auditing을 함께 사용할 때, `@LastModifiedDate`가 설정되면 자동으로 dirty checking이 발생해 `@Version`이 증가하는가?

**Q3.** 테스트에서 `@DataJpaTest`를 사용하고 `@EnableJpaAuditing`이 별도 `@Configuration`에 있을 때, Auditing이 동작하지 않는 이유는? 해결 방법은?

> 💡 **해설**
>
> **Q1.** `modifyOnCreate = false`이면 `@PrePersist`(최초 저장) 시 `@LastModifiedDate` 필드를 설정하지 않는다. 즉 엔티티 생성 직후에는 `createdAt`만 값이 있고 `updatedAt`은 `null`이 된다. `@PreUpdate`가 발생하는 첫 번째 수정 시점에야 `updatedAt`이 채워진다. 생성 시 `updatedAt = null`을 허용하는 스키마라면 의도적으로 사용할 수 있다. 기본값은 `modifyOnCreate = true`로, 생성 시에도 `@LastModifiedDate`를 `@CreatedDate`와 동일한 값으로 채운다.
>
> **Q2.** 직접적으로 증가하지 않는다. `@LastModifiedDate` 설정은 `@PrePersist` / `@PreUpdate` 이벤트에서 리플렉션으로 필드를 직접 수정하는 방식이다. 이 자체가 dirty checking을 트리거하는 것은 아니다. dirty checking은 Hibernate의 `flush()` 시점에 `loadedState`(스냅샷)와 현재 엔티티 상태를 비교할 때 발생한다. `@LastModifiedDate` 필드가 변경된 것이 스냅샷과 다르면 UPDATE가 발생하고, 이때 `@Version` 컬럼도 함께 UPDATE되어 버전이 증가한다. 즉 `@LastModifiedDate`가 바뀌면 Hibernate는 변경을 감지해 UPDATE를 실행하고 `@Version`도 증가한다. `@PreUpdate`에서 설정되는 것이므로 이미 UPDATE가 결정된 이후이므로 추가 UPDATE가 발생하지는 않는다.
>
> **Q3.** `@DataJpaTest`는 JPA 관련 컴포넌트만 로딩하는 슬라이스 테스트다. `@EnableJpaAuditing`이 별도 `@Configuration` 클래스에 있고, 해당 클래스에 다른 빈(예: `@Service`)이 함께 있으면 `@DataJpaTest`가 그 설정 클래스를 제외한다. 해결 방법은 세 가지다: (1) `@EnableJpaAuditing`을 단독 `@Configuration` 클래스에 분리하면 `@DataJpaTest`가 포함할 수 있다. (2) 테스트 클래스에 `@Import(JpaAuditingConfig.class)`를 추가한다. (3) `@MockBean(JpaMetamodelMappingContext.class)`를 테스트에 추가하면 Auditing 관련 빈 요구 오류를 우회할 수 있다. 가장 깔끔한 방법은 (1)번이다.

---

<div align="center">

**[⬅️ 이전: Specifications를 통한 동적 쿼리](./06-specifications-dynamic-query.md)** | **[다음: Custom Repository 구현 패턴 ➡️](./08-custom-repository-pattern.md)**

</div>
