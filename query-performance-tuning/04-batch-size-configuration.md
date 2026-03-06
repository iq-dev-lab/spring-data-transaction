# @BatchSize vs default_batch_fetch_size — IN절 배치 로딩 완전 분석

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@BatchSize`와 `hibernate.default_batch_fetch_size`가 내부적으로 어떻게 IN절을 생성하는가?
- Hibernate가 IN절 파라미터 수를 2의 제곱수로 조정하는 이유는?
- `@BatchSize`와 `default_batch_fetch_size` 중 어느 것이 우선하는가?
- IN절 크기가 너무 크거나 작을 때 발생하는 문제는?
- `@BatchSize`와 Fetch Join을 조합하는 전략은?

---

## 🔍 왜 이게 존재하는가

### 문제: N+1은 Fetch Join으로 해결되지만 컬렉션 2개 이상이면 한계가 있다

```java
// 팀에 멤버와 프로젝트가 있는 경우
@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    private List<User> members;    // N+1 위험

    @OneToMany(mappedBy = "team")
    private List<Project> projects; // N+1 위험
}

// Fetch Join으로 두 컬렉션 동시 로딩 → MultipleBagFetchException!
// → 해결책: @BatchSize로 IN절 배치 로딩
```

```
@BatchSize의 동작 원리:
  팀 10개 조회 → team_id: [1, 2, 3, ..., 10]
  members 접근 시 → SELECT * FROM users WHERE team_id IN (1,2,3,...,10)
  → SQL 1번으로 10개 팀의 멤버 모두 로딩
  → N+1이 1+1(2번)으로 감소

default_batch_fetch_size:
  엔티티 레벨 @BatchSize 없이 전역 적용
  모든 @OneToMany, @ManyToOne에 배치 로딩 자동 적용
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: BatchSize를 크게 설정하면 항상 좋다

```java
// ❌ BatchSize를 1000 이상으로 설정
hibernate.default_batch_fetch_size=1000

// 문제:
// IN절에 파라미터 1000개 → DB 파서 부하 증가
// 일부 DB (Oracle): IN절 최대 1000개 제한 → 예외 발생!
// QueryPlanCache 키가 커짐 → 캐시 효율 저하

// ✅ 적정 범위: 100 ~ 500
hibernate.default_batch_fetch_size=100
// → DB 부하, 캐시 효율, N+1 해소 사이의 균형
```

### Before: @BatchSize는 항상 정확히 size만큼 IN절을 만든다

```java
// ❌ 잘못된 이해: @BatchSize(size=100)이면 항상 IN (?, ?, ...) 100개

// ✅ 실제: Hibernate가 2의 제곱수로 패딩
// 데이터 75개 → 실제 IN절: IN (?, ?, ...) 128개 (75개 이후 null 패딩)
// 이유: QueryPlanCache 재사용 → 파라미터 수가 다르면 다른 캐시 키

// Hibernate 5.x: 2의 제곱수 패딩
// Hibernate 6.x: 개선된 패딩 전략 (더 세밀하게 조정)
```

---

## ✨ 올바른 이해와 패턴

### After: BatchSize 적용 계층과 우선순위

```
우선순위 (높은 것이 우선):

1. @BatchSize(size=N) — 특정 컬렉션/엔티티에 직접 선언
   @OneToMany @BatchSize(size=50)
   private List<User> members;

2. @BatchSize on @Entity — 엔티티 전체에 적용
   @Entity @BatchSize(size=50)
   public class User { ... }

3. hibernate.default_batch_fetch_size — 전역 기본값
   (1, 2가 없는 모든 연관관계에 적용)

적용 범위:
  @OneToMany, @ManyToMany: 컬렉션 초기화 시 IN절
  @ManyToOne, @OneToOne: Lazy 로딩 시 IN절
  (FetchType.LAZY인 경우에만 의미 있음)
```

---

## 🔬 내부 동작 원리 — BatchFetch 소스 추적

### 1. BatchFetch 트리거 — 컬렉션 초기화 시점

