---
title: "JPA 연관관계 종류별 매핑 — 다대일, 일대다, 일대일, 다대다"
date: 2026-11-07 16:00:00 +0900
categories: ["Courses","JPA기본"]
tags: [jpa, hibernate, many-to-one, one-to-many, one-to-one, many-to-many]
description: "JPA의 다대일, 일대다, 일대일, 다대다 연관관계 매핑 방식과 외래 키 관리, 연관관계의 주인, 연결 엔티티를 사용하는 기준을 정리한다."
toc: true
---

## 1. 다대일(N:1) — 기본으로 삼는 관계

`Book`(N)이 `Publisher`(1)를 참조하는 관계다. 외래 키를 가진 엔티티에서 다른 엔티티를 참조하므로 구조가 단순하다.

```java
@Entity
public class Book {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "publisher_id")
    private Publisher publisher;
}
```

외래 키 `publisher_id`는 `book` 테이블에 있으므로, 이 **다대일 매핑의 연관관계 주인**은 `Book.publisher`다.

출판사에서 책 목록도 조회해야 한다면 반대쪽에 양방향 매핑을 추가한다.

```java
@Entity
public class Publisher {
    @OneToMany(mappedBy = "publisher")
    private List<Book> books = new ArrayList<>();
}
```

- `mappedBy = "publisher"`는 `Book.publisher`가 외래 키를 관리한다는 뜻이다.
- `Publisher.books`는 **연관관계의 주인이 아닌 쪽**이다. 컬렉션을 조회하거나 메모리에서 수정할 수 있지만, 그 변경만으로 외래 키가 갱신되지는 않는다.
- 양방향으로 사용할 때는 편의 메서드를 통해 `Book.publisher`와 `Publisher.books`를 함께 맞춰야 한다.

## 2. 일대다(1:N) — 가능하면 다대일로 뒤집기

“출판사가 책 목록을 가진다”를 출판사에서만 참조하는 **일대다 단방향**으로 만들 수도 있다.

```java
@Entity
public class Publisher {
    @OneToMany
    @JoinColumn(name = "publisher_id")
    private List<Book> books = new ArrayList<>();
}
```

이때 외래 키는 `book` 테이블에 있지만, **연관관계의 주인은 `Publisher.books`**다.

```java
Book book = new Book("리팩터링");
em.persist(book);

Publisher publisher = new Publisher("한빛");
publisher.getBooks().add(book);
em.persist(publisher);

em.flush();
```

이런 매핑에서는 자식 엔티티의 INSERT와 별개로, 연관관계를 반영하기 위한 다음 UPDATE가 발생할 수 있다.

```sql
UPDATE book SET publisher_id = ? WHERE id = ?;
```

- `Publisher`의 컬렉션을 변경했는데 `book` 테이블에 UPDATE가 발생해 SQL 흐름을 예측하기 어려워진다.
- 신규 자식의 외래 키 값이 초기 INSERT에 들어가지 않는 경우, `NOT NULL` 제약조건과 충돌할 수 있다.
- 외래 키가 있는 테이블의 엔티티가 직접 연관관계를 관리하도록 구성하는 편이 단순하다.

따라서 일반적으로 **`Book.publisher`를 다대일 주인으로 두고, 필요한 경우 `Publisher.books`를 `mappedBy`로 추가하는 양방향 매핑**을 선택한다.

일대다 쪽을 주인으로 만들기 위해 반대쪽 다대일에 `insertable = false, updatable = false`를 지정하는 우회 구성도 있지만, 양쪽의 역할이 혼란스러워져 권장하지 않는다.

## 3. 일대일(1:1)

`Reader`(이용자) 한 명이 `LibraryCard`(도서관 카드) 한 장을 갖는 관계다.

일대일은 **어느 테이블에 외래 키를 둘지** 결정해야 한다. DB에서도 일대일을 보장하려면 외래 키에 `UNIQUE` 제약조건이 필요하다.

### 3-1. 주 테이블에 외래 키를 두는 방식

자주 조회하는 `reader` 테이블에 `card_id`를 둔다.

```java
@Entity
public class Reader {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "card_id", unique = true)
    private LibraryCard card;
}

@Entity
public class LibraryCard {
    @Id @GeneratedValue
    private Long id;

    private String cardNumber;

    @OneToOne(mappedBy = "card")
    private Reader reader;
}
```

