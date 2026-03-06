# Persistence Context — 1차 캐시 동작 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Persistence Context 내부의 `EntityKey → EntityEntry` 맵 구조는 무엇인가?
- 동일 트랜잭션 내에서 같은 ID로 두 번 조회해도 쿼리가 1개만 발생하는 이유는?
- 엔티티의 4가지 상태(New / Managed / Detached / Removed)는 어떻게 전환되는가?
- `em.merge()`는 `em.persist()`와 어떻게 다르며, 언제 어떤 SQL이 발생하는가?
- OSIV(Open Session In View)가 켜져 있을 때 Persistence Context 범위는 어디까지인가?

---

## 🔍 왜 이게 존재하는가

### 문제: 매번 DB 조회하면 같은 데이터를 반복해서 읽는다

```java
// Persistence Context(1차 캐시) 없는 가정
User user1 = em.find(User.class, 1L); // SELECT 실행
User user2 = em.find(User.class, 1L); // 또 SELECT 실행 (동일 데이터)
User user3 = em.find(User.class, 1L); // 또 SELECT 실행

// user1 == user2 == user3? 아님 (서로 다른 객체)
// DB에는 총 3번 왕복 발생

// Persistence Context 있는 실제 Hibernate:
User user1 = em.find(User.class, 1L); // SELECT 실행 → 1차 캐시에 저장
User user2 = em.find(User.class, 1L); // 1차 캐시 히트 → DB 접근 없음
User user3 = em.find(User.class, 1L); // 1차 캐시 히트 → DB 접근 없음

// user1 == user2 == user3 (동일 객체 보장 — 객체 동일성)
// DB 왕복 1번
```

```
Persistence Context의 두 가지 역할:
  1. 1차 캐시: 동일 트랜잭션 내 같은 엔티티 재조회 시 DB 접근 없음
  2. 변경 감지: 관리(Managed) 상태 엔티티의 변경을 추적 → flush 시 자동 UPDATE
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: em.find()는 항상 DB를 조회한다

```java
// ❌ 잘못된 이해
@Transactional
public void process() {
    User user = userRepository.findById(1L).get(); // SELECT
    // 동일 트랜잭션 내
    User sameUser = userRepository.findById(1L).get(); // SELECT? → 아님!
    // 1차 캐시 히트 → 쿼리 없음

    System.out.println(user == sameUser); // true — 동일 객체
}
```

### Before: JPQL/Criteria 쿼리도 1차 캐시를 사용한다

```java
// ❌ 잘못된 이해: JPQL 결과도 1차 캐시에서 가져온다
User user1 = em.find(User.class, 1L); // 1차 캐시에 저장

// JPQL은 1차 캐시를 bypass하고 항상 DB에 쿼리
List<User> users = em.createQuery("SELECT u FROM User u WHERE u.id = 1", User.class)
                     .getResultList();
// → DB에 SELECT 실행 → 결과를 1차 캐시와 병합
// → user1과 users.get(0)은 동일 객체 (병합 후 캐시 반환)

// 핵심: em.find() → 1차 캐시 우선
//       JPQL/Criteria → 항상 DB 쿼리 (결과는 캐시에 병합)
```

### Before: detached 엔티티에서 Lazy Loading이 된다

```java
// ❌ 트랜잭션 종료 후 Lazy Loading 시도
User user;
try (EntityManager em = emf.createEntityManager()) {
    em.getTransaction().begin();
    user = em.find(User.class, 1L); // Managed 상태
    em.getTransaction().commit();
} // EntityManager 닫힘 → user가 Detached 상태로 전환

// Detached 상태에서 Lazy Loading
user.getOrders().size(); // LazyInitializationException!
// orders는 아직 초기화 안 됨, EntityManager는 닫힘
```

---

## ✨ 올바른 이해와 패턴

### After: 엔티티 상태 전환 전체 흐름

```
New (Transient):
  new User() — Spring/Hibernate가 모름
  → persist() → Managed

