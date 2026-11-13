---
title: "LAZY만으로는 N+1이 해결되지 않는다 — 조회 계획과 페이징을 함께 설계하기"
date: 2026-11-07 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["Courses","JPA기본"]
description: "프록시와 지연 로딩에서 N+1, fetch join, 배치 로딩과 페이징의 조회 계획까지 연결한다."
toc: true
---

대출 목록 API에서 책 제목을 보여 주려고 `loan.getBook().getTitle()`을 호출했습니다. 자바 코드로는 한 줄이지만, 목록의 각 항목에서 SQL이 실행된다면 응답 시간이 데이터 수에 따라 늘어납니다.

이 문제를 `LAZY`와 `EAGER` 중 하나를 고르는 것으로만 접근하면 해결이 어렵습니다. 매핑의 로딩 설정은 기본 동작이고, API마다 필요한 데이터는 다릅니다. **조회 시점에 어떤 연관 데이터를 어떤 단위로 가져올지**를 별도로 설계해야 합니다.

> 기준: Jakarta Persistence 3.1, Hibernate 6.6의 동작을 기준으로 설명합니다. 쿼리 횟수와 행 수는 명시한 가정 아래의 계산 예시이며 실측 성능 결과가 아닙니다. 예제 연관관계는 `Loan N:1 Book`, `Book N:1 Publisher`, `Publisher 1:N Book`입니다.

## 1. LAZY는 조회를 없애는 기능이 아니다

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "book_id")
private Book book;
```

지연 로딩은 연관 상태를 실제로 사용하기 전까지 로딩을 미룰 수 있게 합니다. 사용하지 않으면 비용을 줄일 수 있지만, 결국 모든 항목의 책 제목을 읽는다면 그 데이터는 어딘가에서 가져와야 합니다.

Hibernate의 일반적인 프록시 방식에서는 식별자를 가진 대리 객체를 먼저 제공하고, 상태가 필요할 때 초기화합니다. 다만 이미 관리 중인 객체가 있거나 바이트코드 향상을 사용하는 경우가 있어 모든 참조가 항상 같은 형태의 프록시인 것은 아닙니다.

```java
Book reference = em.getReference(Book.class, bookId);
// 일반적으로 상태 로딩을 미룰 수 있다.
String title = reference.getTitle();
// 상태가 아직 없다면 초기화가 필요하다.
```

`getReference()`는 무조건 SELECT가 없다는 약속으로 사용하면 안 됩니다. 존재하지 않는 ID에 대한 확인 시점도 구현과 접근 방식에 따라 달라집니다. 존재 여부를 즉시 검증해야 하는 요구와 FK 참조만 필요한 요구를 구분해야 합니다.

### EAGER는 조인 한 번이라는 뜻이 아니다

EAGER는 연관 데이터를 즉시 가져와야 한다는 요구이지, 반드시 하나의 SQL 조인으로 가져오라는 지시가 아닙니다. JPQL로 루트를 조회한 뒤 연관 엔티티를 추가 SELECT로 로딩할 수도 있습니다.

따라서 LAZY는 불필요한 자동 로딩을 줄이기 위한 출발점이고, EAGER로 바꾸는 것은 N+1의 일반적인 해결책이 아닙니다. JPA의 LAZY는 힌트이므로 특정 매핑에서 실제 지연 동작이 가능한지도 확인해야 합니다.

## 2. N+1의 N은 목록 길이와 항상 같지 않다

```java
List<Loan> loans = em.createQuery(
    "select l from Loan l order by l.id", Loan.class)
    .getResultList();

