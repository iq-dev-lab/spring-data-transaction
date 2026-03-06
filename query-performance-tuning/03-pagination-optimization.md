# Pagination 최적화 전략 — OFFSET 함정과 Cursor 기반 페이징

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `LIMIT/OFFSET` 방식이 대용량 데이터에서 느린 근본적인 이유는?
- Cursor 기반 Pagination이 OFFSET 방식과 다른 SQL을 생성하는 방식은?
- `Page<T>`와 `Slice<T>`의 내부 차이 — CountQuery가 실행되는 시점은?
- `CountQuery`를 분리 최적화하는 방법과 효과는?
- Covering Index가 Pagination 성능에 미치는 영향은?

---

## 🔍 왜 이게 존재하는가

### 문제: OFFSET이 커질수록 성능이 선형으로 저하된다

```sql
-- 1페이지 (빠름)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 0;
-- → 첫 20행 읽기

-- 500페이지 (느림)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 9980;
-- → 10000행을 읽고 버리고 20행 반환
-- → 인덱스가 있어도 9980행 스캔 비용 발생

-- 문제:
-- OFFSET N은 앞 N행을 물리적으로 스캔한 후 버림
-- 데이터 100만 건, 마지막 페이지 접근 → 거의 전체 스캔
```

```
Spring Data JPA의 페이징 방식:

Page<T>:  SELECT + COUNT (*) — 2번의 쿼리
Slice<T>: SELECT LIMIT+1 — 1번의 쿼리 (다음 페이지 존재 여부만 확인)
List<T>:  SELECT — 1번의 쿼리 (전체 개수 없음)

해결:
  Cursor 기반: WHERE id > :lastId LIMIT 20 → O(log N) (인덱스 활용)
  CountQuery 최적화: 복잡한 JOIN 없는 별도 COUNT 쿼리
  Covering Index: 정렬 컬럼을 인덱스에 포함
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Page<T>를 항상 사용하면 된다

```java
// ❌ 모든 페이징에 Page<T> 사용 — 불필요한 COUNT 쿼리
public Page<Order> getOrders(Pageable pageable) {
    return orderRepository.findAll(pageable);
    // SELECT * FROM orders LIMIT ? OFFSET ?     ← 데이터 조회
    // SELECT COUNT(*) FROM orders               ← 전체 개수 조회 (항상)
    // → 무한 스크롤 UI에서는 전체 개수 불필요!
}

// ✅ 무한 스크롤에는 Slice<T>
public Slice<Order> getOrdersForScroll(Pageable pageable) {
    return orderRepository.findAll(pageable);
    // SELECT * FROM orders LIMIT ?+1 OFFSET ?  ← 1개 더 조회해서 다음 페이지 여부 확인
    // COUNT 쿼리 없음 → 50% 쿼리 절감
}
```

### Before: Cursor 기반 페이징은 복잡하므로 OFFSET으로 충분하다

```java
// ❌ 100만 건 데이터에서 OFFSET 페이징
// → 마지막 페이지로 갈수록 기하급수적으로 느려짐
// → 데이터 추가/삭제 시 페이지 경계가 변동 (데이터 누락/중복)

// ✅ Cursor 기반이 필요한 상황:
// - 데이터 100만 건 이상
// - 무한 스크롤 (SNS 피드, 알림 목록)
// - 실시간으로 데이터가 추가되는 목록
```

---

## ✨ 올바른 이해와 패턴

### After: 상황별 페이징 전략

```
데이터 건수 < 10만, 번호 페이지 UI:
  → Page<T> (LIMIT/OFFSET) — 단순하고 충분

데이터 건수 > 100만, 번호 페이지 UI:
  → No-Offset 페이징 (Covering Index 활용)
  → CountQuery 분리 최적화

무한 스크롤:
  → Slice<T> (COUNT 없음)
  → Cursor 기반 (WHERE id > :lastId)

실시간 데이터 피드:
  → Cursor 기반 (페이지 경계 안정적)
