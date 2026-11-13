---
title: "JPA 연관관계 매핑 기초 — 단방향, 양방향, 연관관계의 주인"
date: 2026-11-07 16:00:00 +0900
categories: ["Courses","JPA기본"]
tags: [jpa, hibernate, relationship-mapping, many-to-one, mappedby]
description: "JPA 단방향·양방향 연관관계의 매핑 방법과 연관관계의 주인, 외래 키 관리, 양방향 객체 참조 동기화 시 주의사항을 Book과 Publisher 예제로 정리한다."
toc: true
---

JPA 연관관계 매핑의 핵심은 **객체에는 서로 다른 방향의 참조가 존재할 수 있지만, 테이블에서는 외래 키 하나로 관계를 관리한다**는 점이다. 이 차이를 이해하면 `mappedBy`와 연관관계의 주인이 필요한 이유도 명확해진다.

> **예제 환경:** Spring Boot 3(Hibernate 6), MySQL 8, JUnit 5. 예제 테스트는 `@SpringBootTest`, `@Transactional`이 선언된 클래스에서 `@PersistenceContext`로 `EntityManager em`을 주입받아 실행한다. `spring.jpa.show-sql=true`로 SQL을 확인하며, 테스트 트랜잭션은 기본적으로 롤백된다. 엔티티의 생성자·접근자 등 반복되는 코드는 일부 생략했다.

예제는 `Book`(다)과 `Publisher`(일)의 관계다. **한 출판사가 여러 책을 낸다**는 다대일(N:1) 관계를 기준으로 살펴본다.

## 1. 외래 키만 필드로 관리하면 불편한 이유

테이블 구조를 그대로 엔티티에 옮기면 외래 키를 단순 값으로 표현할 수 있다.

```java
@Entity
public class Book {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @Column(name = "publisher_id")
    private Long publisherId;
}
```

이 방식도 저장과 조회는 가능하다. 하지만 책의 출판사 이름이 필요하면 출판사를 별도로 조회해야 한다.

```java
Book book = em.find(Book.class, 1L);
Publisher publisher = em.find(Publisher.class, book.getPublisherId());
```

객체 간 관계를 참조로 표현하면 `book.getPublisher().getName()`과 같이 **객체 그래프 탐색**이 가능해진다. 다만 단순 FK 필드 방식 자체가 항상 잘못된 것은 아니며, 객체 간 탐색이 필요한 도메인 모델에서 연관관계 매핑이 유용하다는 뜻이다.

## 2. 단방향 연관관계

```java
@Entity
public class Book {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "publisher_id")
    private Publisher publisher;
}
```

- `@ManyToOne`: `Book`이 다(N) 쪽이고 `Publisher`가 일(1) 쪽인 관계
- `@JoinColumn(name = "publisher_id")`: `book` 테이블에서 관계를 저장할 외래 키 컬럼

JPA는 객체의 `publisher` 참조를 데이터베이스의 외래 키 값과 연결해 준다.

```java
@Test
void 참조를_설정하면_FK로_저장된다() {
    Publisher publisher = new Publisher("한빛미디어");
    em.persist(publisher);

    Book book = new Book("이펙티브 자바");
    book.setPublisher(publisher);
    em.persist(book);

    em.flush();
    em.clear();

    Book found = em.find(Book.class, book.getId());
    assertThat(found.getPublisher().getName()).isEqualTo("한빛미디어");
}
```

`em.persist(book)` 이후 SQL이 실행될 때 `book.publisher_id`에는 `publisher`의 기본 키가 저장된다. 저장 시점은 식별자 생성 전략과 flush 시점 등에 따라 달라질 수 있다.

```java
Book found = em.find(Book.class, book.getId());
found.getPublisher().getName(); // 객체 참조를 통한 탐색
```

출판사를 변경할 때는 `book.setPublisher(otherPublisher)`로 주인 측 참조를 바꾸면 된다. 관리 상태의 엔티티라면 변경 감지에 의해 외래 키가 갱신된다.

