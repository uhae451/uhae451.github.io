---
title: "JPA 영속성 컨텍스트"
date: 2026-11-01 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["jpa", "persistence-context", "dirty-checking", "flush", "first-level-cache"]
description: "JPA 영속성 컨텍스트의 엔티티 상태, 1차 캐시, 쓰기 지연, 변경 감지, flush와 준영속 동작을 테스트 코드로 정리한다."
toc: true
---

JPA를 쓰다 보면 "분명 `save`를 안 불렀는데 UPDATE가 나가요", "`persist`했는데 INSERT가 안 보여요" 같은 상황을 만납니다. 이 동작은 전부 **영속성 컨텍스트** 하나로 설명됩니다. 개념을 길게 설명하기보다, **테스트를 돌려서 SQL 로그가 언제 나가는지** 직접 확인하는 방식으로 정리했습니다.

> 환경: Spring Boot 3(Hibernate 6) + MySQL 8, JUnit 5. 테스트 클래스에 `@SpringBootTest`와 `@Transactional`을 붙이고 `@PersistenceContext EntityManager em`을 주입받아 사용합니다. `spring.jpa.show-sql=true`로 SQL을 출력합니다. `@Transactional` 테스트는 끝나면 롤백되므로, **SQL을 보고 싶은 시점에 `em.flush()`를 직접 호출**합니다.

## 1. 영속성 컨텍스트란

`EntityManager`가 내부에 갖고 있는, **엔티티를 관리하는 메모리 영역**입니다. DB와 애플리케이션 사이의 "작업 공간"이라고 생각하면 됩니다.

- `em.persist(book)`: "DB에 저장해"가 아니라 "이 객체를 작업 공간의 관리 대상으로 올려"에 가깝다.
- 실제 DB 반영은 보통 **flush(커밋 직전 포함) 시점**에 일어난다.

엔티티는 이 영역과의 관계에 따라 네 가지 상태가 됩니다.

| 상태 | 의미 | 대표 동작 |
|---|---|---|
| 비영속 (new) | 컨텍스트와 무관한 그냥 객체 | `new Book(...)` |
| 영속 (managed) | 컨텍스트가 관리 중 | `persist`, `find` 결과 |
| 준영속 (detached) | 관리되다가 분리됨 | `detach`, `clear`, `close` |
| 삭제 (removed) | 삭제 예약 상태 | `remove` |

## 2. 실험용 엔티티

쓰기 지연을 관찰하려면 PK를 미리 알 수 있어야 해서, 이 글의 `Book`은 ID를 직접 지정합니다. (MySQL의 `AUTO_INCREMENT` 방식(IDENTITY)은 번호를 받아야 하므로 `persist` 때 바로 INSERT가 나갑니다.)

```java
@Entity
public class Book {
@Id
private Long id;
private String title;

protected Book() {}
public Book(Long id, String title) { this.id = id; this.title = title; }

public Long getId() { return id; }
public String getTitle() { return title; }
public void changeTitle(String title) { this.title = title; }
}
```

```java
@SpringBootTest
@Transactional
class PersistenceContextTest {

@PersistenceContext
EntityManager em;
// 아래 테스트들이 이 클래스에 들어갑니다.
}
```

## 3. 쓰기 지연: persist 해도 SQL은 바로 안 나간다

```java
@Test
void 쓰기_지연() {
em.persist(new Book(1L, "클린 코드"));
System.out.println("=== persist 직후 ===");

em.flush();
System.out.println("=== flush 직후 ===");
}
```

콘솔에는 `persist 직후` 다음에 INSERT가, 그리고 `flush 직후`가 찍힙니다. 컨텍스트가 변경을 쌓아 뒀다가 flush 때 보낸다는 뜻입니다. 이를 **쓰기 지연**이라고 합니다.

여러 건을 저장할 때 SQL을 모아 JDBC 배치로 한꺼번에 보낼 수도 있습니다. (`hibernate.jdbc.batch_size`)

## 4. 1차 캐시: 같은 걸 두 번 조회하면?

```java
@Test
void 일차_캐시와_동일성() {
em.persist(new Book(1L, "클린 코드"));
em.flush();
em.clear(); // 컨텍스트를 비워서 DB에서 읽게 만든다

Book a = em.find(Book.class, 1L); // SELECT 발생
Book b = em.find(Book.class, 1L); // SELECT 없음

assertThat(a).isSameAs(b); // 같은 인스턴스
}
```

두 번째 `find`는 컨텍스트 안의 **1차 캐시**에서 꺼냅니다. `@Id`를 키로, 엔티티 인스턴스를 값으로 가진 맵이라고 보면 됩니다.

- 캐시에 있으면 바로 반환한다.
- 없으면 DB에서 조회해 캐시에 저장한 뒤 반환한다.

이 캐시는 `EntityManager` 하나(보통 트랜잭션 하나)가 살아 있는 동안만 존재합니다. 그래서 성능보다 **"같은 PK는 같은 객체"라는 동일성 보장**이 더 큰 의미입니다. 같은 행이 객체 두 개로 나뉘어 한쪽만 수정되는 사고를 막아 줍니다.

