# Fetch Join vs @EntityGraph — 연관관계 즉시 로딩의 두 가지 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fetch Join과 `@EntityGraph`가 생성하는 SQL은 동일한가, 다른가?
- `MultipleBagFetchException`은 왜 발생하며 어떻게 회피하는가?
- 컬렉션을 2개 이상 즉시 로딩해야 할 때 어떤 전략을 선택해야 하는가?
- Fetch Join과 Pagination(`Pageable`)을 함께 쓰면 왜 경고가 발생하는가?
- `@EntityGraph`와 Fetch Join의 실무 선택 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: Lazy Loading으로는 N+1 문제가 발생한다

```java
// 팀 목록과 팀원 동시 조회 — Lazy Loading N+1 발생
List<Team> teams = teamRepository.findAll(); // SQL 1번

for (Team team : teams) {
    System.out.println(team.getMembers().size()); // SQL N번 (팀 수만큼)
}
// 팀 10개 → SELECT 총 11번
// 팀 100개 → SELECT 총 101번

// 해결책: 연관관계를 한 번의 JOIN 쿼리로 함께 로딩
// → Fetch Join 또는 @EntityGraph
```

```
두 방식의 공통 목표:
  N+1 쿼리 → 1개의 JOIN 쿼리로 해결
  Lazy 연관관계를 즉시 초기화

차이:
  Fetch Join: JPQL/QueryDSL에서 명시적 선언
  @EntityGraph: 어노테이션 기반 선언적 설정
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 컬렉션 2개를 Fetch Join하면 된다

```java
// ❌ MultipleBagFetchException 발생
@Query("SELECT t FROM Team t " +
       "JOIN FETCH t.members " +
       "JOIN FETCH t.projects")  // ← 컬렉션 2개 동시 Fetch Join → 예외!
List<Team> findAllWithMembersAndProjects();

// org.hibernate.loader.MultipleBagFetchException:
// cannot simultaneously fetch multiple bags: [Team.members, Team.projects]
```

### Before: @EntityGraph는 Fetch Join과 동일한 SQL을 생성한다

```java
// ❌ 잘못된 이해: 두 방식이 완전히 동일

// ✅ 실제 차이:
// JPQL "JOIN FETCH" → 기본적으로 INNER JOIN
// @EntityGraph       → 기본적으로 LEFT OUTER JOIN

@Query("SELECT t FROM Team t JOIN FETCH t.members")
// → INNER JOIN: 멤버 없는 팀은 결과에서 제외

@EntityGraph(attributePaths = {"members"})
@Query("SELECT t FROM Team t")
// → LEFT OUTER JOIN: 멤버 없는 팀도 포함

// LEFT Fetch Join으로 맞추려면:
@Query("SELECT t FROM Team t LEFT JOIN FETCH t.members")
// → LEFT OUTER JOIN → @EntityGraph와 동일한 SQL
```

---

## ✨ 올바른 이해와 패턴

### After: 선택 기준 요약

```
Fetch Join 사용:
  JPQL/QueryDSL 쿼리를 직접 작성하는 경우
  INNER/LEFT 방향을 명시적으로 제어해야 할 때
  동적 쿼리에서 조건에 따라 fetch join 여부를 결정할 때

@EntityGraph 사용:
  Spring Data JPA Repository 메서드에 선언적으로 적용할 때
  기존 쿼리 메서드(findById, findAll 등)에 로딩 전략만 추가할 때
  JPQL 없이 간단하게 N+1을 해결할 때

컬렉션 2개 이상 즉시 로딩:
  한 컬렉션만 Fetch Join + 나머지는 @BatchSize
  또는 DTO Projection으로 전환 (JOIN + 애플리케이션 조립)
```

---

## 🔬 내부 동작 원리 — Fetch Join과 @EntityGraph 소스 추적

### 1. Fetch Join — JPQL에서 SQL로

```java
// Fetch Join JPQL
@Query("SELECT t FROM Team t LEFT JOIN FETCH t.members m WHERE t.status = 'ACTIVE'")
List<Team> findActiveTeamsWithMembers();

