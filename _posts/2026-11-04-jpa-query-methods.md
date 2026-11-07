---
title: "JPA Query Method 문법 정리"
date: 2026-11-04 09:00:00 +0900
categories: ["JPA", "Repository"]
tags: ["spring-data-jpa", "query-method", "derived-query", "paging", "sorting"]
description: "Spring Data JPA Query Method의 이름 규칙과 비교·범위·문자열·정렬·페이징 검색 문법을 예제로 정리한다."
toc: true
---

## 1. 예제 Entity

```java
@Entity
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private String email;
    private int age;
    private boolean active;
    private LocalDateTime createdAt;

    @ManyToOne
    private Team team;
}
```

```java
@Entity
public class Team {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
}
```

Repository:

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

---

## 2. JpaRepository 기본 제공 메서드

`JpaRepository`를 상속하면 기본적인 CRUD 메서드를 제공한다.

```java
Member save(Member member);
// 저장 또는 수정

Optional<Member> findById(Long id);
// ID로 회원 1명 조회

List<Member> findAll();
// 전체 회원 조회

boolean existsById(Long id);
// 해당 ID의 회원 존재 여부 확인

long count();
// 전체 회원 수 조회

void deleteById(Long id);
// ID로 회원 삭제

void delete(Member member);
// 회원 객체 삭제

void deleteAll();
// 전체 회원 삭제
```

---

## 3. Query Method 기본 구조

Query Method는 **메서드 이름을 규칙에 맞게 작성하면 Spring Data JPA가 조회 쿼리를 자동으로 생성하는 방식**이다.

가장 기본적인 형태:

```text
findBy + 조건
```

```java
List<Member> findByName(String name);
// 이름이 정확히 일치하는 회원 조회
```

```text
WHERE name = ?
```

즉,

```text
findBy + Name
```

으로 구성되어 있다.

---

## 4. 기본 조건 검색

```java
List<Member> findByName(String name);
// 이름이 정확히 일치하는 회원 조회

List<Member> findByEmail(String email);
// 이메일이 정확히 일치하는 회원 조회

List<Member> findByAge(int age);
// 나이가 정확히 일치하는 회원 조회

Optional<Member> findOneByEmail(String email);
// 이메일이 하나만 존재한다고 가정하고 회원 1명 조회
```

Java는 반환 타입만 다른 메서드를 같은 인터페이스에 선언할 수 없으므로 단건 예제는 `findOneByEmail`로 구분했다.

조회 결과가 여러 개일 수 있다면 `List`를 사용하고, 단건 조회라면 `Optional` 등을 사용할 수 있다.

---

## 5. 비교 연산

### GreaterThan

```java
List<Member> findByAgeGreaterThan(int age);
// age보다 나이가 많은 회원

// WHERE age > ?
```

### GreaterThanEqual

```java
List<Member> findByAgeGreaterThanEqual(int age);
// age 이상인 회원

// WHERE age >= ?
```

### LessThan

```java
List<Member> findByAgeLessThan(int age);
// age보다 나이가 적은 회원

// WHERE age < ?
```

### LessThanEqual

```java
List<Member> findByAgeLessThanEqual(int age);
// age 이하인 회원

// WHERE age <= ?
```

### Not

```java
List<Member> findByNameNot(String name);
// 해당 이름이 아닌 회원

// WHERE name <> ?
```

---

## 6. 범위 검색

### Between

```java
List<Member> findByAgeBetween(int min, int max);
// 나이가 min 이상 max 이하인 회원

// WHERE age BETWEEN ? AND ?
```

---

## 7. 문자열 검색

### Like

```java
List<Member> findByNameLike(String name);
// LIKE 조건으로 이름 검색

// WHERE name LIKE ?
```

### Containing

```java
List<Member> findByNameContaining(String keyword);
// 이름에 keyword가 포함된 회원

// WHERE name LIKE %keyword%
```

### StartingWith

```java
List<Member> findByNameStartingWith(String prefix);
// 이름이 prefix로 시작하는 회원

// WHERE name LIKE prefix%
```

### EndingWith

```java
List<Member> findByNameEndingWith(String suffix);
// 이름이 suffix로 끝나는 회원

// WHERE name LIKE %suffix
```

### NotContaining

```java
List<Member> findByNameNotContaining(String keyword);
// 이름에 keyword가 포함되지 않은 회원
```

### IgnoreCase

```java
List<Member> findByNameIgnoreCase(String name);
// 대소문자를 구분하지 않고 이름 검색
```

다른 문자열 조건과 조합할 수도 있다.

