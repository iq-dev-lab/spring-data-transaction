# Repository 프록시 생성 과정 — RepositoryFactorySupport와 JDK 동적 프록시

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `interface UserRepository extends JpaRepository<User, Long>`만 선언했는데, 런타임에 어떻게 구현체가 생기는가?
- `RepositoryFactorySupport.getRepository()`는 내부적으로 무슨 일을 하는가?
- `SimpleJpaRepository`는 언제, 어떻게 등장하는가?
- `@EnableJpaRepositories`가 없으면 왜 Repository 빈이 등록되지 않는가?
- Repository 프록시 생성 비용이 애플리케이션 시작 시간에 미치는 영향은?

---

## 🔍 왜 이게 존재하는가

### 문제: 동일한 CRUD 코드를 엔티티마다 반복 작성해야 한다

Spring Data JPA 이전에는 각 엔티티마다 DAO를 직접 구현해야 했다.

```java
// 전통적인 DAO 패턴 — 엔티티마다 이 코드를 반복
@Repository
public class UserDao {
    @PersistenceContext
    private EntityManager em;

    public User findById(Long id) {
        return em.find(User.class, id);
    }

    public List<User> findAll() {
        return em.createQuery("SELECT u FROM User u", User.class).getResultList();
    }

    public User save(User user) {
        if (user.getId() == null) {
            em.persist(user);
            return user;
        }
        return em.merge(user);
    }

    public void delete(User user) {
        em.remove(em.contains(user) ? user : em.merge(user));
    }
    // ... count, exists, deleteById, findAll(Pageable) ...
}

// OrderDao, ProductDao, CategoryDao ... 모두 같은 코드 반복
```

```
반복 코드 문제:
  엔티티 10개 → DAO 10개 → findById, save, delete 등 동일 패턴 10번 반복
  → 타입만 다를 뿐 로직은 동일
  → 제네릭 + 동적 프록시로 런타임에 구현체를 생성하면 해결 가능
```

Spring Data JPA는 이 문제를 **"인터페이스 선언만으로 런타임에 구현체를 자동 생성"** 하는 방식으로 해결한다.

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: "Spring이 인터페이스를 보고 코드를 생성하는 것이다"

```java
// ❌ 잘못된 이해:
// 컴파일 타임이나 빌드 타임에 UserRepositoryImpl.java 같은 클래스가 생성된다

// ✅ 실제:
// 런타임에 JDK 동적 프록시 객체가 생성되며,
// 내부 구현은 SimpleJpaRepository 인스턴스로 위임된다

// 확인 방법
@Autowired
UserRepository userRepository;

System.out.println(userRepository.getClass().getName());
// 출력: com.sun.proxy.$Proxy87  (또는 Spring의 JdkDynamicAopProxy 래퍼)
```

### Before: "@EnableJpaRepositories 없이도 자동으로 된다"

```java
// ❌ 잘못된 이해:
// Spring Boot를 사용하면 항상 Repository가 자동 등록된다

// ✅ 실제:
// Spring Boot의 JpaRepositoriesAutoConfiguration이 @EnableJpaRepositories를 자동으로 등록
// → spring-boot-autoconfigure 의존성이 있을 때만 동작
// → 커스텀 멀티 모듈 구조에서 @EnableJpaRepositories 위치를 직접 지정해야 하는 경우가 있음
```

---

## ✨ 올바른 이해와 패턴

### After: 전체 흐름을 세 단계로 파악

```
[1단계] 스캔 — @EnableJpaRepositories가 Repository 인터페이스를 발견
[2단계] BeanDefinition 등록 — Repository 인터페이스를 팩토리 빈으로 등록
[3단계] 프록시 생성 — getBean() 시점에 JDK 동적 프록시 생성 및 SimpleJpaRepository 위임
```

---

## 🔬 내부 동작 원리 — Spring Data JPA 소스 추적

### 1. @EnableJpaRepositories → 스캔 트리거

```java
// @EnableJpaRepositories 어노테이션
@Import(JpaRepositoriesRegistrar.class)  // ← 핵심
public @interface EnableJpaRepositories {
    String[] basePackages() default {};
    // ...
}

// JpaRepositoriesRegistrar
class JpaRepositoriesRegistrar extends RepositoryBeanDefinitionRegistrarSupport {
    @Override
    protected Class<? extends Annotation> getAnnotation() {
        return EnableJpaRepositories.class;
    }

    @Override
    protected RepositoryConfigurationExtension getExtension() {
        return new JpaRepositoryConfigExtension();  // JPA 전용 확장
    }
}
```