- **장점:** `reader.card_id` 값으로 카드 연결 여부를 알 수 있고, 외래 키를 가진 쪽의 지연 로딩도 구현하기 쉽다.
- **단점:** 카드가 없는 이용자를 허용하면 `card_id`가 `NULL`일 수 있다.

### 3-2. 대상 테이블에 외래 키를 두는 방식

`library_card` 테이블에 `reader_id`를 두고 `LibraryCard`가 주인이 되는 방식이다.

```java
@Entity
public class Reader {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "reader", fetch = FetchType.LAZY)
    private LibraryCard card;
}

@Entity
public class LibraryCard {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reader_id", unique = true)
    private Reader reader;
}
```

`Reader`에서 `LibraryCard`로만 탐색하면서 외래 키는 `library_card`에 두려는 **단방향 매핑**은 일반적인 JPA `@OneToOne` 매핑으로 표현할 수 없다. 이 경우처럼 반대쪽을 주인으로 둔 양방향 관계를 구성하거나 탐색 방향을 바꿔야 한다.

- **장점:** 나중에 이용자 한 명에게 여러 카드를 허용하도록 확장할 때, 외래 키 위치를 유지하면서 제약조건과 매핑을 변경할 수 있다.
- **단점:** 외래 키가 없는 `Reader.card` 쪽에서는 관련 카드의 존재 여부를 알기 어려워 추가 조회가 발생할 수 있다. Hibernate의 버전 및 바이트코드 향상(bytecode enhancement) 설정 등에 따라 지연 로딩의 실제 동작이 달라진다.

외래 키를 어디에 둘지는 **주요 조회 방향, NULL 허용 여부, 관계의 확장 가능성**을 기준으로 결정한다.

## 4. 다대다(N:M) — 연결 엔티티로 풀기

“이용자는 여러 책을 빌리고, 책은 여러 이용자에게 대출될 수 있다”는 관계는 개념적으로 N:M이다.

관계형 DB에서는 중간에 연결 테이블이 필요하며, JPA의 `@ManyToMany`는 이를 엔티티 없이 매핑한다.

```java
@ManyToMany
@JoinTable(
    name = "loan",
    joinColumns = @JoinColumn(name = "reader_id"),
    inverseJoinColumns = @JoinColumn(name = "book_id")
)
private List<Book> books = new ArrayList<>();
```

연결 정보만으로 충분한 단순한 관계에는 사용할 수 있다. 하지만 실제 대출 도메인에는 대출일, 반납 예정일, 반납일 같은 **연결 관계 자체의 속성**이 필요하다.

`@ManyToMany`로 만든 연결 테이블에 이런 속성을 엔티티 필드처럼 직접 매핑할 수는 없다.

이 경우 연결 테이블을 **별도의 엔티티**로 만든다.

```java
@Entity
public class Loan {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reader_id")
    private Reader reader;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id")
    private Book book;

    private LocalDate loanedAt;
    private LocalDate dueDate;
    private LocalDate returnedAt;
}
```

필요하다면 양쪽에도 역방향 관계를 추가한다.

```java
@OneToMany(mappedBy = "reader")
private List<Loan> loans = new ArrayList<>();
```

이 예시는 `Reader` 내부에 선언하는 경우다. `Book`에서는 `mappedBy = "book"`을 사용한다.

이렇게 N:M 관계를 **1:N + N:1**로 풀면 대출 이력 자체를 독립적인 도메인 데이터로 관리할 수 있다.

## 5. 정리

- **다대일:** 외래 키가 있는 엔티티를 주인으로 두는 가장 일반적인 매핑이다.
- **일대다 단방향:** 연관관계 반영을 위한 추가 UPDATE가 발생할 수 있으므로, 다대일 중심의 양방향 매핑을 우선 고려한다.
- **일대일:** 조회 방향과 제약조건을 고려해 외래 키 위치를 정한다.
- **다대다:** 연결 관계에 별도 속성이 필요하다면 `@ManyToMany` 대신 연결 엔티티를 사용한다.
- **연관관계의 주인:** 외래 키 변경을 담당하는 매핑이다. 양방향 다대일·일대다에서는 다대일 쪽이 주인이지만, 일대다 단방향처럼 외래 키가 있는 테이블과 매핑의 주인이 서로 다른 예외도 있다.