Managed:
  1차 캐시에 등록 (EntityKey → EntityEntry)
  변경 감지 대상
  → 트랜잭션 커밋/close() → Detached
  → remove() → Removed

Detached:
  1차 캐시에서 제거 (트랜잭션 종료 or em.detach())
  변경 감지 안 됨
  → merge() → Managed (병합 후)
  → persist() → 예외 (이미 ID 있으면 EntityExistsException)

Removed:
  삭제 예약됨 (1차 캐시에서 제거 예약)
  → flush 시 DELETE SQL 실행
  → commit 후 DB에서 삭제됨
```

---

## 🔬 내부 동작 원리 — Persistence Context 소스 추적

### 1. StatefulPersistenceContext — 1차 캐시 내부 구조

```java
// Hibernate의 Persistence Context 구현체
public class StatefulPersistenceContext implements PersistenceContext {

    // 1차 캐시 핵심 자료구조
    // EntityKey: (entityName, id) 복합 키
    // EntityEntry: 엔티티 상태 정보 + 스냅샷(loadedState)
    private Map<EntityKey, Object> entitiesByKey;
    // entitiesByKey: EntityKey → 실제 엔티티 객체

    private Map<EntityKey, EntityEntry> entityEntryContext;
    // entityEntryContext: EntityKey → EntityEntry (상태 추적 정보)

    // 컬렉션 캐시 (연관 컬렉션)
    private Map<CollectionKey, PersistentCollection<?>> collectionsByKey;
}

// EntityKey — 1차 캐시의 키
class EntityKey {
    private final Object identifier;   // 엔티티 ID (e.g., 1L)
    private final EntityPersister persister; // 엔티티 메타데이터
    // hashCode, equals: identifier + persister 기반
}

// EntityEntry — 엔티티 상태 추적
class EntityEntry {
    private Status status;        // MANAGED, DELETED, GONE, LOADING
    private Object[] loadedState; // 스냅샷 (dirty checking 기준값)
    private Object version;       // @Version 값
    private boolean existsInDatabase;
    private LockMode lockMode;
}
```

### 2. em.find() — 1차 캐시 조회 흐름

```java
// SessionImpl.find() — em.find() 구현
public <T> T find(Class<T> entityClass, Object primaryKey, ...) {

    EntityPersister persister = getEntityPersister(entityClass.getName());

    // [1] EntityKey 생성
    EntityKey key = generateEntityKey(primaryKey, persister);

    // [2] 1차 캐시 조회
    Object cached = persistenceContext.getEntity(key);
    if (cached != null) {
        // 캐시 히트 → DB 조회 없이 반환
        return (T) cached;
    }

    // [3] 캐시 미스 → DB 조회
    Object entity = persister.load(primaryKey, null, lockMode, this);
    // → SELECT 실행

    // [4] 결과를 1차 캐시에 등록
    // (load() 내부에서 persistenceContext.addEntity() 호출)

    return (T) entity;
}
```

### 3. JPQL 쿼리 후 1차 캐시 병합

```java
// JPQL 실행 후 결과 처리 — 1차 캐시 병합
// HqlQueryPlan.performList() → ResultSetProcessingContextImpl

