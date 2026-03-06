# Batch Insert/Update 최적화 — IDENTITY 전략의 한계와 SEQUENCE 조합

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `hibernate.jdbc.batch_size`만 설정해도 Batch INSERT가 안 되는 이유는?
- `IDENTITY` 전략이 Batch INSERT를 원천적으로 막는 이유는?
- `SEQUENCE` 전략과 `allocationSize` 조합으로 Batch INSERT가 가능한 이유는?
- `saveAll()`이 실제로 몇 번의 SQL을 실행하는지 측정하는 방법은?
- JPA Batch INSERT와 `JdbcTemplate.batchUpdate()`를 언제 선택해야 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 대량 INSERT 시 건별 쿼리로 성능이 극히 저하된다

```java
// 문제 상황: 10,000개 엔티티 저장
List<Product> products = generateProducts(10000);
productRepository.saveAll(products);

// 기본 동작 (Batch 없음):
// INSERT INTO products (name, price) VALUES (?, ?) -- 1번
// INSERT INTO products (name, price) VALUES (?, ?) -- 2번
// INSERT INTO products (name, price) VALUES (?, ?) -- 3번
// ... 10,000번 반복
// → 10,000 × DB 왕복 = 수십 초

// Batch INSERT 동작:
// INSERT INTO products (name, price) VALUES (?, ?),(?, ?),(?, ?),...
// 또는 PreparedStatement.addBatch() + executeBatch()
// → 1회 DB 왕복 (또는 수십 회) = 수백 ms
```

```
Batch 처리의 핵심:
  여러 SQL을 하나의 DB 왕복으로 처리
  JDBC: PreparedStatement.addBatch() + executeBatch()
  → DB 왕복 횟수 감소 → 레이턴시 × N → 레이턴시 × (N/batch_size)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: batch_size 설정만 하면 Batch INSERT가 동작한다

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100  # 이것만으로 충분하다고 생각
```

```java
// @GeneratedValue(strategy = GenerationType.IDENTITY) 사용 중이면
// batch_size 설정에도 불구하고 Batch INSERT 안 됨!

@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 문제!
    private Long id;
    private String name;
}

// 이유:
// IDENTITY 전략: INSERT 실행 후 DB가 생성한 ID를 즉시 반환해야 함
// Batch: 여러 INSERT를 묶어서 나중에 실행
// → 충돌: Batch INSERT 중에는 ID를 즉시 알 수 없음
// → Hibernate: IDENTITY 전략이면 Batch 자동 비활성화!
```

### Before: saveAll()은 항상 Batch INSERT를 수행한다

```java
// ❌ 잘못된 이해: saveAll() = Batch INSERT

// ✅ 실제 saveAll() 동작:
// SimpleJpaRepository.saveAll():
public <S extends T> List<S> saveAll(Iterable<S> entities) {
    List<S> result = new ArrayList<>();
    for (S entity : entities) {
        result.add(save(entity));  // 각 엔티티를 개별 save() 호출
    }
    return result;
}
// → save() = isNew() ? persist() : merge()
// → IDENTITY 전략 + persist() → 즉시 INSERT (Batch 없음)
// → SEQUENCE 전략 + persist() → ID 할당 후 Batch 가능

// SEQUENCE + batch_size 설정 시:
// Hibernate가 여러 persist()를 ActionQueue에 쌓음
// flush() 시 Batch로 실행 → Batch INSERT 동작!
```

---

## ✨ 올바른 이해와 패턴

### After: Batch INSERT 동작 조건

```
Batch INSERT가 동작하는 조건 (모두 충족 필요):
  [1] hibernate.jdbc.batch_size > 0 설정
  [2] @GeneratedValue(strategy = SEQUENCE 또는 TABLE)
      ← IDENTITY는 Batch 불가
  [3] 동일 엔티티 타입의 연속 INSERT
      (다른 엔티티 섞이면 Batch 끊김)
  [4] @Version 없거나 hibernate.jdbc.batch_versioned_data=true

Batch UPDATE 동작 조건:
  [1] hibernate.jdbc.batch_size > 0
  [2] IDENTITY도 Batch UPDATE는 가능 (ID 이미 알고 있으므로)
  [3] 동일 엔티티 타입 연속 UPDATE

IDENTITY 사용 시 대안:
  JdbcTemplate.batchUpdate() (순수 JDBC)
  Spring Batch (대용량 배치 처리)
  @Id 직접 할당 (UUID, 비즈니스 키)
```

