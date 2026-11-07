---
title: "N+1 문제"
date: 2026-11-06 09:00:00 +0900
categories: ["JPA", "Performance"]
tags: ["jpa", "n-plus-one", "fetch-join", "entity-graph", "batch-fetch"]
description: "JPA N+1 문제를 재현하고 SQL 횟수를 측정한 뒤 Fetch Join, EntityGraph, Batch Size와 DTO 조회 방법을 비교한다."
toc: true
---

> "Fetch Join 쓰면 되죠?"에서 끝나지 않으려고, **N+1을 직접 재현하고 쿼리 횟수를 숫자로 측정**했습니다.
> 왜 발생하는지, 해결책마다 실제로 어떤 SQL이 나가는지, 각각의 한계는 무엇인지까지 정리했습니다.

---

## 1. N+1 문제란?

**연관된 엔티티를 조회할 때, 처음 1번의 쿼리 때문에 연관 데이터를 가져오기 위한 쿼리가 N번 추가로 실행되는 문제**입니다.

팀(Team) 목록을 조회하고, 각 팀의 멤버(Member)에 접근하면 이렇게 됩니다.

```sql
SELECT * FROM team;                      -- 1번 (팀 N개 조회)
SELECT * FROM member WHERE team_id = 1;  -- N번 시작
SELECT * FROM member WHERE team_id = 2;
SELECT * FROM member WHERE team_id = 3;
...
```

팀이 N개면 쿼리가 **1 + N번** 나갑니다. 데이터가 늘수록 DB 호출 횟수가 선형으로 늘어 성능이 나빠지는데, 코드만 봐서는 눈치채기 어렵다는 점이 문제입니다.

---

## 2. 직접 재현해 보기

### 엔티티

```java
@Entity
public class TeamD1012 {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String name;

  @OneToMany(mappedBy = "team")          // 기본 fetch = LAZY
  private List<MemberD1012> members = new ArrayList<>();
}

@Entity
public class MemberD1012 {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)     // @ManyToOne 기본은 EAGER 라서 LAZY 명시
  @JoinColumn(name = "team_id")
  private TeamD1012 team;
}
```

### `members`는 DB 컬럼이 아니다

처음에 가장 헷갈렸던 부분입니다. `team` 테이블에는 `members`라는 컬럼이 **없습니다.**

```
team 테이블              member 테이블
 id  name              id  name  team_id
```

`@OneToMany(mappedBy = "team") List<Member> members`는 *"member 테이블에서 team_id가 나인 행들을 찾아서 자바 리스트로 보여주겠다"* 는 **자바 쪽 편의 기능**입니다. 그래서 `teamRepository.findAll()`을 해도 `members`는 조회하지 않고, 비어 있는 **프록시(지연 로딩)** 상태로 둡니다. **`members`를 실제로 건드리는 순간** 그 팀의 멤버를 가져오는 쿼리가 나갑니다.

### 쿼리 횟수 측정 방법

SQL 로그를 눈으로 세는 것만으로는 부정확해서, Hibernate의 통계 기능으로 쿼리 수를 셌습니다.

```java
Statistics statistics = em.getEntityManagerFactory()
    .unwrap(SessionFactory.class).getStatistics();   // Hibernate 전용 객체에서 통계 꺼내기
statistics.setStatisticsEnabled(true);               // 기본이 꺼져 있어서 켜야 함
statistics.clear();                                  // 데이터 저장 때 나간 INSERT 는 빼고 0부터 세기
...
statistics.getPrepareStatementCount();               // 지금까지 나간 SQL 수
```

> **트러블슈팅**: 처음에는 `setStatisticsEnabled(true)`를 빼먹어서 쿼리가 분명히 나갔는데도 `expected: 1 but was: 0`으로 테스트가 실패했습니다. 통계는 **기본이 꺼져 있기 때문**입니다.

### 테스트 데이터와 조회 코드

팀 5개, 팀당 멤버 3명을 저장한 뒤 `em.flush(); em.clear();`로 DB에 반영하고 영속성 컨텍스트를 비웁니다. (안 비우면 1차 캐시에서 꺼내와서 N+1이 재현되지 않습니다.)

```java
List<TeamD1012> teams = teamRepository.findAll();            // 팀 조회 1번

for (TeamD1012 team : teams) {
  System.out.println(team.getName() + " 멤버 수 : " + team.getMembers().size());  // 팀마다 쿼리 1번씩
}
```