```java
List<Member> findByNameContainingIgnoreCase(String keyword);
// 대소문자를 구분하지 않고 이름에 keyword가 포함된 회원
```

---

## 8. NULL 검색

```java
List<Member> findByEmailIsNull();
// email이 NULL인 회원

List<Member> findByEmailIsNotNull();
// email이 NULL이 아닌 회원
```

---

## 9. 여러 값 검색

### In

```java
List<Member> findByAgeIn(Collection<Integer> ages);
// 여러 나이 중 하나에 해당하는 회원
```

```java
List<Member> findByNameIn(Collection<String> names);
// 여러 이름 중 하나에 해당하는 회원
```

### NotIn

```java
List<Member> findByAgeNotIn(Collection<Integer> ages);
// 주어진 나이에 해당하지 않는 회원
```

---

## 10. Boolean 검색

`Member`에 다음과 같은 필드가 있다고 가정한다.

```java
private boolean active;
```

```java
List<Member> findByActiveTrue();
// active가 true인 회원

List<Member> findByActiveFalse();
// active가 false인 회원
```

---

## 11. 날짜 / 시간 검색

`createdAt`이 `LocalDateTime`이라고 가정한다.

```java
List<Member> findByCreatedAtAfter(LocalDateTime date);
// date 이후에 생성된 회원

List<Member> findByCreatedAtBefore(LocalDateTime date);
// date 이전에 생성된 회원

List<Member> findByCreatedAtGreaterThanEqual(LocalDateTime date);
// date 이후 또는 같은 시간에 생성된 회원

List<Member> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
// 특정 시간 범위에 생성된 회원
```

---

## 12. AND 조건

`And`를 사용하면 여러 조건을 동시에 만족하는 데이터를 조회한다.

```java
List<Member> findByNameAndAge(String name, int age);
// 이름과 나이가 모두 일치하는 회원
```

```java
List<Member> findByNameAndEmail(String name, String email);
// 이름과 이메일이 모두 일치하는 회원
```

```java
List<Member> findByNameAndEmailAndAge(String name, String email, int age);
// 이름, 이메일, 나이가 모두 일치하는 회원
```

---

## 13. OR 조건

`Or`를 사용하면 조건 중 하나를 만족하는 데이터를 조회한다.

```java
List<Member> findByNameOrEmail(String name, String email);
// 이름 또는 이메일이 일치하는 회원
```

```java
List<Member> findByNameOrAge(String name, int age);
// 이름이 일치하거나 나이가 일치하는 회원
```

---

## 14. AND + OR 조합

```java
List<Member> findByNameAndAgeOrEmail(String name, int age, String email);
// (name = ? AND age = ?) OR email = ?
```

Query Method에서는 복잡한 괄호 조건을 표현하기 어렵다.

예를 들어:

```text
name = ? AND (age = ? OR email = ?)
```

처럼 괄호가 필요한 조건은 `@Query`, Specification, QueryDSL 등의 방법을 사용하는 것이 적절하다.

---

## 15. 정렬

### OrderBy

```java
List<Member> findByNameOrderByAgeAsc(String name);
// 이름이 같은 회원을 나이 오름차순으로 조회
```

```java
List<Member> findByNameOrderByAgeDesc(String name);
// 이름이 같은 회원을 나이 내림차순으로 조회
```

여러 필드로 정렬할 수도 있다.

```java
List<Member> findByActiveTrueOrderByAgeDescNameAsc();
// 활성 회원을 나이 내림차순 → 이름 오름차순으로 조회
```

기본 구조:

```text
findBy + 조건 + OrderBy + 필드 + Asc/Desc
```

---

## 16. Top / First

조회 결과의 일부만 가져올 수 있다.

```java
List<Member> findTop3ByOrderByAgeDesc();
// 나이가 많은 회원 3명 조회
```

```java
List<Member> findFirst3ByOrderByAgeDesc();
// 나이가 많은 회원 3명 조회
```

단건 조회에도 사용할 수 있다.

```java
Optional<Member> findTopByOrderByAgeDesc();
// 나이가 가장 많은 회원 1명 조회
```

```java
Optional<Member> findFirstByOrderByAgeDesc();
// 나이가 가장 많은 회원 1명 조회
```

조건과 함께 사용할 수도 있다.

```java
List<Member> findTop3ByTeamNameOrderByAgeDesc(String teamName);
// 특정 팀에서 나이가 많은 회원 3명
```

---

## 17. Distinct

중복 결과를 제거할 수 있다.

```java
List<Member> findDistinctByName(String name);
// 중복을 제거하고 이름이 일치하는 회원 조회
```

---

## 18. existsBy

데이터의 존재 여부만 확인할 때 사용한다.

