---
title: "JPQL을 조회와 변경의 경계로 이해하기 — DTO, 조인, 페이징, 벌크 연산"
date: 2026-11-10 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["Courses","JPA기본"]
description: "JPQL의 DTO 조회, 조인과 페이지 의미, 벌크 연산의 영속성 컨텍스트와 동시성 경계를 정리한다."
toc: true
---

JPQL 문법을 처음 보면 SQL과 비슷해 금방 사용할 수 있을 것 같습니다. 하지만 엔티티를 조회했을 때의 관리 상태, 조인 조건에 따른 결과 누락, 벌크 수정 후 메모리에 남은 이전 값은 SQL 문법만으로 설명하기 어렵습니다.

JPQL을 쓸 때는 세 가지를 함께 결정해야 합니다. **어떤 대상을 검색할지, 결과를 어떤 형태로 받을지, 조회·변경이 영속성 컨텍스트와 어떤 관계를 갖는지**입니다. 도서관 서비스의 목록 조회와 대출 연장 작업을 예로 살펴보겠습니다.

> 기준: Java 17, Jakarta Persistence 3.1의 JPQL. Hibernate 확장인 HQL은 구분해 설명합니다. 코드는 부분 예제로, import와 일부 엔티티 정의는 생략했으며 실제 실행·성능 측정을 완료한 코드는 아닙니다.

## 1. JPQL의 대상은 테이블이 아니라 매핑된 모델이다

```java
select b from Book b where b.title = :title
```

`Book`은 엔티티 이름이고 `title`은 영속 속성 이름입니다. 실제 테이블이 `library_book`, 컬럼이 `book_title`이어도 JPQL에서는 매핑된 이름을 사용합니다.

이 추상화가 SQL을 몰라도 된다는 의미는 아닙니다. 결과는 결국 SQL로 번역되고 DB가 실행합니다. 어떤 인덱스를 사용하는지, 조인이 얼마나 많은 행을 만드는지, 정렬 비용이 어떤지는 생성된 SQL과 실행 계획에서 확인해야 합니다.

JPQL은 비교적 이식 가능한 엔티티 쿼리가 필요할 때 적합합니다. 동적 조건이 복잡하면 Criteria나 Querydsl 같은 쿼리 구성 도구를, DB 고유 기능과 복잡한 보고서가 핵심이면 네이티브 SQL이나 다른 데이터 접근 수단을 검토할 수 있습니다. 도구의 선택은 문자열을 얼마나 적게 쓰는지보다 요구하는 쿼리를 명확하게 표현하고 검증할 수 있는지로 판단합니다.

### 파라미터 바인딩에도 경계가 있다

```java
List<Book> books = em.createQuery("""
    select b from Book b where b.title = :title
    """, Book.class)
    .setParameter("title", inputTitle)
    .getResultList();
```

입력 값을 쿼리 문자열에 이어 붙이지 않고 파라미터로 전달합니다. 하지만 컬럼명이나 정렬 방향 같은 문법 요소를 값 파라미터로 대체할 수는 없습니다. 사용자에게 정렬 옵션을 받는다면 허용한 속성과 방향만 선택하도록 매핑해야 합니다.

이름 기반 파라미터는 쿼리를 읽을 때 값의 의미를 드러내고, 조건을 추가해도 위치 번호가 밀리지 않는 장점이 있습니다.

## 2. 엔티티를 조회할지 DTO를 조회할지 먼저 정한다

```java
// 업무 규칙을 적용해 수정할 객체가 필요한 경우
Book book = em.createQuery("""
    select b from Book b where b.id = :id
    """, Book.class)
    .setParameter("id", bookId)
    .getSingleResult();

book.changeTitle("개정판");
```

활성 트랜잭션 안에서 조회한 관리 엔티티라면 변경 감지의 대상입니다. 반면 목록 화면이 ID, 제목, 출판사명만 필요하다면 전체 엔티티 그래프를 구성할 이유가 작을 수 있습니다.

