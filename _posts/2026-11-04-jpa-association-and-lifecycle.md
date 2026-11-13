---
title: "연관관계의 주인과 생명주기의 주인은 다르다 — 매핑부터 cascade와 orphanRemoval까지"
date: 2026-11-04 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["Courses","JPA기본"]
description: "외래 키 관리와 생명주기 책임을 구분하고 연관관계, cascade, orphanRemoval의 설계 기준을 설명한다."
toc: true
---

`@ManyToOne`과 `@OneToMany`를 붙이는 방법은 어렵지 않습니다. 실제로 헷갈리는 지점은 “왜 컬렉션에 넣었는데 FK가 저장되지 않을까?”, “부모를 지웠는데 자식까지 지워도 될까?” 같은 질문입니다.

두 질문에 등장하는 ‘주인’은 서로 다릅니다. 하나는 **외래 키를 어떤 필드의 값으로 갱신할지**이고, 다른 하나는 **누가 자식의 생성과 삭제를 책임지는지**입니다. 이 둘을 분리하면 연관관계 매핑과 cascade를 훨씬 명확하게 설계할 수 있습니다.

> 기준: Jakarta Persistence 3.1, Hibernate 6 계열. 코드 일부는 설명에 필요한 필드와 메서드만 제시합니다. DB 결과는 예상 동작이며 실제 실행 기록이 아닙니다.

## 1. 먼저 세 가지 결정을 분리한다

도서관의 책과 출판사를 예로 들어 보겠습니다.

| 결정 | 질문 | 예시 |
|---|---|---|
| 탐색 방향 | 어떤 객체에서 어떤 객체로 이동해야 하나? | 책에서 출판사 이름을 사용한다 |
| FK 관리 | 어떤 필드를 변경해야 FK가 바뀌나? | `Book.publisher` |
| 생명주기 | 한 객체를 삭제하면 다른 객체도 없어져야 하나? | 책 삭제가 출판사 삭제를 뜻하지 않는다 |

출판사가 여러 책과 연결되어 있어도, 출판사를 삭제할 때 대출 이력이 있는 책을 물리 삭제하는 것이 옳다는 결론은 나오지 않습니다. DB 관계의 방향만으로 업무상 삭제 정책을 정할 수 없습니다.

이 글에서 책과 출판사는 **공유되는 독립 엔티티**로, 뒤에서 다룰 주문과 주문 항목은 **부모에 종속된 엔티티**로 구분합니다. 같은 일대다라도 생명주기 설정이 달라지는 이유를 비교하기 위해서입니다.

## 2. 객체의 참조 두 개를 외래 키 하나에 연결하기

```java
// Book의 일부
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "publisher_id")
private Publisher publisher;

// Publisher의 일부
@OneToMany(mappedBy = "publisher")
private List<Book> books = new ArrayList<>();
```

객체에서는 `Book.publisher`와 `Publisher.books`가 서로 다른 참조입니다. DB에는 `book.publisher_id`라는 FK가 하나 있습니다. 두 참조가 서로 모순된 값을 가진다면 어떤 것을 SQL에 반영할지 정해야 합니다.

이 매핑에서는 `Book.publisher`가 FK를 쓰는 쪽이고, `mappedBy = "publisher"`는 반대편 필드의 이름을 가리킵니다. 컬럼명 `publisher_id`가 아닙니다.

```java
publisher.getBooks().add(book);
// book.publisher는 여전히 null
```

이 코드만으로는 FK가 설정되지 않습니다. FK가 NULL 허용이면 관계 없이 저장될 수 있고, NOT NULL이면 저장에 실패할 수 있습니다. “항상 NULL로 정상 저장된다”가 아니라 **매핑과 DB 제약에 따라 실패 양상이 달라진다**고 봐야 합니다.

반대로 `book.publisher`만 바꾸면 DB의 FK는 바뀌더라도 이미 메모리에 있는 `publisher.books`에는 그 변화가 자동으로 반영되지 않습니다. 같은 트랜잭션에서 컬렉션을 사용하는 로직이 있다면 잘못된 판단을 할 수 있습니다.

### mappedBy 쪽은 정말 읽기 전용인가?

