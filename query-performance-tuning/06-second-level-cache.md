# Second-Level Cache — Ehcache / Redis 연동과 캐시 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 1차 캐시(Persistence Context)와 2차 캐시(Second-Level Cache)의 범위와 생명주기 차이는?
- `@Cache` 어노테이션이 Hibernate Region Factory를 통해 Ehcache/Redis에 저장되는 경로는?
- `READ_ONLY` / `READ_WRITE` / `NONSTRICT_READ_WRITE` 전략의 동시성 처리 방식은?
- 2차 캐시를 쓰면서 데이터 정합성을 해치지 않는 캐시 무효화 시점은?
- 쿼리 캐시(Query Cache)와 엔티티 캐시의 차이와 각각의 적용 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: 1차 캐시는 트랜잭션 범위로 한정된다

```java
// 1차 캐시 (Persistence Context):
//   범위: 하나의 EntityManager (트랜잭션 단위)
//   트랜잭션 종료 → 소멸

// 동일 엔티티를 두 요청에서 각각 조회
// 요청 1 (트랜잭션 A):
User user1 = em.find(User.class, 1L); // DB 조회
// 트랜잭션 종료 → 1차 캐시 소멸

// 요청 2 (트랜잭션 B):
User user2 = em.find(User.class, 1L); // 다시 DB 조회! (1차 캐시 없음)

// 문제: 변경이 없는 참조 데이터(공지사항, 설정, 코드 테이블)도
//       요청마다 DB 조회 → 불필요한 DB 부하
```

```
2차 캐시 (Second-Level Cache):
  범위: SessionFactory (애플리케이션 전체 공유)
  생명주기: 애플리케이션 구동 중 유지 (또는 설정된 TTL)
  저장소: JVM 힙 (Ehcache) 또는 외부 스토어 (Redis)

  요청 1 → DB 조회 → 2차 캐시 저장
  요청 2 → 2차 캐시 히트 → DB 조회 없음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 모든 엔티티에 @Cache를 붙이면 성능이 좋아진다

```java
// ❌ 자주 변경되는 엔티티에 @Cache 적용
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Order {  // 주문은 자주 상태가 변경됨!
    private OrderStatus status;
    // 캐시 무효화 빈번 → 캐시 비용 > 캐시 이익
    // 동시 수정 시 캐시 정합성 문제 위험
}

// ✅ 2차 캐시가 적합한 엔티티:
// - 읽기 빈도 높고 수정 빈도 낮음
// - 참조 데이터 (코드, 카테고리, 설정)
// - 수정 시 약간의 지연(eventual consistency) 허용 가능

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Category { ... }  // 카테고리는 거의 변경 없음 → 적합

@Entity
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class ProductInfo { ... } // 가끔 변경, 약간의 지연 허용
```

### Before: 쿼리 캐시는 항상 유용하다

```java
// ❌ 쿼리 캐시 남용
// 쿼리 캐시 = 쿼리 결과의 엔티티 ID 목록을 캐시
// 엔티티 데이터 자체는 엔티티 캐시에서 가져옴

@Query("SELECT u FROM User u WHERE u.status = :status")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<User> findByStatus(@Param("status") Status status);

// 문제: User 엔티티 하나라도 변경되면 → 쿼리 캐시 전체 무효화!
// 활발히 변경되는 엔티티의 쿼리 캐시 → 캐시 적중률 0%에 가까움
// → 쿼리 캐시는 결과가 거의 변하지 않는 쿼리에만 적합
```

---

## ✨ 올바른 이해와 패턴

### After: 캐시 전략 선택 가이드

```
READ_ONLY:
  수정 없는 참조 데이터 (코드 테이블, 고정 설정)
  가장 빠름 (Lock 없음, dirty check 없음)
  수정 시 예외 발생

READ_WRITE:
  수정이 있지만 동시성 보장 필요
  Soft Lock으로 동시 수정 시 다른 스레드 캐시 미스 처리
  성능: READ_ONLY보다 약간 낮음

