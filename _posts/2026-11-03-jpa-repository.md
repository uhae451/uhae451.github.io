---
title: "JpaRepository"
date: 2026-11-03 09:00:00 +0900
categories: ["JPA", "Repository"]
tags: ["spring-data-jpa", "jparepository", "repository", "crud", "paging"]
description: "Spring Data JPA의 JpaRepository 기본 메서드와 리포지토리 작성, 조회 및 페이징 관련 문법을 정리한다."
toc: true
---

> `JpaRepository`를 상속만 했는데 `save()`, `findById()`가 되는 이유와, 기본 메서드로 부족할 때 쓰는 문법들을 코드 중심으로 정리했습니다.

## 0. 예제 엔티티

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

 @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
 private Long id;

 private String name;
 private int age;
 private String email;

 @ManyToOne(fetch = FetchType.LAZY)
 @JoinColumn(name = "team_id")
 private Team team;

 public Member(String name, int age, String email, Team team) {
 this.name = name;
 this.age = age;
 this.email = email;
 this.team = team;
 }

 public void changeName(String name) {
 this.name = name;
 }
}
```

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

 @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
 private Long id;

 private String name;

 public Team(String name) {
 this.name = name;
 }
}
```

## 1. JpaRepository란?

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

`JpaRepository<엔티티 타입, PK 타입>`을 상속하면 **구현 클래스 없이** CRUD가 가능합니다. Spring Data JPA가 애플리케이션 시작 시점에 **프록시 구현체를 만들어 빈으로 등록**해주기 때문입니다.

| 인터페이스 | 제공하는 기능 |
|---|---|
| `Repository` | 마커 인터페이스 (기능 없음) |
| `CrudRepository` | `save`, `findById`, `findAll`, `delete`, `count` 등 기본 CRUD |
| `PagingAndSortingRepository` | 페이징과 정렬 (`findAll(Pageable)`, `findAll(Sort)`) |
| `JpaRepository` | 위 기능 전부 + `flush`, `saveAndFlush`, 배치 삭제 등 JPA 특화 기능 |

## 2. 기본 제공 메서드

```java
// 저장 / 수정
Member saved = memberRepository.save(member);
List<Member> savedAll = memberRepository.saveAll(members);

// 조회
Optional<Member> found = memberRepository.findById(1L);
List<Member> all = memberRepository.findAll();
List<Member> some = memberRepository.findAllById(List.of(1L, 2L, 3L));
boolean exists = memberRepository.existsById(1L);
long count = memberRepository.count();

// 삭제
memberRepository.deleteById(1L);
memberRepository.delete(member);

// JPA 특화
memberRepository.flush(); // 영속성 컨텍스트 → DB 동기화
memberRepository.saveAndFlush(member); // 저장 후 즉시 flush
memberRepository.getReferenceById(1L); // 프록시(지연 로딩) 조회
memberRepository.deleteAllInBatch(); // 한 번의 DELETE 쿼리로 삭제
```

### 2-1. save()는 두 가지로 동작한다

```java
// 새 엔티티 (id == null) → em.persist()
Member newMember = new Member("hyun", 28, "a@a.com", null);
memberRepository.save(newMember);

// 이미 id가 있는 엔티티 → em.merge() (SELECT 후 병합)
memberRepository.save(detachedMember);
```

- 위 예제처럼 `@Version`이 없으면 **`@Id` 값이 null인지**로 판단합니다. 기본 전략은 먼저 nullable `@Version`을 검사하고, 없으면 ID를 검사합니다. `Persistable`을 구현하면 `isNew()`로 판단할 수 있습니다.
- `merge()`는 전달된 객체의 상태를 관리 엔티티에 복사하고 그 관리 엔티티를 반환합니다. 필요하면 SELECT하며, 전달한 준영속 객체 자체가 영속 상태로 바뀌는 것은 아닙니다. null까지 복사될 수 있으므로 부분 수정에는 주의해야 합니다. 수정이 목적이라면 `save()`보다 **조회 후 변경 감지**를 쓰는 편이 안전합니다. (10장 참고)

