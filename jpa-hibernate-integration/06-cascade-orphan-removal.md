# Cascade 타입과 Orphan Removal — 연관관계 전파의 설계 원칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CascadeType` 6가지의 내부 동작 원리와 각각이 전파하는 JPA 이벤트는?
- `orphanRemoval=true`와 `CascadeType.REMOVE`의 차이는 무엇인가?
- `CascadeType.ALL`을 무분별하게 사용하면 어떤 위험이 생기는가?
- 연관관계 설계에서 Cascade를 적용해야 하는 기준은?
- `@ManyToMany`에 `CascadeType.REMOVE`를 설정하면 어떤 일이 벌어지는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 부모 엔티티 조작 시 자식 엔티티도 함께 처리해야 한다

```java
// Cascade 없는 경우 — 모든 자식을 수동으로 처리
Order order = new Order();
order.setStatus("PENDING");

OrderItem item1 = new OrderItem("상품A", 2);
OrderItem item2 = new OrderItem("상품B", 1);

// 각 엔티티를 개별적으로 persist
em.persist(item1);
em.persist(item2);
em.persist(order);
// 순서도 신경 써야 함 (FK 제약)

// Cascade 있는 경우 — 부모만 처리
@OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
private List<OrderItem> items;

order.addItem(item1);
order.addItem(item2);
em.persist(order); // → item1, item2도 자동 persist
```

```
Cascade: JPA 엔티티 연산(persist, merge, remove 등)을 연관 엔티티에 전파
  부모-자식 관계에서 부모 조작 → 자식 자동 조작
  생명주기가 강하게 결합된 엔티티에 적합

주의:
  모든 연관에 Cascade = 위험
  독립 생명주기를 가진 엔티티에는 적용 금지
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: CascadeType.ALL을 항상 설정한다

```java
// ❌ 무분별한 ALL 사용
@ManyToOne(cascade = CascadeType.ALL)  // 절대 금지!
private User user;

// 문제: Order를 삭제하면 User도 삭제됨!
em.remove(order); // → order 삭제 + user 삭제!
// User는 독립 엔티티 — Order와 함께 삭제되어서는 안 됨

// ❌ @ManyToMany + CascadeType.REMOVE
@ManyToMany(cascade = CascadeType.REMOVE)
private List<Tag> tags;

// Article 삭제 → 모든 Tag 삭제
// 다른 Article이 참조하던 Tag도 삭제됨 → 데이터 손상!

// ✅ Cascade 적용 기준:
// "이 연관 엔티티가 부모 없이 존재 의미가 있는가?"
// Yes → Cascade 금지 (독립 생명주기)
// No → Cascade 고려 (종속 생명주기)

// OrderItem은 Order 없이 의미 없음 → Cascade 적합
// User는 Order 없이도 존재 → Cascade 금지
// Tag는 Article 없이도 존재 → Cascade 금지
```

### Before: orphanRemoval=true와 CascadeType.REMOVE는 같다

```java
// ❌ 잘못된 이해: 두 가지가 동일하다

// CascadeType.REMOVE: 부모 remove 시 자식도 remove
em.remove(order); // → OrderItem들도 삭제

// orphanRemoval=true: 부모와의 연관이 끊어진 자식을 삭제
order.getItems().remove(item); // 컬렉션에서 제거 → item이 고아 → 자동 삭제

// 차이:
// CascadeType.REMOVE: 부모 삭제 시에만 자식 삭제
// orphanRemoval: 컬렉션에서 제거(고아 발생) 시 자식 삭제 (부모 삭제와 무관)
// orphanRemoval=true는 CascadeType.REMOVE의 효과도 포함
```

---

## ✨ 올바른 이해와 패턴

### After: Cascade 적용 기준

```
Cascade 적용 조건 (모두 만족해야):
  [1] 부모-자식 관계 (자식이 부모 없이 존재 의미 없음)
  [2] 항상 부모를 통해서만 자식을 관리
  [3] 자식이 다른 엔티티와 공유되지 않음