NONSTRICT_READ_WRITE:
  수정이 있고 약간의 일관성 지연 허용
  Lock 없음 → 수정 직후 다른 스레드가 구버전 읽을 수 있음
  성능: READ_WRITE보다 높음

TRANSACTIONAL:
  JTA 환경에서 완전한 트랜잭션 참여
  2차 캐시와 DB 트랜잭션이 함께 커밋/롤백
  성능: 가장 낮음 (XA 트랜잭션)
```

---

## 🔬 내부 동작 원리 — 2차 캐시 소스 추적

### 1. 2차 캐시 저장/조회 흐름

```java
// EntityManager.find() → 2차 캐시 조회 흐름
// (CacheEnabled = true, @Cache 선언 시)

// SessionImpl.find()
public <T> T find(Class<T> entityClass, Object primaryKey) {
    // [1] 1차 캐시 확인
    T entity = sessionCache.getEntity(new EntityKey(primaryKey, entityDescriptor));
    if (entity != null) return entity;

    // [2] 2차 캐시 확인
    if (entityDescriptor.canUseReferenceCacheEntries()) {
        Object cached = entityDescriptor.getCacheAccessStrategy()
            .get(session, new EntityCacheKey(primaryKey));
        // → RegionFactory → Ehcache / Redis 조회

        if (cached != null) {
            // 캐시 히트 → 엔티티 재구성
            T result = assembleFromCacheEntry(cached, entityDescriptor, session);
            // 1차 캐시에도 등록
            session.getPersistenceContext().addEntity(primaryKey, result);
            return result;
        }
    }

    // [3] DB 조회
    T result = loadFromDatabase(entityClass, primaryKey);
    // DB 조회 후 2차 캐시에 저장
    entityDescriptor.getCacheAccessStrategy().putFromLoad(
        session, new EntityCacheKey(primaryKey), cacheEntry, version);
    return result;
}
```

### 2. Region Factory — Ehcache 연동

```java
// Ehcache 2차 캐시 설정 (Spring Boot)

// build.gradle
// implementation 'org.hibernate.orm:hibernate-jcache'
// implementation 'org.ehcache:ehcache'

// application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            uri: classpath:ehcache.xml   # Ehcache 설정 파일

// ehcache.xml
// <config xmlns='...'
//   <cache alias="com.example.Category">
//     <expiry><ttl unit="hours">24</ttl></expiry>
//     <heap unit="entries">1000</heap>
//   </cache>
// </config>

// 엔티티에 @Cache 선언
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Category {
    @Id Long id;
    String name;
    String code;
}
// → Category 조회 시 Ehcache 캐시 사용
// → Region 이름: "com.example.Category" (패키지 포함 클래스명)
```

### 3. READ_WRITE 전략 — Soft Lock 동작

```java
// READ_WRITE: 수정 시 Soft Lock으로 캐시 정합성 보장
// 수정 흐름:

// [1] 트랜잭션 시작, 엔티티 수정
category.setName("새이름");

// [2] flush() (또는 커밋 전)
// → 2차 캐시에서 항목 제거 + Soft Lock 등록
// Lock: {lockId: UUID, txStart: timestamp, timeout: 5000ms}
// 다른 스레드가 이 항목 조회 시 → Soft Lock 발견 → 캐시 미스 처리 (DB 조회)

// [3] DB UPDATE 실행

// [4] 커밋 후
// → Soft Lock 해제 + 새 값으로 캐시 업데이트
// 이후 조회 → 새 값 캐시 히트

// Soft Lock 타임아웃 (5초) 이내 다른 스레드:
//   캐시에서 Soft Lock 발견 → 캐시 미스 → DB 직접 조회
//   → 정합성 보장 (구버전 캐시 반환 없음)
```

### 4. Redis 기반 2차 캐시 — Spring Cache와의 차이

```java
// Hibernate 2차 캐시 (엔티티 캐시):
// → EntityManager.find() 수준에서 자동으로 캐시
// → 캐시 키: 엔티티 ID
// → 캐시 값: 엔티티 상태 (CacheEntry)
// → 무효화: 엔티티 수정/삭제 시 자동