정확히는 **해당 FK 관계의 쓰기를 결정하는 쪽이 아니다**라는 뜻입니다. 컬렉션 객체를 수정할 수 없다는 의미가 아닙니다. `cascade`나 `orphanRemoval`이 설정되어 있으면 그 컬렉션 변경이 다른 영속성 동작과 연결될 수도 있습니다.

“읽기 전용이니까 아무 영향도 없다”로 이해하면 뒤의 고아 객체 제거를 설명할 수 없습니다.

## 3. 양쪽 참조를 맞추되 불필요한 양방향은 만들지 않는다

출판사 변경이 허용된다는 가정 아래, 기존 관계 해제까지 포함한 메서드는 다음처럼 작성할 수 있습니다.

```java
// Book의 일부. 이 예제는 publisher_id의 NULL을 허용한다.
public void changePublisher(Publisher next) {
    if (this.publisher == next) {
        return;
    }

    Publisher previous = this.publisher;
    this.publisher = next;

    if (previous != null) {
        previous.removeBookInternal(this);
    }
    if (next != null) {
        next.addBookInternal(this);
    }
}

// Publisher의 일부. Book과 같은 패키지에 둔다.
void addBookInternal(Book book) {
    if (!books.contains(book)) {
        books.add(book);
    }
}

void removeBookInternal(Book book) {
    books.remove(book);
}

public List<Book> getBooks() {
    return Collections.unmodifiableList(books);
}
```

외부에는 변경 의도를 가진 메서드를 제공하고, 내부 컬렉션을 임의로 조작하지 못하도록 합니다. 연관 엔티티의 `equals()`에 다른 연관관계를 포함하면 `contains()` 같은 동작에도 재귀나 불필요한 로딩이 생길 수 있으므로 동등성 설계 역시 따로 검토해야 합니다.

이 메서드에도 비용이 있습니다. 지연 로딩 컬렉션에 접근하므로 상황에 따라 책 목록을 로딩할 수 있습니다. 출판사마다 책이 수십만 권인데 출판사 변경 하나를 위해 거대한 컬렉션을 다룬다면 모델을 재검토할 이유가 됩니다.

책에서 출판사로만 이동하면 충분하다면 단방향 `Book.publisher`와 별도 목록 쿼리로 시작할 수 있습니다. 양방향은 관계를 ‘완성’하기 위해 붙이는 옵션이 아니라, **실제로 필요한 역방향 탐색을 제공하는 선택**입니다.

FK를 ID 값으로만 갖는 모델도 무조건 잘못은 아닙니다. 서로 다른 관리 경계를 넘거나 객체 그래프를 제한하려는 의도가 있다면 ID 참조가 적합할 수 있습니다. 대신 직접 조회, 정합성 확인 등 책임이 어디로 이동하는지를 설명할 수 있어야 합니다.

## 4. 네 가지 관계를 외우기보다 FK 위치와 제약을 읽는다

### 다대일과 일대다

양방향 N:1/1:N에서는 다 쪽인 `Book.publisher`를 주인으로 매핑합니다. 자기 테이블의 FK를 자기 필드로 관리하므로 SQL을 해석하기 쉽습니다.

일대다 단방향도 지원됩니다. `@OneToMany`만 두면 기본적으로 연결 테이블을 사용하며, `@JoinColumn`을 추가해 대상 테이블 FK를 관리하는 형태도 있습니다. 후자는 컬렉션을 바꿨는데 다른 엔티티 테이블의 FK가 갱신되고 추가 UPDATE가 발생할 수 있습니다.

따라서 “FK를 가진 엔티티가 언제나 주인”이라는 설명은 모든 매핑에 적용되지 않습니다. 양방향 다대일 매핑의 규칙과 일대다 단방향의 예외를 구분해야 합니다.

### 일대일

한 회원이 카드 하나를 가진다면 FK와 UNIQUE 제약으로 한 카드가 중복 연결되지 않도록 해야 합니다. 다만 UNIQUE만으로 모든 회원이 반드시 카드를 가져야 한다는 요구까지 보장하지는 않습니다. NULL 허용 여부와 카드 발급 시점도 결정해야 합니다.