---

## 🔬 내부 동작 원리 — Batch 처리 소스 추적

### 1. IDENTITY 전략이 Batch를 막는 이유

```java
// AbstractEntityPersister.isInsertCallable()
// Hibernate: IDENTITY 전략이면 Batch 비활성화

// IdentifierGeneratorHelper (MySQL auto_increment):
// INSERT 실행 → Statement.getGeneratedKeys() → ID 반환
// → Batch 실행 시 마지막 ID만 알 수 있음 (개별 ID 불가)
// → Hibernate는 persist() 즉시 INSERT를 실행해야 ID를 얻을 수 있음
// → ActionQueue.insertions에 쌓지 않고 즉시 실행

// BatchingEntityLoaderBuilder의 판단:
protected boolean isBatchingEnabled(EntityPersister persister) {
    // IDENTITY 전략이면 항상 false 반환
    return !(persister.getIdentifierGenerator() instanceof IdentityGenerator);
}

// MySQL에서 IDENTITY를 사용하면서 Batch를 얻는 방법:
// → JdbcTemplate.batchUpdate() 사용
// → MySQL의 executeBatch()는 IDENTITY와 함께 동작 가능 (JDBC 레벨)
// → JPA Batch는 아니지만 DB 왕복은 최소화
```

### 2. SEQUENCE 전략 + allocationSize — Batch 활성화

```java
// SEQUENCE 전략 설정
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(
        name = "product_seq",
        sequenceName = "product_sequence",  // DB의 시퀀스 이름
        allocationSize = 50  // 핵심: 50개씩 ID를 미리 할당
    )
    private Long id;
}

// DB 시퀀스 생성:
// CREATE SEQUENCE product_sequence INCREMENT BY 50 START WITH 1;

// 동작 원리:
// [1] 첫 번째 persist() 시:
//     SELECT NEXTVAL('product_sequence') → 1 반환 (DB 1회 호출)
//     Hibernate: "ID 1~50 확보" (allocationSize=50)
//     → id=1 할당 (DB 왕복 없이)
// [2] 두 번째 persist():
//     id=2 할당 (메모리에서, DB 왕복 없음)
// [3] ...50번째까지 메모리에서 ID 할당
// [4] 51번째 persist():
//     SELECT NEXTVAL('product_sequence') → 51 반환 (다시 DB 1회)
//     id=51~100 확보

// Batch INSERT 실행:
// 100개 persist() → ActionQueue에 쌓임
// flush() 시:
// → PreparedStatement.addBatch() × 100
// → executeBatch() → DB로 한 번에 전송
```

### 3. hibernate.jdbc.batch_size 동작

```java
// BatchingBatch — JDBC Batch 실행 관리
public class BatchingBatch extends AbstractBatchImpl {

    private int batchSize;  // hibernate.jdbc.batch_size 값

    @Override
    public void addToBatch(BatchKey key, ...) {
        // 동일 키(같은 엔티티 타입 + 같은 SQL)이면 addBatch()
        currentBatch.addBatch();
        currentBatchStatementCount++;

        // batch_size에 도달하면 중간 실행
        if (currentBatchStatementCount >= batchSize) {
            performExecution(); // executeBatch() 실행
            currentBatchStatementCount = 0;
        }
    }

    private void performExecution() {
        // PreparedStatement.executeBatch() 호출
        // → 쌓인 SQL을 DB로 한 번에 전송
        for (Map.Entry<String, PreparedStatement> entry : statements.entrySet()) {
            PreparedStatement statement = entry.getValue();
            statement.executeBatch();
        }
    }
}
```

### 4. Batch INSERT 전체 코드 패턴