// Spring Cache (@Cacheable):
// → 메서드 호출 수준에서 캐시
// → 캐시 키: 메서드 파라미터
// → 캐시 값: 메서드 반환값 (DTO, List 등)
// → 무효화: @CacheEvict 명시 필요

// Redis + Hibernate 2차 캐시 설정:
// implementation 'com.github.sonus21:hibernate-l2cache:...'
// 또는 Redisson Hibernate Cache
// implementation 'org.redisson:redisson-hibernate-6:...'

// application.yml (Redisson 기반)
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: org.redisson.hibernate.RedissonRegionFactory
        redisson:
          config: classpath:redisson.yml

// redisson.yml
// singleServerConfig:
//   address: "redis://localhost:6379"

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Redis에 저장
public class ProductInfo { ... }
```

### 5. 캐시 무효화 시점과 전략

```java
// Hibernate 자동 무효화:
// 엔티티 수정 → 해당 엔티티 캐시 무효화
// 벌크 UPDATE/DELETE → 해당 엔티티 타입 전체 캐시 무효화

// 주의: 벌크 연산은 1차 캐시를 거치지 않음
@Modifying
@Query("UPDATE Category c SET c.active = false WHERE c.type = :type")
void deactivateByType(@Param("type") String type);
// → Hibernate가 Category 엔티티 캐시 전체 무효화
// → 다음 조회 시 DB에서 다시 로딩

// 수동 캐시 무효화:
Cache cache = emf.getCache();
cache.evict(Category.class, categoryId); // 특정 엔티티 무효화
cache.evictAll(Category.class);          // 전체 무효화

// SessionFactory 통한 무효화:
SessionFactory sf = emf.unwrap(SessionFactory.class);
sf.getCache().evictEntityData(Category.class);
```

### 6. 컬렉션 캐시

```java
// 연관 컬렉션도 별도 캐시 가능
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Category {
    @Id Long id;

    @OneToMany(mappedBy = "category")
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY)  // 컬렉션도 캐시
    private List<Product> products;
}

// 컬렉션 캐시 Region 이름:
// "com.example.Category.products"
// → 캐시 키: Category ID
// → 캐시 값: Product ID 목록 (실제 엔티티는 엔티티 캐시에서)

// 주의: 컬렉션 캐시는 엔티티 캐시와 함께 사용해야 효과적
// 컬렉션 캐시만 있고 엔티티 캐시 없으면:
// Product ID는 캐시에서 → 각 Product는 DB에서 조회 (N+1!)
```

---

## 💻 실험으로 확인하기

### 실험 1: 2차 캐시 히트 확인

```java
@Autowired
EntityManagerFactory emf;

@Test
void secondLevelCacheHit() {
    SessionFactory sf = emf.unwrap(SessionFactory.class);
    Statistics stats = sf.getStatistics();
    stats.setStatisticsEnabled(true);
    stats.clear();

    // 첫 번째 조회 (트랜잭션 1) — DB 조회 + 캐시 저장
    transactionTemplate.execute(s -> em.find(Category.class, 1L));

    // 두 번째 조회 (트랜잭션 2) — 2차 캐시 히트
    transactionTemplate.execute(s -> em.find(Category.class, 1L));

    System.out.println("SecondLevel cache hits: "
        + stats.getSecondLevelCacheHitCount());   // 1
    System.out.println("SecondLevel cache misses: "
        + stats.getSecondLevelCacheMissCount());  // 1 (첫 조회)
    System.out.println("Entity fetch count: "
        + stats.getEntityFetchCount());           // 1 (DB 조회 1번만)
}
```

### 실험 2: 수정 후 캐시 무효화 확인

```java
@Test
void cacheInvalidationOnUpdate() {
    stats.clear();

    // [1] 첫 조회 → 캐시 저장
    transactionTemplate.execute(s -> em.find(Category.class, 1L));

    // [2] 수정
    transactionTemplate.execute(s -> {
        Category c = em.find(Category.class, 1L);
        c.setName("수정됨");
        return null; // dirty checking → UPDATE + 캐시 무효화
    });

    // [3] 재조회 → 캐시 무효화됐으므로 DB 조회
    transactionTemplate.execute(s -> em.find(Category.class, 1L));

    System.out.println("DB fetches: " + stats.getEntityFetchCount()); // 2
    // (첫 조회 1번 + 수정 후 재조회 1번)
}
```

---

## ⚡ 성능 임팩트

```
2차 캐시 효과 (READ_ONLY, 카테고리 100개 기준):