```java
// Hibernate 내부: 컬렉션 첫 접근 시 BatchFetch 실행
// PersistentBag.initialize() → BatchFetchQueue에 등록 → IN절 실행

// DefaultBatchLoadContext (Hibernate 6.x)
class BatchEntitySelectFetchInitializer {

    // 엔티티의 Lazy 연관관계 초기화 요청 시
    void initializeInstance(RowProcessingState rowProcessingState) {

        // 현재 세션의 BatchFetchQueue에서 대기 중인 ID 수집
        List<Object> batchIds = session.getBatchFetchQueue()
            .getBatchLoadableEntityIds(entityDescriptor, id, batchSize);
        // → 같은 타입, 아직 로딩 안 된 엔티티 ID 최대 batchSize개 수집

        if (batchIds.size() > 1) {
            // IN절으로 배치 로딩
            loadEntitiesByBatchIds(batchIds);
            // SELECT * FROM users WHERE id IN (1, 2, 3, ..., N)
        } else {
            // 단건 로딩
            loadSingleEntity(id);
        }
    }
}
```

### 2. IN절 파라미터 패딩 — QueryPlanCache 최적화

```java
// Hibernate의 IN절 패딩 전략
// 목적: 파라미터 수가 달라도 같은 QueryPlan 재사용

// 예: batchSize=100, 실제 ID 75개
// → 128까지 null 패딩 (다음 2의 제곱수)
// SQL: WHERE id IN (1, 2, 3, ..., 75, null, null, ..., null [128개])
// → 128개짜리 PreparedStatement는 항상 같은 QueryPlan 사용

// 실제 생성되는 SQL (Hibernate 5.x):
// SELECT * FROM users WHERE id IN (?,?,?,?,?,?,?,?,  [128개])
//   AND id IS NOT NULL  ← null 패딩 필터링

// Hibernate 6.x 개선:
// 패딩을 2의 제곱수 계단식으로:
// [1, 2, 3, 4, 5, 6, 7, 8, 10, 12, 14, 16, 20, 24, 28, 32, 40, ...]
// → 더 세밀한 크기 조정, QueryPlanCache 효율 향상
```

### 3. @BatchSize 위치별 동작 차이

```java
// [방법 1] @OneToMany에 직접 — 컬렉션 배치 로딩
@Entity
public class Team {

    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    @BatchSize(size = 100)  // 이 컬렉션만 배치 로딩
    private List<User> members;

    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    @BatchSize(size = 50)   // 다른 크기 적용 가능
    private List<Project> projects;
}

// [방법 2] @Entity에 — 이 엔티티를 참조하는 Lazy 로딩에 적용
@Entity
@BatchSize(size = 100)  // User를 @ManyToOne으로 참조하는 곳에서 배치 로딩
public class User {
    @ManyToOne(fetch = FetchType.LAZY)
    private Team team;
}
// 주의: @ManyToOne은 @BatchSize 대신 Fetch Join이 더 효율적인 경우 많음

// [방법 3] 전역 설정
// application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
// → @BatchSize 없는 모든 Lazy 연관관계에 자동 적용
// → 가장 간편하고 일반적인 방법
```

### 4. Fetch Join + @BatchSize 조합 전략

```java
// 팀 + 멤버 + 프로젝트 동시 조회 전략

// Repository
public interface TeamRepository extends JpaRepository<Team, Long> {

    // 팀과 멤버: Fetch Join (가장 자주 쓰이는 연관관계)
    @Query("SELECT DISTINCT t FROM Team t LEFT JOIN FETCH t.members WHERE t.status = :status")
    List<Team> findByStatusWithMembers(@Param("status") Status status);
}

// @Entity
@Entity
public class Team {
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    @BatchSize(size = 100)  // projects는 @BatchSize로 IN절 배치 로딩
    private List<Project> projects;

    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    // members는 Fetch Join으로 처리하므로 @BatchSize 불필요
    private List<User> members;
}

// Service
public List<TeamDto> getTeams(Status status) {
    // 1번 쿼리: teams + members (Fetch Join)
    List<Team> teams = teamRepository.findByStatusWithMembers(status);

    // projects 접근 시 (2번 쿼리: @BatchSize IN절)
    // SELECT * FROM projects WHERE team_id IN (1, 2, 3, ..., 100)
    teams.forEach(t -> processProjects(t.getProjects()));

    // 총 2번 쿼리 (N+1 완전 해결)
    return teams.stream().map(TeamDto::from).toList();
}
```

---

## 💻 실험으로 확인하기

### 실험 1: BatchSize 적용 전후 SQL 비교

```java
// 설정 없음 (N+1)
List<Team> teams = teamRepository.findAll(); // SQL 1번
teams.forEach(t -> t.getMembers().size());
// SQL N번: SELECT * FROM users WHERE team_id = 1
//          SELECT * FROM users WHERE team_id = 2
//          ...

// @BatchSize(size=10) 적용 후
@OneToMany @BatchSize(size=10)
private List<User> members;

// 동일 코드 실행:
// SQL 1번: SELECT * FROM users WHERE team_id IN (1,2,3,4,5,6,7,8,9,10)
// SQL 1번: SELECT * FROM users WHERE team_id IN (11,12,...,20)
// → 팀 20개 → SQL 2번으로 처리
```

