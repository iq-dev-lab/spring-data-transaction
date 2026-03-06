# N+1 문제 완전 해결 — 5가지 전략별 SQL 쿼리 수 실측

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- N+1 문제가 발생하는 정확한 시점 — Lazy Loading 프록시가 초기화되는 조건은?
- Fetch Join과 `@EntityGraph`는 어떤 SQL을 생성하며, 무엇이 다른가?
- `@BatchSize`로 IN 절을 활용하면 N+1이 아닌 몇 개의 쿼리가 발생하는가?
- `FetchMode.SUBSELECT`는 IN 절과 어떻게 다른가?
- DTO Projection은 왜 N+1 자체를 발생시키지 않는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 연관 엔티티를 조회하면 예상치 못한 쿼리 폭발이 발생한다

```java
// Team 1개에 User N명 (OneToMany)
@Entity
public class Team {
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    private List<User> users; // Lazy — 접근 시 쿼리 발생
}

// N+1 발생 코드
@Transactional
public void printTeamUsers() {
    List<Team> teams = teamRepository.findAll(); // 쿼리 1: SELECT * FROM teams

    for (Team team : teams) {
        // 각 team.getUsers() 접근 시 → Lazy 프록시 초기화 → DB 조회
        System.out.println(team.getUsers().size()); // 쿼리 2~N+1: SELECT * FROM users WHERE team_id=?
    }
}

// 팀이 100개라면:
// findAll(): 1번
// team.getUsers(): 100번
// 총 101번 쿼리 → N+1 (여기서 N=100)
```

```
N+1 발생 조건:
  [1] @OneToMany, @ManyToMany → Lazy Loading (기본)
  [2] 컬렉션/연관 접근 시 → 프록시 초기화 → SELECT
  [3] N개 엔티티 × 각 연관 → N번 추가 쿼리

해결 전략 5가지:
  1. Fetch Join        → JOIN + 한 번의 쿼리
  2. @EntityGraph      → JOIN + 한 번의 쿼리 (선언적)
  3. @BatchSize        → IN 절 분할 쿼리
  4. FetchMode.SUBSELECT → 서브쿼리로 한 번 처리
  5. DTO Projection    → 필요한 데이터만 직접 쿼리
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @ManyToOne도 N+1이 발생한다

```java
// ❌ 잘못된 이해: @ManyToOne은 Eager이라 N+1 없다

// @ManyToOne 기본값: EAGER (JPA 스펙)
@ManyToOne(fetch = FetchType.EAGER)  // 기본값
private Team team;

// JPQL로 User 조회 시 EAGER 연관은 어떻게 로드될까?
List<User> users = em.createQuery("SELECT u FROM User u", User.class).getResultList();
// 기대: JOIN으로 한 번에 로드
// 실제 (Hibernate 동작): User N개 로드 후 각 team을 별도 쿼리로 로드
// → EAGER도 N+1 발생! (JPQL은 연관관계 힌트를 무시하고 직접 쿼리)

// 해결: @ManyToOne도 LAZY로 설정 + 필요 시 Fetch Join
@ManyToOne(fetch = FetchType.LAZY)
private Team team;
```

### Before: Fetch Join만 하면 모든 N+1이 해결된다

```java
// ❌ Fetch Join으로 해결 안 되는 상황
// 1. 컬렉션 2개 이상 Fetch Join → MultipleBagFetchException
@Query("SELECT t FROM Team t JOIN FETCH t.users JOIN FETCH t.managers")
List<Team> findTeamsWithUsersAndManagers();
// → org.hibernate.loader.MultipleBagFetchException: cannot simultaneously fetch multiple bags

// 2. Fetch Join + Pagination → 경고 + 메모리에서 페이지 처리 (위험)
@Query("SELECT t FROM Team t JOIN FETCH t.users")
Page<Team> findTeamsWithUsers(Pageable pageable);
// → HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!
// → 전체 데이터를 메모리에 올린 후 페이지 처리 → OOM 위험