```
Spring Boot 자동 설정 경로:
  JpaRepositoriesAutoConfiguration
    → @Import(JpaRepositoriesRegistrar.class)
    → 별도의 @EnableJpaRepositories 없이도 동작
    → spring.data.jpa.repositories.enabled=false 로 비활성화 가능
```

### 2. Repository 인터페이스 스캔 → BeanDefinition 등록

```java
// RepositoryBeanDefinitionRegistrarSupport 내부
public void registerBeanDefinitions(
        AnnotationMetadata metadata, BeanDefinitionRegistry registry) {

    RepositoryConfigurationDelegate delegate =
        new RepositoryConfigurationDelegate(configSource, resourceLoader, environment);

    // Repository 인터페이스들을 스캔하여 BeanDefinition 등록
    delegate.registerRepositoriesIn(registry, extension);
}

// 등록되는 BeanDefinition의 실제 타입:
// → JpaRepositoryFactoryBean (FactoryBean 패턴!)
// → UserRepository 자체가 아닌, UserRepository를 만들어낼 팩토리 빈이 등록됨

// BeanDefinition 내용 (개념적 표현)
BeanDefinition bd = new RootBeanDefinition(JpaRepositoryFactoryBean.class);
bd.getConstructorArgumentValues().addGenericArgumentValue(UserRepository.class);
registry.registerBeanDefinition("userRepository", bd);
```

```
핵심 포인트:
  등록되는 것: JpaRepositoryFactoryBean (FactoryBean)
  반환하는 것: UserRepository 타입의 JDK 동적 프록시
  → getBean("userRepository") → JpaRepositoryFactoryBean.getObject() 호출
```

### 3. JpaRepositoryFactoryBean → RepositoryFactorySupport

```java
// JpaRepositoryFactoryBean (FactoryBean 구현체)
public class JpaRepositoryFactoryBean<R extends JpaRepository<T, ID>, T, ID>
        extends TransactionalRepositoryFactoryBeanSupport<R, T, ID> {

    @Override
    protected RepositoryFactorySupport doCreateRepositoryFactory() {
        return new JpaRepositoryFactory(entityManager);  // ← 핵심 팩토리
    }
}

// TransactionalRepositoryFactoryBeanSupport.afterPropertiesSet()
@Override
public void afterPropertiesSet() {
    // ...
    this.factory = doCreateRepositoryFactory();  // JpaRepositoryFactory 생성

    // 핵심: 실제 Repository 프록시 생성
    T repository = this.factory.getRepository(repositoryInterface, customImplementation);
    this.repository = repository;
}
```

### 4. RepositoryFactorySupport.getRepository() — 프록시 생성 핵심

```java
// RepositoryFactorySupport (Spring Data Commons)
public <T> T getRepository(Class<T> repositoryInterface,
                            RepositoryFragments fragments) {

    // [1] Repository 메타데이터 추출
    RepositoryMetadata metadata = getRepositoryMetadata(repositoryInterface);

    // [2] 커스텀 구현 조합 (MyRepositoryImpl 등)
    RepositoryComposition composition = getRepositoryComposition(metadata, fragments);

    // [3] 핵심 구현체 생성 → SimpleJpaRepository
    Object target = getTargetRepository(information);
    // JpaRepositoryFactory.getTargetRepositoryViaReflection(SimpleJpaRepository.class, ...)

    // [4] Advice 구성 (트랜잭션, 쿼리 실행 등)
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.add(new QueryExecutorMethodInterceptor(information, ...));

    // [5] JDK 동적 프록시 생성
    ProxyFactory result = new ProxyFactory();
    result.setTarget(target);                          // SimpleJpaRepository
    result.setInterfaces(repositoryInterface, ...);    // UserRepository 인터페이스
    result.addAdvisor(ExposeInvocationInterceptor.ADVISOR);
    interceptors.forEach(result::addAdvice);

    return (T) result.getProxy(classLoader);           // JDK Proxy 반환
}
```

### 5. SimpleJpaRepository — 실제 구현체

```java
// SimpleJpaRepository — Spring Data JPA가 제공하는 기본 구현
@Repository
@Transactional(readOnly = true)  // ← 기본적으로 readOnly!
public class SimpleJpaRepository<T, ID>
        implements JpaRepositoryImplementation<T, ID> {

    private final JpaEntityInformation<T, ?> entityInformation;
    private final EntityManager em;

    @Override
    public Optional<T> findById(ID id) {
        return Optional.ofNullable(em.find(getDomainClass(), id));
    }

    @Override
    @Transactional  // ← 쓰기 작업은 readOnly 해제
    public <S extends T> S save(S entity) {
        if (entityInformation.isNew(entity)) {
            em.persist(entity);
            return entity;
        }
        return em.merge(entity);
    }

    @Override
    @Transactional
    public void deleteById(ID id) {
        findById(id).ifPresent(this::delete);
    }
    // ... 나머지 CRUD 구현
}
```