for (Loan loan : loans) {
    String title = loan.getBook().getTitle();
}
```

대출 100건이 서로 다른 책 100권을 참조하고, 컨텍스트가 비어 있으며, 배치 로딩과 2차 캐시가 없다면 대출 조회 1회와 책 조회 최대 100회를 예상할 수 있습니다.

반면 100건이 같은 책 10권만 반복 참조한다면 1차 캐시 때문에 추가 조회는 줄어들 수 있습니다. 실제 수는 이미 로딩된 상태와 조회 설정에도 영향을 받습니다.

이 차이는 문제를 재현할 때 중요합니다. 테스트 데이터를 모두 같은 책으로 만들면 N+1의 증가 양상이 잘 드러나지 않습니다. 반대로 “100건이면 반드시 101번”이라고 단정하면 실제 로그와 다른 이유를 찾지 못합니다.

N+1의 핵심은 정확한 숫자 자체보다 **루트 목록 조회 이후 항목별 연관 조회가 반복되는 구조**입니다.

## 3. 일반 join과 fetch join은 조회 목적이 다르다

다음 쿼리는 특정 제목의 책을 대출한 이력을 찾습니다.

```java
select l from Loan l
join l.book b
where b.title = :title
```

`Book`을 조건에 사용했지만 SELECT 대상은 `Loan`입니다. 이 조인만으로 `Loan.book`이 사용 가능한 상태로 초기화된다고 기대하면 안 됩니다.

반면 목록에서 책 상태까지 함께 사용할 목적이라면 fetch join을 고려합니다.

```java
select l from Loan l
join fetch l.book
order by l.id
```

fetch join은 연관 엔티티의 상태도 함께 로딩해 객체 그래프를 채우려는 의도입니다. “프록시가 절대 없다”는 식보다 **연관 상태가 로딩되어 추가 초기화 조회가 필요하지 않다**고 이해하는 편이 정확합니다.

단일 값 연관인 N:1은 정상적인 FK 관계라면 루트 한 건이 조인 때문에 여러 건으로 늘어나지 않습니다. 따라서 다음처럼 대출 기준 페이징과 함께 사용하기 좋습니다.

```java
List<Loan> page = em.createQuery("""
    select l from Loan l
    join fetch l.book
    order by l.id desc
    """, Loan.class)
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();
```

이 예제는 모든 대출에 책이 존재한다는 전제입니다. 선택적 연관관계에서 내부 조인을 사용하면 대상이 없는 루트가 결과에서 빠질 수 있으므로 `left join fetch`가 필요한지도 확인합니다.

## 4. 컬렉션 fetch join에서는 SQL 행 수를 계산해야 한다

출판사와 출판사의 책 목록을 한 번에 가져오는 쿼리입니다.

```java
select p from Publisher p
left join fetch p.books
```

출판사 하나에 책이 30권이면 그 출판사는 SQL 결과에서 30행에 걸쳐 표현됩니다. 출판사 컬럼도 행마다 반복됩니다. 애플리케이션에서 부모 객체가 하나로 정리되더라도 DB가 반환한 행과 전송량은 그대로 존재합니다.

Hibernate 6은 fetch join으로 생기는 루트 엔티티 결과 중복을 자동으로 제거합니다. 따라서 이 목적만으로 `distinct`를 추가할 필요는 없습니다. 하지만 결과 리스트 중복 제거가 DB의 행 증폭까지 없애 주는 것은 아닙니다. [Hibernate 6.6의 DISTINCT 설명](https://docs.hibernate.org/orm/6.6/querylanguage/html_single/#distinct)

여기에 직원 컬렉션까지 동시에 조인한다고 합시다. 출판사 한 곳의 책이 30권이고 직원이 10명이면 해당 출판사에서 최대 300개 조합이 생깁니다. 쿼리 횟수는 한 번이어도 데이터 처리 비용은 커집니다.

다중 bag fetch는 Hibernate에서 `MultipleBagFetchException`으로 거절될 수 있습니다. 모든 다중 컬렉션 fetch가 문법적으로 불가능한 것은 아니지만, Set으로 바꿔 예외만 피하더라도 곱으로 늘어나는 결과 행 문제는 남습니다. 컬렉션 자료구조는 예외 회피가 아니라 중복·순서에 대한 업무 요구로 결정해야 합니다.

## 5. 컬렉션 fetch join과 페이징이 충돌하는 이유

“출판사 20개”를 조회하려는데 SQL 결과는 “출판사와 책의 조합 행”입니다. DB에서 조합 행 20개만 자르면 첫 번째 출판사의 책 일부만 포함될 수도 있습니다. 부모 단위 페이지와 SQL 행 단위 제한이 서로 다릅니다.

Hibernate는 이 상황에서 전체 결과를 가져온 뒤 메모리에서 페이지를 자를 수 있습니다. 개발 DB에서는 빨라 보이다가 데이터가 늘면 문제가 드러나는 유형입니다.

실수 탐지용으로 다음 설정을 검토할 수 있습니다.

```yaml
spring:
  jpa:
    properties:
      hibernate.query.fail_on_pagination_over_collection_fetch: true