### 2-2. findById vs getReferenceById

```java
// 1차 캐시에 없으면 보통 SELECT, 없으면 Optional.empty()
Member member = memberRepository.findById(1L)
 .orElseThrow(() -> new EntityNotFoundException("회원이 없습니다."));

// SELECT 없이 프록시만 반환 → 실제 필드를 사용할 때 조회
Team teamRef = teamRepository.getReferenceById(1L);
Member member = new Member("hyun", 28, "a@a.com", teamRef); // FK만 필요할 때 유용
```

연관관계 FK만 채우면 되는 상황이라면 `getReferenceById`로 **불필요한 SELECT를 줄일 수 있습니다.**

### 2-3. 삭제 메서드는 방식이 다르다

| 메서드 | 동작 |
|---|---|
| `deleteAll()` | 전체 조회 후 **엔티티를 하나씩 `remove`** (쿼리 N+1) |
| `deleteAllInBatch()` | `DELETE FROM member` **한 번**. 영속성 컨텍스트를 거치지 않음 |
| `deleteAllByIdInBatch(ids)` | `DELETE ... WHERE id IN (...)` 한 번 |

데이터가 많을 때 `deleteAll()`은 느리니 주의하세요.

---

## 3. 쿼리 메서드: 메서드 이름으로 쿼리 만들기

기본 메서드로 부족한 조회는 **메서드 이름 규칙**에 맞춰 선언만 하면 쿼리가 만들어집니다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

 List<Member> findByName(String name);

 Optional<Member> findByEmail(String email);

 List<Member> findByNameAndAge(String name, int age);
}
```

```sql
-- findByNameAndAge("hyun", 28)
select m.id, m.age, m.email, m.name, m.team_id
from member m
where m.name = ? and m.age = ?
```

### 3-1. 자주 쓰는 키워드

| 키워드 | 예시 | 의미 |
|---|---|---|
| `And`, `Or` | `findByNameAndAge` | 조건 결합 |
| `Between` | `findByAgeBetween(20, 30)` | 범위 |
| `LessThan`, `GreaterThan` | `findByAgeGreaterThanEqual(20)` | 비교 (`Equal` 붙이면 이상/이하) |
| `Like`, `Containing` | `findByNameContaining("hy")` | `%hy%` 검색 |
| `StartingWith`, `EndingWith` | `findByNameStartingWith("h")` | 접두/접미 검색 |
| `In` | `findByAgeIn(List.of(20, 30))` | IN 조건 |
| `IsNull`, `IsNotNull` | `findByTeamIsNull()` | NULL 체크 |
| `OrderBy` | `findByAgeOrderByNameDesc(28)` | 정렬 |
| `Top`, `First` | `findTop3ByOrderByAgeDesc()` | 상위 N개 |
| `Distinct` | `findDistinctByName("hyun")` | 중복 제거 |

```java
List<Member> findByAgeBetween(int from, int to);
List<Member> findByNameContaining(String keyword);
List<Member> findByAgeIn(Collection<Integer> ages);
List<Member> findTop3ByOrderByAgeDesc();
Member findFirstByOrderByAgeDesc();
```

### 3-2. 조회 외 접두어: exists, count, delete

```java
boolean existsByEmail(String email); // 중복 체크에 유용
long countByTeam(Team team);