```java
package library.dto;

public record BookRow(Long id, String title, String publisherName) {}
```

```java
List<BookRow> rows = em.createQuery("""
    select new library.dto.BookRow(b.id, b.title, p.name)
    from Book b
    left join b.publisher p
    order by b.id desc
    """, BookRow.class)
    .setMaxResults(20)
    .getResultList();
```

생성자 표현식의 클래스명은 패키지를 포함하고, 인자의 순서와 타입이 맞아야 합니다. 이 예제처럼 스칼라 값만 담은 DTO는 영속성 컨텍스트가 수정 대상으로 관리하지 않습니다.

DTO 안에 엔티티 자체를 인자로 넣으면 그 엔티티는 여전히 관리 대상일 수 있습니다. 따라서 “DTO로 감쌌으니 모든 값이 비영속”이라고 일반화하면 안 됩니다.

결과가 여러 컬럼일 때 `Object[]`도 가능하지만 인덱스와 캐스팅에 의존합니다. DTO는 응답 계약을 이름과 타입으로 드러내므로 유지보수에 유리합니다. Tuple 지원도 쿼리 API와 구현체에 따라 구분해야 하며, 문자열 JPQL의 모든 사용 방식에서 동일하게 이식 가능하다고 전제하지 않습니다.

### 단건 조회의 실패도 API 계약이다

Jakarta Persistence 3.1의 `getSingleResult()`는 결과가 없으면 `NoResultException`, 여러 건이면 `NonUniqueResultException`을 발생시킵니다. “한 건일 것”이라는 기대와 실제 유일성 제약이 일치하는지 확인해야 합니다.

PK 조회라면 `find()`가 더 직접적이고, 검색 API라면 빈 목록을 정상 결과로 둘 수도 있습니다. 반환 메서드를 고르기 전에 결과 없음이 오류인지 정상인지 결정하는 것이 먼저입니다.

## 3. 조인은 성능뿐 아니라 결과의 의미를 바꾼다

출판사가 없는 책도 목록에 보여야 한다면 내부 조인은 맞지 않습니다.

```java
select b from Book b join b.publisher p
```

위 쿼리는 연결된 출판사가 있는 책만 반환합니다. `left join`으로 바꾸면 출판사가 없어도 책을 남길 수 있습니다. 다만 WHERE 조건에 따라 그 의미가 다시 바뀝니다.

```java
// 출판사 이름이 일치하는 책만 남는다.
select b from Book b
left join b.publisher p
where p.name = :name
```

NULL인 `p.name`은 해당 비교 조건을 만족하지 않으므로 출판사 없는 책은 빠집니다.

반면 다음은 모든 책을 유지하되 조건에 맞는 출판사만 조인합니다.

```java
select new library.dto.BookRow(b.id, b.title, p.name)
from Book b
left join b.publisher p on p.name = :name
```

조건에 맞지 않는 출판사를 가진 책도 결과에 남으며 DTO의 출판사명은 NULL이 됩니다. 실제 연결된 출판사가 없다는 뜻이 아니라 **이 쿼리의 조인 조건에 맞는 대상이 없었다**는 뜻입니다. 응답 필드의 의미까지 생각해야 합니다.

### 묵시적 조인은 짧지만 의도를 숨길 수 있다

```java
select b from Book b where b.publisher.name = :name
```

단일 값 연관 경로 탐색은 내부 조인을 유발할 수 있습니다. 명시적 join을 쓰면 관계를 따라가며 어떤 행을 남길지 더 잘 드러납니다. 다만 `b.publisher.id`처럼 FK로 해석할 수 있는 경우는 추가 조인이 생기지 않을 수 있어, 모든 점 표현식을 SQL 조인으로 단정하지 않습니다.

컬렉션 요소의 필드를 탐색하려면 조인으로 요소에 별칭을 부여합니다.

```java
select b.title from Publisher p join p.books b
```