```java
// 패턴 1: SEQUENCE + batch_size (JPA Batch)
@Configuration
public class JpaConfig {
    // application.yml로도 설정 가능
}
```

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 100
          order_inserts: true   # 같은 엔티티 INSERT끼리 묶음 (순서 최적화)
          order_updates: true   # 같은 엔티티 UPDATE끼리 묶음
```

```java
// SEQUENCE 전략 엔티티
@Entity
@SequenceGenerator(name = "seq", sequenceName = "product_seq", allocationSize = 100)
public class Product {
    @Id @GeneratedValue(strategy = SEQUENCE, generator = "seq")
    private Long id;
    private String name;
    private int price;
}

// Batch INSERT 서비스
@Service
@RequiredArgsConstructor
public class ProductBatchService {

    private final EntityManager em;

    @Transactional
    public void saveProducts(List<ProductDto> dtos) {
        int batchSize = 100;

        for (int i = 0; i < dtos.size(); i++) {
            Product p = new Product(dtos.get(i));
            em.persist(p);  // ActionQueue에 쌓임

            // batchSize마다 flush + clear
            if (i % batchSize == batchSize - 1) {
                em.flush();   // Batch INSERT 실행
                em.clear();   // 1차 캐시 비움 (메모리 관리)
            }
        }
        // 마지막 남은 것 flush
        em.flush();
        em.clear();
    }
}

// 패턴 2: IDENTITY + JdbcTemplate.batchUpdate() (MySQL 환경)
@Service
@RequiredArgsConstructor
public class JdbcBatchService {

    private final JdbcTemplate jdbcTemplate;

    @Transactional
    public void saveProductsJdbc(List<ProductDto> dtos) {
        String sql = "INSERT INTO products (name, price) VALUES (?, ?)";

        jdbcTemplate.batchUpdate(sql, dtos, 100, (ps, dto) -> {
            ps.setString(1, dto.getName());
            ps.setInt(2, dto.getPrice());
        });
        // → PreparedStatement.addBatch() × N, executeBatch() per 100
        // IDENTITY와 함께 사용 가능
    }
}
```

### 5. Batch INSERT 성능 측정 — Statistics 활용

```java
// Hibernate Statistics로 Batch 효과 측정
@Configuration
public class HibernateConfig {
    @Bean
    public HibernateStatisticsFactoryBean statistics(EntityManagerFactory emf) {
        HibernateStatisticsFactoryBean stats = new HibernateStatisticsFactoryBean();
        stats.setEntityManagerFactory(emf);
        return stats;
    }
}

// 측정 코드
@Test
void measureBatchPerformance() {
    Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();

    long start = System.currentTimeMillis();
    batchService.saveProducts(generateDtos(10000));
    long elapsed = System.currentTimeMillis() - start;

    System.out.println("소요 시간: " + elapsed + "ms");
    System.out.println("JDBC Batch 실행 횟수: " + stats.getQueryExecutionCount());
    System.out.println("INSERT SQL 수: " + stats.getEntityInsertCount());
    // INSERT SQL 수 = 10000 (엔티티 수)
    // JDBC executeBatch 횟수 = 100 (10000 / 100 batchSize)
}
```

---

## 💻 실험으로 확인하기

### 실험: IDENTITY vs SEQUENCE Batch 비교

```java
@SpringBootTest
class BatchInsertTest {

    @Test
    void identityVsSequenceBatch() {
        int count = 10000;

        // IDENTITY 전략 (Batch 불가)
        long start = System.currentTimeMillis();
        identityProductService.saveAll(count);
        long identityTime = System.currentTimeMillis() - start;

        // SEQUENCE 전략 (Batch 가능)
        start = System.currentTimeMillis();
        sequenceProductService.saveAll(count);
        long sequenceTime = System.currentTimeMillis() - start;

        // JdbcTemplate Batch
        start = System.currentTimeMillis();
        jdbcBatchService.saveAll(count);
        long jdbcTime = System.currentTimeMillis() - start;

        System.out.printf("IDENTITY:  %dms%n", identityTime);
        System.out.printf("SEQUENCE:  %dms%n", sequenceTime);
        System.out.printf("JDBC Batch:%dms%n", jdbcTime);
    }
}
```

```
실측 결과 예시 (10,000건, H2 인메모리):

