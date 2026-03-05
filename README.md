<div align="center">

# 🗄️ Spring Data & Transaction Deep Dive

**"Repository 인터페이스만으로 어떻게 구현체가 생기는가"**

<br/>

> *"Repository를 만드는 것과, Repository가 어떻게 작동하는지 아는 것은 다르다"*

동적 프록시 기반 Repository 생성부터 Propagation 7가지, N+1 해결, HikariCP 튜닝까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring Data / JPA / Hibernate 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring](https://img.shields.io/badge/Spring-6.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io)
[![Hibernate](https://img.shields.io/badge/Hibernate-6.x-59666C?style=flat-square&logo=hibernate&logoColor=white)](https://hibernate.org)
[![Docs](https://img.shields.io/badge/Docs-45개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring Data JPA에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`JpaRepository`를 상속하면 CRUD가 자동으로 됩니다" | `RepositoryFactorySupport`가 동적 프록시를 생성하고 `SimpleJpaRepository`로 위임하는 전체 과정 |
| "`@Transactional`을 붙이면 트랜잭션이 적용됩니다" | Propagation 7가지가 각각 `AbstractPlatformTransactionManager`에서 어떻게 분기되는가 |
| "N+1 문제는 `fetch join`으로 해결합니다" | Hibernate의 Lazy Loading 프록시 생성 원리, 5가지 해결 전략별 실제 SQL 쿼리 수 비교 |
| "`readOnly=true`를 붙이면 성능이 좋아집니다" | Hibernate FlushMode 변경, JDBC Connection `readOnly` 힌트, 스냅샷 생략으로 절약되는 실측 비용 |
| "`private` 메서드에는 `@Transactional`이 안 됩니다" | CGLIB 오버라이딩 불가 원리 + Self-Invocation 함정의 바이트코드 수준 설명 |
| 이론 나열 | 실행 가능한 코드 + Hibernate `show_sql` 출력 + 쿼리 수 실측 비교 |

> **선행 학습 권장**: [Spring Core Deep Dive](https://github.com/dev-book-lab/spring-core-deep-dive) — IoC, DI, AOP 프록시 원리를 이해한 후 이 레포를 학습하면 효과가 배가됩니다.

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Spring Data JPA](https://img.shields.io/badge/🔹_Ch1-Repository_프록시_생성_과정-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./spring-data-jpa-internals/01-repository-proxy-creation.md)
[![Transaction](https://img.shields.io/badge/🔹_Ch2-PlatformTransactionManager_구조-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./transaction-management/01-platform-transaction-manager.md)
[![JPA Hibernate](https://img.shields.io/badge/🔹_Ch3-EntityManager_vs_Session-59666C?style=for-the-badge&logo=hibernate&logoColor=white)](./jpa-hibernate-integration/01-entitymanager-vs-session.md)
[![Query Tuning](https://img.shields.io/badge/🔹_Ch4-JPQL_vs_Criteria_vs_QueryDSL-FF6B35?style=for-the-badge)](./query-performance-tuning/01-jpql-criteria-querydsl.md)
[![Spring JDBC](https://img.shields.io/badge/🔹_Ch5-JdbcTemplate_내부_구조-0052CC?style=for-the-badge)](./spring-jdbc/01-jdbctemplate-internals.md)
[![Connection Pool](https://img.shields.io/badge/🔹_Ch6-HikariCP_설정과_최적화-181717?style=for-the-badge)](./connection-pool-management/01-hikaricp-configuration.md)
[![Testing](https://img.shields.io/badge/🔹_Ch7-@DataJpaTest_범위와_제약-DC143C?style=for-the-badge)](./testing-data-layer/01-datajpatest-scope.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Spring Data JPA Internals

> **핵심 질문:** `JpaRepository` 인터페이스만 선언했는데, 런타임에 어떻게 구현체가 생기는가?

<details>
<summary><b>동적 프록시 Repository 생성부터 Query Method 파싱, Auditing까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Repository 프록시 생성 과정](./spring-data-jpa-internals/01-repository-proxy-creation.md) | `RepositoryFactorySupport.getRepository()`가 JDK 동적 프록시를 생성하는 과정, `SimpleJpaRepository`로 위임되는 구조, `@EnableJpaRepositories` 트리거 |
| [02. Query Method 파싱 메커니즘](./spring-data-jpa-internals/02-query-method-parsing.md) | `findByNameAndAge()`가 JPQL로 변환되는 `PartTree` 파싱 과정, Subject / Predicate 분리, 지원 키워드 목록과 한계 |
| [03. @Query JPQL vs Native Query 처리](./spring-data-jpa-internals/03-query-annotation-jpql-native.md) | `@Query` 어노테이션이 `NamedQuery`와 다르게 처리되는 이유, Native Query 실행 경로, `nativeQuery=true`일 때 결과 매핑 방식 |
| [04. Projection — Interface vs DTO](./spring-data-jpa-internals/04-projection-interface-dto.md) | Closed / Open Interface Projection이 Hibernate 프록시로 구현되는 방식, DTO Projection이 인터페이스 방식보다 성능상 유리한 조건 |
| [05. QueryDSL 통합 원리](./spring-data-jpa-internals/05-querydsl-integration.md) | `QuerydslPredicateExecutor`가 Spring Data에 통합되는 방식, `JPAQueryFactory` 설정과 `Q타입` 생성 메커니즘, 타입 안전 동적 쿼리 |
| [06. Specifications를 통한 동적 쿼리](./spring-data-jpa-internals/06-specifications-dynamic-query.md) | `JpaSpecificationExecutor`와 `Specification<T>` 인터페이스, Criteria API를 래핑하는 구조, Specification 조합(`and`, `or`) 원리 |
| [07. Auditing 동작 원리](./spring-data-jpa-internals/07-auditing-internals.md) | `@CreatedDate` / `@LastModifiedDate`가 `AuditingEntityListener`와 `AuditorAware`를 통해 주입되는 과정, `@EnableJpaAuditing` 설정 체인 |
| [08. Custom Repository 구현 패턴](./spring-data-jpa-internals/08-custom-repository-pattern.md) | `Impl` 네이밍 컨벤션의 동작 원리, `JpaRepositoryImplementation` 위임 구조, `EntityManager` 직접 사용이 필요한 경우와 설계 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 2: Transaction Management

> **핵심 질문:** `@Transactional(propagation=REQUIRES_NEW)`는 내부적으로 무슨 일을 하는가?

<details>
<summary><b>PlatformTransactionManager부터 Propagation 7가지, Rollback 규칙까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. PlatformTransactionManager 구조와 구현체](./transaction-management/01-platform-transaction-manager.md) | `PlatformTransactionManager` 인터페이스 설계, `JpaTransactionManager` vs `DataSourceTransactionManager` 구현 차이, `TransactionStatus` 객체 역할 |
| [02. @Transactional 프록시 생성 메커니즘](./transaction-management/02-transactional-proxy-mechanism.md) | `TransactionInterceptor`가 AOP Advice로 등록되는 과정, `TransactionAttributeSource`의 어노테이션 파싱, 트랜잭션 시작/커밋/롤백 흐름 |
| [03. Propagation 7가지 완전 분석](./transaction-management/03-propagation-seven-types.md) | `REQUIRED` / `REQUIRES_NEW` / `NESTED` / `SUPPORTS` / `NOT_SUPPORTED` / `MANDATORY` / `NEVER` 각각의 분기 조건과 내부 동작, 물리 트랜잭션 vs 논리 트랜잭션 구분 |
| [04. Isolation Level과 Database Lock](./transaction-management/04-isolation-level-lock.md) | READ_UNCOMMITTED ~ SERIALIZABLE 4단계의 JDBC 설정 경로, Phantom Read / Non-Repeatable Read 재현 코드, DB별 기본 Isolation Level과 Lock 전략 |
| [05. private 메서드에 @Transactional이 안 되는 이유](./transaction-management/05-why-private-transactional-fails.md) | CGLIB 오버라이딩 불가 원리, Self-Invocation 함정 (`this.method()` 호출 시 프록시 우회), `AopContext.currentProxy()` 우회 방법과 트레이드오프 |
| [06. readOnly=true의 실제 효과](./transaction-management/06-readonly-true-effects.md) | Hibernate `FlushMode.MANUAL` 설정, 스냅샷 비교 생략 비용 절감, JDBC `Connection.setReadOnly()` 힌트, MySQL/PostgreSQL 리플리카 라우팅 연계 |
| [07. Rollback 규칙 — checked vs unchecked](./transaction-management/07-rollback-rules.md) | Spring 기본 롤백 정책 (`RuntimeException` 기준), `rollbackFor` / `noRollbackFor` 설정 방법, checked exception으로 롤백하지 않아 데이터가 오염되는 실제 사례 |
| [08. TransactionSynchronization 활용](./transaction-management/08-transaction-synchronization.md) | `TransactionSynchronizationManager`의 ThreadLocal 기반 자원 관리, `afterCommit()` / `afterCompletion()` 훅 활용 패턴, `@TransactionalEventListener`와의 연결 |

</details>

<br/>

### 🔹 Chapter 3: JPA & Hibernate Integration

> **핵심 질문:** Hibernate는 어떻게 변경을 감지하고, 언제 쿼리를 날리는가?

<details>
<summary><b>Persistence Context부터 N+1 완전 해결, Batch Insert 최적화까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. EntityManager vs Hibernate Session](./jpa-hibernate-integration/01-entitymanager-vs-session.md) | JPA 표준 `EntityManager`와 Hibernate `Session`의 관계, `EntityManager.unwrap(Session.class)` 필요한 상황, Spring의 `SharedEntityManagerCreator` 프록시 구조 |
| [02. Persistence Context — 1차 캐시 동작 원리](./jpa-hibernate-integration/02-persistence-context-first-cache.md) | `PersistenceContext` 내부 `EntityKey → EntityEntry` 맵 구조, 동일 트랜잭션 내 동일 ID 조회 시 쿼리가 발생하지 않는 이유, Detached / Managed / Removed 상태 전환 |
| [03. Dirty Checking 메커니즘](./jpa-hibernate-integration/03-dirty-checking-mechanism.md) | `ActionQueue`와 `EntityEntry.loadedState` 스냅샷 비교 방식, `flush()` 시점에 변경 감지가 일어나는 내부 코드, FlushMode 별 동작 차이 (`AUTO` vs `COMMIT`) |
| [04. N+1 문제 완전 해결](./jpa-hibernate-integration/04-n-plus-one-complete-solution.md) | N+1 발생 원리 (Lazy Loading 프록시 초기화 시점), `@EntityGraph` / `Fetch Join` / `@BatchSize` / `FetchMode.SUBSELECT` / DTO Projection 5가지 해결 전략별 실제 SQL 쿼리 수 비교 |
| [05. Lazy Loading 프록시 생성 과정](./jpa-hibernate-integration/05-lazy-loading-proxy.md) | Hibernate `ByteBuddyProxyFactory`가 엔티티 서브클래스를 생성하는 방식, `LazyInitializationException` 발생 조건, OSIV(Open Session In View) 패턴의 트레이드오프 |
| [06. Cascade 타입과 Orphan Removal](./jpa-hibernate-integration/06-cascade-orphan-removal.md) | `CascadeType` 6가지의 내부 동작 원리, `orphanRemoval=true`와 `CascadeType.REMOVE`의 차이, 무분별한 `ALL` 사용의 위험성과 연관관계 설계 기준 |
| [07. Batch Insert/Update 최적화](./jpa-hibernate-integration/07-batch-insert-update.md) | `hibernate.jdbc.batch_size` 설정만으로 Batch가 안 되는 이유 (`IDENTITY` 전략 한계), `SEQUENCE` 전략과 `allocationSize` 조합, `saveAll()` 호출 시 실제 발생하는 SQL 수 실측 |

</details>

<br/>

### 🔹 Chapter 4: Query Performance Tuning

> **핵심 질문:** 같은 결과를 내는 쿼리인데, 왜 어떤 방법은 SQL이 1개고 어떤 방법은 N+1개인가?

<details>
<summary><b>JPQL/QueryDSL 비교부터 Pagination, Second-Level Cache까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. JPQL vs Criteria API vs QueryDSL 비교](./query-performance-tuning/01-jpql-criteria-querydsl.md) | 세 방식의 컴파일 타임 안전성 / 동적 쿼리 지원 / 성능 비교, 각각이 내부에서 `CriteriaQuery`와 `NativeQuery`로 변환되는 경로 |
| [02. Fetch Join vs @EntityGraph 선택 기준](./query-performance-tuning/02-fetch-join-vs-entity-graph.md) | Fetch Join과 `@EntityGraph`가 생성하는 SQL 비교, `MultipleBagFetchException` 발생 조건과 회피 전략, 컬렉션 2개 이상 조인 시 선택 기준 |
| [03. Pagination 최적화 전략](./query-performance-tuning/03-pagination-optimization.md) | `Pageable`을 사용한 `LIMIT/OFFSET` 방식의 성능 함정 (대용량 오프셋), Cursor 기반 Pagination, `CountQuery` 분리 최적화, `@QueryHints`로 페이지 카운트 캐싱 |
| [04. @BatchSize vs default_batch_fetch_size](./query-performance-tuning/04-batch-size-configuration.md) | `@BatchSize`(엔티티 레벨)와 `hibernate.default_batch_fetch_size`(전역 설정)의 차이, IN 절 파라미터 수가 성능에 미치는 영향, 적정 값 설정 기준 |
| [05. Query Plan Cache 동작](./query-performance-tuning/05-query-plan-cache.md) | Hibernate가 JPQL을 파싱해 `QueryPlan`으로 캐싱하는 구조, `hibernate.query.plan_cache_max_size` 튜닝, 파라미터 바인딩 방식(`?` vs `:name`)이 캐시 히트율에 미치는 영향 |
| [06. Second-Level Cache — Ehcache / Redis](./query-performance-tuning/06-second-level-cache.md) | `@Cache` 어노테이션이 Hibernate Region Factory를 통해 Ehcache/Redis에 저장되는 구조, `READ_ONLY` / `READ_WRITE` / `NONSTRICT_READ_WRITE` 전략 차이, 캐시 무효화 시점 |

</details>

<br/>

### 🔹 Chapter 5: Spring JDBC

> **핵심 질문:** `JdbcTemplate`은 어떻게 반복되는 JDBC 보일러플레이트를 제거하는가?

<details>
<summary><b>JdbcTemplate 내부 구조부터 Batch 처리까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. JdbcTemplate 내부 구조](./spring-jdbc/01-jdbctemplate-internals.md) | Template Method 패턴으로 `Connection` 획득 / `PreparedStatement` 생성 / 예외 변환 / `Connection` 반환이 추상화되는 방식, `DataSourceUtils`를 통한 트랜잭션 컨텍스트 연계 |
| [02. RowMapper vs ResultSetExtractor](./spring-jdbc/02-rowmapper-vs-resultsetextractor.md) | `RowMapper`(행 단위 변환)와 `ResultSetExtractor`(전체 `ResultSet` 제어)의 사용 시점, `BeanPropertyRowMapper`의 리플렉션 비용과 커스텀 `RowMapper`와의 성능 비교 |
| [03. NamedParameterJdbcTemplate 활용](./spring-jdbc/03-named-parameter-jdbctemplate.md) | `:name` 바인딩이 내부적으로 `?` 치환되는 파싱 과정, `MapSqlParameterSource` vs `BeanPropertySqlParameterSource` 차이, SQL Injection 방지 원리 |
| [04. SimpleJdbcInsert로 단순 삽입](./spring-jdbc/04-simple-jdbc-insert.md) | `SimpleJdbcInsert`가 DB 메타데이터를 조회해 컬럼 목록을 자동 감지하는 방식, `usingGeneratedKeyColumns()`로 자동 생성 키를 받는 과정 |
| [05. Batch 처리 최적화](./spring-jdbc/05-batch-processing.md) | `batchUpdate()`가 `PreparedStatement.addBatch()` / `executeBatch()`를 감싸는 구조, Chunk 단위 처리 패턴, JPA `saveAll()`과의 실측 처리 속도 비교 |

</details>

<br/>

### 🔹 Chapter 6: Connection Pool Management

> **핵심 질문:** 적정 Pool Size는 어떻게 결정하며, Connection Leak은 어디서 발생하는가?

<details>
<summary><b>HikariCP 내부 구조부터 Leak 탐지, Statement 캐싱까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. HikariCP 설정과 최적화](./connection-pool-management/01-hikaricp-configuration.md) | `HikariPool` 내부 `ConcurrentBag` 자료구조, 주요 설정값(`maximumPoolSize`, `minimumIdle`, `connectionTimeout`) 의미와 상호작용, Spring Boot 자동 설정 경로 |
| [02. Connection Leak 탐지와 디버깅](./connection-pool-management/02-connection-leak-detection.md) | `leakDetectionThreshold` 설정 시 `ProxyLeakTask`가 스택 트레이스를 캡처하는 방식, 트랜잭션 밖에서 `EntityManager`를 직접 사용할 때 발생하는 Leak 패턴 |
| [03. Pool Size 튜닝 공식](./connection-pool-management/03-pool-size-tuning.md) | `Connections = cores × 2 + effective_spindle_count` 공식의 이론적 배경, I/O 바운드 vs CPU 바운드 서비스에서의 적정값 차이, 과도한 Pool Size가 오히려 성능을 낮추는 이유 |
| [04. Connection Health Check 전략](./connection-pool-management/04-connection-health-check.md) | `connectionTestQuery` vs `connectionInitSql` vs JDBC4 `isValid()` 방식 비교, `keepaliveTime`으로 유휴 Connection 유효성 유지, DB 재시작 후 Pool 복구 동작 |
| [05. Statement Caching 효과](./connection-pool-management/05-statement-caching.md) | HikariCP `cachePrepStmts` 설정이 DB 드라이버 레벨 캐싱과 연동되는 방식, MySQL `prepStmtCacheSize`와 `prepStmtCacheSqlLimit` 튜닝 포인트, 캐싱 전후 파싱 비용 실측 |
| [06. Connection Timeout vs Idle Timeout](./connection-pool-management/06-timeout-configuration.md) | `connectionTimeout`(연결 대기) / `idleTimeout`(유휴 제거) / `maxLifetime`(최대 수명) 세 타임아웃의 역할 차이, DB 방화벽 강제 종료 시나리오에서 `maxLifetime` 설정의 중요성 |

</details>

<br/>

### 🔹 Chapter 7: Testing Data Layer

> **핵심 질문:** `@DataJpaTest`는 무엇을 로딩하고, 테스트의 `@Transactional`은 왜 함정인가?

<details>
<summary><b>슬라이스 테스트부터 Testcontainers, @Sql 관리까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. @DataJpaTest 범위와 제약](./testing-data-layer/01-datajpatest-scope.md) | `@DataJpaTest`가 로딩하는 슬라이스 컨텍스트 범위, 기본 H2 인메모리 DB 설정 경로, `@Service` 빈이 로딩되지 않는 이유와 `@Import`로 추가하는 방법 |
| [02. @Transactional in Test의 함정](./testing-data-layer/02-transactional-test-pitfall.md) | 테스트 메서드의 `@Transactional` 자동 롤백이 실제 트랜잭션 동작을 은폐하는 방식, `REQUIRES_NEW` 동작이 테스트에서 다르게 보이는 이유, 통합 테스트에서 `@Rollback(false)` 사용 기준 |
| [03. Testcontainers vs H2 선택 기준](./testing-data-layer/03-testcontainers-vs-h2.md) | H2 호환 모드의 한계(JSON 타입, DB 고유 함수), Testcontainers `@DynamicPropertySource`로 실제 DB를 띄우는 방식, CI 환경에서의 컨테이너 재사용 전략 |
| [04. Repository 테스트 전략](./testing-data-layer/04-repository-test-strategy.md) | Query Method 단위 테스트 / 복잡한 JPQL 통합 테스트 / Custom Repository 격리 테스트 전략 구분, `TestEntityManager`를 활용한 fixture 데이터 준비 패턴 |
| [05. @Sql 스크립트 관리](./testing-data-layer/05-sql-script-management.md) | `@Sql`의 `executionPhase`(`BEFORE_TEST_METHOD` / `AFTER_TEST_METHOD`) 동작 순서, SQL 스크립트 격리 전략, `@SqlConfig`로 트랜잭션 처리 방식 제어 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "@Transactional이 뭔지 정확히 알고 싶다" — 면접 준비 / 실무 의문 해소 (2주)</b></summary>

<br/>

**Week 1 — Repository와 트랜잭션의 작동 방식**
```
Ch1-01  Repository 프록시 생성 과정
Ch2-01  PlatformTransactionManager 구조
Ch2-02  @Transactional 프록시 생성 메커니즘
Ch2-05  private 메서드에 @Transactional이 안 되는 이유
```

**Week 2 — 실무 혼동 포인트와 성능 이슈**
```
Ch2-03  Propagation 7가지 완전 분석 (면접 단골)
Ch2-06  readOnly=true의 실제 효과
Ch2-07  Rollback 규칙 — checked vs unchecked
Ch3-04  N+1 문제 완전 해결
```

</details>

<details>
<summary><b>🔵 JPA/Hibernate 내부를 원리로 이해하고 싶은 개발자 (6주)</b></summary>

<br/>

```
Week 1  Chapter 1 전체 — Spring Data JPA 동적 프록시 Repository 생성
Week 2  Chapter 2 전체 — 트랜잭션 관리 내부 구조
Week 3  Chapter 3 전체 — Hibernate Persistence Context / Dirty Checking / N+1
Week 4  Chapter 4 전체 — 쿼리 성능 튜닝 (QueryDSL, Pagination, 2차 캐시)
Week 5  Chapter 5 전체 + Chapter 6 전체 — JDBC Template + Connection Pool
Week 6  Chapter 7 전체 — 데이터 레이어 테스트 전략
```

</details>

<details>
<summary><b>🔴 성능 최적화가 목표인 개발자 (집중 코스)</b></summary>

<br/>

```
핵심 경로 (쿼리 수와 응답 시간 직접 개선)

Step 1  Ch3-04  N+1 문제 완전 해결 → 즉각적인 쿼리 수 감소
Step 2  Ch4-02  Fetch Join vs @EntityGraph 선택 기준
Step 3  Ch4-03  Pagination 최적화 (OFFSET 함정 → Cursor 전환)
Step 4  Ch3-07  Batch Insert/Update → 대량 저장 속도 개선
Step 5  Ch6-03  Pool Size 튜닝 공식 → 커넥션 대기 시간 제거
Step 6  Ch4-06  Second-Level Cache → 반복 조회 비용 제거
Step 7  Ch2-06  readOnly=true → 스냅샷 비용 절감

각 문서의 "⚡ 성능 임팩트" 섹션에서 Before/After 쿼리 수와 실행 시간을 확인하세요.
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이게 존재하는가** | 문제 상황과 설계 배경 |
| 😱 **흔한 오해 또는 잘못된 사용** | Before — 많은 개발자가 틀리는 방식 |
| ✨ **올바른 이해와 사용** | After — 원리를 알고 난 후의 올바른 접근 |
| 🔬 **내부 동작 원리** | Spring Data / Hibernate 소스코드 직접 추적 + 다이어그램 |
| 💻 **실험으로 확인하기** | 직접 실행 가능한 코드 + `hibernate.show_sql` 출력 예시 |
| ⚡ **성능 임팩트** | Before/After 쿼리 수, 실행 시간 비교 |
| 🤔 **트레이드오프** | 이 설계의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🙏 Reference

- [Spring Data JPA Reference Documentation](https://docs.spring.io/spring-data/jpa/reference/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/)
- [Spring Framework Reference — Data Access](https://docs.spring.io/spring-framework/reference/data-access.html)
- [High-Performance Java Persistence — Vlad Mihalcea](https://vladmihalcea.com/books/high-performance-java-persistence/)
- [HikariCP GitHub](https://github.com/brettwooldridge/HikariCP)
- [Baeldung Spring Data JPA Guides](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Repository를 만드는 것과, Repository가 어떻게 작동하는지 아는 것은 다르다"*

</div>