일반 join은 검색과 프로젝션을 위한 것이고 fetch join은 연관 상태 로딩을 위한 것입니다. DTO로 필요한 컬럼만 고르는 쿼리에 무작정 fetch join을 붙일 이유는 없습니다.

## 4. 페이지는 정렬과 결과 단위가 있어야 정의된다

```java
List<Book> page = em.createQuery("""
    select b from Book b order by b.publishedDate desc, b.id desc
    """, Book.class)
    .setFirstResult(20)
    .setMaxResults(20)
    .getResultList();
```

출간일이 같은 책들이 있을 수 있으므로 ID를 보조 정렬 기준으로 넣었습니다. 이 예제는 출간일이 NULL이 아니라는 전제입니다. NULL을 허용한다면 DB별 NULL 정렬과 원하는 표시 순서도 고려해야 합니다.

안정적인 정렬을 사용해도 페이지 요청 사이에 새 데이터가 들어오면 offset 방식은 중복이나 누락이 생길 수 있습니다. 큰 offset에서는 DB가 앞부분을 처리하는 비용도 커질 수 있습니다.

ID 내림차순의 단순 목록이라면 다음 페이지를 마지막 ID 이후로 제한하는 방식이 가능합니다.

```java
select b from Book b
where b.id < :lastId
order by b.id desc
```

여기에 `setMaxResults(pageSize)`를 적용합니다. 첫 페이지는 `lastId` 조건 없이 조회하는 식으로 구분합니다. 정렬이 출간일과 ID 조합이라면 커서 조건도 그 복합 순서에 맞춰야 합니다. ID만 비교하면서 화면은 출간일 순으로 정렬하면 페이지 계약이 깨집니다.

커서 방식도 무조건 대체재는 아닙니다. 임의 페이지 번호로 이동해야 하는 관리 화면과 순차 탐색하는 피드는 요구가 다릅니다.

### count 쿼리는 목록과 같은 단위를 세는가?

출판사 목록을 책과 조인하면 책 수만큼 부모 행이 늘어날 수 있습니다. 페이지 개수가 출판사 수를 뜻한다면 `count(p)`만으로 정확하지 않을 수 있습니다. 조건의 성격에 따라 `count(distinct p.id)`나 exists 조건을 사용해 부모 수를 세도록 설계합니다.

컬렉션 fetch join과 페이지 제한은 부모 단위 페이징을 깨뜨릴 수 있으므로 부모 ID 페이지와 상세 조회를 분리하거나 배치 로딩을 검토합니다. 페이지 크기뿐 아니라 **무엇을 한 건으로 세는지**가 먼저입니다.

## 5. 문법 선택은 표준과 구현체 확장을 구분한다

자주 쓰는 표현은 별도 암기 목록보다 조회 의도에 연결하면 기억하기 쉽습니다.

| 의도 | 표현 예시 | 확인할 점 |
|---|---|---|
| 대출 이력이 있는 책 | `exists (select l.id from Loan l where l.book = b)` | 존재 여부만 필요하면 전체 자식 로딩 불필요 |
| NULL 대신 표시 값 | `coalesce(b.subtitle, '부제 없음')` | 저장 값을 수정하는 것은 아님 |
| 범주별 표시 | `case when b.pages >= 500 then '장편' else '일반' end` | 화면 표현인지 업무 규칙인지 구분 |
| 조건 집합 | `b.status in :statuses` | 빈 집합 입력은 호출 전에 정책 결정 |
| DB 고유 함수 | `function('함수명', ...)` | 함수 존재 여부와 이식성 확인 |

