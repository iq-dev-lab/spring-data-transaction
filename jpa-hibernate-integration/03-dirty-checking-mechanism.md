# Dirty Checking 메커니즘 — 변경 감지가 일어나는 내부 코드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `loadedState`(스냅샷)는 언제 생성되고, 어떤 자료구조로 저장되는가?
- `flush()` 시 dirty checking이 수행되는 정확한 코드 경로는?
- `FlushMode.AUTO`와 `FlushMode.COMMIT`은 flush 발생 시점이 어떻게 다른가?
- `@DynamicUpdate`는 dirty checking과 어떻게 연계되어 UPDATE SQL을 줄이는가?
- save()를 호출하지 않아도 엔티티 변경이 DB에 반영되는 이유는?

---

## 🔍 왜 이게 존재하는가

### 문제: 엔티티를 수정할 때마다 save()를 호출해야 하는가

```java
// save() 없이도 UPDATE가 발생하는 이유를 이해하지 못하면:
@Transactional
public void updateUserName(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName);
    userRepository.save(user); // ← 이 save() 필요한가?
}

// 실제로 save()는 불필요하다 — dirty checking이 자동 처리
@Transactional
public void updateUserName(Long id, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(newName);
    // 트랜잭션 커밋 시 flush() → dirty checking → UPDATE 자동 실행
}
```

```
Dirty Checking의 동작 원리:
  [1] 엔티티 로드 시: loadedState(스냅샷) = 현재 필드값 복사본 저장
  [2] 트랜잭션 커밋 전 flush() 호출
  [3] Managed 엔티티 순회 → 현재값 vs loadedState 비교
  [4] 다르면 → UPDATE SQL 생성 및 실행
  [5] 같으면 → SQL 없음
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Transactional 없으면 dirty checking이 작동하지 않는다

```java
// ❌ 잘못된 이해: 트랜잭션이 없어서 dirty checking도 없다

// ✅ 실제:
// Dirty checking은 Persistence Context(EntityManager)에 종속
// @Transactional 없이 EntityManager가 열려있어도 동작
// 하지만 flush()/commit()이 없으면 DB에 반영되지 않을 뿐

// OSIV 환경: 트랜잭션 종료 후에도 EntityManager 열려있음
// → 하지만 write-only 메서드 호출 불가 (Transaction 필요)
// → 보통 트랜잭션 종료 시 flush + close
```

### Before: Hibernate는 변경된 필드만 UPDATE한다

```java
// ❌ 잘못된 이해: user.setName()만 하면 name 컬럼만 UPDATE

// 기본 동작: 전체 컬럼 UPDATE
// UPDATE users SET name=?, email=?, age=?, status=?, ... WHERE id=?
// → 변경하지 않은 컬럼도 포함 (loadedState와 다른 필드만 골라내는 것은 SQL 아닌 dirty check)

// ✅ 변경된 컬럼만 UPDATE하려면 @DynamicUpdate 필요
@Entity
@DynamicUpdate  // ← 추가
public class User {
    // ...
}
// → UPDATE users SET name=? WHERE id=?  (변경된 name만)
```

### Before: save()를 호출해야 UPDATE가 발생한다

```java
// ❌ Managed 상태 엔티티에 save() 중복 호출
@Transactional
public User updateUser(Long id, String name) {
    User user = userRepository.findById(id).orElseThrow(); // Managed
    user.setName(name);
    return userRepository.save(user); // 불필요한 save() — merge() 내부 로직 수행
    // SimpleJpaRepository.save(): isNew() 확인 → false → em.merge() 호출
    // merge(): 이미 Managed 상태이면 그냥 반환 (추가 쿼리 없음)
    // 단, merge() 호출 자체의 약간의 오버헤드 존재
}

// ✅ Managed 상태에서는 save() 불필요
@Transactional
public User updateUser(Long id, String name) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(name);
    return user; // 트랜잭션 종료 시 자동 flush → UPDATE
}
```

---

## ✨ 올바른 이해와 패턴

### After: Dirty Checking 전체 흐름

```
em.find() / JPQL 실행
  → 엔티티 로드
  → EntityEntry 생성: loadedState = Object[] {필드값들 복사본}
  → entitiesByKey에 등록 (Managed 상태)