적합한 예:
  Order → OrderItem: ✅ (OrderItem은 Order의 일부)
  Post → Comment:    ✅ (Comment는 Post에 종속)
  User → Address:    ✅ (Address는 User의 일부)

부적합한 예:
  Order → User:      ❌ (User는 독립 엔티티)
  Article → Tag:     ❌ (Tag는 여러 Article에서 공유)
  Team → Member:     ❌ (Member는 여러 Team에 속할 수 있음)

orphanRemoval 적합:
  위 Cascade 조건 만족 + 컬렉션에서 직접 제거로 삭제 처리할 때
  → @OneToMany + orphanRemoval=true + CascadeType.PERSIST/MERGE
```

---

## 🔬 내부 동작 원리 — Cascade 소스 추적

### 1. CascadeType 6가지 동작 원리

```java
// CascadeType별 전파되는 JPA 이벤트

// CascadeType.PERSIST
// em.persist(parent) → parent의 연관(children)에도 persist 전파
// DefaultPersistEventListener.cascadeBeforeSave() 호출
// → CascadingAction.PERSIST.cascade(eventSource, child)

@OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
private List<OrderItem> items;

Order order = new Order();
order.addItem(new OrderItem("A", 1));
order.addItem(new OrderItem("B", 2));
em.persist(order);
// → INSERT INTO orders ...
// → INSERT INTO order_items ... (×2)

// CascadeType.MERGE
// em.merge(parent) → 연관 엔티티에도 merge 전파
// Detached 상태의 부모를 merge할 때 자식도 merge됨

@OneToMany(cascade = CascadeType.MERGE)
private List<OrderItem> items;

// detachedOrder.items에 변경 후:
Order managed = em.merge(detachedOrder);
// → order merge + items merge (각 item의 변경사항 반영)

// CascadeType.REMOVE
// em.remove(parent) → 연관 엔티티에도 remove 전파
// DefaultDeleteEventListener.deleteEntity() 내 cascade

@OneToMany(mappedBy = "order", cascade = CascadeType.REMOVE)
private List<OrderItem> items;

em.remove(order);
// → DELETE FROM order_items WHERE order_id=?
// → DELETE FROM orders WHERE id=?
// (순서: 자식 먼저 삭제 후 부모 삭제 → FK 제약 준수)

// CascadeType.REFRESH
// em.refresh(parent) → DB에서 최신 상태로 재로드, 연관도 재로드

// CascadeType.DETACH
// em.detach(parent) → 연관도 Detached 상태로 전환

// CascadeType.ALL
// 위 6가지 모두 포함
// @OneToMany(cascade = CascadeType.ALL) = PERSIST + MERGE + REMOVE + REFRESH + DETACH
```

### 2. Cascade 전파 내부 코드

```java
// CascadeStyles — Cascade 전파 여부 판단
public abstract class CascadeStyles {

    // 각 CascadeType이 전파하는 Action
    private static final CascadeStyle PERSIST_STYLE = new BaseCascadeStyle() {
        @Override
        public boolean doCascade(CascadingAction action) {
            return action == CascadingAction.PERSIST
                || action == CascadingAction.PERSIST_ON_FLUSH;
        }
    };
}

// Cascade.cascade() — 실제 전파 실행
public class Cascade {

    public static void cascade(
            CascadingAction action,
            CascadePoint cascadePoint,
            EventSource session,
            EntityPersister persister,
            Object parent) {

        // 이 엔티티의 모든 연관관계 순회
        for (int i = 0; i < persister.countSubclassProperties(); i++) {
            Type type = persister.getSubclassPropertyType(i);

            if (type.isAssociationType()) {
                AssociationType associationType = (AssociationType) type;

                // 이 연관에 해당 Cascade가 설정되어 있는가?
                if (cascadeStyle.doCascade(action)) {
                    Object child = persister.getPropertyValue(parent, i);
                    // child에 action 전파
                    action.cascade(session, child, parent, ...);
                }
            }
        }
    }
}
```

### 3. orphanRemoval 동작 원리

```java
// orphanRemoval=true 설정
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;