```java
boolean existsByName(String name);
// 해당 이름의 회원이 존재하는지 확인
```

```java
boolean existsByEmail(String email);
// 해당 이메일의 회원이 존재하는지 확인
```

조건을 조합할 수도 있다.

```java
boolean existsByNameAndEmail(String name, String email);
// 해당 이름과 이메일을 가진 회원이 존재하는지 확인
```

---

## 19. countBy

조건에 해당하는 데이터의 개수를 조회한다.

```java
long countByAge(int age);
// 특정 나이의 회원 수
```

```java
long countByActiveTrue();
// 활성 회원 수
```

```java
long countByTeamName(String teamName);
// 특정 팀에 속한 회원 수
```

---

## 20. deleteBy

조건에 해당하는 데이터를 삭제할 수 있다.

```java
void deleteByName(String name);
// 이름이 일치하는 회원 삭제
```

```java
void deleteByAge(int age);
// 특정 나이의 회원 삭제
```

삭제된 개수를 반환하도록 작성할 수도 있다.

```java
long deleteByActiveFalse();
// 비활성 회원을 삭제하고 삭제된 개수 반환
```

삭제 Query Method는 실제 프로젝트에서 트랜잭션 처리 등을 함께 고려해야 한다.

---

## 21. 연관관계 탐색

`Member`가 `Team`과 `@ManyToOne` 관계라고 가정한다.

```java
@ManyToOne
private Team team;
```

Team의 `name`을 기준으로 회원을 조회할 수 있다.

```java
List<Member> findByTeamName(String teamName);
// Member의 team을 거쳐 Team.name이 일치하는 회원
```

구조:

```text
Member
 └── team
      └── name
```

따라서:

```java
findByTeamName(...)
```

은 다음과 같은 의미다.

```text
member.team.name = ?
```

Team의 ID로도 조회할 수 있다.

```java
List<Member> findByTeamId(Long teamId);
// Member의 team.id가 일치하는 회원
```

---

## 22. `_`를 사용한 연관관계 명시

연관관계를 명확하게 표현하고 싶다면 `_`를 사용할 수 있다.

```java
List<Member> findByTeam_Name(String teamName);
// Member.team.name이 일치하는 회원
```

다음 두 메서드는 같은 의미로 사용할 수 있다.

```java
List<Member> findByTeamName(String teamName);
// Team.name 기준 조회
```

```java
List<Member> findByTeam_Name(String teamName);
// Team.name 기준 조회
```

`_`는 연관관계를 따라가는 경계를 명확하게 표현할 때 사용할 수 있다.

---

## 23. 여러 조건 조합

Query Method는 지금까지 배운 키워드를 조합할 수 있다.

```java
List<Member> findByTeamNameAndAgeGreaterThan(String teamName, int age);
// 특정 팀에 속하면서 특정 나이보다 많은 회원
```

```java
List<Member> findByTeamNameInAndAgeBetween(Collection<String> teamNames, int minAge, int maxAge);
// 여러 팀 중 하나에 속하면서 나이가 특정 범위인 회원
```

```java
List<Member> findTop3ByTeamNameOrderByAgeDesc(String teamName);
// 특정 팀에서 나이가 많은 회원 3명
```

```java
boolean existsByNameAndTeamName(String name, String teamName);
// 특정 팀에 같은 이름의 회원이 존재하는지 확인
```

---

## 24. Pageable 사용

많은 데이터를 한 번에 조회하지 않고 페이지 단위로 조회할 수 있다.

```java
Page<Member> findByAgeGreaterThan(int age, Pageable pageable);
// 특정 나이보다 많은 회원을 페이지 단위로 조회
```

사용:

```java
Pageable pageable = PageRequest.of(0, 10);
Page<Member> members = memberRepository.findByAgeGreaterThan(20, pageable);
// 20살 초과 회원을 한 페이지에 10명씩 조회
```

---

## 25. Sort 사용

정렬 조건을 메서드 이름에 고정하지 않고 외부에서 전달할 수도 있다.

```java
List<Member> findByAgeGreaterThan(int age, Sort sort);
// 특정 나이보다 많은 회원을 원하는 정렬 조건으로 조회
```

사용:

```java
Sort sort = Sort.by(Sort.Direction.DESC, "age");
List<Member> members = memberRepository.findByAgeGreaterThan(20, sort);
// 20살 초과 회원을 나이 내림차순으로 조회
```

---

## 26. Slice 사용

전체 데이터 개수보다 다음 페이지 존재 여부가 필요한 경우 `Slice`를 사용할 수 있다.