// Hibernate가 생성하는 SQL:
// SELECT t.id, t.name, t.status,
//        m.id, m.name, m.team_id
// FROM teams t
// LEFT OUTER JOIN users m ON m.team_id = t.id
// WHERE t.status = 'ACTIVE'

// 결과 처리:
// 팀 A (멤버 3명) → 결과 행 3개, Hibernate가 Team 1개 + Member 3개로 조합
// 팀 B (멤버 0명) → 결과 행 1개 (LEFT JOIN이므로 NULL 멤버)
// → 중복 Team 행 제거는 Hibernate가 1차 캐시에서 처리

// 주의: SQL 결과 행 수 = 멤버 총 수 (카테시안 곱 X, JOIN은 1:N)
// 팀 3개, 각 10명 → SQL 결과 30행 → Hibernate가 3개 Team으로 조합
```

### 2. @EntityGraph — 내부 동작

```java
// @EntityGraph 선언
public interface TeamRepository extends JpaRepository<Team, Long> {

    // 방법 1: attributePaths 직접 지정
    @EntityGraph(attributePaths = {"members", "members.roles"})
    List<Team> findByStatus(Status status);
    // → LEFT OUTER JOIN users m ON m.team_id = t.id
    // → LEFT OUTER JOIN roles r ON r.user_id = m.id

    // 방법 2: @NamedEntityGraph 참조
    @EntityGraph("Team.withMembers")
    List<Team> findByName(String name);
}

// @NamedEntityGraph 선언 (엔티티에)
@Entity
@NamedEntityGraph(
    name = "Team.withMembers",
    attributeNodes = @NamedAttributeNode(
        value = "members",
        subgraph = "members.roles"
    ),
    subgraphs = @NamedSubgraph(
        name = "members.roles",
        attributeNodes = @NamedAttributeNode("roles")
    )
)
public class Team { ... }
```

```java
// @EntityGraph 처리 내부 — JpaEntityGraph → Hibernate LoadGraph
// AbstractJpaQuery.createJpaQuery() 내부:
void applyEntityGraphQueryHint(String type, JpaEntityGraph graph, Query query) {
    // "javax.persistence.loadgraph" 또는 "javax.persistence.fetchgraph" 힌트로 변환
    query.setHint(type, graph.toEntityGraph(em.getEntityManagerFactory()));
    // → Hibernate QueryPlan에 LoadGraph 추가
    // → LEFT OUTER JOIN 생성
}
```

### 3. MultipleBagFetchException — 원인과 해결

```java
// 원인: Java의 Bag 컬렉션(List, 순서 없고 중복 허용)을 2개 동시 Fetch Join
// Hibernate가 카테시안 곱 결과에서 올바른 매핑을 보장할 수 없어 거부

@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    private List<User> members;   // Bag (List, 순서 없음)

    @OneToMany(mappedBy = "team")
    private List<Project> projects; // Bag (List, 순서 없음)
    // 두 Bag 동시 fetch join → MultipleBagFetchException
}

// 해결책 1: Set으로 변경 (순서 보장 불필요한 경우)
@OneToMany(mappedBy = "team")
private Set<User> members;   // Set → 중복 없음 → 동시 fetch join 가능

@OneToMany(mappedBy = "team")
private Set<Project> projects; // Set → 가능
// → 두 Set 동시 fetch join 허용

// 해결책 2: @OrderColumn 추가 (순서 있는 List)
@OneToMany(mappedBy = "team")
@OrderColumn(name = "member_order")
private List<User> members;  // OrderedList → 동시 fetch join 가능

// 해결책 3 (권장): 하나만 Fetch Join + 나머지는 @BatchSize
@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    @BatchSize(size = 100)  // IN절로 배치 로딩
    private List<User> members;

    @OneToMany(mappedBy = "team")
    @BatchSize(size = 100)
    private List<Project> projects;
}