// 케이스 1: 컬렉션에서 제거
@Transactional
public void removeItem(Long orderId, Long itemId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.getItems().removeIf(item -> item.getId().equals(itemId));
    // → flush 시: OrderItem이 Order의 컬렉션에서 제거됨 → 고아 발생
    // → DELETE FROM order_items WHERE id=?
    // orphanRemoval이 없으면: 제거해도 DB에서 삭제 안 됨 (FK만 null로 변경 시도)
}

// 케이스 2: 부모 삭제
@Transactional
public void deleteOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    orderRepository.delete(order);
    // → orphanRemoval: ORDER의 items가 모두 고아 → 자동 삭제
    // → CascadeType.REMOVE도 있으면: 같은 효과
}

// 내부 동작:
// DefaultFlushEntityEventListener가 컬렉션 변경 감지
// → 컬렉션에서 제거된 항목 → OrphanRemovalAction 등록
// → flush 시: DELETE 실행

// AbstractCollectionPersister — 고아 감지
// collection.wasInitialized() && !collection.isInverseCollection()
// → deletedEntries = oldEntries - currentEntries
// → 차집합이 고아 → remove 처리
```

### 4. CascadeType.REMOVE의 삭제 순서

```java
// 삭제 순서: 자식 → 부모 (FK 제약 준수)

// 예: Order → OrderItem (order_items.order_id FK)
em.remove(order);

// Hibernate ActionQueue 실행 순서:
// 1. 컬렉션 원소 삭제: DELETE FROM order_items WHERE order_id=?
// 2. 부모 삭제: DELETE FROM orders WHERE id=?

// 단, Cascade REMOVE는 1차 캐시에 로드된 자식만 삭제
// 컬렉션이 Lazy이고 아직 초기화 안 됐으면:
Order order = em.find(Order.class, 1L);
// order.items는 아직 Lazy (초기화 안 됨)
em.remove(order);
// → Hibernate: items 컬렉션을 로드 후 각 item에 remove 전파
// (컬렉션 초기화 → cascade remove 전파)
// → SELECT order_items WHERE order_id=1
// → DELETE FROM order_items WHERE id=? (각 item마다)
// → DELETE FROM orders WHERE id=1
```

### 5. @ManyToMany Cascade 위험 패턴

```java
// ❌ 위험한 @ManyToMany CascadeType.REMOVE
@Entity
public class Article {
    @ManyToMany(cascade = CascadeType.REMOVE)  // 절대 금지!
    @JoinTable(name = "article_tag",
               joinColumns = @JoinColumn(name = "article_id"),
               inverseJoinColumns = @JoinColumn(name = "tag_id"))
    private List<Tag> tags;
}

// Article 삭제 시:
em.remove(article);
// → Cascade REMOVE → tags의 각 Tag에 remove 전파
// → DELETE FROM tags WHERE id=? (Tag 자체가 삭제됨!)
// 다른 Article이 사용하던 Tag도 삭제 → 데이터 손상

// ✅ @ManyToMany 올바른 설정
@Entity
public class Article {
    @ManyToMany  // Cascade 없음
    @JoinTable(...)
    private List<Tag> tags;
}

// Article 삭제 시 중간 테이블(article_tag)만 삭제
// Tag 자체는 보존
// → article_tag 레코드: JPA가 자동으로 정리 (연관 테이블이므로)
```

---

## 💻 실험으로 확인하기

### 실험 1: orphanRemoval vs CascadeType.REMOVE 차이

```java
@Test
@Transactional
void orphanRemovalVsCascadeRemove() {
    // orphanRemoval=true 케이스
    Order order = orderRepository.findById(1L).get();
    OrderItem itemToRemove = order.getItems().get(0);

    order.getItems().remove(itemToRemove); // 컬렉션에서 제거
    // → flush 시: DELETE FROM order_items WHERE id=?
    // orphanRemoval 없으면: DELETE 안 됨 (FK null로 변경 시도 또는 예외)

    // CascadeType.REMOVE만 있는 케이스
    // → 컬렉션에서 제거해도 DELETE 안 됨
    // → em.remove(order) 해야 자식 삭제
}