// 해결: Pagination + 컬렉션 → @BatchSize 또는 @EntityGraph(attributePaths 단일) 사용
```

---

## ✨ 올바른 이해와 패턴

### After: 5가지 전략 선택 기준

```
전략 비교표:
  전략             쿼리 수    페이지네이션    컬렉션 2개+    SQL 형태
  ─────────────────────────────────────────────────────────────
  Fetch Join       1개       ⚠️ 위험       ❌ 불가       LEFT OUTER JOIN
  @EntityGraph     1개       ⚠️ 위험       ❌ 불가       LEFT OUTER JOIN
  @BatchSize       1+N/K개   ✅ 안전       ✅ 가능       IN (?, ?, ...)
  SUBSELECT        2개       ✅ 안전       ✅ 가능       WHERE IN (서브쿼리)
  DTO Projection   1개       ✅ 안전       ✅ 가능       JOIN + SELECT 지정

추천 우선순위:
  단순 조회 (페이지 없음, 컬렉션 1개): Fetch Join 또는 @EntityGraph
  페이지네이션 + 컬렉션: @BatchSize (전역 설정) + 별도 count 쿼리
  복잡한 DTO 조회: QueryDSL Projection
  컬렉션 2개 이상 + 페이지: DTO + @BatchSize 조합
```

---

## 🔬 내부 동작 원리 — 5가지 전략 소스 추적

### 1. N+1 발생 원리 — Lazy 프록시 초기화

```java
// Lazy 컬렉션: PersistentBag (List의 Hibernate 구현)
// users 필드 = PersistentBag 인스턴스 (아직 초기화 안 됨)

public class PersistentBag extends AbstractPersistentCollection {

    // size(), get(), iterator() 등 접근 시 초기화
    @Override
    public int size() {
        // initialized인지 확인
        if (!isInitialized()) {
            // 초기화 → DB 쿼리 실행
            initialize(false);
            // → session.initializeCollection(this, false)
            // → DefaultInitializeCollectionEventListener.onInitializeCollection()
            // → CollectionLoader.loadCollection()
            // → SELECT users WHERE team_id = ?
        }
        return bag.size();
    }
}
```

### 2. 전략 1: Fetch Join

```java
// JPQL Fetch Join
@Query("SELECT DISTINCT t FROM Team t LEFT JOIN FETCH t.users")
List<Team> findAllWithUsers();

// 생성 SQL:
// SELECT DISTINCT t.*, u.*
// FROM teams t
// LEFT OUTER JOIN users u ON u.team_id = t.id

// DISTINCT 필요 이유:
// JOIN 결과: 팀 A (user 3명) → 팀 A 행이 3개
// DISTINCT(Java): 팀 A가 1개로 줄어듦
// SQL DISTINCT도 추가해야 DB에서도 중복 제거 (네트워크 감소)

// 주의: 컬렉션 Fetch Join 시 Pagination 금지
// → HHH90003004 경고 + 전체 메모리 로드
```

### 3. 전략 2: @EntityGraph

```java
// 인터페이스에서 선언
public interface TeamRepository extends JpaRepository<Team, Long> {

    @EntityGraph(attributePaths = {"users"})  // 로드할 연관 지정
    List<Team> findAll();

    @EntityGraph(attributePaths = {"users", "users.address"})  // 중첩 연관
    Optional<Team> findById(Long id);
}

// 생성 SQL: Fetch Join과 동일한 LEFT OUTER JOIN
// SELECT t.*, u.* FROM teams t LEFT OUTER JOIN users u ON u.team_id = t.id

// @EntityGraph vs Fetch Join:
// Fetch Join: JPQL에 직접 작성, 재사용 어려움
// @EntityGraph: 선언적, Repository 메서드에 적용, 재사용 용이
// 생성 SQL은 동일 (내부적으로 Fetch Join으로 변환)

// Named EntityGraph (엔티티에 정의)
@Entity
@NamedEntityGraph(
    name = "Team.withUsers",
    attributeNodes = @NamedAttributeNode("users")
)
public class Team { ... }

@EntityGraph("Team.withUsers")
List<Team> findByName(String name);
```

### 4. 전략 3: @BatchSize — IN 절 분할

```java
// @BatchSize: Lazy 초기화 시 IN 절로 묶어서 처리
@Entity
public class Team {
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    @BatchSize(size = 100)  // 한 번에 100개 팀의 users를 IN으로 조회
    private List<User> users;
}