@Transactional
long deleteByName(String name); // 먼저 조회 → 엔티티를 하나씩 삭제
```

`deleteBy...`는 **조회 후 하나씩 삭제**하므로, 대량 삭제에는 5장의 `@Modifying` 쿼리를 쓰는 게 좋습니다.

### 3-3. 연관관계 필드 접근

```java
List<Member> findByTeamName(String teamName); // member.team.name 으로 탐색 (자동 조인)
List<Member> findByTeam(Team team);
```

### 3-4. 반환 타입

```java
Member findByName(String name); // 결과 없으면 null, 2건 이상이면 예외
Optional<Member> findByEmail(String email); // 단건 조회는 Optional 권장
List<Member> findByAge(int age); // 결과 없으면 빈 리스트 (null 아님)
```

> 💡 쿼리 메서드는 조건이 2~3개를 넘어가면 `findByNameAndAgeGreaterThanAndTeamNameOrderByAgeDesc` 같은 괴물 이름이 됩니다. 그때부터는 `@Query`로 넘어가세요.

---

## 4. @Query: 직접 JPQL 작성하기

```java
@Query("select m from Member m where m.name = :name and m.age >= :age")
List<Member> findUsers(@Param("name") String name, @Param("age") int age);
```

- 대상은 테이블이 아니라 **엔티티와 필드**입니다. (`member`가 아니라 `Member`)
- 파라미터는 `:이름` + `@Param`으로 바인딩합니다. 순서 기반(`?1`)보다 이름 기반이 안전합니다.
- JPQL 문법은 보통 **Repository 초기화 시점에 검증**됩니다. 다만 지연 초기화 설정이나 실제 DB의 SQL·타입 문제까지 모두 보장하지는 않으므로 실행 테스트도 필요합니다.

### 4-1. 엔티티 대신 필요한 값만 DTO로 조회

```java
public record MemberTeamDto(Long id, String name, String teamName) {}
```

```java
@Query("""
 select new com.example.dto.MemberTeamDto(m.id, m.name, t.name)
 from Member m
 join m.team t
 where m.age >= :age
 """)
List<MemberTeamDto> findMemberTeam(@Param("age") int age);
```

DTO 생성자 표현식은 **패키지 경로를 포함한 풀네임**을 써야 합니다.

### 4-2. 네이티브 쿼리

```java
@Query(value = "select * from member where email like %:domain", nativeQuery = true)
List<Member> findByEmailDomain(@Param("domain") String domain);
```

DB 전용 함수가 꼭 필요한 경우가 아니라면 JPQL을 쓰는 게 DB 변경에 유리합니다.

---

## 5. @Modifying: 벌크 수정과 삭제

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Member m set m.age = m.age + 1 where m.age >= :age")
int bulkAgePlus(@Param("age") int age);

@Modifying(clearAutomatically = true)
@Query("delete from Member m where m.age < :age")
int bulkDeleteYoung(@Param("age") int age);
```

- 반환값은 **영향받은 row 수**입니다.
- 벌크 연산은 **영속성 컨텍스트를 거치지 않고 DB에 바로 실행**됩니다. 그래서 컨텍스트에 이미 올라와 있던 엔티티는 **변경 전 값**을 그대로 들고 있게 됩니다.

```java
@Transactional
public void test() {
 Member member = memberRepository.save(new Member("hyun", 28, "a@a.com", null));

 memberRepository.bulkAgePlus(20); // DB는 29로 변경

 Member found = memberRepository.findById(member.getId()).get();
 found.getAge(); // clearAutomatically = false 라면 1차 캐시의 28이 반환됨
}
```

- `clearAutomatically = true`: 실행 후 영속성 컨텍스트를 비워서 이후 조회가 DB 값을 보게 합니다.
- `flushAutomatically = true`: 실행 전에 아직 반영되지 않은 변경을 먼저 DB에 반영합니다.
- 반드시 **`@Transactional` 안에서** 실행해야 합니다.

---

## 6. 페이징과 정렬

```java
Page<Member> findByAge(int age, Pageable pageable); // 전체 개수 쿼리 포함
Slice<Member> findByName(String name, Pageable pageable); // 다음 페이지 유무만 확인
List<Member> findByName(String name, Sort sort); // 정렬만
```