엔티티 필드 수정 (user.setName("새이름"))
  → 단순 Java 필드 수정
  → Hibernate 개입 없음 (일반 setter 호출)

flush() 호출 (트랜잭션 커밋 직전 또는 FlushMode.AUTO에서 JPQL 실행 전)
  → DefaultFlushEntityEventListener.onFlushEntity()
  → currentState = persister.getPropertyValues(entity) (현재 필드값)
  → loadedState = entityEntry.getLoadedState()
  → 값 비교: Arrays.deepEquals(currentState, loadedState)
  → 다른 필드 있음 → EntityUpdateAction 생성 → ActionQueue에 추가

ActionQueue.executeActions()
  → SQL 생성 및 실행: UPDATE users SET ... WHERE id=?
  → loadedState 갱신 (현재값으로 업데이트)
```

---

## 🔬 내부 동작 원리 — Dirty Checking 소스 추적

### 1. 엔티티 로드 시 loadedState 저장

```java
// AbstractEntityPersister.hydrate() — 엔티티 로드 시 스냅샷 생성
public Object[] hydrate(ResultSet rs, Object id, Object object,
                         Loadable rootLoadable, ...) {

    // DB에서 읽어온 각 컬럼 값을 Java 타입으로 변환
    Object[] values = new Object[getPropertySpan()];
    for (int i = 0; i < getPropertySpan(); i++) {
        values[i] = getPropertyTypes()[i].hydrate(rs, aliases[i], session, object);
    }
    return values; // → loadedState로 저장
}

// TwoPhaseLoad.addUninitializedEntity() 이후
// EntityEntry 생성 시 loadedState 등록
EntityEntry entry = persistenceContext.addEntry(
    entity,
    Status.LOADING,
    values,      // ← loadedState = 이 배열
    rowId,
    id,
    version,
    lockMode,
    existsInDatabase,
    persister,
    false
);
```

### 2. DefaultFlushEntityEventListener — Dirty Checking 핵심

```java
// DefaultFlushEntityEventListener.onFlushEntity() — dirty checking 핵심
public class DefaultFlushEntityEventListener implements FlushEntityEventListener {

    @Override
    public void onFlushEntity(FlushEntityEvent event) {

        final Object entity = event.getEntity();
        final EntityEntry entry = event.getEntityEntry();
        final EventSource session = event.getSession();

        // [1] 현재 엔티티의 모든 필드 값 조회
        final Object[] values = entry.getPersister().getPropertyValues(entity);
        // → 리플렉션으로 각 필드 getter 호출

        // [2] 스냅샷 가져오기
        final Object[] loadedState = entry.getLoadedState();

        // [3] @Version 처리 (Optimistic Lock)
        // version 필드가 있으면 자동 증가

        // [4] Dirty checking — 변경된 필드 찾기
        final int[] dirtyProperties = session.getInterceptor().findDirty(
            entity, entry.getId(), values, loadedState,
            entry.getPersister().getPropertyNames(),
            entry.getPersister().getPropertyTypes()
        );
        // dirtyProperties: 변경된 필드의 인덱스 배열
        // null → 변경 없음
        // [2, 5] → 인덱스 2, 5번 필드 변경됨

        // [5] 변경 있으면 UPDATE 예약
        if (dirtyProperties != null && dirtyProperties.length > 0) {
            event.setDirtyProperties(dirtyProperties);
            // → ActionQueue에 EntityUpdateAction 추가
        }
    }
}