// JPQL 결과의 각 행에 대해:
for (Object[] row : rows) {
    Object id = extractIdentifier(row);
    EntityKey key = generateEntityKey(id, persister);

    // 1차 캐시에 이미 있는가?
    Object existing = persistenceContext.getEntity(key);
    if (existing != null) {
        // 있으면 DB 결과 무시, 캐시의 엔티티 반환 (동일성 보장)
        result.add(existing);
    } else {
        // 없으면 새 엔티티 생성 → 1차 캐시에 등록
        Object entity = createAndInitialize(row, key, ...);
        persistenceContext.addEntity(key, entity);
        result.add(entity);
    }
}
// JPQL이 DB에서 새로운 데이터를 가져왔더라도,
// 1차 캐시에 같은 ID의 엔티티가 있으면 캐시 버전을 우선 (Dirty Read 방지)
```

### 4. em.merge() — Detached 엔티티 병합

```java
// SessionImpl.merge() — Detached 엔티티를 Managed로 전환
@Override
public <T> T merge(T object) {

    // [1] 대상 엔티티의 ID 조회
    Object id = getIdentifier(object);

    // [2] 1차 캐시 조회
    Object managed = persistenceContext.getEntity(generateEntityKey(id, persister));

    if (managed == null) {
        // [3] 1차 캐시에 없음 → DB에서 조회
        managed = em.find(entityClass, id);
        // 없으면 새로 생성 (INSERT 예약)
    }

    // [4] detached → managed 상태의 엔티티로 값 복사
    copyValues(object, managed);
    // → detached 엔티티의 필드값을 managed 엔티티에 복사

    // [5] managed 엔티티 반환 (이것이 1차 캐시에 등록된 객체)
    return (T) managed;
}

// merge() vs persist():
// persist(): New 엔티티를 Managed로 (ID 없음 → INSERT 예약)
// merge(): Detached 엔티티를 Managed로 (ID 있음 → DB 조회 후 병합)
```

### 5. 엔티티 상태별 flush 동작

```java
// flush() 시 1차 캐시 내 엔티티 처리
// SessionImpl.flush() → DefaultFlushEventListener.onFlush()

// 모든 Managed 엔티티를 순회:
for (EntityEntry entry : persistenceContext.reentrantSafeEntityEntries()) {

    Object entity = persistenceContext.getEntity(entry.getEntityKey());

    if (entry.getStatus() == Status.MANAGED) {
        // Dirty Checking: loadedState(스냅샷)와 현재 상태 비교
        Object[] currentState = entry.getPersister().getPropertyValues(entity);
        Object[] loadedState = entry.getLoadedState();

        if (!Arrays.equals(currentState, loadedState)) {
            // 변경 감지 → UPDATE SQL 예약
            actionQueue.addAction(new EntityUpdateAction(entity, ...));
        }
    }

    if (entry.getStatus() == Status.DELETED) {
        // remove() 호출된 엔티티 → DELETE SQL 예약
        actionQueue.addAction(new EntityDeleteAction(entity, ...));
    }
}

// 예약된 SQL 실행 순서:
// 1. INSERT (persist된 것)
// 2. UPDATE (dirty checked)
// 3. DELETE (removed)
```

### 6. Persistence Context 범위 — 트랜잭션 vs OSIV

```
기본 (Transaction-scoped):
  트랜잭션 시작 → Persistence Context 생성
  트랜잭션 종료 → Persistence Context 닫힘 (모든 엔티티 Detached)

OSIV (Open Session In View, spring.jpa.open-in-view=true):
  HTTP 요청 시작 → OpenEntityManagerInViewInterceptor → Persistence Context 생성
  @Transactional 메서드 → 같은 Persistence Context 재사용
  HTTP 요청 종료 → Persistence Context 닫힘

OSIV의 함정:
  서비스 트랜잭션 종료 후에도 Persistence Context 열려있음
  Controller에서 Lazy Loading 가능 (편의)
  단, 트랜잭션 없이 DB Connection 점유 → Connection Pool 낭비
  대규모 트래픽 환경에서 Connection 고갈 위험
  → spring.jpa.open-in-view=false 권장 (명시적 Fetch 전략 사용)
```

---

## 💻 실험으로 확인하기

### 실험 1: 1차 캐시 히트 확인

```java
@Transactional
@Test
void firstLevelCacheHit() {
    // 첫 조회 — SQL 발생
    User user1 = userRepository.findById(1L).get();
    // SELECT u FROM users WHERE id = 1

    // 두 번째 조회 — SQL 없음 (1차 캐시)
    User user2 = userRepository.findById(1L).get();
    // SQL 없음!

    // 동일 객체 보장
    assertSame(user1, user2); // ✅ 같은 참조
}