### 실험 2: IN절 패딩 확인

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

```java
// 팀 7개의 멤버 배치 로딩 시 (@BatchSize(size=10)):
// 예상: IN (?, ?, ?, ?, ?, ?, ?) 7개
// 실제 (Hibernate 5.x): IN (?,?,?,?,?,?,?,?,) 8개 (다음 2의 제곱수)
//   바인딩: (1,2,3,4,5,6,7,null) → null은 필터링
// Hibernate 6.x: 더 정교하게 조정

// SQL 로그로 확인:
// select m.* from users m where m.team_id in(?,?,?,?,?,?,?,?)
// binding parameter [1] as [BIGINT] - [1]
// binding parameter [2] as [BIGINT] - [2]
// ...
// binding parameter [8] as [BIGINT] - [null]
```

### 실험 3: default_batch_fetch_size 전역 효과

```java
// 설정:
// hibernate.default_batch_fetch_size=100

// Team, User, Project 모두 @BatchSize 없어도
// 모든 Lazy 컬렉션에 자동으로 배치 로딩 적용

@Test
void globalBatchSizeEffect() {
    List<Team> teams = teamRepository.findAll(); // SQL 1번
    teams.forEach(t -> {
        t.getMembers().size();    // IN절 배치 (전역 설정)
        t.getProjects().size();   // IN절 배치 (전역 설정)
    });
    // SQL 총 3번 (팀 1 + 멤버 1 + 프로젝트 1)
}
```

---

## ⚡ 성능 임팩트

```
팀 100개, 팀당 멤버 10명, 팀당 프로젝트 5개 조회:

N+1 (BatchSize 없음):
  SQL 201번 (팀 1 + 멤버 100 + 프로젝트 100)
  네트워크 왕복 201번 → 수백 ms

@BatchSize(size=100):
  SQL 3번 (팀 1 + 멤버 IN절 1 + 프로젝트 IN절 1)
  → 201번 → 3번 (98.5% 감소)

Fetch Join (멤버) + @BatchSize(프로젝트):
  SQL 2번 (팀+멤버 JOIN 1 + 프로젝트 IN절 1)
  → 카테시안 곱 없음 (Fetch Join은 멤버만)

BatchSize 크기별 성능:
  size=10 : 팀 100개 → 10번 쿼리 (100/10)
  size=100: 팀 100개 → 1번 쿼리 (100/100)
  size=1000: 팀 100개 → 1번 쿼리 (같음, but IN 1000개 파싱 부하)
  권장: size=100~500 (DB 부하와 쿼리 수의 균형)

QueryPlanCache 영향:
  패딩 없이 가변 IN절 → 캐시 미스 → 매번 파싱
  패딩 있는 고정 크기 IN절 → 캐시 히트 → 파싱 생략
  → 패딩이 성능 향상에 기여
```

---

## 🤔 트레이드오프

```
@BatchSize vs Fetch Join:
  @BatchSize: 컬렉션 2개 이상 안전, N+K번 쿼리 (K=컬렉션 수)
  Fetch Join: 1번 쿼리, 카테시안 곱 위험, Pagination 제한
  → 단순 단일 컬렉션 → Fetch Join
  → 복수 컬렉션 → @BatchSize (또는 1개 Fetch Join + 나머지 @BatchSize)

@BatchSize vs DTO Projection:
  @BatchSize: 엔티티 그래프 탐색 가능, dirty checking 비용
  DTO Projection: 필요한 데이터만 조회, 읽기 최적화
  → 도메인 로직 필요 → @BatchSize
  → 조회 전용, 성능 최우선 → DTO Projection

BatchSize 크기 선택:
  너무 작음 → 여러 번 IN절 쿼리 (N+1 완전 해결 안 됨)
  적당함 → 1~2번 IN절 (권장: 100)
  너무 큼 → IN절 파싱 부하, Oracle 1000개 제한
  → 실제 사용 데이터 건수 + DB 특성 고려

@Entity 레벨 @BatchSize vs @OneToMany 레벨:
  @Entity 레벨: 이 타입을 Lazy로 참조하는 모든 곳에 적용
  @OneToMany 레벨: 이 컬렉션에만 적용 (더 세밀한 제어)
  → 일관성 원하면 전역 설정, 개별 조정 원하면 @OneToMany 레벨
```