```java
Slice<Member> findByAgeGreaterThan(int age, Pageable pageable);
// 특정 나이보다 많은 회원을 Slice 단위로 조회
```

`Page`는 전체 개수 확인이 필요할 수 있고, `Slice`는 다음 데이터가 존재하는지 확인하는 용도로 사용할 수 있다.

---

## 27. Query Method 전체 구조

Query Method는 다음과 같은 형태로 조합해서 생각하면 된다.

```text
조회 종류
↓
find / exists / count / delete
↓
Distinct, Top / First 등 (조회 목적에 따라 선택)
↓
By
↓
조건 (And / Or로 연결)
↓
OrderBy + 필드 + Asc / Desc (선택)
```

예를 들어:

```java
List<Member> findTop3ByTeamNameAndAgeGreaterThanOrderByAgeDesc(String teamName, int age);
// 특정 팀이면서 특정 나이보다 많은 회원 중 나이순 상위 3명
```

메서드를 나누어 보면:

```text
find
→ 조회

Top3
→ 3개까지

By
→ 조건 시작

TeamName
→ 팀 이름 조건

And
→ AND

AgeGreaterThan
→ 나이가 특정 값보다 큼

OrderByAgeDesc
→ 나이 내림차순
```

---

## 28. 자주 사용하는 키워드 정리

| 키워드 | 의미 |
|---|---|
| `findBy` | 조건으로 조회 |
| `existsBy` | 존재 여부 |
| `countBy` | 개수 조회 |
| `deleteBy` | 조건으로 삭제 |
| `And` | AND |
| `Or` | OR |
| `Not` | 같지 않음 |
| `GreaterThan` | `>` |
| `GreaterThanEqual` | `>=` |
| `LessThan` | `<` |
| `LessThanEqual` | `<=` |
| `Between` | 범위 |
| `Like` | LIKE |
| `Containing` | 포함 |
| `StartingWith` | ~로 시작 |
| `EndingWith` | ~로 끝남 |
| `NotContaining` | 포함하지 않음 |
| `IgnoreCase` | 대소문자 무시 |
| `IsNull` | NULL |
| `IsNotNull` | NULL이 아님 |
| `In` | IN |
| `NotIn` | NOT IN |
| `True` | true |
| `False` | false |
| `OrderBy` | 정렬 |
| `Asc` | 오름차순 |
| `Desc` | 내림차순 |
| `Top` | 상위 N개 |
| `First` | 첫 N개 |
| `Distinct` | 중복 제거 |

---

## 29. Query Method를 사용할 때 주의할 점

#### ① 메서드 이름이 너무 길어지는 경우

```java
List<Member> findByTeamNameAndAgeGreaterThanAndActiveTrueAndCreatedAtBetweenOrderByAgeDesc(String teamName, int age, LocalDateTime start, LocalDateTime end);
```

조건이 많아질수록 메서드 이름이 지나치게 길어지고 가독성이 떨어진다.

이런 경우에는 `@Query`, Specification, QueryDSL 등의 방법을 고려한다.

#### ② 복잡한 괄호 조건

```text
name = ? AND (age = ? OR email = ?)
```

처럼 복잡한 조건은 Query Method보다 `@Query` 등을 사용하는 것이 적합하다.

#### ③ 너무 복잡한 조회

단순한 조건 검색에는 Query Method가 편하지만, JOIN이 많거나 복잡한 검색 조건과 DTO 조회가 필요한 경우에는 Query Method만으로 해결하려 하지 않는 것이 좋다.

---

## 30. 핵심 패턴

Query Method를 메서드 전체로 하나씩 외우기보다 다음 구조를 이해하는 것이 중요하다.

```text
findBy + 필드명 + 조건 키워드
```

예:

```java
List<Member> findByAgeGreaterThan(int age);
// findBy + Age + GreaterThan
```

조건을 추가하면:

```java
List<Member> findByTeamNameAndAgeGreaterThan(String teamName, int age);
// findBy + TeamName + And + AgeGreaterThan
```

정렬과 개수 제한까지 추가하면:

```java
List<Member> findTop3ByTeamNameOrderByAgeDesc(String teamName);
// find + Top3 + By + TeamName + OrderBy + Age + Desc
```

결국 Query Method의 핵심은 **메서드 이름을 정해진 규칙에 따라 조립해서 원하는 조회 조건을 표현하는 것**이다.

#### 오늘 배운 것 한 줄 요약

**JPA Query Method는 `findBy`, `And`, `Or`, `GreaterThan`, `Between`, `OrderBy`, `Top` 등의 키워드를 조합해 메서드 이름만으로 조회 조건을 표현하는 기능이다.**