// 동작:
// [1] findAll(): SELECT * FROM teams → 200개 팀 로드
// [2] 첫 번째 team.getUsers() 접근 → 프록시 초기화 시작
// [3] Hibernate: "아직 초기화 안 된 팀 ID 최대 100개 모아서 IN 절로 조회"
//     → SELECT * FROM users WHERE team_id IN (1, 2, ..., 100)
// [4] 나머지 101~200번 팀 접근 시 또 IN 절
//     → SELECT * FROM users WHERE team_id IN (101, 102, ..., 200)
// 총 쿼리 수: 1(팀 조회) + ceil(200/100)(users 조회) = 3

// 전역 설정 (모든 Lazy 컬렉션에 적용):
// application.yml:
// spring.jpa.properties.hibernate.default_batch_fetch_size: 100
// → 모든 @OneToMany, @ManyToMany에 자동 적용

// IN 절 파라미터 수 최적화:
// Hibernate는 IN 절 크기를 2의 거듭제곱으로 패딩
// size=100 → 실제로 1, 2, 4, 8, ..., 64, 128 크기의 IN 절 사용
// → Query Plan Cache 재활용을 위한 최적화
```

### 5. 전략 4: FetchMode.SUBSELECT

```java
// SUBSELECT: 원본 쿼리를 서브쿼리로 사용해 연관 한 번에 로드
@Entity
public class Team {
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SUBSELECT)  // Hibernate 전용
    private List<User> users;
}

// 동작:
// [1] findAll(): SELECT * FROM teams → 팀 전체 로드
// [2] 첫 번째 team.getUsers() 접근 시:
//     → SELECT * FROM users WHERE team_id IN
//         (SELECT id FROM teams [원본 쿼리 조건])
//     → 한 번에 모든 팀의 users 로드
// 총 쿼리 수: 2 (팀 쿼리 + users 서브쿼리)

// @BatchSize vs SUBSELECT:
// @BatchSize: 최대 K개씩 분할 (메모리 제어 가능)
// SUBSELECT: 항상 전체를 한 번에 (원본 조건 동적 활용)
// 팀이 1000개이면 SUBSELECT는 1000개 모두 한 번에 로드 → 메모리 주의
// @BatchSize가 더 안전한 선택
```

### 6. 전략 5: DTO Projection

```java
// DTO Projection: 엔티티 로드 없이 필요한 데이터만 쿼리
// N+1 자체가 발생하지 않음 (연관 엔티티를 Lazy로 로드할 일이 없음)

// QueryDSL 기반 DTO Projection
@Repository
@RequiredArgsConstructor
public class TeamQueryRepository {

    private final JPAQueryFactory queryFactory;

    public List<TeamUserDto> findTeamsWithUserCount() {
        return queryFactory
            .select(Projections.constructor(TeamUserDto.class,
                team.id,
                team.name,
                user.count()
            ))
            .from(team)
            .leftJoin(team.users, user)
            .groupBy(team.id, team.name)
            .fetch();
        // SQL: SELECT t.id, t.name, COUNT(u.id)
        //      FROM teams t LEFT JOIN users u ON u.team_id = t.id
        //      GROUP BY t.id, t.name
        // 엔티티 없이 직접 DTO → N+1 불가능
    }

    // 중첩 DTO (팀 + 사용자 목록)
    public List<TeamWithUsersDto> findTeamsWithUsers(List<Long> teamIds) {
        // 방법 1: 팀 조회 후 사용자 별도 조회 (2 쿼리)
        List<TeamDto> teams = queryFactory
            .select(Projections.constructor(TeamDto.class, team.id, team.name))
            .from(team)
            .where(team.id.in(teamIds))
            .fetch();

        Map<Long, List<UserDto>> usersByTeam = queryFactory
            .select(Projections.constructor(UserDto.class, user.id, user.name, team.id))
            .from(user)
            .where(user.team.id.in(teamIds))
            .fetch()
            .stream()
            .collect(Collectors.groupingBy(UserDto::getTeamId));

        return teams.stream()
            .map(t -> new TeamWithUsersDto(t, usersByTeam.getOrDefault(t.getId(), emptyList())))
            .toList();
        // 총 쿼리: 2 (팀 + 사용자) — N+1 없음
    }
}
```

---

## 💻 실험으로 확인하기

### 실험: 전략별 쿼리 수 실측

```java
// 준비: 팀 10개, 팀당 사용자 5명 (총 50명)
@Test
void compareStrategies() {

    // [N+1 — 기본]
    List<Team> teams = teamRepository.findAll();  // 1 쿼리
    teams.forEach(t -> t.getUsers().size());       // 10 쿼리
    // 총: 11쿼리

    // [Fetch Join]
    teams = teamRepository.findAllWithUsers();  // 1 쿼리
    teams.forEach(t -> t.getUsers().size());    // 0 추가 쿼리
    // 총: 1쿼리

    // [@BatchSize(size=5)]
    teams = teamRepository.findAll();           // 1 쿼리
    teams.forEach(t -> t.getUsers().size());    // ceil(10/5) = 2 쿼리
    // 총: 3쿼리

    // [SUBSELECT]
    teams = teamRepository.findAll();           // 1 쿼리
    teams.get(0).getUsers().size();             // 1 쿼리 (전체 팀의 users 한 번에)
    teams.get(1).getUsers().size();             // 0 추가 쿼리 (이미 로드됨)
    // 총: 2쿼리
}
```

```
실측 결과 (팀 100개 기준):