```
중요한 설계 포인트:
  SimpleJpaRepository의 @Transactional(readOnly = true) 기본값
  → findById, findAll 등 조회 메서드: readOnly=true (FlushMode 최적화)
  → save, delete 등 변경 메서드: @Transactional (readOnly 해제)
  → 개발자가 별도 @Transactional 없어도 기본 CRUD는 트랜잭션 보장
```

### 6. QueryExecutorMethodInterceptor — 쿼리 메서드 처리

```java
// QueryExecutorMethodInterceptor — findByName 같은 쿼리 메서드를 처리하는 Advice
class QueryExecutorMethodInterceptor implements MethodInterceptor {

    private final Map<Method, RepositoryQuery> queries; // 메서드 → 쿼리 매핑 (캐시)

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();

        // CRUD 메서드(findById 등)는 SimpleJpaRepository로 위임
        if (hasQueryFor(method)) {
            RepositoryQuery query = queries.get(method);
            return query.execute(invocation.getArguments());  // JPQL 실행
        }

        return invocation.proceed();  // SimpleJpaRepository 위임
    }
}
```

### 7. 전체 구조 요약 다이어그램

```
@Autowired UserRepository userRepository
       ↓
  JDK Proxy (UserRepository 인터페이스 구현)
       ↓
  QueryExecutorMethodInterceptor (Advice)
  ├── findByName() → PartTree 파싱 → JPQL 실행
  ├── @Query 메서드 → NamedQuery 실행
  └── findById(), save() 등 → 아래로 위임
       ↓
  SimpleJpaRepository (실제 CRUD 구현, target)
       ↓
  EntityManager (Hibernate Session)
       ↓
  Database
```

---

## 💻 실험으로 확인하기

### 실험 1: Repository 실제 타입 확인

```java
@SpringBootTest
class RepositoryProxyTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void repositoryIsProxy() {
        // 프록시 타입 확인
        System.out.println("타입: " + userRepository.getClass().getName());
        // 출력: com.sun.proxy.$Proxy87 (또는 Jdk...)

        // AOP 프록시 여부
        System.out.println("AOP 프록시: " + AopUtils.isAopProxy(userRepository));
        // 출력: true

        System.out.println("JDK 프록시: " + AopUtils.isJdkDynamicProxy(userRepository));
        // 출력: true  (Repository는 인터페이스 기반 → JDK Proxy)

        // 실제 구현체 확인
        Object target = ((Advised) userRepository).getTargetSource().getTarget();
        System.out.println("실제 구현체: " + target.getClass().getName());
        // 출력: org.springframework.data.jpa.repository.support.SimpleJpaRepository
    }
}
```

### 실험 2: FactoryBean 직접 접근

```java
@Autowired
ApplicationContext context;

@Test
void getFactoryBean() {
    // &를 prefix로 붙이면 FactoryBean 자체를 꺼낼 수 있음
    Object factoryBean = context.getBean("&userRepository");
    System.out.println(factoryBean.getClass().getName());
    // 출력: org.springframework.data.jpa.repository.support.JpaRepositoryFactoryBean
}
```

### 실험 3: 기본 트랜잭션 설정 확인

```java
@Test
void defaultTransactionSettings() throws Exception {
    // SimpleJpaRepository의 save 메서드에 붙은 @Transactional 확인
    Method saveMethod = SimpleJpaRepository.class.getMethod("save", Object.class);
    Transactional tx = saveMethod.getAnnotation(Transactional.class);
    System.out.println("save readOnly: " + tx.readOnly());
    // 출력: false

    // findById는 클래스 레벨 @Transactional(readOnly=true) 상속
    Transactional classTx = SimpleJpaRepository.class.getAnnotation(Transactional.class);
    System.out.println("class readOnly: " + classTx.readOnly());
    // 출력: true
}
```

---

## ⚡ 성능 임팩트

```
프록시 생성 비용 (애플리케이션 시작 시):

Repository 1개당 비용:
  - RepositoryMetadata 파싱:      ~1ms
  - Query Method 파싱 (캐시):     ~5~20ms (메서드 수에 비례)
  - JDK Proxy 생성:               ~0.5ms
  합계: 약 7~22ms per repository

Repository 50개 프로젝트 예시:
  → 시작 시간에 약 350ms~1100ms 추가
  → Spring Boot 3.x의 Lazy Initialization (@Lazy) 로 지연 가능

런타임 오버헤드 (요청 처리 시):
  - QueryExecutorMethodInterceptor: 첫 호출 후 캐시
  - 이후: 캐시된 RepositoryQuery 조회 → 무시 가능한 수준 (~수 마이크로초)
```