> **LAZY를 명시하는 이유:** `@ManyToOne`의 기본 fetch 전략은 `EAGER`다. 불필요한 연관 엔티티 조회나 N+1 문제를 피할 수 있도록 보통 `LAZY`를 기본으로 두고, 필요한 조회에서 fetch join 등으로 함께 가져온다.

## 3. 양방향이 필요해질 때

이번에는 `Publisher`에서 자신이 출판한 책 목록을 탐색해야 한다고 가정한다.

관계형 데이터베이스는 `book.publisher_id` 외래 키만으로 양쪽 방향의 조회가 가능하므로, 별도의 외래 키를 추가할 필요가 없다.

```sql
SELECT p.*, b.*
FROM publisher p
JOIN book b ON b.publisher_id = p.id;
```

객체에서는 참조가 한 방향이므로, `Publisher`에서 `Book` 목록으로 탐색하려면 반대 방향의 참조를 추가해야 한다.

```java
@Entity
public class Publisher {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "publisher")
    private List<Book> books = new ArrayList<>();
}
```

이제 `publisher.getBooks()`로 출판사의 책 목록을 탐색할 수 있다.

여기서 중요한 질문은 다음과 같다.

> `Book.publisher`와 `Publisher.books`가 같은 `publisher_id`를 표현한다면, 어느 쪽의 값을 기준으로 외래 키를 변경할까?

## 4. 연관관계의 주인 = 외래 키를 관리하는 쪽

객체와 테이블은 관계를 표현하는 방식이 다르다.

| 구분 | 관계 표현 |
|---|---|
| 객체 | `Book.publisher`, `Publisher.books`라는 두 방향의 참조 |
| 테이블 | `book.publisher_id`라는 하나의 외래 키 |

양방향 매핑에서는 두 참조 중 **한쪽을 연관관계의 주인(owning side)**으로 지정해 외래 키 변경을 관리한다. 반대편은 `mappedBy`로 주인을 가리킨다.

```java
// Book.publisher: 연관관계의 주인
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "publisher_id")
private Publisher publisher;

// Publisher.books: 주인이 아닌 반대편
@OneToMany(mappedBy = "publisher")
private List<Book> books = new ArrayList<>();
```

`mappedBy = "publisher"`의 `publisher`는 **DB 컬럼명이 아니라 `Book` 엔티티의 필드명**이다.

주인이 아닌 `Publisher.books` 컬렉션을 변경하는 것만으로는 `book.publisher_id`의 값이 변경되지 않는다. 여기서 '읽기 전용'이란 **외래 키 관리 권한이 없다**는 뜻이지, 컬렉션을 읽거나 메모리에서 변경할 수 없다는 뜻은 아니다.

### 주인을 결정하는 기준

**다대일·일대다 양방향 매핑에서는 외래 키가 있는 테이블에 대응하는 다대일 쪽이 주인**이다. 이 예제에서는 `book` 테이블의 외래 키를 관리하는 `Book.publisher`가 주인이다.

도메인상 출판사가 책을 소유한다고 해서 `Publisher.books`를 주인으로 정하는 것은 아니다.

> 이 기준은 지금 다루는 **다대일·일대다 양방향 관계**에 대한 설명이다. 모든 JPA 관계에서 무조건 외래 키가 있는 쪽만 주인이 되는 것은 아니다. 예를 들어 일대다 단방향 매핑은 반대쪽 테이블의 FK를 관리할 수도 있다.

## 5. 가장 흔한 실수: 주인이 아닌 쪽에만 값 넣기

```java
@Test
void 반대편에만_추가하면_FK가_설정되지_않는다() {
    Publisher publisher = new Publisher("한빛미디어");
    em.persist(publisher);

    Book book = new Book("이펙티브 자바");
    publisher.getBooks().add(book); // 반대편 컬렉션만 변경
    em.persist(book);

    em.flush();
    em.clear();

    Book found = em.find(Book.class, book.getId());
    assertThat(found.getPublisher()).isNull();
}
```