전략             쿼리 수    실행 시간(예시)
─────────────────────────────────────────
N+1 (기본)      101개      1,234ms
Fetch Join        1개         45ms
@EntityGraph      1개         48ms
@BatchSize(100)   2개         52ms
SUBSELECT         2개         51ms
DTO Projection    1~2개       38ms
```

---

## ⚡ 성능 임팩트

```
N+1 비용:
  DB 왕복 N번 → 각 왕복 ~1~5ms
  N=100: 100~500ms 추가 지연
  N=1000: 1~5초 추가 지연 → 서비스 장애 수준

Fetch Join 주의:
  컬렉션 JOIN → 데이터 뻥튀기 (팀 1 × 사용자 5 = 5행)
  DISTINCT 없으면 팀 중복 반환
  대용량 컬렉션 JOIN → 결과 행 수 폭발적 증가 → 네트워크/메모리 부하

@BatchSize 최적 값:
  너무 작음 → IN 절 여러 번 (N/K 쿼리)
  너무 큼 → IN 절 파라미터 너무 많음 (DB 제한, 성능 저하)
  권장: 100~1000 (DB 설정과 엔티티 수 고려)
  MySQL IN 절 한계: 수천 개 (쿼리 크기 제한 약 16MB)

DTO Projection 우위:
  엔티티 로드 없음 → 스냅샷 없음 → dirty checking 없음
  필요한 컬럼만 SELECT → 네트워크 최소화
  페이지네이션 + 복잡한 집계에 최적
```

---

## 🤔 트레이드오프

```
Fetch Join / @EntityGraph:
  ✅ 단일 쿼리로 연관 로드 (빠름)
  ❌ 컬렉션 2개 이상 동시 불가 (MultipleBagFetchException)
  ❌ Pagination과 함께 사용 시 OOM 위험
  → 단일 컬렉션, 페이지 없는 경우에 최적

@BatchSize (전역 설정 권장):
  ✅ 컬렉션 여러 개 처리 가능
  ✅ Pagination과 안전하게 사용
  ✅ 코드 변경 최소 (전역 설정)
  ❌ Fetch Join보다 쿼리 수 많음 (1+N/K)
  → 대부분의 일반 조회에 적합한 기본 전략

DTO Projection:
  ✅ 가장 최적화된 쿼리 (필요한 것만)
  ✅ N+1 구조적 불가
  ❌ 엔티티 대신 DTO 관리 → 코드량 증가
  ❌ 복잡한 조인 쿼리 작성 필요
  → 성능이 중요한 조회 API, 복잡한 통계 쿼리

실무 조합 권장:
  기본: @BatchSize 전역 설정 (spring.jpa.properties.hibernate.default_batch_fetch_size=100)
  성능 중요 API: DTO Projection (QueryDSL)
  단순 상세 조회: Fetch Join 또는 @EntityGraph
```

---

## 📌 핵심 정리

```
N+1 발생 원인
  Lazy 컬렉션 접근 → PersistentBag 초기화 → SELECT WHERE fk=?
  엔티티 N개 × 연관 접근 = 1 + N 쿼리