## 5. 변경 감지: update를 안 불러도 UPDATE가 나간다

```java
@Test
void 변경_감지() {
em.persist(new Book(1L, "클린 코드"));
em.flush();
em.clear();

Book book = em.find(Book.class, 1L);
book.changeTitle("클린 코드 (개정판)"); // 필드만 바꿨다

em.flush(); // UPDATE 가 나간다
}
```

원리는 **스냅샷 비교**입니다.

1. 엔티티가 컨텍스트에 올라올 때 **원본 상태를 스냅샷으로 따로 저장**한다.
2. flush 시점에 현재 상태와 스냅샷을 비교한다.
3. 다르면 UPDATE SQL을 만든다.

알아 둘 점입니다.

- `em.update()` 같은 메서드는 없습니다. **영속 상태 엔티티를 수정하면 끝**입니다.
- 기본적으로 바뀌지 않은 컬럼까지 포함해 전체 컬럼을 UPDATE 합니다. 컬럼이 아주 많다면 Hibernate의 `@DynamicUpdate`로 바뀐 컬럼만 보낼 수 있습니다.
- **준영속 엔티티는 변경 감지 대상이 아닙니다.** (아래 7번 참고)

삭제도 같습니다. `em.remove(book)`은 삭제 예약일 뿐이고, DELETE는 flush 때 나갑니다.

## 6. flush: 컨텍스트와 DB를 맞추는 순간

flush는 **컨텍스트의 변경 내용을 DB로 보내는 동작**입니다.

1. 변경 감지로 UPDATE 대상을 찾는다.
2. 쌓인 INSERT/UPDATE/DELETE를 순서대로 DB에 전송한다.

**flush는 컨텍스트를 비우지 않습니다.** 1차 캐시는 그대로고, 변경분만 DB로 보냅니다. flush로 보낸 SQL도 아직 트랜잭션 안에 있어서 롤백하면 사라집니다.

flush가 일어나는 경우는 세 가지입니다.

| 경우 | 비고 |
|---|---|
| `em.flush()` 직접 호출 | 테스트·디버깅에서 주로 사용 |
| 트랜잭션 커밋 직전 | 가장 일반적 |
| JPQL(쿼리) 실행 직전 | `AUTO` 모드에서 미반영 변경이 조회 결과에 영향을 줄 때 |

세 번째가 실무에서 중요합니다.

```java
@Test
void 쿼리_실행_전에_자동_flush() {
em.persist(new Book(1L, "A"));
em.persist(new Book(2L, "B"));
// 아직 INSERT 가 나가지 않은 상태

List<Book> books = em.createQuery("select b from Book b", Book.class)
.getResultList();

assertThat(books).hasSize(2); // 조회 직전에 자동 flush 되었기 때문
}
```

JPQL은 DB에 직접 SQL을 보내므로 flush 없이 실행하면 방금 `persist`한 데이터가 결과에 안 나옵니다. 이 예제처럼 미반영 변경이 조회 대상과 겹치면 Hibernate가 쿼리 전에 flush하여 일관성을 맞춥니다. 모든 JPQL 실행마다 무조건 flush하는 것은 아닙니다.

## 7. 준영속: 관리에서 빠지면 모든 기능이 꺼진다

컨텍스트 관리에서 엔티티를 빼는 방법입니다.

```java
em.detach(book); // 특정 엔티티만 분리
em.clear(); // 컨텍스트 전체 초기화 (1차 캐시 비움)
em.close(); // 컨텍스트 종료
```

```java
@Test
void 준영속이면_변경_감지가_안_된다() {
em.persist(new Book(1L, "클린 코드"));
em.flush();
em.clear();

Book book = em.find(Book.class, 1L);
book.changeTitle("바뀐 제목");
em.detach(book); // 관리에서 분리

em.flush(); // UPDATE 없음

em.clear();
assertThat(em.find(Book.class, 1L).getTitle()).isEqualTo("클린 코드");
}
```

`clear()`는 "1차 캐시 때문에 SQL이 안 나가서 확인이 안 될 때" 테스트에서 특히 유용합니다. 위 코드들에서 `flush()` 다음 `clear()`를 반복한 이유입니다.

## 8. 정리

- 영속성 컨텍스트는 `EntityManager`마다 있는 엔티티 관리 영역이다.
- **1차 캐시**: 같은 PK는 같은 인스턴스를 돌려줘서 동일성을 보장한다
- **쓰기 지연**: 변경을 모았다가 flush 때 전송 (IDENTITY 전략은 예외)
- **변경 감지**: 스냅샷 비교로 UPDATE 생성. 수정 후 별도 저장 호출이 필요 없다.
- **flush**: 컨텍스트의 변경 내용을 DB에 동기화한다. 커밋·쿼리 직전에 자동 실행. 컨텍스트는 비워지지 않는다.
- 준영속이 되면 위 기능이 모두 적용되지 않는다.
- 테스트에서는 `flush()`, `clear()`, `find()`를 차례로 호출해 "진짜 DB 값"을 확인하는 패턴을 자주 쓴다.