// hibernate.show_sql=true 출력:
// Hibernate: select u.id, u.name from users u where u.id=?
// (두 번째 find는 SQL 없음)
```

### 실험 2: JPQL vs find() 1차 캐시 동작 차이

```java
@Transactional
@Test
void jpqlVsFind() {
    // JPQL은 항상 DB 쿼리
    User fromJpql = em.createQuery(
        "SELECT u FROM User u WHERE u.id = 1", User.class).getSingleResult();
    // SQL: SELECT ...

    // find()는 이미 캐시에 있으므로 쿼리 없음
    User fromFind = em.find(User.class, 1L);
    // SQL: 없음

    // 동일 객체 (JPQL 결과가 캐시의 엔티티로 대체됨)
    assertSame(fromJpql, fromFind);
}
```

### 실험 3: 상태 전환 확인

```java
@Test
void entityStateTransition() {
    // New
    User user = new User("홍길동");
    // Hibernate가 모름 — ID 없음

    // Managed
    userRepository.save(user); // persist() → Managed, INSERT 예약
    Long userId = user.getId(); // ID 채워짐

    // Detached (트랜잭션 종료 후)
    // @Test에서 @Transactional 없으면 자동으로 Detached

    // Managed again (새 트랜잭션에서 find)
    @Transactional
    User managed = userRepository.findById(userId).get();
    // 다시 Managed → 변경 감지 대상
}
```

---

## ⚡ 성능 임팩트

```
1차 캐시 효과:

동일 엔티티 N번 조회 시:
  캐시 없음: N번 DB 왕복 (N × ~수 ms)
  1차 캐시: 1번 DB 왕복 + (N-1) × ~수 ns (맵 조회)
  → 같은 트랜잭션에서 반복 조회가 많을수록 효과 큼

1차 캐시의 한계:
  트랜잭션 범위 내에서만 유효 (요청 간 공유 불가)
  2차 캐시(Ehcache, Redis)로 트랜잭션 간 캐시 가능

flush() 비용:
  Managed 엔티티 수에 비례한 dirty checking
  1000개 엔티티 × 20 fields = 20,000 값 비교
  → 대량 처리 시 flush/clear 주기적 수행 필요

merge() 비용:
  1차 캐시 조회 → 없으면 DB 조회 (SELECT 발생!)
  → Detached 엔티티를 반복 merge() 시 N번 SELECT 발생
  → 신규 엔티티는 persist(), 기존 수정은 변경 감지 활용
```

---

## 🤔 트레이드오프

```
OSIV 활성화 (spring.jpa.open-in-view=true, 기본값):
  ✅ Controller에서 Lazy Loading 가능
  ✅ "N+1 고민 없이" 편리하게 사용
  ❌ 서비스 종료 후에도 Connection 점유
  ❌ 대규모 트래픽 → Connection Pool 고갈
  ❌ 느슨한 레이어 경계 (Controller가 DB 접근)
  → 소규모 서비스, 프로토타입에 적합

OSIV 비활성화 (spring.jpa.open-in-view=false):
  ✅ 트랜잭션 범위 내에서만 Connection 사용
  ✅ 레이어 경계 명확 (Controller는 DTO만)
  ❌ Lazy Loading → LazyInitializationException 주의
  ❌ 명시적 Fetch 전략 필요 (EntityGraph, Fetch Join)
  → 프로덕션 권장, 특히 고트래픽 서비스

1차 캐시 범위 확장:
  2차 캐시 (Ehcache, Redis) → Ch4-06 참조
  읽기 전용 자주 조회되는 엔티티에 적합
  쓰기가 많은 엔티티에는 캐시 무효화 비용 주의