---

## 📌 핵심 정리

```
BatchFetch 동작
  Lazy 컬렉션 첫 접근 → BatchFetchQueue에 같은 타입 ID 수집
  → IN (id1, id2, ..., idN) 한 번의 쿼리로 배치 로딩
  → N+1 → N/batchSize + 1번으로 감소

IN절 패딩 이유
  파라미터 수가 다르면 다른 QueryPlan (캐시 미스)
  → 2의 제곱수로 패딩 → 같은 크기의 PreparedStatement 재사용
  → QueryPlanCache 히트율 향상

우선순위
  @BatchSize(컬렉션) > @BatchSize(@Entity) > default_batch_fetch_size

권장 전략
  Spring Boot: default_batch_fetch_size=100 전역 설정 (가장 단순)
  세밀한 제어: 주요 컬렉션에 @BatchSize 직접 선언
  복수 컬렉션: 1개 Fetch Join + 나머지 @BatchSize

BatchSize 적정 범위
  Oracle: 최대 1000개 (IN절 제한)
  MySQL/PostgreSQL: 제한 없지만 100~500 권장
  Hibernate 6.x: 자동 최적화 패딩 (더 세밀)
```

---

## 🤔 생각해볼 문제

**Q1.** `hibernate.default_batch_fetch_size=100`을 설정했을 때, `@ManyToOne(fetch=EAGER)`인 연관관계에도 IN절 배치 로딩이 적용되는가?

**Q2.** 팀 150개를 조회하고 멤버를 접근할 때 `@BatchSize(size=100)`이 설정된 경우 실제로 몇 번의 IN절 쿼리가 발생하는가? Hibernate의 패딩 전략을 고려하면?

**Q3.** `@BatchSize`와 2차 캐시(Second-Level Cache)를 함께 사용할 때 어떤 상호작용이 발생하는가? IN절로 조회한 엔티티들이 2차 캐시에 저장되는가?

> 💡 **해설**
>
> **Q1.** 적용되지 않는다. `EAGER` 로딩은 연관 엔티티를 즉시 로딩하는 방식인데, JPA 구현체는 EAGER를 JOIN 쿼리 또는 즉시 SELECT로 처리한다. `default_batch_fetch_size`는 LAZY 연관관계의 초기화(프록시 초기화, 컬렉션 초기화)에만 적용된다. EAGER 연관관계는 이미 로딩 전략이 결정되어 있어 BatchFetch가 개입하지 않는다. 따라서 N+1 문제가 있는 EAGER 연관관계는 `@BatchSize`로 해결할 수 없고, LAZY로 변경 후 `@BatchSize` 또는 Fetch Join을 적용해야 한다.
>
> **Q2.** 이론적으로는 150 / 100 = 2번 쿼리가 예상된다. 하지만 Hibernate 5.x의 패딩 전략을 고려하면: 첫 번째 배치 100개 → IN절 128개짜리(다음 2의 제곱수) PreparedStatement 사용, 두 번째 배치 50개 → IN절 64개짜리(다음 2의 제곱수) PreparedStatement 사용. 실제로는 2번의 IN절 쿼리가 발생하며, 각각 128개와 64개 파라미터를 가진다. 나머지 자리는 null로 패딩되고 `WHERE id IN (...) AND id IS NOT NULL` 조건으로 null을 필터링한다. Hibernate 6.x에서는 패딩 전략이 개선되어 다소 다를 수 있다.
>
> **Q3.** 상호작용이 있으며 긍정적으로 결합된다. IN절로 조회된 각 엔티티는 개별적으로 2차 캐시에 저장된다. 다음 번 같은 ID의 엔티티를 로딩할 때 2차 캐시에서 꺼낸다. 단, IN절 배치 로딩 시 Hibernate는 먼저 2차 캐시에서 각 ID를 확인하고, 캐시에 없는 ID만 IN절에 포함시켜 DB 조회를 최소화한다. 따라서 2차 캐시 히트율이 높을수록 IN절에 포함되는 ID 수가 줄어들고 DB 부하가 감소한다. 이 두 기법의 조합은 캐시 웜업 이후 DB 접근을 크게 줄일 수 있는 효과적인 패턴이다.

---

<div align="center">

**[⬅️ 이전: Pagination 최적화 전략](./03-pagination-optimization.md)** | **[다음: Query Plan Cache 동작 ➡️](./05-query-plan-cache.md)**

</div>