```

이 옵션은 잘못된 페이징을 최적화해 주는 설정이 아닙니다. 메모리 페이징이 필요한 쿼리에 예외를 발생시켜 발견하기 쉽게 합니다. [Hibernate의 해당 설정 설명](https://docs.hibernate.org/orm/6.6/userguide/html_single/#settings-hibernate.query.fail_on_pagination_over_collection_fetch)

### 대안 A: 부모를 페이지로 조회한 뒤 컬렉션 배치 로딩

부모 목록은 DB에서 페이징하고, 필요한 부모들의 컬렉션을 묶어 로딩합니다.

```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 50
```

```java
List<Publisher> page = em.createQuery("""
    select p from Publisher p order by p.id desc
    """, Publisher.class)
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();

for (Publisher publisher : page) {
    publisher.getBooks().size();
}
```

적용 조건이 맞으면 여러 출판사의 책을 IN 조건 등으로 묶어 로딩할 수 있습니다. 실제 쿼리 분할은 방언, 컨텍스트에 있는 프록시·컬렉션, 배치 크기에 따라 달라지므로 특정 횟수를 보장하는 공식처럼 사용하지 않습니다.

여기서 `default_batch_fetch_size`는 **조회** 최적화입니다. `jdbc.batch_size`는 주로 **쓰기** SQL의 JDBC 배치와 관련된 설정입니다. 이름이 비슷해도 해결하는 문제가 다릅니다.

부모 20개로 페이지를 제한했어도 각 부모의 자식이 수만 건이면 여전히 큰 응답입니다. 목록에 책 개수만 필요하다면 컬렉션 전체보다 집계 쿼리가 적합합니다.

### 대안 B: 부모 ID 페이지와 상세 조회를 분리

```java
List<Long> ids = em.createQuery("""
    select p.id from Publisher p order by p.id desc
    """, Long.class)
    .setFirstResult(0)
    .setMaxResults(20)
    .getResultList();