// Fetch Join으로 Team 로딩 후
// members, projects는 @BatchSize로 배치 로딩
// Team 10개 → members: 1번의 IN절 쿼리, projects: 1번의 IN절 쿼리
// 총 3번 (Fetch Join 1 + BatchSize 2) → 실용적 해결
```

### 4. Fetch Join + Pagination 문제

```java
// ❌ 컬렉션 Fetch Join + Pageable → 경고 + 메모리 페이징
@Query("SELECT t FROM Team t LEFT JOIN FETCH t.members")
Page<Team> findAllWithMembers(Pageable pageable);

// Hibernate 경고:
// HHH90003004: firstResult/maxResults specified with collection fetch;
// applying in memory → 전체 결과를 메모리에 올린 후 페이징!

// 이유:
// SQL LIMIT/OFFSET을 적용하면 Team 단위가 아닌 JOIN 결과 행 단위로 페이징됨
// 팀 A(멤버 5명) → 5행, LIMIT 3이면 팀 A의 멤버 3명만 → 불완전한 팀 데이터
// → Hibernate가 안전하게 전체 조회 후 메모리에서 Team 단위 페이징

// ✅ 해결책 1: CountQuery 분리 + ID 기반 두 단계 조회
@Query(
    value = "SELECT DISTINCT t FROM Team t LEFT JOIN FETCH t.members",
    countQuery = "SELECT COUNT(DISTINCT t) FROM Team t"
)
Page<Team> findAllWithMembersOptimized(Pageable pageable);
// 여전히 메모리 페이징 문제 있음

// ✅ 해결책 2 (권장): 먼저 ID 페이징, 이후 Fetch Join
@Query("SELECT t FROM Team t")
Page<Team> findAllPaged(Pageable pageable); // ID만 페이징

// Service 레이어에서 두 단계로 처리
public Page<Team> findTeamsWithMembers(Pageable pageable) {
    Page<Team> teamPage = teamRepository.findAllPaged(pageable);
    List<Long> teamIds = teamPage.map(Team::getId).toList();

    // ID 목록으로 Fetch Join (페이징 결과에 대해서만)
    List<Team> teamsWithMembers = teamRepository.findByIdInWithMembers(teamIds);
    // → 정확한 페이징 + N+1 해결

    return new PageImpl<>(teamsWithMembers, pageable, teamPage.getTotalElements());
}

@Query("SELECT t FROM Team t LEFT JOIN FETCH t.members WHERE t.id IN :ids")
List<Team> findByIdInWithMembers(@Param("ids") List<Long> ids);
```

### 5. @EntityGraph type — LOAD vs FETCH

```java
// EntityGraph.EntityGraphType.FETCH (기본값)
// → 그래프에 명시된 속성: EAGER
// → 명시되지 않은 속성: LAZY (FetchType 설정 무관)

// EntityGraph.EntityGraphType.LOAD
// → 그래프에 명시된 속성: EAGER
// → 명시되지 않은 속성: 엔티티의 FetchType 설정 따름 (기본값)

@EntityGraph(attributePaths = {"members"}, type = EntityGraph.EntityGraphType.FETCH)
List<Team> findByStatus(Status status);
// → members: EAGER (JOIN), team.projects: LAZY (명시 안 했으므로)

@EntityGraph(attributePaths = {"members"}, type = EntityGraph.EntityGraphType.LOAD)
List<Team> findByStatus(Status status);
// → members: EAGER (JOIN)
// → team.projects: 엔티티의 FetchType 따름 (LAZY면 LAZY)
```

---

## 💻 실험으로 확인하기

### 실험 1: Fetch Join vs @EntityGraph SQL 비교

```java
@Test
void compareSqlGeneration() {
    // Fetch Join: INNER JOIN (기본)
    teamRepository.findAllWithMembersFetchJoin();
    // Hibernate SQL:
    // select t.*, m.* from teams t
    // inner join users m on m.team_id = t.id

    // LEFT Fetch Join
    teamRepository.findAllWithMembersLeftFetchJoin();
    // Hibernate SQL:
    // select t.*, m.* from teams t
    // left outer join users m on m.team_id = t.id

    // @EntityGraph (기본 LEFT OUTER)
    teamRepository.findAllWithMembersEntityGraph();
    // Hibernate SQL:
    // select t.*, m.* from teams t
    // left outer join users m on m.team_id = t.id
}
```

### 실험 2: 중복 결과 처리 — DISTINCT

```java
// Fetch Join 시 결과 중복 (JOIN으로 인한 행 증가)
@Query("SELECT t FROM Team t LEFT JOIN FETCH t.members")
List<Team> findAllWithMembers(); // 멤버 10명이면 Team이 10번 반복될 수 있음