주인인 `book.publisher`에 값이 없으므로 `publisher_id`는 `NULL`로 저장된다. **단, FK 컬럼이 `NOT NULL`이라면 이 예제는 정상 저장되지 않고 제약조건 오류가 발생한다.**

```java
book.setPublisher(publisher); // 주인 쪽 참조를 설정해야 FK가 반영된다.
```

그러나 주인 쪽에만 설정하면 동일한 영속성 컨텍스트에서 이미 가지고 있는 `publisher.books` 컬렉션은 메모리 상태가 일치하지 않을 수 있다. 따라서 **양방향 참조를 함께 동기화하는 편의 메서드**를 두는 것이 좋다.

### 양방향 연관관계 편의 메서드

출판사를 변경하거나 관계를 해제할 때 기존 출판사의 컬렉션도 함께 정리한다.

```java
// Book 엔티티 내부
public void assignPublisher(Publisher newPublisher) {
    if (this.publisher == newPublisher) {
        return;
    }

    if (this.publisher != null) {
        this.publisher.getBooks().remove(this);
    }

    this.publisher = newPublisher;

    if (newPublisher != null) {
        newPublisher.getBooks().add(this);
    }
}
```

- 연관관계 변경 로직은 한곳에 모아 두고 양쪽에서 서로 호출하지 않도록 한다.
- 다른 출판사로 변경할 때는 이전 컬렉션에서도 제거한다.
- 컬렉션 필드는 `new ArrayList<>()`로 초기화한다.

위 예제는 관계 동기화의 핵심만 표현한 코드다. 실제 도메인에서는 컬렉션을 외부에서 임의로 수정하지 못하도록 접근 범위도 고려할 수 있다.

## 6. 양방향에서 조심할 것

### 무한 순환 참조

`Book.toString()`이 `publisher`를 출력하고 `Publisher.toString()`이 다시 `books`를 출력하면 무한 순환으로 `StackOverflowError`가 발생할 수 있다. Lombok의 `@ToString`, JSON 직렬화 과정에서도 문제가 될 수 있다.

- `toString()` 대상에서 연관관계 필드를 제외한다.
- API 응답에는 엔티티를 그대로 반환하기보다 DTO로 변환해 전달한다.

### 양방향은 필요할 때 추가하기

양방향 매핑은 객체에서 반대 방향으로 탐색할 수 있게 하는 기능이다. 일반적인 다대일·일대다 양방향 매핑에서는 **단방향 매핑에 반대편 참조를 추가해도 테이블 구조는 바뀌지 않는다.**

처음에는 단방향으로 시작하고 `publisher.getBooks()` 같은 탐색이 필요한 시점에 양방향을 추가해도 된다. 단순 목록 조회라면 JPQL로 `Book`을 검색하는 방법도 고려할 수 있다.

## 7. 정리

| 항목 | 내용 |
|---|---|
| 단방향 | `@ManyToOne`과 `@JoinColumn`으로 객체 참조를 외래 키에 매핑 |
| 양방향 | 서로 반대 방향을 가리키는 객체 참조 두 개를 구성 |
| 연관관계의 주인 | 외래 키 변경을 관리하는 쪽. 다대일·일대다 양방향에서는 다대일 쪽 |
| `mappedBy` | 반대편에서 주인 필드명을 지정하며 외래 키를 직접 변경하지 않음 |
| 실수 방지 | 주인 쪽 값을 설정하고 편의 메서드로 양쪽 참조 동기화 |
| 주의 사항 | 순환 `toString()`·JSON 직렬화, 엔티티 직접 반환에 유의 |

**핵심:** 데이터베이스의 외래 키는 하나지만, 양방향 객체 참조는 두 개이므로 **외래 키를 변경할 주인을 지정하고 양쪽 객체 상태를 일치시켜야 한다.**
