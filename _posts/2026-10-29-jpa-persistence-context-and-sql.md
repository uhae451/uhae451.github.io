---
title: "JPA는 언제 SQL을 실행할까 — 영속성 컨텍스트부터 식별자 전략까지"
date: 2026-10-29 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["Courses","JPA기본"]
description: "영속성 컨텍스트와 식별자 전략을 연결해 persist, flush, commit과 SQL 실행 시점을 설명한다."
toc: true
---

JPA를 쓰면 CRUD 코드가 짧아집니다. 그런데 코드가 줄었다는 사실만으로는 `save()` 없이 UPDATE가 나가는 이유나, 같은 `persist()`인데 INSERT 시점이 달라지는 이유를 설명하기 어렵습니다.

JPA를 이해하려면 메서드와 SQL을 일대일로 대응시키는 생각에서 벗어나야 합니다. JPA는 객체를 관리하다가 그 상태를 데이터베이스와 동기화합니다. 이 글에서는 도서관 서비스의 `Book`을 통해 **무엇을 관리하고, 언제 동기화하며, 어디까지 보장하는지**를 연결해 보겠습니다.

> 기준: Java 17, Jakarta Persistence 3.1, Hibernate 6 계열. SQL 관련 구현 차이는 Hibernate 6.6 문서로 확인했습니다. 아래 코드는 핵심 동작을 설명하는 예제이며, 이 문서 작성 과정에서 실행한 결과는 아닙니다. 테스트 코드는 Spring Boot 3 프로젝트와 독립된 테스트 DB를 전제로 합니다.

## 1. JPA가 줄여 주는 일과 여전히 개발자가 해야 하는 일

JDBC에서 책을 저장하려면 INSERT를 작성하고 객체의 필드를 파라미터로 풀어야 합니다. 조회할 때는 반대로 결과 행을 읽어 객체를 조립합니다. `Book`에 `Publisher` 참조가 있다면 출판사 객체를 만들고 두 객체를 연결하는 작업도 필요합니다.

ORM은 이 반복적인 변환을 매핑 정보에 따라 처리합니다. JPA는 그 동작의 표준이고, Hibernate는 구현체입니다. Spring Data JPA의 Repository는 그 위에서 CRUD 등의 사용 방식을 간소화합니다.

하지만 ORM이 조회 요구사항까지 결정해 주지는 않습니다. 상세 화면에서 필요한 정보와 목록 화면에서 필요한 정보는 다릅니다. `book.getPublisher()`라는 탐색이 가능해져도, 그 탐색을 위해 SQL을 몇 번 실행할지는 별도로 설계해야 합니다.

또한 엔티티에 필드를 추가하면 생성되는 SQL은 달라지지만, 운영 DB의 컬럼까지 안전하게 추가되는 것은 아닙니다. 스키마 변경과 데이터 마이그레이션은 여전히 개발자의 책임입니다.

제가 JPA의 이점을 판단하는 기준은 다음과 같습니다.

| 작업 | JPA가 유리한 이유 | 별도로 고려할 것 |
|---|---|---|
| 엔티티를 조회하고 업무 규칙에 따라 수정 | 객체 상태 변화와 저장을 연결하기 쉽다 | 트랜잭션 범위, 동시 수정 |
| 연관된 객체를 함께 다루기 | 참조와 외래 키의 변환을 맡길 수 있다 | 조회 계획, 객체 그래프의 크기 |
| 통계·집계·대량 일괄 처리 | 일부는 JPQL로 표현 가능 | DTO, 네이티브 SQL, JDBC 등의 대안 |

JPA를 채택했다고 모든 작업을 엔티티 단위로 처리할 필요는 없습니다.

## 2. 영속성 컨텍스트는 단순한 조회 캐시가 아니다

영속성 컨텍스트는 관리 중인 엔티티의 동일성과 상태를 추적하는 작업 공간입니다. 같은 컨텍스트 안에서 같은 엔티티 타입과 식별자를 가리키는 관리 객체는 하나로 유지됩니다.

```java
Book first = em.find(Book.class, bookId);
Book second = em.find(Book.class, bookId);

assertThat(first).isSameAs(second);
```

첫 조회에서 DB 접근이 필요했다면 엔티티를 읽어 관리하고, 두 번째 `find()`에서는 이미 관리 중인 인스턴스를 사용할 수 있습니다. 중요한 점은 단순히 SELECT 한 번을 아낀다는 데 있지 않습니다. 같은 책을 가리키는 두 객체가 서로 다른 상태로 수정되는 문제를 줄여 줍니다.

그렇다고 모든 조회가 캐시에서 끝나지는 않습니다.

```java
em.find(Book.class, bookId);

em.createQuery("select b from Book b where b.id = :id", Book.class)
    .setParameter("id", bookId)
    .getResultList();
```

별도 쿼리 캐시를 가정하지 않으면, 두 번째 JPQL은 일반적으로 SQL을 실행합니다. 쿼리 조건에 맞는 행을 DB에서 찾는 과정과, 그 결과를 기존 관리 객체로 연결하는 과정은 다른 작업입니다. **1차 캐시는 임의의 검색 조건에 대한 결과 캐시가 아닙니다.**