```

---

## 🔬 내부 동작 원리 — Spring Data JPA 페이징 소스 추적

### 1. Page<T> — CountQuery 실행 메커니즘

```java
// SimpleJpaRepository.findAll(Pageable pageable)
@Override
public Page<T> findAll(Pageable pageable) {

    if (isUnpaged(pageable)) {
        return new PageImpl<>(findAll()); // 페이징 없음
    }

    return findAll((Specification<T>) null, pageable);
}

// JpaSpecificationExecutor.findAll(Specification, Pageable)
public Page<T> findAll(@Nullable Specification<T> spec, Pageable pageable) {

    TypedQuery<T> query = getQuery(spec, pageable);
    // → JPQL: SELECT e FROM Entity e ORDER BY ? LIMIT ? OFFSET ?

    // Page 반환 시 CountQuery 실행
    return isUnpaged(pageable) ? new PageImpl<>(query.getResultList())
        : readPage(query, getDomainClass(), pageable, spec);
}

// readPage() — COUNT 쿼리 실행
protected <S extends T> Page<S> readPage(TypedQuery<S> query, Class<S> domainClass,
                                           Pageable pageable, @Nullable Specification<S> spec) {
    // [1] 데이터 조회 (LIMIT/OFFSET)
    List<S> content = getPageableQuery(query, pageable).getResultList();

    // [2] 전체 개수 조회 (항상 실행됨, 단 결과가 LIMIT보다 적으면 생략 가능)
    // Spring Data 최적화: 첫 페이지에서 결과가 LIMIT보다 적으면 COUNT 생략
    Supplier<Long> countSupplier = () -> executeCountQuery(
        getCountQuery(spec, domainClass));

    return PageableExecutionUtils.getPage(content, pageable, countSupplier);
}

// PageableExecutionUtils.getPage() — COUNT 생략 최적화
public static <T> Page<T> getPage(List<T> content, Pageable pageable, LongSupplier totalSupplier) {
    // 최적화: content가 한 페이지 미만이면 COUNT 쿼리 생략
    if (pageable.isUnpaged() || pageable.getOffset() == 0) {
        if (content.size() < pageable.getPageSize()) {
            return new PageImpl<>(content, pageable, content.size());
            // → COUNT 쿼리 생략!
        }
    }
    // 그 외 → COUNT 쿼리 실행
    return new PageImpl<>(content, pageable, totalSupplier.getAsLong());
}
```

### 2. CountQuery 분리 최적화

```java
// ❌ JOIN이 포함된 쿼리 — COUNT도 JOIN 포함 → 느림
@Query("SELECT o FROM Order o " +
       "LEFT JOIN FETCH o.items i " +
       "LEFT JOIN FETCH o.customer c " +
       "WHERE o.status = :status")
Page<Order> findByStatus(@Param("status") Status status, Pageable pageable);
// COUNT 쿼리도 자동으로 JOIN 포함:
// SELECT COUNT(o) FROM Order o
//   LEFT JOIN o.items i LEFT JOIN o.customer c
//   WHERE o.status = ?
// → JOIN이 있어 COUNT가 느림

// ✅ countQuery 분리 — JOIN 없는 단순 COUNT
@Query(
    value = "SELECT o FROM Order o " +
            "LEFT JOIN FETCH o.items i " +
            "LEFT JOIN FETCH o.customer c " +
            "WHERE o.status = :status",
    countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status"
    // → JOIN 없는 단순 COUNT → 훨씬 빠름
)
Page<Order> findByStatusOptimized(@Param("status") Status status, Pageable pageable);
```

### 3. No-Offset 페이징 — OFFSET 제거

```java
// ✅ No-Offset 페이징 — WHERE 조건으로 OFFSET 대체
public interface OrderRepository extends JpaRepository<Order, Long> {

    // 다음 페이지: 마지막으로 본 ID보다 작은 것 조회
    @Query("SELECT o FROM Order o " +
           "WHERE o.id < :lastId " +
           "ORDER BY o.id DESC")
    List<Order> findNextPage(
        @Param("lastId") Long lastId,
        Pageable pageable  // LIMIT만 사용, OFFSET은 0
    );
}