전략                  시간      DB 왕복
─────────────────────────────────────────
IDENTITY (기본)       12,450ms  10,000회
SEQUENCE + batch=100  380ms     100회 + 시퀀스 조회
JdbcTemplate.batch    220ms     100회
JPA IDENTITY + flush  8,200ms   10,000회 (flush 최적화 없음)
```

---

## ⚡ 성능 임팩트

```
Batch INSERT 성능 비교 (MySQL, 10,000건):

건별 INSERT (IDENTITY 기본):
  INSERT 10,000회 × ~1ms = ~10,000ms (10초)
  네트워크 왕복이 병목

JPA Batch (SEQUENCE, batch_size=100):
  executeBatch 100회 × ~5ms = ~500ms
  시퀀스 조회: 10,000/100 = 100회 × ~0.5ms = ~50ms
  총: ~550ms (18배 향상)

JdbcTemplate.batchUpdate():
  executeBatch 100회 × ~3ms = ~300ms
  JPA 오버헤드 없음
  총: ~300ms (33배 향상)

batch_size 최적화:
  너무 작음 (10): 효과 미미
  적정 (100~500): 대부분 환경에서 최적
  너무 큼 (10000): 메모리 부담, 트랜잭션 길어짐

order_inserts=true 효과:
  다른 엔티티 타입이 섞이면 Batch가 끊김
  order_inserts=true: 같은 엔티티 타입의 INSERT를 연속 배치
  복잡한 Cascade INSERT에서 큰 효과
```

---

## 🤔 트레이드오프

```
JPA Batch (SEQUENCE + batch_size):
  ✅ JPA 레벨 추상화 유지
  ✅ Cascade, dirty checking 등 JPA 기능 활용
  ✅ @Transactional + EntityManager 통합
  ❌ SEQUENCE 전략 필요 (IDENTITY 사용 불가)
  ❌ MySQL에서 SEQUENCE는 별도 설정 필요
  → PostgreSQL, Oracle 환경에 적합

JdbcTemplate.batchUpdate():
  ✅ IDENTITY 전략과 함께 사용 가능
  ✅ JPA보다 빠름 (엔티티 생명주기 관리 없음)
  ✅ MySQL에서 자연스러움
  ❌ JPA 추상화 포기 (raw SQL 작성)
  ❌ 엔티티 캐시/dirty checking과 별개 동작
  → 고성능 대량 INSERT가 최우선일 때

UUID @Id 직접 할당:
  ✅ IDENTITY 문제 없음, JPA Batch 가능
  ✅ 분산 환경에서 ID 충돌 없음
  ❌ UUID 저장 공간 크고 인덱스 비효율
  ❌ 순서 없는 ID → 페이지 분할 (INSERT 성능 저하)
  → 최신: UUID v7 (시간 순서 정렬) 고려

flush/clear 주기:
  batchSize와 동일하게 설정 권장
  너무 자주: flush 오버헤드 증가
  너무 드물게: 1차 캐시 메모리 부족 (OOM)
```

---

## 📌 핵심 정리

```
JPA Batch INSERT 조건
  hibernate.jdbc.batch_size > 0
  SEQUENCE 또는 TABLE 전략 (IDENTITY 불가)
  동일 엔티티 타입 연속 INSERT (order_inserts=true로 보조)

IDENTITY가 Batch를 막는 이유
  INSERT 직후 DB 생성 ID를 알아야 함
  Batch: 나중에 실행 → ID를 즉시 알 수 없음 충돌
  Hibernate가 IDENTITY 감지 시 Batch 자동 비활성화

SEQUENCE + allocationSize
  allocationSize개의 ID를 한 번에 예약 (DB 호출 최소화)
  메모리에서 ID 할당 → persist()가 즉시 ID를 가짐
  → ActionQueue에 쌓아 Batch flush 가능