### 실제 실행 로그

```sql
-- ① findAll() : 팀 조회 (1번)
select td1_0.id, td1_0.name from teams_d1012 td1_0

-- ② 반복문에서 team.getMembers() 에 접근할 때마다 (팀 수만큼, 5번)
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id=?
팀1 멤버 수 : 3
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id=?
팀2 멤버 수 : 3
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id=?
팀3 멤버 수 : 3
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id=?
팀4 멤버 수 : 3
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id=?
팀5 멤버 수 : 3

쿼리 수 : 6
```

**팀 5개 → 쿼리 6번 (1 + N).** 위 로그에서 `where m1_0.team_id=?` 쿼리가 팀마다 한 번씩 반복되는 것이 보입니다.

> 참고로 `members`를 건드리지 않고 `findAll()`만 하면 쿼리는 **1번**뿐이었습니다. N+1은 `findAll()` 시점이 아니라 **연관 데이터에 접근하는 시점**에 발생합니다.

---

## 3. 왜 발생할까? (원인)

핵심은 **연관 데이터의 로딩 시점과 SQL로 가져오는 방법은 별개**라는 점입니다. Spring Data JPA의 기본 `findAll()`은 Criteria API를 사용하지만, 연관관계를 조인으로 한 번에 가져오지 않으면 JPQL 조회와 마찬가지로 추가 쿼리가 발생할 수 있습니다.

1. 이 예제의 `findAll()`은 **팀 목록을 먼저 조회**합니다.
2. `LAZY`는 사용 시점까지 로딩을 미룰 수 있고, `EAGER`는 반환 전에 로딩을 보장해야 합니다. EAGER가 반드시 SQL JOIN을 뜻하지는 않습니다.
3. 필요한 멤버를 개별 SELECT로 로딩하면 팀마다 추가 쿼리가 발생합니다.

### 즉시 로딩(EAGER)으로 바꾸면 해결될까?
```java
@OneToMany(mappedBy = "team", fetch = FetchType.EAGER)
```

**EAGER만으로 해결을 보장하지 않습니다.** 먼저 팀을 조회하고 EAGER 조건을 충족하기 위해 반환 전에 팀마다 멤버를 추가 조회할 수 있습니다. 오히려 필요 없는 곳에서도 항상 조회되어 더 위험합니다.

> **결론: 모든 연관관계는 `LAZY`로 설정하고, 필요한 곳에서만 한 번에 가져온다.**
> (`@ManyToOne`, `@OneToOne`은 기본값이 EAGER이므로 **명시적으로 LAZY 지정**해야 합니다.)

---

## 4. 해결 방법과 실제 결과

### 4-1. Fetch Join

JPQL에서 `join fetch`를 쓰면 연관 엔티티를 **한 번의 쿼리**로 함께 조회합니다.

```java
@Query("select distinct t from TeamD1012 t join fetch t.members")
List<TeamD1012> findAllWithMembers();
```

**실제 실행된 SQL**

```sql
select distinct td1_0.id, m1_0.team_id, m1_0.id, m1_0.name, td1_0.name
from teams_d1012 td1_0
join members_d1012 m1_0 on td1_0.id=m1_0.team_id
```

**쿼리 수: 6 → 1**

> `ORDER BY`가 없는 `DISTINCT` 쿼리라서 결과 순서가 보장되지 않습니다. 실제로 출력 순서가 `팀3, 팀2, 팀4, 팀1, 팀5`로 뒤섞여 나왔습니다. 순서가 중요하면 `order by`를 명시해야 합니다.

**주의할 점**

1. **컬렉션 페치 조인은 페이징이 안전하지 않습니다.**
   - 1:N 조인은 결과 row가 N쪽 기준으로 늘어납니다. (팀 1개 + 멤버 3명 → 3 row)
   - 이 상태에서 `Pageable`을 쓰면 Hibernate가 경고(`HHH000104`)를 내고 **모든 데이터를 메모리로 읽어서 페이징**합니다. → OutOfMemory 위험
2. **여러 bag 컬렉션을 동시에 페치 조인하면** Hibernate에서 `MultipleBagFetchException`이 발생합니다. Set이나 인덱스가 있는 List 등은 가능할 수 있지만, 여러 ToMany를 동시에 조인하면 결과 행이 곱으로 늘어나 성능 문제가 생길 수 있습니다.
3. 페치 조인 대상에는 **별칭을 주고 조건을 거는 것을 지양**합니다. 연관 데이터가 일부만 담긴 채로 영속성 컨텍스트에 올라가 일관성이 깨질 수 있습니다.