또한 이 동일성은 다른 트랜잭션의 갱신을 차단하지 않습니다. 동시 수정으로 인한 갱신 유실은 `@Version`이나 DB 잠금 등 별도의 동시성 제어 문제입니다.

## 3. 엔티티의 상태를 알아야 수정이 저장되는 이유가 보인다

| 상태 | 의미 | 대표적인 진입 경로 |
|---|---|---|
| 비영속 | 아직 관리되지 않는 새 객체 | `new Book(...)` |
| 영속 | 컨텍스트가 관리하는 객체 | `persist()`, `find()` |
| 준영속 | 관리되다가 분리된 객체 | `detach()`, `clear()`, 컨텍스트 종료 |
| 삭제 | 삭제 처리가 예약된 관리 객체 | `remove()` |

보통의 Spring 트랜잭션 서비스에서는 트랜잭션 안에서 조회한 엔티티를 수정합니다.

```java
@Transactional
public void renameBook(Long bookId, String title) {
    Book book = em.find(Book.class, bookId);
    if (book == null) {
        throw new IllegalArgumentException("책을 찾을 수 없습니다.");
    }
    book.changeTitle(title);
}
```

관리 중인 객체라면 수정 후 다시 `persist()`할 필요가 없습니다. Hibernate는 일반적으로 로딩 당시 상태와 현재 상태를 비교해 변경을 찾고, flush 시점에 UPDATE를 생성합니다. 바이트코드 향상 등 설정에 따라 변경 추적 구현은 달라질 수 있지만, 개발자가 의존할 핵심 계약은 **영속 상태의 변경이 동기화 대상이 된다**는 점입니다.

반면 컨텍스트에서 분리된 객체를 수정해도 자동으로 저장되지 않습니다. `merge()`도 전달한 객체 자체를 다시 관리한다기보다, 상태를 관리 객체에 복사하고 그 관리 객체를 반환하는 연산입니다. 요청 DTO를 엔티티로 만든 뒤 통째로 병합하면, 전달되지 않은 필드까지 덮어쓰는 설계가 되기 쉽습니다.

수정 API에서는 식별자로 관리 엔티티를 찾고 허용된 변경 메서드만 호출하는 방식이 변경 범위를 드러내기 좋습니다.

## 4. persist, flush, commit은 서로 다른 단계다

`persist()`는 새 객체를 영속 상태로 만드는 연산입니다. `flush()`는 관리 상태의 변경을 SQL로 DB에 동기화합니다. `commit()`은 트랜잭션을 확정합니다.

```text
객체 생성 → persist → 영속 상태에서 변경 → flush → commit
                                      SQL 실행   트랜잭션 확정
```

이 그림은 기본 흐름이며 모든 SQL이 마지막까지 미뤄진다는 의미는 아닙니다. 식별자 생성이나 쿼리 실행 때문에 더 일찍 SQL이 나갈 수 있습니다.

flush 이후에도 트랜잭션은 진행 중이므로 롤백할 수 있습니다. 다만 SQL을 이미 실행했으므로 제약 조건 위반이 드러나거나 DB 잠금이 잡힐 수 있습니다. 따라서 “커밋 전이니 DB에는 아무 영향도 없다”는 설명도 정확하지 않습니다.

그리고 flush는 컨텍스트를 비우지 않습니다. 엔티티를 분리하는 `clear()`와 구분해야 합니다.

```java
book.changeTitle("개정판");
em.clear();
em.flush();
```

이 순서라면 변경이 동기화되기 전에 관리 대상이 사라집니다. 변경을 DB에 보내고 작업 공간을 비우려면 `flush()` 다음 `clear()`입니다.

### JPQL을 실행할 때마다 flush가 일어나는가?