flush/clear 패턴
  batchSize마다 em.flush() + em.clear()
  flush: Batch INSERT 실행
  clear: 1차 캐시 비움 (메모리 관리)

IDENTITY 환경 대안
  JdbcTemplate.batchUpdate() → DB 왕복 최소화
  Spring Batch + ItemWriter → 대용량 배치
  UUID v7 직접 할당 → JPA Batch 가능
```

---

## 🤔 생각해볼 문제

**Q1.** `order_inserts=true`와 `order_updates=true`를 설정하지 않으면 어떤 상황에서 Batch가 끊기는가? Cascade PERSIST로 부모-자식을 함께 저장할 때 어떤 문제가 발생하는가?

**Q2.** `allocationSize=50`으로 설정하고 애플리케이션을 재시작하면 ID 시퀀스에 어떤 일이 생기는가? 재시작마다 50개씩 ID가 낭비되는가?

**Q3.** `saveAll(list)`를 호출할 때 리스트의 일부 엔티티는 `New` 상태이고 일부는 `Detached` 상태라면 어떻게 처리되는가? Batch INSERT에 영향이 있는가?

> 💡 **해설**
>
> **Q1.** `order_inserts=false`(기본값)일 때, ActionQueue는 FIFO 순서로 SQL을 실행한다. `Order`와 `OrderItem`을 Cascade PERSIST로 함께 저장하면: INSERT order(1) → INSERT order_item(1.1) → INSERT order_item(1.2) → INSERT order(2) → INSERT order_item(2.1) 순서로 ActionQueue에 쌓인다. Batch는 연속된 동일 SQL끼리만 묶을 수 있으므로: `order INSERT` × 1 → `order_item INSERT` × 2 → `order INSERT` × 1 → `order_item INSERT` × 1 처럼 Batch가 계속 끊긴다. `order_inserts=true` 설정 시 Hibernate가 ActionQueue를 재정렬하여 `order INSERT` × N → `order_item INSERT` × M 순서로 그룹화하므로 Batch 효율이 극대화된다.
>
> **Q2.** 재시작마다 ID가 낭비된다. `allocationSize=50` 설정 시 Hibernate는 메모리에서 50개 ID 범위를 추적한다. 애플리케이션이 재시작되면 메모리의 ID 할당 정보가 초기화되고, DB 시퀀스에서 다시 NEXTVAL을 호출한다. 만약 이전에 1~50을 예약하고 30개만 사용했다면, 재시작 후 51~100이 예약되어 31~50의 ID는 영구 누락된다. 이것은 SEQUENCE 전략의 알려진 트레이드오프다. ID 연속성이 중요하다면 `allocationSize=1`로 설정하거나(성능 저하) IDENTITY를 사용해야 한다. 대부분의 비즈니스 로직에서 ID 연속성은 중요하지 않으므로 `allocationSize=50~100`이 권장된다.
>
> **Q3.** `SimpleJpaRepository.saveAll()`은 각 엔티티에 대해 `save(entity)`를 호출하고, `save()`는 `isNew()` 여부에 따라 `persist()` 또는 `merge()`를 선택한다. `New` 엔티티 → `persist()` → ActionQueue에 INSERT 등록, `Detached` 엔티티 → `merge()` → SELECT(DB 조회) + UPDATE 예약. Batch INSERT에 영향이 생긴다: `persist()`로 쌓인 INSERT Batch는 중간에 `merge()`로 인한 SELECT가 끼어들면서 Batch가 끊길 수 있다. 또한 `merge()`는 SELECT를 발생시키므로 N개의 Detached 엔티티가 섞이면 N번의 추가 SELECT가 발생해 성능이 저하된다. 권장: 대량 저장 시 모두 `New` 엔티티로 구성하거나, Detached 엔티티는 별도로 처리한다.

---

<div align="center">

**[⬅️ 이전: Cascade 타입과 Orphan Removal](./06-cascade-orphan-removal.md)** | **[홈으로 🏠](../README.md)**

</div>