### 4-2. @EntityGraph

애너테이션만 붙여서 페치 조인과 같은 효과를 냅니다.

```java
@EntityGraph(attributePaths = "members")
@Query("select t from TeamD1012 t")
List<TeamD1012> findAllWithMembersByGraph();
```

**실제 실행된 SQL** (내부적으로 **LEFT OUTER JOIN**)

```sql
select td1_0.id, m1_0.team_id, m1_0.id, m1_0.name, td1_0.name
from teams_d1012 td1_0
left join members_d1012 m1_0 on td1_0.id=m1_0.team_id
```

**쿼리 수: 6 → 1**

이 예제의 `join fetch`는 INNER JOIN이고 EntityGraph는 LEFT JOIN SQL로 실행되어 멤버가 없는 팀도 포함합니다. Fetch Join도 `left join fetch`로 작성할 수 있으며, EntityGraph는 로딩 대상을 지정하는 기능이므로 SQL 조인 종류를 명세로 고정하지 않습니다. 여기처럼 컬렉션을 조인으로 로딩하면 **컬렉션 페이징 문제도 동일**합니다.

### 4-3. Batch Size (컬렉션 N+1의 실무 해법)

지연 로딩 시 연관 데이터를 **여러 개씩 묶어서** 한 번에 가져옵니다.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

테스트에서는 전역 설정이 다른 테스트에 영향을 주지 않도록 세션에만 적용했습니다.

```java
em.unwrap(Session.class).setFetchBatchSize(100);
```

**실제 실행된 SQL**

```sql
-- ① 팀 조회 (1번)
select td1_0.id, td1_0.name from teams_d1012 td1_0

-- ② 팀 5개의 멤버를 한 번에 조회 (1번)
select m1_0.team_id, m1_0.id, m1_0.name from members_d1012 m1_0 where m1_0.team_id = any (?)
```

**쿼리 수: 6 → 2**

> 흔히 `IN (1, 2, 3, ...)`으로 설명하지만, **PostgreSQL에서는 `= any (?)`** 형태로 배열 파라미터 하나로 나갔습니다. DB와 Hibernate 버전에 따라 SQL 모양은 달라도 "여러 팀을 묶어서 한 번에 조회한다"는 원리는 같습니다.

**장점**
- 컬렉션이 있어도 **페이징이 정상 동작**합니다. (팀을 DB에서 페이징 → 가져온 팀들의 멤버만 묶어서 조회)
- 둘 이상의 컬렉션도 문제없습니다.
- 설정 한 줄로 전체에 적용할 수 있습니다.

이처럼 한 종류의 컬렉션 N개를 같은 영속성 컨텍스트에서 배치 크기만큼 묶어 로딩하면 대략 `1 + ⌈N / batch size⌉`번입니다. 캐시, 접근 순서, 실제 배치 구성에 따라 달라지며 Page의 count 쿼리는 별도입니다.

### 4-4. DTO 직접 조회

엔티티가 아닌 **필요한 컬럼만 DTO로 바로 조회**합니다.

```java
@Query("select new com.example.dto.TeamMemberDto(t.name, m.name) " +
       "from Team t join t.members m")
List<TeamMemberDto> findTeamMemberDtos();
```

- 장점: 필요한 컬럼만 가져와 성능이 좋고, 영속성 컨텍스트를 거치지 않아 N+1이 일어나지 않습니다.
- 단점: 엔티티를 재사용하기 어렵고 조회 전용에 한정됩니다. 화면에 딱 맞춘 **조회 API**에 적합합니다.

---

## 5. 측정 결과 한눈에 보기

팀 5개, 팀당 멤버 3명 기준입니다.

| 방법 | 쿼리 수 | 실제 SQL 특징 |
|---|---|---|
| 아무것도 안 함 (N+1) | **6** | `where team_id=?` 가 팀마다 반복 |
| Fetch Join | **1** | `join` 한 번 (INNER JOIN, `distinct`) |
| EntityGraph | **1** | `left join` 한 번 |
| Batch Size (100) | **2** | `where team_id = any (?)` 로 묶어서 조회 |