FK를 회원에 둘지 카드에 둘지는 다음을 함께 봅니다.

- 어느 쪽이 먼저 생성되는가?
- 카드 없이 회원이 존재할 수 있는가?
- 카드 여러 장으로 확장될 가능성이 있는가?
- 어느 방향의 조회가 자주 발생하는가?

비주인 쪽 일대일은 대상 존재 여부를 알아내기 위해 추가 조회가 필요할 수 있습니다. `LAZY`만 적으면 항상 조회가 미뤄진다고 가정하지 말고, Hibernate 버전과 바이트코드 향상 설정을 포함해 확인해야 합니다.

### 다대다

연결 테이블에 상태와 시간이 필요하다면 연결 자체를 엔티티로 표현합니다.

```java
@Entity
public class Loan {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reader_id", nullable = false)
    private Reader reader;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "book_id", nullable = false)
    private Book book;

    private LocalDate loanedAt;
    private LocalDate dueDate;
    private LocalDate returnedAt;

    protected Loan() {}
}
```

`Loan`은 단순한 연결을 넘어 대출 이력을 식별하는 객체입니다. 같은 회원이 같은 책을 나중에 다시 빌릴 수 있다면 `(reader_id, book_id)`를 무조건 유일하게 만드는 것도 맞지 않습니다. 동시 대출 제한과 과거 이력 보존을 별도 요구로 다뤄야 합니다.

`@ManyToMany` 자체가 사용 불가능한 기능은 아닙니다. 관계에 추가 속성이나 독립 생명주기가 생길 가능성이 크다면 연결 엔티티가 더 유연하다는 판단입니다.

## 5. cascade는 호출의 전파이고 소유권 선언은 아니다

`cascade = PERSIST`는 부모에 대한 persist가 자식으로 전파되도록 합니다. FK 주인을 바꾸거나 양방향 참조를 자동 동기화하지는 않습니다.

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
private List<OrderLine> lines = new ArrayList<>();
```

이 설정이 있어도 `OrderLine.order`를 빠뜨리면 FK 관계는 올바르게 저장되지 않습니다. 저장 대상에 포함되는 것과 올바른 FK 값을 갖는 것은 별개입니다.

| 옵션 | 전파되는 대표 동작 | 판단할 질문 |
|---|---|---|
| `PERSIST` | 신규 영속화 | 부모 저장 경로에서 새 자식도 등록해야 하나? |
| `MERGE` | 상태 병합 | 그래프 전체 병합이 의도된 동작인가? |
| `REMOVE` | 삭제 | 부모 삭제가 자식 삭제를 의미하나? |
| `ALL` | 모든 cascade 동작 | 각 동작을 전부 전파할 근거가 있나? |

특히 `Loan.book`에 `REMOVE`나 `ALL`을 붙이면 대출 삭제가 도서 삭제로 전파될 수 있습니다. 책은 다른 대출이 공유하거나 기록을 위해 유지해야 할 대상이므로 부적절합니다.

모든 cascade를 동일하게 취급하지 않는 것도 중요합니다. 독점 소유 여부는 특히 REMOVE와 orphanRemoval에서 핵심이고, PERSIST는 실제 생성 흐름에 맞춰 더 좁게 판단할 수 있습니다. “편하니까 ALL” 대신 필요한 연산만 선택하는 편이 의도를 설명하기 좋습니다.

## 6. orphanRemoval은 관계 해제를 삭제로 해석한다

주문 항목은 주문에서 제거되면 그 주문의 항목으로 남아 있을 이유가 없다고 가정해 보겠습니다. 이때는 컬렉션에서 빠지는 것을 자식 삭제로 연결하는 것이 자연스럽습니다.

```java
// PurchaseOrder의 일부
@OneToMany(mappedBy = "order",
    cascade = CascadeType.PERSIST,
    orphanRemoval = true)
private List<OrderLine> lines = new ArrayList<>();

public void addLine(OrderLine line) {
    line.attachTo(this); // 다른 주문에 소속되지 않은 새 항목만 허용
    lines.add(line);
}