// Service에서 사용
public List<Order> getOrders(Long lastOrderId, int pageSize) {
    Pageable pageable = PageRequest.of(0, pageSize); // offset = 0

    if (lastOrderId == null) {
        // 첫 페이지
        return orderRepository.findAll(
            PageRequest.of(0, pageSize, Sort.by(Sort.Direction.DESC, "id"))
        ).getContent();
    }

    return orderRepository.findNextPage(lastOrderId, pageable);
    // SQL: SELECT * FROM orders WHERE id < ? ORDER BY id DESC LIMIT ?
    // → 인덱스(id)를 직접 활용 → O(log N) 탐색
    // OFFSET 없음 → 100만 번째 페이지도 빠름
}
```

### 4. Cursor 기반 — 복합 정렬 처리

```java
// 복합 정렬 (created_at DESC, id DESC) — Cursor 페이징
// created_at만으로는 동일 시각 데이터가 많을 때 중복/누락 발생
// created_at + id 복합 조건으로 안전한 cursor 구현

@Query("SELECT o FROM Order o " +
       "WHERE (o.createdAt < :lastCreatedAt) " +
       "   OR (o.createdAt = :lastCreatedAt AND o.id < :lastId) " +
       "ORDER BY o.createdAt DESC, o.id DESC")
List<Order> findNextPageByCursor(
    @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
    @Param("lastId") Long lastId,
    Pageable pageable
);

// 또는 QueryDSL로 더 깔끔하게
public List<Order> findNextPage(OrderCursor cursor, int pageSize) {
    BooleanExpression cursorCondition = cursor == null ? null :
        order.createdAt.lt(cursor.getCreatedAt())
            .or(order.createdAt.eq(cursor.getCreatedAt())
                .and(order.id.lt(cursor.getId())));

    return queryFactory
        .selectFrom(order)
        .where(cursorCondition)
        .orderBy(order.createdAt.desc(), order.id.desc())
        .limit(pageSize)
        .fetch();
}

// Cursor DTO
@Value
public class OrderCursor {
    LocalDateTime createdAt;
    Long id;

    public static OrderCursor from(Order lastOrder) {
        return new OrderCursor(lastOrder.getCreatedAt(), lastOrder.getId());
    }
}
```

### 5. Slice<T> 내부 동작

```java
// Slice<T> — LIMIT+1 트릭으로 다음 페이지 여부 확인
// SimpleJpaRepository에서 Slice 반환 시:

TypedQuery<T> query = getQuery(spec, pageable);
int pageSize = pageable.getPageSize();

// LIMIT을 pageSize+1로 설정
query.setMaxResults(pageSize + 1);  // ← 1개 더 조회

List<T> resultList = query.getResultList();

boolean hasNext = resultList.size() > pageSize;  // 1개 더 있으면 다음 페이지 존재
if (hasNext) {
    resultList = resultList.subList(0, pageSize); // 실제 반환은 pageSize개
}

return new SliceImpl<>(resultList, pageable, hasNext);
// → COUNT 쿼리 없음!
// → hasNext()만 제공, getTotalElements() 없음
```

---

## 💻 실험으로 확인하기

### 실험 1: OFFSET 성능 측정

```java
@Test
void offsetPerformanceDegradation() {
    // 100만 건 데이터 가정
    long start1 = System.currentTimeMillis();
    orderRepository.findAll(PageRequest.of(0, 20)); // 1페이지
    System.out.println("1페이지: " + (System.currentTimeMillis() - start1) + "ms");
    // → ~수 ms

    long start2 = System.currentTimeMillis();
    orderRepository.findAll(PageRequest.of(49999, 20)); // 50000페이지 (OFFSET 999980)
    System.out.println("50000페이지: " + (System.currentTimeMillis() - start2) + "ms");
    // → ~수 초 (OFFSET이 커서 느림)
}
```

### 실험 2: COUNT 생략 최적화 확인

```java
@Test
void countQueryOptimization() {
    // 결과가 pageSize보다 적은 첫 페이지
    Page<Order> page = orderRepository.findAll(PageRequest.of(0, 100));
    // 데이터 5건만 있으면:
    // SELECT * FROM orders LIMIT 100 OFFSET 0 → 5건 반환
    // COUNT 쿼리 생략! (5 < 100)

    // 로그에서 COUNT 쿼리가 없음을 확인
    // Hibernate SQL: select o.* from orders o limit 100
    // (count 쿼리 없음)
}
```

### 실험 3: No-Offset vs OFFSET 쿼리 비교

```sql
-- EXPLAIN ANALYZE로 성능 비교 (PostgreSQL)