이 예제에서 Fetch Join과 EntityGraph는 1번이지만, Batch Size는 팀 수가 배치 크기를 넘으면 추가 조회가 늘어납니다. **2번은 팀 5개·배치 크기 100인 이번 측정 결과**입니다.

---

## 6. 그래서 뭘 써야 할까?

| 상황 | 추천 |
|---|---|
| **ToOne** (`@ManyToOne`, `@OneToOne`) 연관 | **Fetch Join** (페이징 영향 없음) |
| **ToMany** (`@OneToMany`) 연관 + 페이징 | **`default_batch_fetch_size`** + 필요 시 ToOne만 fetch join |
| 컬렉션을 한 번에, 페이징 없음 | Fetch Join / EntityGraph |
| 조회 전용 복잡한 화면 | DTO 직접 조회 (QueryDSL 등) |

### 실무의 정석 패턴

1. 모든 연관관계를 **LAZY**로 설정한다.
2. `default_batch_fetch_size`를 **전역으로 설정**해 둔다. (컬렉션 N+1의 방어선)
3. **ToOne 관계는 Fetch Join**으로 한 번에 가져온다.
4. 복잡한 조회는 **DTO 조회**로 분리한다.

---

## 7. 정리

- N+1은 연관 데이터를 개별 조회하면서 추가 쿼리가 반복되는 문제입니다. 이 LAZY 예제에서는 접근 시 발생하고, EAGER에서는 최초 조회 결과를 반환하기 전에도 발생할 수 있습니다.
- **EAGER로 바꾼다고 해결되지 않습니다.** (오히려 악화) 항상 LAZY를 기본으로 합니다.
- 해결책은 **Fetch Join, EntityGraph, Batch Size, DTO 조회**입니다.
- **컬렉션 페치 조인 + 페이징 = 메모리 위험**, 이 경우엔 Batch Size를 사용합니다.
- 성능 문제는 **쿼리 로그와 쿼리 횟수를 직접 측정**하고 검증하는 습관이 중요합니다.

---

## 8. 면접 대비 질문과 모범 답안

### Q1. N+1 문제가 무엇이고 왜 발생하나요?

**A.** 연관 관계가 있는 엔티티를 조회할 때, 처음 조회 쿼리 1번 이후에 **연관된 엔티티를 가져오기 위한 쿼리가 N번 추가로 실행**되는 문제입니다. 예를 들어 팀 5개를 조회한 뒤 각 팀의 멤버에 접근하면 쿼리가 6번 나갑니다. 원인은 필요한 연관 데이터를 한 번에 가져오지 않고 개별 SELECT로 로딩하기 때문입니다. 먼저 팀만 조회하고, 이후 지연 로딩된 연관 데이터에 접근할 때 건건이 추가 쿼리가 실행됩니다.

### Q2. 즉시 로딩(EAGER)으로 바꾸면 N+1이 해결되나요?

**A.** EAGER만으로는 해결되지 않습니다. EAGER는 로딩을 보장하지만 JOIN을 강제하지 않아 **반환 전에 개별 SELECT로 연관 엔티티를 조회**하면서 N+1이 발생할 수 있습니다. 또한 연관 데이터가 필요 없는 상황에서도 항상 조회되어 불필요한 쿼리가 생깁니다. 그래서 **모든 연관관계는 LAZY로 두고**, 필요한 시점에 Fetch Join 등으로 함께 조회하는 것이 좋습니다.

### Q3. N+1을 해결하는 방법에는 어떤 것이 있나요?

**A.** 네 가지를 주로 씁니다.
1. **Fetch Join**: JPQL에서 연관 엔티티를 한 번의 쿼리로 함께 조회합니다.
2. **@EntityGraph**: 조회 시 함께 로딩할 연관관계를 선언합니다. 이번 Hibernate 예제에서는 LEFT JOIN으로 실행됐습니다.
3. **Batch Size**: 지연 로딩 시 연관 데이터를 묶어서 조회해 쿼리 수를 `1+N`에서 `1+⌈N/size⌉`로 줄입니다.
4. **DTO 직접 조회**: 필요한 컬럼만 DTO로 조회해 엔티티 연관 로딩 자체를 피합니다.

직접 팀 5개로 측정했을 때 N+1은 6번, Fetch Join과 EntityGraph는 1번, Batch Size는 2번이었습니다.

### Q4. Fetch Join의 단점이나 주의할 점은 무엇인가요?