캐시 없음:
  요청마다 SELECT FROM categories WHERE id = ?
  초당 1000 요청 → 초당 1000번 DB 쿼리
  DB CPU: 높음, 응답 시간: DB 왕복 포함 (~수 ms)

Ehcache (JVM 힙):
  캐시 히트율 99% → 초당 10번 DB 쿼리
  응답 시간: ~수십 μs (JVM 메모리 접근)
  → 100배 DB 부하 감소

Redis (외부 캐시):
  캐시 히트율 99% → 초당 10번 DB 쿼리
  응답 시간: ~수십 μs~수 ms (네트워크)
  → DB보다 빠름, Ehcache보다 느림
  → 분산 환경에서 일관성 보장

캐시 전략별 오버헤드:
  READ_ONLY:            가장 낮음 (Lock 없음)
  NONSTRICT_READ_WRITE: 낮음 (타임스탬프 비교만)
  READ_WRITE:           중간 (Soft Lock 관리)
  TRANSACTIONAL:        높음 (XA 트랜잭션 참여)
```

---

## 🤔 트레이드오프

```
Ehcache (JVM 힙 캐시):
  ✅ 가장 빠름 (메모리 직접 접근)
  ✅ 별도 인프라 불필요
  ❌ JVM 인스턴스별 독립 → 멀티 인스턴스 불일치
  ❌ JVM 힙 사용 → GC 압박
  → 단일 인스턴스, 개발/테스트 환경

Redis (외부 캐시):
  ✅ 멀티 인스턴스 공유 (분산 환경 일관성)
  ✅ JVM 힙 독립 (GC 영향 없음)
  ✅ TTL 설정, 영속성 지원
  ❌ 네트워크 왕복 (~수십 μs)
  ❌ Redis 인프라 관리 필요
  → 멀티 인스턴스 운영 환경

2차 캐시 vs Spring @Cacheable:
  2차 캐시: EntityManager.find() 수준 자동 캐시
            캐시 키 = 엔티티 ID, 무효화 자동
  @Cacheable: 메서드 반환값 캐시
              캐시 키 = 파라미터, 무효화 수동(@CacheEvict)
  → 단순 엔티티 조회 → 2차 캐시
  → 복잡한 쿼리 결과, DTO → @Cacheable

2차 캐시 적용 기준:
  읽기 빈도 / 쓰기 빈도 비율 > 10 → 캐시 고려
  데이터 변경 시 약간의 지연 허용 → NONSTRICT_READ_WRITE
  완전한 정합성 필요 → READ_WRITE (or 캐시 사용 안 함)
```

---

## 📌 핵심 정리

```
1차 캐시 vs 2차 캐시
  1차: EntityManager 범위 (트랜잭션)
  2차: SessionFactory 범위 (애플리케이션 전체)
  2차는 1차 미스 시에만 조회됨

@Cache 어노테이션 위치
  @Entity 레벨: 엔티티 캐시
  @OneToMany/@ManyToMany: 컬렉션 캐시 (엔티티 캐시와 함께 사용)

캐시 전략
  READ_ONLY:            변경 없는 참조 데이터
  READ_WRITE:           변경 있음, 정합성 보장 (Soft Lock)
  NONSTRICT_READ_WRITE: 변경 있음, 약간의 지연 허용
  TRANSACTIONAL:        JTA 환경만