Jakarta Persistence 3.1 표준의 서브쿼리는 WHERE와 HAVING에 제한됩니다. Hibernate의 HQL은 표준보다 넓은 기능을 제공하며, Hibernate 6.6에는 FROM 절의 derived root 등이 있습니다. 이를 “JPQL 표준이 모두 지원한다”거나 “Hibernate 6이면 모든 세부 버전에서 같다”고 표현하면 안 됩니다. 사용할 기능의 정확한 버전과 API를 확인합니다. [Hibernate HQL의 derived root 설명](https://docs.hibernate.org/orm/6.6/querylanguage/html_single/#derived-roots)

NamedQuery나 리포지토리의 선언형 쿼리는 쿼리를 한곳에 모으고 검증을 앞당기는 데 도움이 될 수 있습니다. 다만 문법 검사로 실행 계획, 데이터에 따른 결과 수, 동적 쿼리 전체까지 검증되는 것은 아닙니다. 대표 데이터로 결과 의미를 검증하는 작업은 여전히 필요합니다.

## 6. 벌크 연산은 객체별 수정을 집합 연산으로 바꾼다

아직 반납되지 않은 특정 기한의 대출을 모두 일주일 연장한다고 합시다. 일반적인 변경 감지는 대출 엔티티를 조회하고 각각 메서드를 호출한 뒤 UPDATE를 수행합니다.

각 대출마다 권한, 연장 횟수, 예약 여부를 검사해야 한다면 이 방식이 업무 규칙을 표현하기 좋습니다. 반면 동일한 조건에 동일한 변경을 적용하는 운영 작업이라면 집합 UPDATE가 적합할 수 있습니다.

```java
@Transactional
public int extendDueDates(LocalDate oldDueDate, LocalDate newDueDate) {
    if (!newDueDate.isAfter(oldDueDate)) {
        throw new IllegalArgumentException("새 기한은 기존 기한 이후여야 합니다.");
    }

    em.flush();
    int affected = em.createQuery("""
        update Loan l
        set l.dueDate = :newDueDate
        where l.returnedAt is null
          and l.dueDate = :oldDueDate
        """)
        .setParameter("newDueDate", newDueDate)
        .setParameter("oldDueDate", oldDueDate)
        .executeUpdate();
    em.clear();
    return affected;
}
```

이 예제는 연장 가능 여부가 위 조건으로 충분하다는 단순화가 있습니다. 실제 규칙에 예약 여부나 연장 횟수 제한이 있다면 쿼리에 반영하거나 엔티티별 처리 방식을 유지해야 합니다. 빠르게 수정되는 것과 올바른 업무 처리는 다른 문제입니다.

### 왜 flush와 clear를 넣었을까?

flush는 기존 관리 객체의 미반영 변경을 DB에 먼저 동기화하려는 의도입니다. AUTO 모드에서 자동 동기화가 일어날 수 있어도, 예제에서는 순서를 명시적으로 드러냈습니다.

벌크 UPDATE는 DB를 직접 변경하며 이미 로딩한 엔티티 상태를 그 결과로 갱신하지 않습니다. 그래서 작업 후 clear로 기존 상태를 분리하고, 이후 필요하면 다시 조회합니다.

```java
Loan before = em.find(Loan.class, loanId); // 기존 기한을 가진 관리 객체

// 벌크 쿼리 실행 후 clear가 호출되었다고 가정
Loan after = em.find(Loan.class, loanId);  // 변경된 DB 상태를 재조회
```

`clear()`를 했다고 `before` 변수의 필드가 바뀌는 것은 아닙니다. 기존 객체는 이전 값을 가진 준영속 객체로 남습니다. 따라서 벌크 작업 후에는 기존 참조를 계속 사용하지 않아야 합니다.

또한 clear는 영향받은 엔티티 하나만이 아니라 **컨텍스트 전체**에 적용됩니다. 위 메서드를 큰 트랜잭션 도중 호출하면 호출자가 관리 중이라고 생각하던 다른 객체도 준영속 상태가 될 수 있습니다. 같은 트랜잭션에 참여하는 서비스끼리는 이런 부작용을 계약으로 드러내거나, 벌크 작업 자체를 독립적인 작업 흐름으로 설계하는 편이 낫습니다.

## 7. 벌크 처리에서는 엔티티 연산의 보장을 다시 확인한다

벌크 작업은 단순히 변경 감지를 빠르게 실행하는 기능이 아닙니다. 일반적인 엔티티 연산의 경로를 우회합니다.

- 엔티티 메서드에 넣은 검증이 자동 실행되지 않습니다.
- 엔티티별 수정·삭제 콜백에 의존할 수 없습니다.
- JPQL 벌크 삭제가 연관 엔티티의 JPA REMOVE cascade를 수행하지는 않습니다.
- 일반적인 `@Version` 기반 낙관적 잠금 검사와 버전 증가가 자동으로 적용되는 것은 아닙니다.

이 특성은 [Jakarta Persistence의 벌크 UPDATE·DELETE 규칙](https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1)에 명시된 DB 직접 처리와 컨텍스트 비동기화 원칙에 연결됩니다.

버전 컬럼이 있는 엔티티를 벌크로 갱신해야 한다면, 버전을 증가시켜 기존 엔티티 기반 쓰기를 충돌로 감지할지, 벌크 작업 자체에 기대 버전 조건을 넣을지 등을 따로 설계해야 합니다. 버전 값을 증가시키는 것과 “기대한 버전일 때만 수정한다”는 조건은 같은 보장이 아닙니다.

삭제할 부모에 자식 FK가 있다면 삭제 순서와 DB 제약도 검토해야 합니다. JPA cascade, DB cascade, 명시적 자식 삭제는 서로 다른 방식입니다.

### 영향받은 건수는 유용하지만 충분하지 않다

반환 건수가 예상보다 많으면 WHERE 조건이 넓었을 수 있고, 0이면 대상 상태가 이미 바뀌었을 수 있습니다. 건수 확인은 좋은 방어선이지만 모든 행이 올바른 업무 조건을 만족했는지는 별도로 검증해야 합니다.

대량 변경에서는 잠금 유지 시간, 로그 발생량, 다른 트랜잭션과의 경합도 확인합니다. SQL 한 번으로 표현된다는 이유로 시스템 비용까지 작다고 볼 수 없습니다. 청크 처리나 재시도 정책이 필요하다면 중간 실패 후 어느 범위까지 적용됐는지도 추적할 수 있어야 합니다.

## 8. 테스트는 반환값과 저장 상태를 함께 본다

조회 테스트에서는 “결과가 나온다”보다 다음 경계를 확인하는 것이 유용합니다.

| 사례 | 확인하려는 오류 |
|---|---|
| 연관 대상이 없는 루트 | 내부 조인 때문에 누락되는지 |
| 정렬 값이 같은 여러 행 | 페이지 순서가 불안정한지 |
| 자식이 여러 개인 부모 | 목록·count가 서로 다른 단위를 세는지 |
| 조회 전 이미 관리 중인 객체 | 캐시 상태가 SQL 관찰을 왜곡하는지 |

벌크 테스트는 먼저 엔티티를 조회해 이전 값을 관리하게 한 뒤, 벌크 변경 후 기존 참조와 재조회 값을 비교하면 경계가 잘 드러납니다. 검증용 재조회는 flush/clear와 트랜잭션 조건을 명확히 하고, 업무 조건에 해당하지 않는 행은 유지되는지도 확인합니다.

쿼리의 전문성은 복잡한 문법보다 “이 결과가 왜 맞는지, 이 변경에서 무엇이 자동으로 보장되지 않는지”를 설명하는 데 있습니다.

**확인 질문:** 벌크 UPDATE 후 `clear()`를 호출했는데도 이전에 조회한 객체의 값이 그대로인 이유와, 그 객체를 계속 수정할 때의 문제는 무엇일까요?

**오늘 배운 것 한 줄 요약:** JPQL은 결과의 형태와 행의 의미를 설계하는 도구이며, 벌크 연산을 선택하면 컨텍스트·검증·동시성의 경계를 직접 관리해야 한다.