```

---

## 📌 핵심 정리

```
Persistence Context = 1차 캐시 + 변경 감지
  entitiesByKey: Map<EntityKey, Object> (ID → 엔티티 객체)
  entityEntryContext: Map<EntityKey, EntityEntry> (ID → 상태 + 스냅샷)

1차 캐시 동작
  em.find(): 캐시 우선 → 미스 시 DB 조회
  JPQL: 항상 DB 조회 → 결과를 캐시에 병합
  캐시 우선 원칙: 동일 ID는 항상 동일 객체

엔티티 4가지 상태
  New → persist() → Managed
  Managed → 트랜잭션 종료 → Detached
  Managed → remove() → Removed
  Detached → merge() → Managed (SELECT 발생 가능)

OSIV
  활성화: 요청 범위 Persistence Context (편리, 위험)
  비활성화: 트랜잭션 범위 Persistence Context (권장)
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 트랜잭션에서 `em.find(User.class, 1L)`로 엔티티를 가져온 후, `em.createQuery("SELECT u FROM User u WHERE u.id = 1").getSingleResult()`를 실행했다. 두 번째 쿼리가 DB에서 가져온 최신 데이터로 기존 Managed 엔티티를 덮어쓰는가?

**Q2.** `@Transactional` 서비스 메서드에서 `userRepository.findAll()`로 10,000개를 조회한 후, 수정 없이 메서드를 종료하면 flush()가 발생하는가?

**Q3.** `em.detach(user)`를 호출한 후 해당 `user`의 필드를 수정하면, 이후 `em.merge(user)`를 호출했을 때 UPDATE SQL이 발생하는가? 발생한다면 언제인가?

> 💡 **해설**
>
> **Q1.** 덮어쓰지 않는다. JPQL 결과를 1차 캐시에 병합할 때, 이미 같은 `EntityKey`로 등록된 엔티티가 있으면 DB 결과를 무시하고 캐시의 엔티티를 반환한다. 이를 "Repeatable Read" 보장이라고 하며, 같은 트랜잭션 안에서 동일 엔티티는 항상 동일 객체를 반환한다. DB에서 다른 트랜잭션이 값을 수정했더라도 현재 트랜잭션의 1차 캐시는 처음 로드한 스냅샷을 유지한다. 이 동작은 `FlushMode`와 무관하며 Persistence Context의 근본 특성이다.
>
> **Q2.** `flush()`는 발생하지만 SQL은 실행되지 않는다. 트랜잭션 커밋 전 `flush()`가 호출되어 dirty checking을 수행하지만, 10,000개 엔티티 중 변경된 것이 없으므로 UPDATE SQL이 없다. 단, dirty checking 자체(스냅샷 비교)는 실행되므로 10,000개 × 필드 수만큼의 값 비교가 발생한다. 이것이 `@Transactional(readOnly = true)`가 성능에 도움이 되는 이유다 — `FlushMode.MANUAL`로 설정되어 dirty checking 자체를 생략한다.
>
> **Q3.** UPDATE SQL이 발생하며, 발생 시점은 `flush()` 또는 트랜잭션 커밋 시점이다. `merge()`를 호출하면 1차 캐시에 해당 ID의 엔티티가 없으므로 DB에서 `SELECT`를 실행해 최신 값을 가져온다. 그 후 `detach` 상태 객체의 필드값을 `merge()`가 반환한 Managed 엔티티에 복사한다. 이때 `loadedState`(스냅샷)는 DB에서 가져온 값이고, 복사된 현재 값은 `detach` 후 수정된 값이다. `flush()` 시 dirty checking에서 차이가 감지되어 UPDATE SQL이 생성된다.

---

<div align="center">

**[⬅️ 이전: EntityManager vs Hibernate Session](./01-entitymanager-vs-session.md)** | **[다음: Dirty Checking 메커니즘 ➡️](./03-dirty-checking-mechanism.md)**

</div>