// hibernate.show_sql=true 출력:
// orphanRemoval: Hibernate: delete from order_items where id=?
// CascadeType.REMOVE only: SQL 없음 (컬렉션 제거만으로는)
```

### 실험 2: CascadeType.ALL + @ManyToOne 위험 확인

```java
// ❌ 위험한 설정 테스트
@ManyToOne(cascade = CascadeType.ALL)  // 실수
private User user;

@Test
@Transactional
void cascadeAllOnManyToOneDanger() {
    Order order = orderRepository.findById(1L).get();
    Long userId = order.getUser().getId();

    orderRepository.delete(order); // Order 삭제
    // CascadeType.REMOVE → User도 삭제됨!

    assertFalse(userRepository.existsById(userId)); // User가 삭제됨 (위험!)
}
```

### 실험 3: 올바른 Cascade 사용 — Order + OrderItem

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order",
               cascade = {CascadeType.PERSIST, CascadeType.MERGE},
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Test
@Transactional
void correctCascadeUsage() {
    Order order = new Order();
    order.addItem(new OrderItem("A", 1));
    orderRepository.save(order); // → INSERT order + INSERT order_item

    // 아이템 추가
    order.addItem(new OrderItem("B", 2));
    // → 트랜잭션 종료 시 INSERT order_item (dirty checking)

    // 아이템 제거
    order.removeItem(order.getItems().get(0));
    // → 트랜잭션 종료 시 DELETE order_item (orphanRemoval)
}
```

---

## ⚡ 성능 임팩트

```
CascadeType.REMOVE 성능:
  자식 엔티티를 1차 캐시에 로드 후 각각 remove
  자식 100개: SELECT (로드) + DELETE × 100
  → 대량 자식 삭제 시 성능 문제

  최적화: @Query("DELETE FROM OrderItem i WHERE i.order.id = :orderId")
  → 단일 DELETE 쿼리 (단, orphanRemoval과 충돌 주의)
  → @Modifying + clearAutomatically=true 필수

orphanRemoval 비용:
  컬렉션 변경 감지 → 고아 식별 → DELETE
  대량 제거 시: 개별 DELETE N번 발생
  대량 제거: JPQL DELETE 사용 후 em.clear()

Cascade PERSIST 비용:
  부모 persist → 자식 순회 → 각 자식 persist
  INSERT 수: 자식 수만큼 발생
  Batch Insert와 조합 시 성능 향상 (Ch3-07 참조)
```

---

## 🤔 트레이드오프

```
CascadeType.PERSIST + CascadeType.MERGE (권장 조합):
  ✅ 부모 저장 시 자식 자동 저장
  ✅ Detached 병합 시 자식도 병합
  ❌ REMOVE 없음 → 삭제는 명시적 처리 (안전)
  → 대부분의 부모-자식 관계에 적합

orphanRemoval=true + CascadeType.PERSIST/MERGE:
  ✅ 컬렉션 관리로 자식 생명주기 제어
  ✅ 자연스러운 도메인 모델 (order.removeItem())
  ❌ 컬렉션 로드 없이는 동작 안 함
  → 컬렉션 수가 작고 항상 로드되는 경우

CascadeType.ALL:
  ✅ 편의성 최대
  ❌ REMOVE 포함 → 의도치 않은 삭제 위험
  → @OneToMany에서 주의하여 사용
  → @ManyToOne에는 절대 사용 금지
  → @ManyToMany에는 절대 사용 금지

연관 삭제 전략:
  자식 수 적음 (< 100): Cascade REMOVE 또는 orphanRemoval
  자식 수 많음 (> 1000): JPQL DELETE + @Modifying
```

---

## 📌 핵심 정리