List<Publisher> publishers;
if (ids.isEmpty()) {
    publishers = List.of();
} else {
    publishers = em.createQuery("""
        select p from Publisher p
        left join fetch p.books
        where p.id in :ids
        order by p.id desc
        """, Publisher.class)
        .setParameter("ids", ids)
        .getResultList();
}
```

페이지 제한은 첫 쿼리에만 적용합니다. 두 번째는 선택한 부모의 관계를 로딩합니다. 이 예제는 ID 내림차순이므로 두 쿼리에서 같은 순서를 표현했지만, 일반적으로 IN 조건은 전달한 ID 목록의 순서를 보장하지 않습니다. 복잡한 정렬이면 결과 순서를 별도로 복원해야 합니다.

또한 두 쿼리 사이의 동시 변경을 어떻게 볼지는 트랜잭션 격리 수준과 요구하는 일관성에 따라 달라집니다. 목록 응답에서 허용할 변동과 정확한 스냅샷이 필요한 조회를 구분해야 합니다.

부모 페이지 요구를 자식 페이지로 바꾸는 것은 동등한 해결이 아닙니다. 출판사 20곳이 필요한 화면을 책 20권 조회로 대체하면 페이지의 의미가 바뀝니다.

## 6. LazyInitializationException은 조회 경계를 점검할 신호다

연관 상태가 아직 로딩되지 않았는데 컨텍스트가 닫힌 뒤 접근하면 초기화할 수 없습니다. 이미 로딩한 값까지 모두 읽을 수 없다는 뜻은 아닙니다.

이때 모든 관계를 EAGER로 바꾸면 예외가 사라지는 대신 사용하지 않는 데이터까지 로딩할 수 있습니다. API가 필요한 데이터를 서비스에서 로딩하고 DTO로 변환해 반환하는 방식이 조회 범위를 명확하게 만듭니다.

```java
@Transactional(readOnly = true)
public List<LoanRow> findLoanRows() {
    return em.createQuery("""
        select new library.dto.LoanRow(l.id, b.title, l.dueDate)
        from Loan l
        join l.book b
        order by l.id desc
        """, LoanRow.class)
        .setMaxResults(20)
        .getResultList();
}
```

이 예제의 DTO는 `library.dto` 패키지에 `record LoanRow(Long id, String title, LocalDate dueDate)`로 정의한다고 가정합니다.

OSIV가 활성화되어 있으면 웹 계층에서도 컨텍스트가 열려 있어 증상이 다르게 보일 수 있습니다. 그래서 “트랜잭션이 끝나면 반드시 같은 예외”라고 단정하지 않습니다. 팀에서 조회를 어느 계층에서 끝낼지 정하고 해당 설정 아래의 동작을 검증해야 합니다.

DTO가 항상 엔티티보다 우월한 것도 아닙니다. 업무 규칙을 실행하고 수정할 대상이라면 관리 엔티티가 적합하고, 목록에 필요한 값만 반환한다면 DTO 프로젝션이 단순할 수 있습니다.

## 7. 최적화는 횟수와 양을 함께 비교한다

성능 검증에서는 먼저 재현 조건을 고정합니다.

1. 서로 다른 연관 대상이 충분히 포함된 데이터를 준비합니다.
2. 준비 데이터를 flush한 뒤 clear하여 기존 1차 캐시의 영향을 줄입니다.
3. 2차 캐시와 배치 로딩 설정을 기록합니다.
4. 목록 조회뿐 아니라 DTO 변환과 실제 연관 필드 접근까지 측정 범위에 포함합니다.
5. SQL 횟수, 반환 행 수, 응답 크기와 지연 시간을 함께 비교합니다.

예를 들어 쿼리를 21회에서 1회로 줄였어도 다중 컬렉션 조인으로 반환 행이 수천 배 늘었다면 좋은 변경이라고 결론 내릴 수 없습니다. count 쿼리가 있는 Page와 count 없는 Slice도 같은 조건으로 비교해야 합니다.

| 요구 | 우선 검토할 접근 |
|---|---|
| 대출 목록 + 각 책 제목 | 단일 값 fetch join 또는 DTO 프로젝션 |
| 출판사 상세 + 적당한 책 목록 | 한 컬렉션 fetch join |
| 출판사 페이지 + 각 책 목록 | 부모 페이징 + 배치 로딩 또는 ID 2단계 조회 |
| 출판사 목록 + 책 개수 | 집계 DTO |
| 여러 큰 컬렉션 | 쿼리 분리, 하위 목록 별도 페이징 |

**확인 질문:** 출판사 페이지의 SQL이 한 번만 실행되는데도 메모리를 많이 사용한다면, 쿼리 횟수 외에 어떤 두 가지를 먼저 확인할까요?

**오늘 배운 것 한 줄 요약:** 조회 성능은 LAZY 설정만으로 결정되지 않으며, 필요한 그래프와 SQL 행 수, 페이지 단위를 함께 설계해야 한다.