// TypeHelper.findDirty() — 실제 값 비교
public static int[] findDirty(Object[] x, Object[] y, Type[] types) {
    int[] results = null;
    int count = 0;

    for (int i = 0; i < types.length; i++) {
        // 타입별 비교 (String: equals, Date: getTime 비교, ...)
        if (!types[i].isSame(x[i], y[i])) {
            if (results == null) results = new int[types.length];
            results[count++] = i;
        }
    }

    return count == 0 ? null : Arrays.copyOf(results, count);
}
```

### 3. FlushMode 별 flush 발생 시점

```java
// FlushMode.AUTO (기본값):
//   1. 트랜잭션 커밋 전
//   2. JPQL/Criteria 쿼리 실행 전 (영향 줄 수 있는 엔티티 변경 있을 때)

// FlushMode.COMMIT:
//   1. 트랜잭션 커밋 전만 (JPQL 실행 전 flush 없음)
//   → 쿼리 일관성 깨질 수 있음 (변경 중인 엔티티와 쿼리 결과 불일치)

// FlushMode.MANUAL:
//   자동 flush 없음 → 명시적 em.flush() 만 (readOnly 트랜잭션에서 설정)

// FlushMode.ALWAYS:
//   모든 쿼리 전 항상 flush (성능에 불리)

// AUTO에서 JPQL 실행 전 flush 조건:
// AutoFlushEventListener가 판단:
//   현재 dirty 엔티티가 쿼리의 대상 테이블에 영향 주는가?
//   → 영향 준다면 flush → DB와 일관성 유지 후 쿼리 실행
//   → 영향 안 준다면 flush 생략 (최적화)
class AutoFlushEventListener {
    public void onAutoFlush(AutoFlushEvent event) {
        Set<String> querySpaces = event.getQuerySpaces(); // 쿼리가 접근하는 테이블들
        // dirty 엔티티의 테이블과 querySpaces 교집합이 있으면 flush
        if (collectionIsEmpty(querySpaces) || persistenceContext.hasNonReadOnlyEntities()) {
            flushEverythingToExecutions(event);
        }
    }
}
```

### 4. @DynamicUpdate — 변경된 컬럼만 UPDATE

```java
// @DynamicUpdate 없는 기본 UPDATE
// UPDATE users SET name=?, email=?, age=?, status=?, created_at=? WHERE id=?
// → 변경한 것이 name뿐이어도 전체 컬럼 포함

// @DynamicUpdate 있는 UPDATE
@Entity
@DynamicUpdate
public class User {
    private String name;
    private String email;
    private int age;
}

// user.setName("새이름")만 변경 시:
// UPDATE users SET name=? WHERE id=?
// → name만 포함

// 내부 동작:
// AbstractEntityPersister.generateUpdateString(int[] dirtyFields)
// → @DynamicUpdate: dirtyFields 인덱스에 해당하는 컬럼만 포함한 SQL 생성 (런타임)
// → 기본: 미리 컴파일된 전체 컬럼 UPDATE SQL 사용 (시작 시 캐시)

// @DynamicUpdate 트레이드오프:
// ✅ 필드 수 많은 엔티티에서 네트워크 + DB 부하 감소
// ✅ 낙관적 락 충돌 감소 (다른 컬럼 수정 시 충돌 없음)
// ❌ 매번 SQL 동적 생성 → PreparedStatement 캐시 활용 어려움
// → 컬럼 수가 많고(20개 이상) 부분 업데이트가 잦을 때 사용 고려
```

### 5. ActionQueue — SQL 실행 순서 제어

```java
// ActionQueue — dirty checking 결과로 쌓인 SQL 실행 관리
public class ActionQueue {

    // 실행 순서: INSERT → UPDATE → DELETE
    private ExecutableList<AbstractEntityInsertAction> insertions;
    private ExecutableList<EntityUpdateAction> updates;
    private ExecutableList<EntityDeleteAction> deletions;
    private ExecutableList<CollectionUpdateAction> collectionUpdates;