```
CascadeType 6가지
  PERSIST:  부모 persist → 자식 persist
  MERGE:    부모 merge → 자식 merge
  REMOVE:   부모 remove → 자식 remove (자식 먼저 삭제)
  REFRESH:  부모 refresh → 자식 refresh
  DETACH:   부모 detach → 자식 detach
  ALL:      위 6가지 모두

orphanRemoval vs CascadeType.REMOVE
  REMOVE: em.remove(parent) 시에만 전파
  orphanRemoval: 컬렉션에서 제거(고아) 시에도 삭제
  orphanRemoval=true는 REMOVE 효과 포함

Cascade 적용 기준
  자식이 부모 없이 존재 의미 없음 → 적합
  자식이 독립 생명주기 or 공유됨 → 금지
  @ManyToOne cascade = ALL → 절대 금지
  @ManyToMany cascade = REMOVE → 절대 금지

권장 패턴
  @OneToMany: cascade = {PERSIST, MERGE}, orphanRemoval = true
  @ManyToOne: cascade 없음
  @ManyToMany: cascade 없음 (중간 테이블만 관리)
```

---

## 🤔 생각해볼 문제

**Q1.** `@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)`에서 부모를 삭제할 때 내부적으로 어떤 순서로 DELETE가 실행되는가? 자식 컬렉션이 Lazy이고 아직 초기화되지 않았다면?

**Q2.** `orphanRemoval = true` 설정 후 `order.setItems(new ArrayList<>())`를 호출하면 기존 아이템들이 삭제되는가?

**Q3.** `CascadeType.PERSIST`가 설정된 `@OneToMany` 컬렉션에 이미 Managed 상태의 엔티티를 추가하면 어떻게 되는가? 이미 존재하는 엔티티에 persist를 cascade하면?

> 💡 **해설**
>
> **Q1.** 자식 컬렉션이 Lazy이더라도 Hibernate는 cascade REMOVE를 처리하기 위해 자식을 로드한다. 구체적으로 `DefaultDeleteEventListener`가 `cascade(CascadingAction.DELETE, ...)`를 호출하면서 Lazy 컬렉션을 초기화한다(`SELECT order_items WHERE order_id=?`). 로드된 각 `OrderItem`에 대해 개별 `DELETE`가 실행된다. 마지막으로 부모 `Order`가 삭제된다. `orphanRemoval`도 설정되어 있으면 동일하게 동작한다. 실행 순서: (1) SELECT order_items WHERE order_id=? (컬렉션 초기화), (2) DELETE FROM order_items WHERE id=? (각 item마다), (3) DELETE FROM orders WHERE id=?
>
> **Q2.** 삭제된다. `orphanRemoval=true`에서 컬렉션 자체를 새 리스트로 교체(`setItems(new ArrayList<>())`)하면, Hibernate는 기존 컬렉션의 모든 항목이 새 컬렉션에 없음을 감지한다. 이 항목들은 "고아(orphan)"가 되어 flush 시 DELETE 쿼리가 실행된다. 단, Hibernate가 이를 감지하려면 컬렉션이 로드되어 있어야 한다. Lazy 상태에서 컬렉션 교체는 Hibernate가 변경을 감지하지 못할 수 있으므로 주의가 필요하다. 권장 패턴: 컬렉션을 교체하는 대신 `clear()` 후 `addAll()`을 사용한다.
>
> **Q3.** 이미 Managed 상태의 엔티티에 `cascade PERSIST`가 전파되면 `EntityState.PERSISTENT`임을 감지하고 아무 일도 하지 않는다. `persist()`는 `New` 상태의 엔티티를 `Managed`로 전환하는 것이 목적이며, 이미 `Managed` 상태이면 무시된다. 단, 이미 다른 Persistence Context에서 Detached된 엔티티를 추가하고 `cascade PERSIST`가 전파되면 `EntityExistsException`이 발생할 수 있다(Detached 엔티티에 persist를 시도하면 예외). 이런 경우 `cascade MERGE`가 올바른 선택이다.

---

<div align="center">

**[⬅️ 이전: Lazy Loading 프록시 생성 과정](./05-lazy-loading-proxy.md)** | **[다음: Batch Insert/Update 최적화 ➡️](./07-batch-insert-update.md)**

</div>