---

## 🤔 트레이드오프

```
JDK 동적 프록시 선택 이유:
  Repository는 인터페이스 → JDK Proxy 적용 가능
  CGLIB 서브클래스 생성보다 프록시 생성 비용 낮음
  인터페이스 기반 설계 강제 → 테스트 시 Mock 교체 용이

SimpleJpaRepository 위임 구조의 장점:
  표준 CRUD 구현이 한 곳에 집중 → 버그 수정 시 전체 적용
  @Transactional(readOnly=true) 기본값 → 성능 기본 최적화

단점 및 주의사항:
  SimpleJpaRepository의 @Transactional이 커스텀 서비스 트랜잭션과 충돌 가능
  → 서비스 레이어에서 트랜잭션 관리 시 Propagation 이해 필요
  Repository 인터페이스에 @Transactional 추가 시 의미 재검토 필요
  → 이미 SimpleJpaRepository에 설정된 경우 중복 가능
```

---

## 📌 핵심 정리

```
Repository 프록시 생성 흐름
  @EnableJpaRepositories
    → JpaRepositoriesRegistrar (ImportBeanDefinitionRegistrar)
    → JpaRepositoryFactoryBean (FactoryBean) 등록
    → getBean() 시점에 JpaRepositoryFactory.getRepository() 호출
    → JDK 동적 프록시 생성 (UserRepository 인터페이스 구현)
    → target: SimpleJpaRepository (실제 CRUD 구현)
    → Advice: QueryExecutorMethodInterceptor (쿼리 메서드 처리)

핵심 클래스 역할
  JpaRepositoryFactoryBean  — FactoryBean, 프록시 생성 트리거
  RepositoryFactorySupport  — 프록시 조립 (인터페이스 + 구현체 + Advice)
  SimpleJpaRepository       — 실제 CRUD 구현 (EntityManager 사용)
  QueryExecutorMethodInterceptor — Query Method → JPQL 실행

기본 트랜잭션 설정
  SimpleJpaRepository: @Transactional(readOnly=true) (클래스 레벨)
  save/delete 메서드:  @Transactional (readOnly 해제)
  → 개발자가 별도 설정 없어도 기본 CRUD는 트랜잭션 보장
```

---

## 🤔 생각해볼 문제

**Q1.** `UserRepository`가 `JpaRepository`가 아닌 `CrudRepository`를 상속한다면, 생성되는 프록시의 구조는 달라지는가?

**Q2.** 같은 `UserRepository`를 두 개의 다른 `EntityManager`(예: 멀티 데이터소스)로 연결하고 싶다면 어떻게 해야 하는가?

**Q3.** `SimpleJpaRepository`가 아닌 커스텀 베이스 구현체를 전체 Repository의 기본 구현으로 교체하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** 달라지지 않는다. `CrudRepository`, `PagingAndSortingRepository`, `JpaRepository` 모두 Spring Data Commons의 `Repository` 마커 인터페이스 계층에 속한다. `RepositoryFactorySupport.getRepository()`는 인터페이스 계층과 무관하게 동일한 JDK Proxy + SimpleJpaRepository 구조를 생성한다. 차이는 프록시가 구현하는 인터페이스 목록과, SimpleJpaRepository가 지원하는 메서드 범위뿐이다.
>
> **Q2.** `@EnableJpaRepositories`를 두 번 선언하되, 각각 `basePackages`와 `entityManagerFactoryRef`, `transactionManagerRef`를 다르게 지정한다. Spring Data는 `@EnableJpaRepositories(basePackages = "com.example.user", entityManagerFactoryRef = "userEntityManagerFactory")`처럼 팩토리 빈 이름을 명시할 수 있다. 각 팩토리 빈이 별도의 `JpaRepositoryFactory`를 생성하고, 각 Repository는 지정된 `EntityManager`를 사용하게 된다.
>
> **Q3.** `@EnableJpaRepositories(repositoryBaseClass = MyBaseRepository.class)`로 기본 구현체를 교체할 수 있다. `MyBaseRepository`는 `SimpleJpaRepository`를 상속하고 필요한 메서드를 오버라이딩하면 된다. `JpaRepositoryFactory.getTargetRepositoryViaReflection()`이 `repositoryBaseClass`를 참조하여 인스턴스를 생성하므로, 전체 Repository에 공통 동작(예: 논리 삭제, 감사 로그)을 한 곳에서 적용할 수 있다.

---

<div align="center">

**[다음: Query Method 파싱 메커니즘 ➡️](./02-query-method-parsing.md)**

</div>