5가지 해결 전략
  Fetch Join:   LEFT OUTER JOIN, 단일 쿼리, Pagination 금지
  @EntityGraph: 선언적 Fetch Join, 동일 SQL
  @BatchSize:   IN 절 분할, 1+N/K 쿼리, Pagination 안전
  SUBSELECT:    서브쿼리, 2 쿼리, 전체 한 번에
  DTO:          엔티티 없이 직접 SELECT, N+1 구조적 불가

MultipleBagFetchException 회피
  컬렉션 2개 이상 Fetch Join 불가
  → Set으로 타입 변경 (Bag → Set은 가능)
  → 또는 @BatchSize 사용

Pagination + 컬렉션
  Fetch Join 사용 금지 (HHH90003004)
  @BatchSize 사용 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `@ManyToOne(fetch = FetchType.EAGER)`로 설정된 연관에서 JPQL `SELECT u FROM User u`를 실행하면 어떤 SQL이 발생하는가? Eager임에도 N+1이 발생하는 이유는?

**Q2.** Fetch Join으로 팀과 사용자를 조회할 때 `DISTINCT` 키워드가 없으면 어떤 결과가 반환되는가? SQL DISTINCT와 JPQL DISTINCT의 차이는?

**Q3.** `spring.jpa.properties.hibernate.default_batch_fetch_size=100`으로 설정했을 때, 팀이 250개이고 각 팀에 사용자가 있다면 몇 번의 쿼리가 실행되는가? Hibernate의 IN 절 크기 패딩이 적용될 때 실제 IN 절 크기는 어떻게 결정되는가?

> 💡 **해설**
>
> **Q1.** N+1이 발생한다. JPQL은 작성된 쿼리 문자열을 그대로 SQL로 변환하므로 `SELECT u FROM User u`는 `SELECT * FROM users`만 실행한다. Fetch 전략(`EAGER`/`LAZY`)은 JPA가 `em.find()`로 엔티티를 로드할 때 적용되는 힌트이며, JPQL은 이를 무시한다. `EAGER`로 설정된 `@ManyToOne`은 User 로드 후 각 `team`에 대해 별도 SELECT가 발생한다. 이것이 모든 연관을 `LAZY`로 설정하고 필요한 곳에서 Fetch Join/BatchSize를 명시하는 것이 권장되는 이유다.
>
> **Q2.** DISTINCT 없이 Fetch Join 시 팀 1개에 사용자 3명이면 팀이 3번 반복된 리스트가 반환된다. SQL DISTINCT는 DB에서 중복 행을 제거하지만, 팀과 사용자 컬럼을 합친 행은 사실 다른 행이므로 SQL DISTINCT만으로는 팀 중복이 제거되지 않는다. JPQL DISTINCT는 두 가지를 함께 수행한다: SQL에 `DISTINCT`를 추가하고, 추가로 Hibernate가 Java 레벨에서 반환된 엔티티 목록의 중복을 `EntityKey`(ID) 기준으로 제거한다. 따라서 JPQL `SELECT DISTINCT t FROM Team t JOIN FETCH t.users`를 사용해야 팀 중복이 제거된다.
>
> **Q3.** Hibernate는 IN 절 크기를 2의 거듭제곱 배열로 패딩한다(1, 2, 4, 8, 16, 32, 64, 128 등). `default_batch_fetch_size=100`이면 실제 사용되는 IN 절 크기는 128(100보다 크면서 가장 가까운 2의 거듭제곱)이다. 팀 250개의 경우: 1차 요청 시 250개 ID 중 최대 128개 → `IN (...128개...)` 쿼리 1번, 나머지 122개 중 최대 64개 → `IN (...64개...)` 쿼리 1번, 나머지 58개 → `IN (...32개...)` + `IN (...16개...)` + `IN (...8개...)` + `IN (...2개...)` 등으로 분할된다. 총 쿼리 수는 1(팀 조회) + 약 3~4번(users 분할 조회) = 4~5번이다. 이 패딩 방식은 동일한 IN 절 크기의 PreparedStatement를 재사용해 Query Plan Cache 효율을 높이기 위함이다.

---

<div align="center">

**[⬅️ 이전: Dirty Checking 메커니즘](./03-dirty-checking-mechanism.md)** | **[다음: Lazy Loading 프록시 생성 과정 ➡️](./05-lazy-loading-proxy.md)**

</div>