`AUTO` 모드에서 Hibernate는 대기 중인 변경이 JPQL/HQL의 조회 결과에 영향을 주는지 판단합니다. 책을 추가한 뒤 책 목록을 읽는다면 조회 전 동기화가 필요하지만, 관련 없는 테이블을 읽는 경우에는 생략할 수 있습니다. 따라서 “모든 JPQL 직전에 무조건 flush”로 외우면 로그를 잘못 해석하게 됩니다. [Hibernate의 AUTO flush 설명](https://docs.hibernate.org/orm/6.6/userguide/html_single/#flushing-auto-flush-jpql)

## 5. 같은 persist인데 INSERT 시점이 다른 이유

쓰기 지연을 이해했다면 다음 질문은 자연스럽습니다.

> MySQL에서 `IDENTITY`를 쓰는데 왜 `persist()` 직후 INSERT가 보일까?

관리 객체를 식별하려면 ID를 알아야 합니다. DB의 `AUTO_INCREMENT`에 ID 생성을 맡기면 INSERT 결과를 받아야 ID를 알 수 있습니다. 따라서 활성 트랜잭션 안의 일반적인 Hibernate 사용에서는 `persist()` 과정에서 INSERT가 실행됩니다.

| 방식 | ID를 얻는 방법 | INSERT와의 관계 |
|---|---|---|
| 직접 할당 | 애플리케이션이 ID 지정 | INSERT 전에 ID를 안다 |
| `IDENTITY` | INSERT 후 DB에서 생성 값 반환 | ID 확보가 INSERT에 연결됨 |
| `SEQUENCE` | DB 시퀀스 등에서 ID 선확보 | 엔티티 INSERT를 뒤로 미룰 수 있음 |
| `TABLE` | 키 생성 테이블 이용 | ID 선확보가 가능하지만 경합 비용 검토 필요 |

기본 키 전략은 취향만의 문제가 아닙니다. Hibernate의 `IDENTITY`는 해당 엔티티의 JDBC INSERT 배치를 제한합니다. MySQL에서 대량 입력이 핵심이라면 `batch_size`만 올려 해결될 것으로 기대하지 말고 실제 SQL과 드라이버 동작을 확인해야 합니다. [Hibernate의 배치 처리 설명](https://docs.hibernate.org/orm/6.6/userguide/html_single/#batch-session-batch)

PostgreSQL처럼 시퀀스를 지원하는 DB에서는 `SEQUENCE`와 ID 할당 최적화를 검토할 수 있습니다. 다만 `allocationSize`와 실제 시퀀스 설정이 맞아야 하며, 생성 번호가 연속적일 것으로 기대해서는 안 됩니다. PK를 업무용 연속 번호로 사용하는 것은 별도 요구사항을 키 생성기에 떠넘기는 셈입니다.

## 6. 실행 시점을 확인하는 최소 실험

쓰기 지연만 관찰하기 위해 ID를 직접 할당합니다. 다음 `Book`은 이 실험용 클래스이며 다른 글의 `Book` 매핑과 합쳐 사용하는 코드는 아닙니다.

```java
@Entity
public class Book {
    @Id
    private Long id;
    private String title;

    protected Book() {}

    public Book(Long id, String title) {
        this.id = id;
        this.title = title;
    }

    public void changeTitle(String title) { this.title = title; }
    public String getTitle() { return title; }
}
```

```java
@SpringBootTest
@Transactional
class PersistenceContextTest {
    @PersistenceContext
    EntityManager em;

    @Test
    void 변경을_동기화하고_DB에서_다시_읽는다() {
        // 비어 있는 테스트 DB를 전제로 한다.
        em.persist(new Book(101L, "초판"));
        em.flush();
        em.clear();

        Book first = em.find(Book.class, 101L);
        Book second = em.find(Book.class, 101L);
        assertThat(first).isSameAs(second);

        first.changeTitle("개정판");
        em.flush();
        em.clear();

        assertThat(em.find(Book.class, 101L).getTitle())
            .isEqualTo("개정판");
    }
}
```

`@Test`는 JUnit 5, `assertThat`은 AssertJ를 사용합니다. 예상 관찰 순서는 최초 flush의 INSERT, 첫 `find()`의 SELECT, 변경 후 flush의 UPDATE, clear 이후 재조회의 SELECT입니다. 실제 SQL 모양과 세부 실행은 설정에 따라 달라질 수 있습니다.

여기서 `clear()` 없이 마지막 값을 확인하면, DB 저장이 올바른지보다 메모리 객체의 값이 바뀌었는지만 확인할 수 있습니다. 반대로 이 테스트가 통과해도 커밋 성공이나 다른 트랜잭션에서의 가시성까지 검증한 것은 아닙니다. Spring의 트랜잭션 테스트는 기본적으로 종료 시 롤백하기 때문입니다.

## 7. 실무에서 확인할 경계

“왜 SQL이 안 나왔지?”라는 질문에는 먼저 객체가 영속 상태인지, 실제 트랜잭션이 열렸는지, flush 시점에 도달했는지를 확인합니다. “왜 예상보다 빨리 나왔지?”라면 ID 전략과 그 사이 실행된 쿼리를 봅니다.

쓰기 지연과 JDBC 배치도 구분해야 합니다. 쓰기 지연은 실행을 미루는 것이고, 배치는 여러 SQL 실행을 효율적으로 전달하는 별도의 기능입니다. SQL을 모아 놓았다고 자동으로 네트워크 왕복이 한 번이 되는 것은 아닙니다.

한 번에 수십만 엔티티를 관리하는 작업은 1차 캐시와 변경 추적의 메모리 비용도 고려해야 합니다. 일정 단위의 flush/clear는 관리 객체 수를 줄이는 수단이지만, 트랜잭션 크기를 줄이는 것과는 다릅니다. 장애 시 재처리 단위까지 고려해 청크 경계를 정해야 합니다.

**확인 질문:** 같은 트랜잭션에서 책 제목을 바꾼 뒤 `clear()`하고 커밋했다면, 어떤 조건에서 변경이 저장되지 않을까요?

**오늘 배운 것 한 줄 요약:** JPA의 SQL 실행은 메서드 이름보다 영속 상태, flush 시점, 식별자 생성 방식으로 설명해야 한다.