public void removeLine(OrderLine line) {
    if (lines.remove(line)) {
        line.detachFrom(this);
    }
}

// OrderLine의 일부
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "order_id", nullable = false)
private PurchaseOrder order;

void attachTo(PurchaseOrder order) {
    if (this.order != null) {
        throw new IllegalStateException("이미 주문에 소속된 항목입니다.");
    }
    this.order = Objects.requireNonNull(order);
}

void detachFrom(PurchaseOrder order) {
    if (this.order != order) {
        throw new IllegalStateException("다른 주문의 항목입니다.");
    }
    this.order = null;
}
```

위 매핑에서 관리 중인 주문의 기존 항목을 제거하면 flush 과정에서 항목 삭제가 적용됩니다. `order`를 null로 만드는 것은 메모리 그래프를 맞추는 작업이고, 의도된 DB 결과는 고아 항목의 DELETE입니다. 단순히 FK를 NULL로 바꾼 채 행을 유지하는 모델과 다릅니다.

고아 제거는 부모가 사적으로 소유하는 자식을 위한 기능입니다. 관계에서 제거한 객체를 다른 부모에 재할당하는 동작에 의존하지 않는 것이 이식 가능한 설계입니다. 부모를 옮겨 다니는 객체라면 이 옵션이 요구와 맞는지부터 재검토해야 합니다. [Jakarta Persistence의 orphanRemoval 규칙](https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1)

부모 삭제와 관계 해제도 구분해야 합니다. `REMOVE`는 부모의 remove 전파이고, `orphanRemoval`은 부모가 살아 있어도 관계에서 빠진 자식의 삭제를 처리합니다. orphanRemoval은 부모 remove 시의 자식 삭제도 포함하므로 그 효과만을 위해 ALL을 추가할 필요는 없습니다.

### DB의 ON DELETE CASCADE와는 다르다

JPA cascade는 엔티티 연산을 전파합니다. DB의 FK `ON DELETE CASCADE`는 DB가 행 삭제를 연쇄 처리합니다. 후자는 JPA의 관리 객체 상태나 콜백 처리까지 자동으로 맞춰 준다는 뜻이 아닙니다. 삭제를 어느 계층에서 실행할지에 따라 메모리 상태와 이벤트 처리도 확인해야 합니다.

## 7. 매핑 테스트는 메모리와 DB를 나눠 확인한다

편의 메서드 테스트에서는 먼저 순수 객체 상태를 확인합니다. 출판사를 A에서 B로 바꿨다면 책이 A 컬렉션에서 사라지고 B 컬렉션에 들어가야 합니다.

DB 매핑 검증은 별도입니다.

```java
book.changePublisher(nextPublisher);
em.flush();
em.clear();

Book reloaded = em.find(Book.class, bookId);
assertThat(reloaded.getPublisher().getId()).isEqualTo(nextPublisherId);
```

이 코드는 관련 엔티티가 미리 저장되어 있고, `book`이 영속 상태이며, 활성 트랜잭션 안이라는 전제입니다. flush/clear 없이 확인하면 FK가 저장되지 않아도 메모리 참조만 보고 통과할 수 있습니다.

삭제 정책도 “삭제가 된다” 하나만 확인하지 않습니다. 주문 항목 제거는 항목을 지워야 하지만 대출 제거는 책을 지우지 않아야 합니다. **삭제해야 할 대상과 유지해야 할 대상을 함께 검증해야 정책이 드러납니다.**

마지막으로, 엔티티를 API 응답으로 그대로 노출하면 양방향 순환과 지연 로딩이 직렬화에 섞입니다. 조회 목적에 맞는 DTO로 변환하면 응답 형식과 연관관계의 내부 표현을 분리하기 쉽습니다.

**확인 질문:** `mappedBy`가 있는 컬렉션에서 항목을 제거했는데 DELETE가 실행된다면, FK 관리 규칙과 어떤 별도 설정을 구분해서 설명해야 할까요?

**오늘 배운 것 한 줄 요약:** 연관관계의 주인은 FK 쓰기 규칙이고, cascade와 orphanRemoval은 업무상 생성·삭제 정책에 따라 따로 결정한다.