@Query("SELECT DISTINCT t FROM Team t LEFT JOIN FETCH t.members")
List<Team> findAllWithMembersDistinct(); // 중복 Team 제거

// Hibernate 6.x 이후:
// DISTINCT가 SQL에 전달되지 않고 Hibernate 레벨에서 처리
// → SQL은 DISTINCT 없이, Hibernate가 1차 캐시 기반 중복 제거
// (Hibernate 5.x에서는 SQL에 DISTINCT 포함)
```

### 실험 3: @BatchSize로 N+1 해결

```yaml
# 전역 설정 (application.yml)
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

```java
// @BatchSize(size=100) 또는 전역 default_batch_fetch_size=100 설정 시:
List<Team> teams = teamRepository.findAll(); // SQL 1번

// teams.get(0).getMembers() 접근 시:
// → teams 목록의 ID를 수집해 IN절로 한 번에 조회
// SELECT * FROM users WHERE team_id IN (1, 2, 3, ..., 100)
// → SQL 1번으로 100개 팀의 멤버 모두 로딩
```

---

## ⚡ 성능 임팩트

```
팀 10개, 팀당 멤버 10명 (총 100명) 조회 시나리오:

N+1 (Lazy Loading):
  SQL 11번 (팀 1번 + 멤버 10번)
  네트워크 왕복 11번 → ~수십 ms

Fetch Join / @EntityGraph:
  SQL 1번 (LEFT JOIN)
  결과 100행 → 네트워크 전송 증가 (중복 팀 데이터)
  → ~수 ms (1번 왕복)

@BatchSize(size=100):
  SQL 2번 (팀 1번 + 멤버 IN절 1번)
  카테시안 곱 없음 → 네트워크 효율적
  → ~수 ms (2번 왕복)

컬렉션 2개 이상 로딩 시나리오:
  Fetch Join (Set) 2개: 카테시안 곱 → 100*100=10000행
  @BatchSize 2개: 3번 SQL → 네트워크 효율적
  → 컬렉션 2개 이상은 @BatchSize가 유리

Pagination 포함 시나리오:
  컬렉션 Fetch Join + Pageable: 전체 메모리 로딩 → 위험
  ID 페이징 + Fetch Join: 2번 쿼리, 정확한 페이징
  @BatchSize + Pageable: 2번 쿼리, 가장 안전
```

---

## 🤔 트레이드오프

```
Fetch Join:
  ✅ 1번 쿼리로 해결 (JOIN)
  ✅ 명시적 제어 (INNER/LEFT 선택)
  ❌ 컬렉션 2개 동시 → MultipleBagFetchException
  ❌ Pagination과 함께 → 메모리 페이징 위험
  ❌ 카테시안 곱으로 결과 행 증가

@EntityGraph:
  ✅ 선언적, Repository 메서드에 간단히 적용
  ✅ 기존 findById, findAll에 오버라이드 가능
  ❌ 항상 LEFT OUTER JOIN (INNER가 필요한 경우 제한)
  ❌ 복잡한 조건 표현 제한
  ❌ Pagination과 함께 → 동일한 메모리 페이징 문제

@BatchSize:
  ✅ 컬렉션 2개 이상도 안전하게 처리
  ✅ Pagination과 함께 사용 가능
  ✅ 카테시안 곱 없음 (IN절 분리 쿼리)
  ❌ 2번 이상 쿼리 (Fetch Join의 1번보다 많음)
  ❌ 배치 크기 튜닝 필요

실무 권장 조합:
  단순 @ManyToOne 조회 → Fetch Join 또는 @EntityGraph
  @OneToMany 1개 → LEFT JOIN FETCH
  @OneToMany 2개 이상 → 1개 Fetch Join + 나머지 @BatchSize
  Pagination + 컬렉션 → ID 페이징 후 Fetch Join
```