-- OFFSET 방식 (10만 번째 페이지)
EXPLAIN ANALYZE
SELECT * FROM orders ORDER BY id DESC LIMIT 20 OFFSET 1999980;
-- → Seq Scan 또는 Index Scan 2000000행 → 느림

-- No-Offset 방식
EXPLAIN ANALYZE
SELECT * FROM orders WHERE id < 1000000 ORDER BY id DESC LIMIT 20;
-- → Index Scan (id 인덱스) → 빠름
-- Rows Removed by Filter: 0 (스캔 최소화)
```

---

## ⚡ 성능 임팩트

```
100만 건 orders 테이블, id 인덱스 있음, 정렬 기준 created_at:

1페이지 (OFFSET 0):
  LIMIT/OFFSET: ~1ms
  No-Offset:    ~1ms (차이 없음)

10000페이지 (OFFSET 199980):
  LIMIT/OFFSET: ~500ms (20만 행 스캔)
  No-Offset:    ~2ms (인덱스 직접 접근)
  → 250배 차이

CountQuery 분리 효과 (JOIN 쿼리):
  JOIN 포함 COUNT: ~200ms
  JOIN 없는 COUNT: ~10ms
  → 20배 차이

Page<T> vs Slice<T>:
  Page<T>:  2번 쿼리 (데이터 + COUNT)
  Slice<T>: 1번 쿼리 (데이터만, LIMIT+1)
  → 무한 스크롤에서 50% 쿼리 절감

PageableExecutionUtils.getPage() 최적화:
  첫 페이지, 결과 < pageSize → COUNT 생략
  → 소량 데이터 첫 페이지에서 추가 50% 절감
```

---

## 🤔 트레이드오프

```
LIMIT/OFFSET (Page<T>):
  ✅ 구현 단순, 번호 페이지 UI 자연스러움
  ✅ 임의 페이지 이동 가능 (7페이지로 바로 이동)
  ❌ 대용량 데이터에서 성능 저하 (선형)
  ❌ 데이터 추가/삭제 시 페이지 경계 변동
  → 소규모 데이터, 관리자 페이지

Cursor 기반 (No-Offset):
  ✅ 대용량 데이터에서도 O(log N) 성능
  ✅ 실시간 데이터 추가에도 안정적
  ❌ 임의 페이지 이동 불가 (다음/이전만)
  ❌ 복합 정렬 시 Cursor 구성 복잡
  → 무한 스크롤, SNS 피드, 알림 목록

Slice<T>:
  ✅ COUNT 쿼리 없음 → 성능 향상
  ❌ 전체 개수 모름 (총 페이지 수 표시 불가)
  → 무한 스크롤, 더 보기 버튼

CountQuery 분리:
  ✅ JOIN 없는 단순 COUNT → 빠름
  ❌ 쿼리 두 곳 관리 (데이터 쿼리와 COUNT 쿼리 동기화 필요)
  → 복잡한 JOIN이 있는 페이징 쿼리에 적용
```

---

## 📌 핵심 정리

```
OFFSET 성능 문제
  OFFSET N → N행 물리 스캔 후 버림
  데이터 100만 건, 마지막 페이지 → 거의 전체 스캔
  인덱스가 있어도 OFFSET 이전 행은 스캔 필요

Page<T> COUNT 생략 최적화
  PageableExecutionUtils.getPage():
    첫 페이지 + 결과 < pageSize → COUNT 생략
  countQuery 분리: JOIN 없는 단순 COUNT
  → Fetch Join과 함께 쓸 때 필수

No-Offset 페이징
  WHERE id < :lastId LIMIT ? (OFFSET 없음)
  → 인덱스 직접 활용 → O(log N)
  → 대용량 + 무한 스크롤에 적합
  복합 정렬: (createdAt, id) 복합 조건 필요