```java
// 페이지 번호는 0부터 시작, 한 페이지 크기는 10
Pageable pageable = PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "name"));

Page<Member> page = memberRepository.findByAge(28, pageable);

page.getContent(); // 현재 페이지 데이터 (List<Member>)
page.getTotalElements(); // 전체 데이터 수
page.getTotalPages(); // 전체 페이지 수
page.hasNext(); // 다음 페이지 존재 여부
```

| 타입 | 특징 |
|---|---|
| `Page` | **count 쿼리를 추가로 실행**해서 전체 개수와 페이지 수를 제공 |
| `Slice` | count 없이 `size + 1`개를 조회해 다음 페이지 유무만 판단. 무한 스크롤에 적합 |

### 6-1. 조인이 있는 @Query의 count 쿼리 분리

```java
@Query(value = "select m from Member m left join m.team t where m.age >= :age",
 countQuery = "select count(m) from Member m where m.age >= :age")
Page<Member> findPageWithTeam(@Param("age") int age, Pageable pageable);
```

조인이 들어간 쿼리를 그대로 count에 쓰면 **불필요한 조인 때문에 느려질** 수 있어서, `countQuery`로 단순화합니다.

### 6-2. 컨트롤러에서 바로 받기

```java
@GetMapping("/members")
public Page<MemberDto> list(@PageableDefault(size = 10, sort = "id") Pageable pageable) {
 return memberRepository.findAll(pageable).map(MemberDto::from); // 엔티티 → DTO 변환
}
```

`GET /members?page=0&size=10&sort=name,desc` 요청이 `Pageable`로 자동 변환됩니다.

---

## 7. 연관관계를 함께 조회하기: fetch join과 @EntityGraph

회원 목록을 조회하며 `member.getTeam().getName()`을 호출하면 회원마다 Team 조회 쿼리가 추가로 나가는 **N+1 문제**가 생깁니다.

```java
// ① JPQL fetch join
@Query("select m from Member m join fetch m.team")
List<Member> findAllWithTeam();

// ② @EntityGraph: JPQL 없이 연관 엔티티를 함께 조회
@EntityGraph(attributePaths = {"team"})
List<Member> findByAgeGreaterThan(int age);

// ③ 기본 메서드에도 적용 가능
@Override
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();
```

- `@EntityGraph`는 쿼리 메서드에 **어노테이션만 추가**하면 돼서 간단한 경우에 편합니다. (내부적으로는 outer join)
- 조건이 복잡하면 `fetch join` JPQL을 직접 쓰는 편이 명확합니다.
- ⚠️ **컬렉션(`@OneToMany`) fetch join에 페이징을 함께 쓰면** Hibernate가 DB가 아니라 **메모리에서 페이징**을 합니다. 데이터가 많으면 위험하니, 이 경우엔 `batch size` 설정으로 푸는 방법을 씁니다.

---

## 8. Projection: 필요한 컬럼만 조회

```java
// 인터페이스 기반 (Closed Projection)
public interface MemberNameOnly {
 String getName();
 int getAge();
}
```

```java
List<MemberNameOnly> findByAgeGreaterThan(int age);
// select m.name, m.age from member m where m.age > ? ← 필요한 컬럼만 SELECT
```

```java
// 클래스(record) 기반 DTO
public record MemberSummary(String name, int age) {}

List<MemberSummary> findByTeamName(String teamName);
```

```java
// 호출 시점에 타입 지정 (Dynamic Projection)
<T> List<T> findByName(String name, Class<T> type);

memberRepository.findByName("hyun", MemberNameOnly.class);
memberRepository.findByName("hyun", Member.class);
```

엔티티 전체가 필요 없는 **조회 전용 API**에서 쓰면 SELECT 컬럼이 줄고, 엔티티가 영속성 컨텍스트에 올라가지 않아 가볍습니다.

---

## 9. 커스텀 Repository: 직접 구현이 필요할 때

