---
title: "JPA 영속성 전이와 고아 객체 제거"
date: 2026-11-07 16:00:00 +0900
categories: [JPA, Core]
tags: [jpa, cascade, orphan-removal, entity-lifecycle, hibernate]
description: "JPA의 영속성 전이(cascade)와 고아 객체 제거(orphanRemoval)의 동작 방식, 사용 조건, 삭제 시 주의사항을 Publisher와 Book 예제로 정리한다."
toc: true
---

부모를 저장할 때 자식도 같이 저장하고, 부모에서 자식을 빼면 자식도 지워 주는 기능이 `cascade`와 `orphanRemoval`이다. 편리하지만 **잘못 적용하면 의도하지 않은 데이터 삭제가 발생할 수 있으므로** 사용 조건을 분명히 알아야 한다.

예제는 `Publisher`(출판사)가 `Book`(책) 목록을 가지는 관계다.

## 1. 영속성 전이 (cascade)

새 `Publisher`를 생성하면서 `Book` 3권도 함께 저장해야 하는 상황을 가정한다.

```java
Publisher publisher = new Publisher("한빛");
publisher.addBook(new Book("A"));
publisher.addBook(new Book("B"));
publisher.addBook(new Book("C"));

em.persist(publisher);
```

별도의 영속성 전이 설정이 없다면 `em.persist(publisher)`만으로 새 `Book`이 함께 저장되지는 않는다. 자식도 저장하려면 각각 `persist`해야 한다.

`cascade`를 설정하면 **부모 엔티티에서 수행한 영속성 작업을 연관된 자식 엔티티에 전파**할 수 있다.

```java
@OneToMany(mappedBy = "publisher", cascade = CascadeType.PERSIST)
private List<Book> books = new ArrayList<>();
```

이제 `em.persist(publisher)` 한 번으로 출판사와 책들을 함께 저장할 수 있다.

```java
@Test
void 부모를_저장하면_자식도_저장된다() {
    Publisher publisher = new Publisher("한빛");
    publisher.addBook(new Book("A"));
    publisher.addBook(new Book("B"));

    em.persist(publisher);  // cascade = PERSIST
    em.flush();             // Publisher INSERT, Book INSERT 2건
}
```

> `addBook()`은 `Publisher.books`에 자식을 추가하는 동시에 `Book.publisher`도 설정하여 양방향 연관관계를 동기화한다고 가정한다.

| 종류 | 전파되는 동작 |
|---|---|
| `PERSIST` | 저장 |
| `REMOVE` | 삭제 |
| `MERGE` | 병합 |
| `REFRESH` | 새로고침 |
| `DETACH` | 영속성 컨텍스트에서 분리 |
| `ALL` | 위 작업 전체 |

### 연관관계 매핑과는 별개의 기능

`cascade`는 연관관계의 주인을 바꾸거나 외래 키를 설정하는 기능이 아니다. **영속성 작업을 연관된 엔티티에 전파하는 기능**이다.

### 언제 사용하는가?

**자식의 생명주기가 부모에 종속되고, 다른 엔티티가 해당 자식을 독립적으로 관리하지 않는 관계**에서 사용하는 것이 안전하다.

- **적합:** `Order`와 `OrderLine`처럼 주문과 함께 생성·삭제되는 관계
- **주의:** `Loan`이 참조하는 `Book`처럼 여러 업무에서 공유되는 엔티티. `Loan` 삭제 시 `Book`까지 삭제되도록 `REMOVE`를 전파하면 데이터 손실이 발생할 수 있다.

## 2. 고아 객체 제거 (orphanRemoval)

`orphanRemoval`은 **부모와의 연관관계가 끊어진 자식을 삭제 대상으로 처리**하는 옵션이다.

```java
@OneToMany(mappedBy = "publisher", orphanRemoval = true)
private List<Book> books = new ArrayList<>();
```

```java
@Test
void 컬렉션에서_빼면_자식이_삭제된다() {
    Publisher publisher = em.find(Publisher.class, publisherId);

    publisher.getBooks().remove(0);
    em.flush();  // DELETE FROM book WHERE id = ?
}
```

- 자식이 한 부모에게 종속되는 관계에서 사용한다.
- `@OneToOne`, `@OneToMany` 관계에서 사용할 수 있다.
- 부모 엔티티 자체를 삭제해도 해당 관계의 자식이 삭제 대상이 된다.
- 양방향 관계라면 컬렉션 제거와 자식의 부모 참조 해제를 함께 처리하는 편의 메서드를 두는 것이 좋다.

### `cascade = ALL` + `orphanRemoval = true`

두 옵션을 함께 사용하면 부모를 저장·삭제할 때 자식에게 작업을 전파하고, 부모 컬렉션에서 빠진 자식을 자동 삭제하도록 구성할 수 있다.

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderLine> lines = new ArrayList<>();
```

이 구조는 부모가 자식의 생명주기를 관리하는 방식으로, 도메인 주도 설계의 애그리거트 루트(aggregate root)와 그 내부 엔티티 관계에 활용할 수 있다.

**장점**은 부모를 통해 자식의 생성·삭제를 일관되게 관리하기 쉽다는 점이고, **주의할 점**은 컬렉션 변경만으로 삭제가 발생할 수 있다는 것이다. 자식의 소유 관계가 분명한 경우에만 적용한다.

## 3. 정리

- `cascade`는 연관관계 매핑과 별개로 `persist`, `remove` 등의 영속성 작업을 전파한다.
- `orphanRemoval`은 부모와의 관계에서 제거된 자식을 삭제 대상으로 처리한다.
- 두 옵션 모두 **자식의 생명주기가 부모에 종속되는 관계인지** 확인하고 사용한다.
- `cascade = ALL`과 `orphanRemoval = true`를 함께 적용하면 부모를 중심으로 자식의 생명주기를 관리할 수 있지만, 의도치 않은 삭제에 주의해야 한다.