    // executeActions() — flush 시 SQL 실행
    public void executeActions() {
        // 1. Orphan deletion (orphanRemoval) 먼저
        // 2. INSERT
        executeActions(insertions);
        // 3. UPDATE
        executeActions(updates);
        // 4. Collection 변경
        executeActions(collectionUpdates);
        // 5. DELETE
        executeActions(deletions);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: save() 없이 UPDATE 발생 확인

```java
@Transactional
@Test
void dirtyCheckingWithoutSave() {
    User user = userRepository.findById(1L).orElseThrow();
    // SELECT 실행, loadedState 저장

    user.setName("새이름"); // loadedState는 "원래이름"

    // save() 호출 없음
    // 트랜잭션 종료 시:
    // flush() → dirty checking → name 변경 감지 → UPDATE 실행

    // hibernate.show_sql=true 출력:
    // Hibernate: select ... from users where id=?
    // Hibernate: update users set name=?, email=?, ... where id=?
}
```

### 실험 2: 변경 없으면 UPDATE 없음

```java
@Transactional
@Test
void noDirtyCheckingWhenUnchanged() {
    User user = userRepository.findById(1L).orElseThrow();
    String originalName = user.getName();
    user.setName(originalName); // 동일한 값으로 "변경"

    // 트랜잭션 종료 시: dirty checking → 변경 없음 → UPDATE 없음
    // hibernate.show_sql: UPDATE 미출력
}
```

### 실험 3: FlushMode.COMMIT vs AUTO 차이

```java
@Test
void flushModeAutoVsCommit() {
    EntityManager em = emf.createEntityManager();
    em.getTransaction().begin();

    User user = em.find(User.class, 1L);
    user.setName("변경됨");
    // FlushMode.AUTO (기본):
    // JPQL 실행 전 flush 가능
    em.createQuery("SELECT u FROM User u").getResultList();
    // → 위 쿼리가 users 테이블을 접근 → 먼저 flush() → UPDATE 실행 후 SELECT

    // FlushMode.COMMIT으로 변경 시:
    // em.setFlushMode(FlushModeType.COMMIT);
    // → JPQL 실행 전 flush 없음 → "변경됨"이 SELECT 결과에 반영 안 될 수 있음

    em.getTransaction().commit(); // flush + commit
}
```

---

## ⚡ 성능 임팩트

```
Dirty Checking 비용:

엔티티 1개당:
  getPropertyValues(): 리플렉션 N번 (N = 필드 수)
  값 비교: N번 타입별 equals()
  합계: ~수 μs per entity

1000개 엔티티 × 20 필드:
  ~20,000 리플렉션 호출 + 값 비교
  → ~수십 ms
  → 대량 처리 시 bottleneck 가능

최적화 방법:
  readOnly=true (@Transactional(readOnly=true)):
    FlushMode.MANUAL → dirty checking 자체 생략
  Bytecode Enhancement (Hibernate):
    setter 호출 시 변경 플래그 설정 → dirty 필드 즉시 파악 (리플렉션 불필요)
    build.gradle: hibernate.bytecode.use_reflection_optimizer=true
  @DynamicUpdate:
    변경된 필드만 UPDATE → 네트워크/DB 부하 감소
    단, SQL 동적 생성으로 PreparedStatement 캐시 효율 감소

flush() 빈도 최적화:
  배치 처리 시 flush/clear 주기적 호출
  추천 단위: 500~1000건마다 flush() + clear()
  → 1차 캐시 크기 제어 + dirty checking 대상 수 제한
```

---

## 🤔 트레이드오프

```
Dirty Checking (자동 변경 감지):
  ✅ save() 호출 없이 변경사항 자동 반영 (편리)
  ✅ 변경 없으면 UPDATE 없음 (효율적)
  ❌ 대량 로드 시 스냅샷 메모리 + dirty checking 비용
  ❌ 의도치 않은 UPDATE 발생 가능 (실수로 필드 수정)

@DynamicUpdate:
  ✅ 변경 필드만 UPDATE → 부분 업데이트 충돌 감소
  ✅ 네트워크 부하 감소 (컬럼 수 많을 때)
  ❌ 동적 SQL 생성 → 쿼리 플랜 캐시 활용 어려움
  → 컬럼 30개 이상, 동시 업데이트 많을 때 고려

FlushMode 선택:
  AUTO (기본): 안전하지만 불필요한 flush 가능
  COMMIT: 성능↑, 쿼리 일관성 위험
  MANUAL: readOnly에서 사용, 완전 수동 제어
```

---

## 📌 핵심 정리

```
loadedState(스냅샷)
  엔티티 로드 시 EntityEntry에 저장 (Object[])
  각 필드의 Java 값 복사본
  flush() 시 현재값과 비교 기준

Dirty Checking 흐름
  flush() → DefaultFlushEntityEventListener
    → getPropertyValues() (현재값)
    → getLoadedState() (스냅샷)
    → 타입별 값 비교 → dirtyProperties[]
    → EntityUpdateAction → ActionQueue → UPDATE SQL

FlushMode
  AUTO: 커밋 전 + JPQL 전 (조건부)
  COMMIT: 커밋 전만
  MANUAL: 수동 flush만 (readOnly)

@DynamicUpdate
  변경된 필드만 UPDATE SQL에 포함
  런타임 SQL 동적 생성 → PreparedStatement 캐시 효율↓
  컬럼 많고 부분 업데이트 잦은 엔티티에 적합
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 메서드 내에서 동일 엔티티를 두 번 수정하면(`user.setName("A")` 후 `user.setName("B")`) UPDATE SQL이 두 번 발생하는가?

**Q2.** Hibernate Bytecode Enhancement를 사용하면 dirty checking 방식이 어떻게 바뀌는가? `loadedState` 스냅샷이 여전히 필요한가?

**Q3.** `@Transactional` 메서드에서 JPQL `UPDATE` 문을 직접 실행(`@Modifying @Query`)했을 때, 이미 1차 캐시에 Managed 상태로 있는 같은 엔티티의 상태는 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** UPDATE SQL은 한 번만 발생한다. 트랜잭션 내에서 몇 번을 수정해도 flush() 시점에 `loadedState`(로드 시 스냅샷 = "원래값")와 현재 값("B")를 비교한다. 중간 변경("A")은 추적하지 않으며, flush() 시점의 최종 값과 스냅샷의 차이만 비교해 UPDATE SQL을 한 번 생성한다. "A"로 변경했다가 다시 "원래값"으로 바꾸면 dirty checking에서 변경 없음으로 판단해 UPDATE가 발생하지 않는다.
>
> **Q2.** Bytecode Enhancement를 사용하면 각 엔티티 클래스의 setter에 "변경 플래그 설정" 코드가 주입된다(`SelfDirtinessTracker` 구현). 필드가 변경될 때마다 변경된 필드 인덱스를 내부 집합에 기록한다. flush() 시 모든 필드를 순회하며 비교하는 대신, 변경 플래그가 있는 필드만 확인하므로 리플렉션 비용이 크게 감소한다. 단, `loadedState` 스냅샷은 여전히 유지된다 — dirty checking 외에도 `@Version` 확인, merge() 시 값 비교 등에 사용되기 때문이다. Bytecode Enhancement는 dirty checking 속도를 개선하지만 스냅샷 메모리는 그대로다.
>
> **Q3.** 1차 캐시의 Managed 엔티티와 DB 사이에 불일치가 발생한다. `@Modifying` JPQL UPDATE는 Persistence Context를 거치지 않고 직접 DB를 수정한다. 이미 1차 캐시에 있는 엔티티의 `loadedState`나 현재 필드값은 갱신되지 않는다. 이 상태로 트랜잭션이 계속되면 JPQL로 수정한 내용을 `em.find()`로 조회해도 캐시에서 오래된 값을 반환한다. 해결책은 `@Modifying(clearAutomatically = true)`를 설정하면 쿼리 실행 후 `em.clear()`가 호출되어 1차 캐시를 비운다. 또는 수동으로 `em.refresh(entity)`로 특정 엔티티를 재로드한다.

---

<div align="center">

**[⬅️ 이전: Persistence Context](./02-persistence-context-first-cache.md)** | **[다음: N+1 문제 완전 해결 ➡️](./04-n-plus-one-complete-solution.md)**

</div>