---

## 📌 핵심 정리

```
Fetch Join vs @EntityGraph SQL 차이
  JPQL JOIN FETCH → 기본 INNER JOIN
  LEFT JOIN FETCH → LEFT OUTER JOIN
  @EntityGraph   → 항상 LEFT OUTER JOIN

MultipleBagFetchException 해결
  List → Set 변경 (중복 불필요한 경우)
  @OrderColumn (순서 있는 List)
  하나만 Fetch Join + 나머지 @BatchSize (권장)

Fetch Join + Pagination 경고
  컬렉션 JOIN → SQL LIMIT 적용 불가 (행 단위 페이징)
  Hibernate가 전체 메모리 로딩 후 페이징 → 위험
  해결: ID 페이징 → Fetch Join (2단계 쿼리)

@EntityGraph 타입
  FETCH: 미명시 속성 → 무조건 LAZY
  LOAD: 미명시 속성 → 엔티티 FetchType 따름 (기본값)
```

---

## 🤔 생각해볼 문제

**Q1.** `@EntityGraph(attributePaths = {"members"})`를 `findById()`에 적용했을 때, 이미 1차 캐시에 해당 Team이 있다면 JOIN 쿼리가 발생하는가?

**Q2.** Fetch Join으로 가져온 Team 엔티티에서 `team.getMembers()`를 호출하면 1차 캐시에 Members가 저장되어 있는가? 트랜잭션 종료 후 Detached 상태에서도 접근 가능한가?

**Q3.** `@BatchSize`의 `size` 파라미터와 `hibernate.default_batch_fetch_size`가 동시에 설정되어 있을 때 어떤 값이 우선하는가? IN절 크기가 성능에 미치는 영향은?

> 💡 **해설**
>
> **Q1.** 1차 캐시에 Team이 있더라도 `@EntityGraph`가 적용된 메서드를 호출하면 JOIN 쿼리가 발생한다. 이유는 1차 캐시의 Team 엔티티가 members를 LAZY로 갖고 있어 초기화 여부를 모르기 때문에 Hibernate가 새 쿼리로 확인한다. 단, 1차 캐시에 Team이 이미 있으면 Team 자체는 새 인스턴스를 만들지 않고 기존 캐시 객체를 반환하며, members 컬렉션만 초기화(채움)한다. 즉 SQL은 실행되지만 Team 객체는 1차 캐시에서 가져온다.
>
> **Q2.** Fetch Join으로 로딩된 Members는 1차 캐시에 각각의 EntityEntry로 저장된다. `team.getMembers()`는 이미 초기화된 컬렉션을 반환하며 추가 SQL이 발생하지 않는다. 트랜잭션 종료 후 Detached 상태에서는 members 컬렉션이 이미 초기화되어 있으므로 `LazyInitializationException` 없이 접근 가능하다. Fetch Join의 핵심 장점 중 하나가 이것이다 — OSIV 없이도 View 레이어에서 Detached 엔티티의 컬렉션에 접근할 수 있다.
>
> **Q3.** `@BatchSize` 어노테이션이 `hibernate.default_batch_fetch_size`보다 우선한다. 특정 컬렉션에 `@BatchSize`를 명시하면 해당 컬렉션에만 그 값이 적용되고, 나머지는 전역 설정을 따른다. IN절 크기의 성능 영향: 너무 작으면(예: 10) 여러 번의 IN절 쿼리가 발생하고, 너무 크면(예: 10000) IN절 파라미터 파싱 비용과 일부 DB의 IN절 크기 제한에 걸릴 수 있다. 일반적으로 100~1000이 권장되며, Hibernate는 내부적으로 2의 제곱수(128, 256 등)로 IN절 크기를 조정해 QueryPlanCache 효율을 높인다.

---

<div align="center">

**[⬅️ 이전: JPQL vs Criteria API vs QueryDSL](./01-jpql-criteria-querydsl.md)** | **[다음: Pagination 최적화 전략 ➡️](./03-pagination-optimization.md)**

</div>