**A.** 가장 큰 문제는 **컬렉션(1:N) 페치 조인과 페이징을 함께 쓸 수 없다**는 점입니다. 1:N 조인은 결과 row가 늘어나기 때문에, Hibernate가 DB에서 페이징하지 못하고 **모든 데이터를 메모리로 가져와 페이징**합니다. 데이터가 많으면 OutOfMemory 위험이 있습니다. 또한 **여러 bag의 동시 페치 조인은 제한되며**(`MultipleBagFetchException`), 다른 컬렉션도 결과 행의 곱 증가에 주의해야 합니다. 별칭을 사용해 조건을 거는 것은 영속성 컨텍스트의 일관성을 해칠 수 있어 지양합니다.

### Q5. 컬렉션 조회에서 페이징이 필요하면 어떻게 하나요?

**A.** 두 단계로 접근합니다. 먼저 **ToOne 관계는 Fetch Join**으로 가져오고(row가 늘어나지 않아 페이징에 영향이 없습니다), **컬렉션은 LAZY로 두고 `default_batch_fetch_size`(또는 `@BatchSize`)를 설정**합니다. 그러면 본 엔티티는 DB에서 정상적으로 페이징되고, 조회된 엔티티들의 컬렉션만 묶어서 한 번에 가져오므로 쿼리 수도 크게 줄어듭니다.

### Q6. Fetch Join과 @EntityGraph의 차이는 무엇인가요?

**A.** Fetch Join은 JPQL에서 `join fetch` 또는 `left join fetch`로 조인 방식을 직접 표현합니다. EntityGraph는 함께 로딩할 속성을 선언하며 SQL 생성 방식은 구현체가 결정합니다. 이번 예제는 각각 INNER JOIN과 LEFT JOIN으로 실행됐지만 이를 모든 환경의 규칙으로 일반화하면 안 됩니다. 컬렉션을 조인으로 로딩할 때는 둘 다 페이징에 주의해야 합니다.

### Q7. N+1 문제를 어떻게 발견하고 예방하나요?

**A.** 개발 단계에서 **SQL 로그를 켜서** 실행되는 쿼리 개수를 직접 확인하고, Hibernate Statistics나 p6spy로 **쿼리 횟수를 테스트로 검증**하는 방법이 있습니다. 저는 Hibernate Statistics의 `getPrepareStatementCount()`로 쿼리 수를 세어 N+1 재현과 해결 여부를 테스트 코드의 `assertThat`으로 검증했습니다. 예방 차원에서는 모든 연관관계를 LAZY로 설정하고, `default_batch_fetch_size`를 전역으로 지정하며, ToOne은 Fetch Join, 복잡한 조회는 DTO로 분리하는 원칙을 세워 둡니다. 운영 환경에서는 APM으로 API당 쿼리 수를 모니터링하는 것도 좋습니다.

### Q8. `@ManyToOne`의 기본 fetch 전략은 무엇이고, 왜 LAZY로 바꿔야 하나요?

**A.** `@ManyToOne`과 `@OneToOne`은 기본이 **EAGER**입니다. (`@OneToMany`, `@ManyToMany`는 LAZY) EAGER는 연관 엔티티가 필요 없는 경우에도 항상 조회하고, JPQL에서 N+1을 유발하며, 연관관계가 얽히면 예상치 못한 쿼리가 많이 발생합니다. 그래서 `fetch = FetchType.LAZY`로 **명시적으로 지연 로딩을 지정**하고, 필요한 시점에 Fetch Join으로 가져오는 것이 권장됩니다.

---

## 마치며

N+1은 JPA를 쓰는 개발자라면 반드시 한 번은 만나는 문제입니다. 개념을 아는 것에서 그치지 않고 **직접 재현해서 쿼리 수를 세어 보니**, `findAll()`에서는 쿼리가 1번인데 연관 데이터에 접근하는 순간 팀 수만큼 늘어난다는 점이 훨씬 선명하게 이해됐습니다.

테스트 코드(`NPlusOneTest`)는 팀 수를 상수 하나(`TEAM_COUNT`)로 바꿀 수 있어서, 팀 수를 늘리며 쿼리 수 차이를 비교해 볼 수도 있습니다.

## 참고 문서

- [Hibernate ORM 6.6 User Guide](https://docs.hibernate.org/orm/6.6/userguide/html_single/)