동적 쿼리처럼 메서드 이름이나 `@Query`로 표현하기 어려운 경우, **인터페이스 + 구현체**를 만들어 `JpaRepository`에 합칩니다.

```java
// ① 커스텀 인터페이스
public interface MemberRepositoryCustom {
 List<Member> search(String name, Integer age);
}
```

```java
// ② 기본 구현체 탐색 규칙: "커스텀 인터페이스명 + Impl"
@RequiredArgsConstructor
public class MemberRepositoryCustomImpl implements MemberRepositoryCustom {

 private final EntityManager em;

 @Override
 public List<Member> search(String name, Integer age) {
 StringBuilder jpql = new StringBuilder("select m from Member m where 1 = 1");
 if (name != null) jpql.append(" and m.name = :name");
 if (age != null) jpql.append(" and m.age >= :age");

 TypedQuery<Member> query = em.createQuery(jpql.toString(), Member.class);
 if (name != null) query.setParameter("name", name);
 if (age != null) query.setParameter("age", age);

 return query.getResultList();
 }
}
```

```java
// ③ 기존 Repository에 합치기
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
}
```

`memberRepository.search("hyun", 20)`처럼 **기본 메서드와 똑같이** 호출할 수 있습니다. 조건이 복잡한 동적 쿼리는 문자열 조립보다 **QueryDSL**을 쓰는 것이 일반적이고, 위 구조에 그대로 적용됩니다.

---

## 10. 서비스에서의 실제 사용

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // 기본은 읽기 전용
public class MemberService {

 private final MemberRepository memberRepository;

 public MemberDto find(Long id) {
 Member member = memberRepository.findById(id)
 .orElseThrow(() -> new EntityNotFoundException("회원이 없습니다."));
 return MemberDto.from(member);
 }

 @Transactional // 쓰기 작업만 readOnly 해제
 public void rename(Long id, String newName) {
 Member member = memberRepository.findById(id).orElseThrow();
 member.changeName(newName); // save() 호출 없이 변경 감지로 UPDATE
 }
}
```

- 조회 메서드는 `readOnly = true`로 두면 **변경 감지를 위한 스냅샷 비교를 생략**해 성능에 유리합니다.
- 수정은 `save()`가 아니라 **조회 → 값 변경**이 기본 패턴입니다.

---

## 11. 정리

| 상황 | 선택 |
|---|---|
| 단순 CRUD | 기본 제공 메서드 |
| 조건 1~2개의 단순 조회 | **쿼리 메서드** (`findByNameAndAge`) |
| 조건이 길거나 조인이 필요 | **`@Query`** (JPQL) |
| 일괄 UPDATE / DELETE | **`@Modifying`** + `clearAutomatically = true` |
| 목록 + 페이지 정보 / 무한 스크롤 | `Page` / `Slice` |
| 연관 엔티티를 함께 조회 (N+1 방지) | fetch join, `@EntityGraph` |
| 일부 컬럼만 조회 | Projection, DTO 조회 |
| 동적 쿼리 | 커스텀 Repository + QueryDSL |

**자주 하는 실수**

- 수정할 때 `save()`를 습관적으로 호출한다. 영속 상태라면 변경 감지로 충분하고, 준영속이면 `merge`로 전체가 덮어써진다.
- `deleteAll()`로 대량 삭제를 한다. `deleteAllInBatch()`나 `@Modifying` 쿼리를 고려한다. 단, 벌크 삭제는 엔티티별 cascade·삭제 콜백을 실행하지 않으므로 FK와 연관 데이터 처리도 확인한다.
- 벌크 쿼리 후 `clearAutomatically`를 빼먹어 오래된 값을 읽는다.
- 컬렉션 fetch join과 페이징을 같이 쓴다.

## 참고 문서

- [Spring Data JPA 엔티티 저장과 신규 판별](https://docs.spring.io/spring-data/jpa/reference/jpa/entity-persistence.html)