Slice<T> vs Page<T>
  Page: 전체 개수 있음 (COUNT 쿼리)
  Slice: 다음 페이지 여부만 (LIMIT+1 트릭)
  → 번호 페이지 → Page, 무한 스크롤 → Slice
```

---

## 🤔 생각해볼 문제

**Q1.** `PageableExecutionUtils.getPage()`의 COUNT 생략 최적화는 `offset = 0`인 첫 페이지에서만 동작한다고 했다. 두 번째 페이지(`offset > 0`)에서 결과가 pageSize보다 적을 때 COUNT가 생략되지 않는 이유는?

**Q2.** No-Offset 페이징에서 `WHERE id < :lastId ORDER BY id DESC`를 사용할 때, 사용자가 임의의 페이지(예: 50페이지)로 바로 이동하고 싶다면 어떻게 구현할 수 있는가?

**Q3.** Cursor 기반 페이징에서 `(createdAt DESC, id DESC)` 복합 정렬을 사용할 때, DB 인덱스를 어떻게 설계해야 최적 성능이 나는가?

> 💡 **해설**
>
> **Q1.** 두 번째 페이지 이후(`offset > 0`)에서 결과가 pageSize보다 적다면, 그것이 마지막 페이지임을 알 수 있다. 하지만 `totalElements`를 정확히 알려면 이전 페이지들의 데이터도 고려해야 한다. 예를 들어 pageSize=10, offset=20이고 결과가 5개라면 totalElements는 25인데, COUNT 없이는 이를 알 수 없다. `PageableExecutionUtils`는 `offset == 0`일 때만 `content.size()`로 totalElements를 추정할 수 있다 — 이때는 "전체 데이터가 한 페이지 안에 들어왔다"고 확신할 수 있기 때문이다. offset이 있으면 totalElements = offset + content.size()로 계산하면 되지 않냐고 생각할 수 있지만, Spring Data는 보수적으로 이 경우에도 COUNT를 실행한다.
>
> **Q2.** No-Offset 방식은 기본적으로 임의 페이지 이동을 지원하지 않는다. 우회 방법은 두 가지다. (1) 50페이지의 시작 ID를 미리 계산: `SELECT id FROM orders ORDER BY id DESC LIMIT 1 OFFSET 980`으로 980번째 ID를 구한 뒤 그 ID를 Cursor로 사용한다 — 이 쿼리 자체는 여전히 OFFSET 문제가 있지만 일회성 조회이고, 이후 페이지 이동은 Cursor 방식이다. (2) 별도 인덱스 테이블: 일정 간격(예: 100페이지마다)으로 Checkpoint ID를 저장해두고 가장 가까운 Checkpoint에서 Cursor 방식으로 탐색한다. 실무에서 SNS 앱들이 "마지막 30페이지까지만 이동 가능"같은 제한을 두는 것도 이 이유다.
>
> **Q3.** `(created_at DESC, id DESC)` 복합 정렬에 최적인 인덱스는 `CREATE INDEX idx_orders_created_id ON orders (created_at DESC, id DESC)`다. 이 인덱스는 정렬 방향이 쿼리와 일치하므로 별도의 정렬 작업(filesort) 없이 인덱스를 순서대로 읽는다. Cursor 조건인 `(created_at < :lastCreatedAt) OR (created_at = :lastCreatedAt AND id < :lastId)`도 이 인덱스로 효율적으로 처리된다. PostgreSQL의 경우 인덱스 컬럼 방향을 명시하고(`DESC`), MySQL InnoDB는 내림차순 인덱스를 8.0부터 지원하므로 버전에 따라 설계가 달라진다. 인덱스에 SELECT 컬럼까지 포함하면(Covering Index) 테이블 랜덤 접근 없이 인덱스만으로 쿼리 해결이 가능해 더욱 빠르다.

---

<div align="center">

**[⬅️ 이전: Fetch Join vs @EntityGraph](./02-fetch-join-vs-entity-graph.md)** | **[다음: @BatchSize vs default_batch_fetch_size ➡️](./04-batch-size-configuration.md)**

</div>