캐시 무효화
  엔티티 수정 → 해당 항목 자동 무효화
  벌크 연산 → 해당 타입 전체 무효화
  수동: emf.getCache().evict(), evictAll()

2차 캐시 적합 엔티티
  읽기 多, 쓰기 少 (비율 10:1 이상)
  카테고리, 코드 테이블, 설정 데이터
  자주 변경되는 Order, Payment → 적합하지 않음
```

---

## 🤔 생각해볼 문제

**Q1.** 멀티 인스턴스(복수 서버) 환경에서 Ehcache(JVM 힙)를 2차 캐시로 사용할 때 발생하는 정합성 문제의 구체적인 시나리오는? 어떻게 해결할 수 있는가?

**Q2.** `@Cache(usage = READ_WRITE)` 엔티티에 대해 `@Modifying @Query("UPDATE ...")`로 벌크 업데이트를 실행하면 캐시가 올바르게 무효화되는가? `em.clear()`와의 관계는?

**Q3.** 쿼리 캐시(Query Cache)를 활성화하고 `SELECT u FROM User u WHERE u.status = 'ACTIVE'`를 캐시했다. 이후 새로운 User가 추가(INSERT)됐을 때 쿼리 캐시가 무효화되는가?

> 💡 **해설**
>
> **Q1.** 구체적 시나리오: 서버 A에서 Category(id=1)를 수정 → 서버 A의 Ehcache에서 무효화. 그러나 서버 B의 Ehcache에는 구버전 캐시가 남아있음 → 서버 B에 요청이 들어오면 구버전 데이터 반환. 이를 캐시 불일치(Cache Staleness) 문제라고 한다. 해결 방법: (1) Redis 등 외부 공유 캐시 사용 — 모든 인스턴스가 같은 캐시 저장소 공유. (2) Ehcache의 Cluster 모드 또는 Terracotta — JVM 간 캐시 동기화. (3) 짧은 TTL 설정 — 불일치 허용 기간을 최소화 (예: 5분). (4) 캐시 무효화 이벤트 브로드캐스트 — Kafka/Redis Pub/Sub으로 모든 인스턴스에 무효화 알림.
>
> **Q2.** 벌크 `@Modifying @Query`는 1차 캐시를 거치지 않고 DB에 직접 UPDATE를 실행한다. Hibernate는 이 경우 해당 엔티티 타입(Category)의 2차 캐시 전체를 무효화한다(`evictEntityData`). 따라서 2차 캐시는 자동으로 무효화된다. `em.clear()`는 1차 캐시(Persistence Context)를 비우는 것이고 2차 캐시와는 무관하다. 벌크 연산 후 `em.clear()`를 하는 이유는 1차 캐시에 남은 구버전 엔티티 인스턴스를 제거해서 이후 `find()`가 DB(또는 2차 캐시)에서 최신 데이터를 가져오게 하기 위함이다.
>
> **Q3.** 무효화된다. Hibernate의 쿼리 캐시는 쿼리 결과를 캐시하면서, 해당 결과에 사용된 엔티티 타입(여기서는 `User`)의 "timestamp"도 함께 저장한다. `User` 엔티티에 INSERT/UPDATE/DELETE가 발생하면 Hibernate는 `UpdateTimestampsCache`에 해당 타입의 타임스탬프를 갱신한다. 쿼리 캐시 히트 시 저장된 타임스탬프와 UpdateTimestampsCache의 타임스탬프를 비교해, 엔티티 타입이 수정됐으면 쿼리 캐시를 무효화(미스 처리)한다. 따라서 User INSERT → User 타임스탬프 갱신 → 쿼리 캐시 미스 → DB에서 재조회. 이 때문에 활발히 변경되는 엔티티의 쿼리 캐시는 적중률이 매우 낮아 실용적이지 않다.

---

<div align="center">

**[⬅️ 이전: Query Plan Cache 동작](./05-query-plan-cache.md)** | **[홈으로 🏠](../README.md)**

</div>
