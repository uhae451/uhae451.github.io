---
title: "Spring Boot 계층별 테스트 작성 기준 - Controller, Service, Repository"
date: 2026-11-09 02:00:00 +0900
categories: [Spring, Testing]
tags: [spring-boot, junit, mockito, mockmvc, datajpatest, unit-test]
description: "Controller, Service, Repository 테스트에서 어떤 도구를 선택하고 어떤 순서로 검증하는지 JUnit, Mockito, MockMvc, JPA 예제로 정리한다."
toc: true
---

Spring Boot에서 테스트 코드를 작성할 때 먼저 결정할 것은 **무슨 문법을 쓸지**가 아니라 **어디까지 실제로 실행해 검증할지**이다.

테스트는 검증 대상의 책임을 확인하는 코드다. 대상 밖의 의존성까지 무조건 실행하면 테스트가 느려지고, 실패 원인도 파악하기 어려워진다.

> 예제는 Java, JUnit Jupiter, Mockito, AssertJ, Spring MVC, Spring Data JPA를 기준으로 한다. 핵심 흐름에 집중하기 위해 `import`와 일부 도메인 세부 구현은 생략했다. `@MockitoBean`은 Spring Boot 3.4 이상에서 사용하는 방식이며, Spring Boot 4에서는 일부 테스트 어노테이션의 패키지 경로가 다르다.

## 1. 테스트 종류는 어떻게 선택할까?

| 검증하려는 것 | 선택 | 실제로 실행하는 범위 | 대체하는 대상 |
|---|---|---|---|
| URL 매핑, 요청 데이터 처리, 응답 JSON | `@WebMvcTest` + `MockMvc` | Controller와 MVC 관련 구성 | Service를 `@MockitoBean`으로 대체 |
| 계산, 조건 분기, 예외 등 비즈니스 규칙 | JUnit + Mockito | 실제 Service 객체 | Repository 등을 `@Mock`으로 대체 |
| JPA 저장·조회, 쿼리, 엔티티 매핑 | `@DataJpaTest` | 실제 JPA와 테스트 DB | Repository를 Mock으로 대체하지 않음 |
| Controller부터 DB까지 연결된 동작 | `@SpringBootTest` | 필요한 Spring 애플리케이션 구성 | 테스트 목적에 따라 결정 |

이 차이를 이해하면 어노테이션 선택이 쉬워진다.

- **Controller 테스트:** 요청을 보내고 응답을 확인한다. Service 내부 로직은 검증하지 않는다.
- **Service 테스트:** 메서드를 직접 호출하고 결과 또는 상태를 확인한다. DB 연결은 필요 없다.
- **Repository 테스트:** 실제 Repository 메서드를 실행하고 DB에서 결과를 확인한다.

---

## 2. Controller 테스트 — 요청과 응답이 올바른가?

### 테스트 목적과 기술 선택

Controller는 URL과 요청값을 받아 Service에 전달하고, 결과를 응답으로 변환한다. 따라서 **Controller가 요청을 정확히 처리하는지**를 검증한다.

`@WebMvcTest`는 MVC 관련 테스트 환경을 구성하고, `MockMvc`는 서버 포트를 열지 않고 가상의 HTTP 요청을 실행한다. Service 의존성은 `@MockitoBean`으로 만든 Mock Bean으로 연결한다.

### 예제: 도서 상세 조회

테스트 대상 Controller와 DTO의 핵심 코드:

```java
record BookResponse(Long id, String title) {}

@RestController
@RequestMapping("/books")
class BookController {
    private final BookQueryService bookQueryService;

    BookController(BookQueryService bookQueryService) {
        this.bookQueryService = bookQueryService;
    }

    @GetMapping("/{id}")
    BookResponse find(@PathVariable Long id) {
        return bookQueryService.find(id);
    }
}

interface BookQueryService {
    BookResponse find(Long id);
}
```

테스트 코드:

```java
@WebMvcTest(BookController.class)
class BookControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    BookQueryService bookQueryService;

    @Test
    void findBook() throws Exception {
        // given: Service가 돌려줄 값 준비
        given(bookQueryService.find(1L))
                .willReturn(new BookResponse(1L, "클린 코드"));

        // when & then: 요청 실행과 응답 검증
        mockMvc.perform(get("/books/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.title").value("클린 코드"));
    }
}
```

### 내부 실행 흐름

1. `@WebMvcTest`가 `BookController`와 필요한 MVC 구성 요소를 준비한다.
2. `@MockitoBean`이 `BookQueryService`의 Mock Bean을 등록한다. 같은 Bean이 이미 존재하면 대체한다.
3. `given()`은 `find(1L)`이 호출될 때 반환할 값을 **미리 설정**한다. 이때 Service 메서드가 실행되는 것은 아니다.
4. `perform()`이 요청을 MVC 처리 과정에 전달하고, 실제 Controller 메서드가 호출된다.
5. Controller가 Mock Service를 호출하면 미리 설정한 DTO가 반환된다.
6. `andExpect()`로 응답 결과를 검증한다. **실제 Service 로직과 DB는 실행되지 않는다.**

### 핵심 문법과 사용 시점

| 문법 | 언제 쓰는가? |
|---|---|
| `@WebMvcTest(BookController.class)` | 특정 Controller의 MVC 동작만 검증할 때 |
| `@Autowired MockMvc` | Controller에 가상 HTTP 요청을 보낼 때 |
| `@MockitoBean` | Spring 테스트 컨텍스트에 Mock Bean을 등록하거나 기존 Bean을 대체할 때 |
| `given(...).willReturn(...)` | Controller가 의존 객체로부터 받을 응답을 정할 때 |
| `perform(get(...))` | GET 요청을 실행할 때 |
| `andExpect(status()...)` | 응답 상태를 검증할 때 |
| `jsonPath("$.title")` | JSON 응답의 `title` 필드를 검증할 때 (`$`는 JSON 최상위) |

POST 요청에서 JSON 본문을 전달할 때는 다음 문법을 사용한다. 아래는 **별도의 `POST /books` 엔드포인트가 구현되어 있다는 가정**의 요청 부분이다.

```java
BookCreateRequest request = new BookCreateRequest("클린 코드");

mockMvc.perform(post("/books")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(request)));
```

- `contentType()` : 요청 본문 형식 지정
- `content()` : 요청 본문 전달
- `writeValueAsString()` : Java 객체를 JSON 문자열로 변환 (`ObjectMapper` 사용)

**판단 기준:** 요청 매핑, 입력 검증, JSON 구조 등 웹 계층의 동작을 확인한다. 재고 계산이나 DB 조회 결과의 정확성은 다른 계층에서 테스트한다. Spring Security를 사용한다면 인증·인가 필터 설정이 테스트 결과에 영향을 줄 수 있다.

---

## 3. Service 테스트 — 비즈니스 규칙이 올바른가?

### 테스트 목적과 기술 선택

Service는 도메인 규칙을 실행하고 필요한 의존 객체를 호출한다. **규칙에 따른 결과가 맞는지** 확인하기 위해 Service는 실제 객체로, Repository는 Mock으로 사용한다.

Spring 컨텍스트가 필요 없는 순수 단위 테스트라면 `@SpringBootTest` 대신 `@ExtendWith(MockitoExtension.class)`를 사용한다.

### 예제: 상품 재고 차감

테스트할 코드의 핵심 구현:

```java
class Product {
    private int stock;

    Product(int stock) { this.stock = stock; }
    int getStock() { return stock; }
    void decrease(int amount) { stock -= amount; }
}

interface ProductRepository {
    Optional<Product> findById(Long id);
}

class StockService {
    private final ProductRepository productRepository;

    StockService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    void decrease(Long productId, int amount) {
        Product product = productRepository.findById(productId).orElseThrow();
        if (product.getStock() < amount) {
            throw new IllegalArgumentException("재고 부족");
        }
        product.decrease(amount);
    }
}
```

테스트 코드:

```java
@ExtendWith(MockitoExtension.class)
class StockServiceTest {

    @Mock
    ProductRepository productRepository;

    @InjectMocks
    StockService stockService;

    @Test
    void decreaseStock() {
        // given
        Product product = new Product(10);
        given(productRepository.findById(1L)).willReturn(Optional.of(product));

        // when
        stockService.decrease(1L, 3);

        // then
        assertThat(product.getStock()).isEqualTo(7);
    }

    @Test
    void rejectWhenStockIsInsufficient() {
        // given
        Product product = new Product(2);
        given(productRepository.findById(1L)).willReturn(Optional.of(product));

        // when & then
        assertThatThrownBy(() -> stockService.decrease(1L, 3))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("재고 부족");
        assertThat(product.getStock()).isEqualTo(2);
    }
}
```

### 내부 실행 흐름

1. `MockitoExtension`이 `@Mock` 필드에 Mock 객체를 만든다.
2. `@InjectMocks`가 **실제 `StockService` 객체**를 만들고 Mock Repository를 주입한다.
3. `given()`으로 Repository의 `findById()` 호출 결과를 지정한다.
4. `stockService.decrease()`는 실제로 실행된다. 단, Repository의 `findById()`는 Mock이 대신 응답한다.
5. `assertThat()`으로 변경된 재고를 검증한다. 재고가 부족한 경우에는 예외와 변경되지 않은 상태를 검증한다.

### 핵심 문법과 사용 시점

| 문법 | 언제 쓰는가? |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | Spring 없이 Mockito 기반 단위 테스트를 할 때 |
| `@Mock` | DB, 외부 API 등 테스트 대상 밖의 의존성을 가짜 객체로 만들 때 |
| `@InjectMocks` | Mock을 주입한 실제 테스트 대상 객체를 만들 때 |
| `given(...).willReturn(...)` | 의존 객체의 반환값을 준비할 때 |
| `assertThat(...).isEqualTo(...)` | 결과값이나 변경된 상태를 확인할 때 |
| `assertThatThrownBy(...)` | 특정 입력에서 예외가 발생해야 할 때 |
| `verify(mock).method()` | **호출 자체가 요구사항**인 상호작용을 검증할 때 |

`verify()`는 모든 테스트에 필요한 것이 아니다. 예를 들어 `findById()`가 호출되었는지 확인하려면 다음처럼 작성할 수 있다.

```java
verify(productRepository).findById(1L);           // 정확히 1번 호출
verify(productRepository, times(2)).findById(1L); // 정확히 2번 호출
```

두 코드는 서로 다른 호출 횟수를 검증하는 **독립적인 사용 예시**다. 호출 여부보다 최종 재고가 중요한 테스트라면 `assertThat()`을 우선한다.

**판단 기준:** 비즈니스 계산과 분기 처리를 검증할 때는 단위 테스트가 적합하다. 다만 Mockito 기반 테스트만으로 `@Transactional`의 실제 적용이나 DB 반영 여부를 증명할 수는 없다.

---

## 4. Repository 테스트 — JPA와 DB 동작이 올바른가?

### 테스트 목적과 기술 선택

Repository는 데이터 접근을 담당한다. 메서드 이름 기반 쿼리, JPQL, 엔티티 매핑을 확인하려면 **Repository를 Mock으로 만들면 안 된다.** 실제 JPA와 DB를 실행해야 하기 때문이다.

`@DataJpaTest`는 JPA 관련 테스트 구성을 제공하며, 기본적으로 테스트가 끝나면 트랜잭션을 Rollback한다.

### 예제: 제목에 키워드가 포함된 게시글 조회

테스트할 엔티티와 Repository:

```java
@Entity
class Post {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    protected Post() {}
    Post(String title) { this.title = title; }
    String getTitle() { return title; }
}

interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByTitleContaining(String keyword);
}
```

테스트 코드:

```java
@DataJpaTest
class PostRepositoryTest {

    @Autowired
    PostRepository postRepository;

    @Autowired
    TestEntityManager entityManager;

    @Test
    void findByTitleContaining() {
        // given: 실제 DB에 테스트 데이터 준비
        entityManager.persist(new Post("JPA 테스트"));
        entityManager.persist(new Post("MVC 테스트"));
        entityManager.flush();
        entityManager.clear();

        // when: 실제 Repository 쿼리 실행
        List<Post> posts = postRepository.findByTitleContaining("JPA");

        // then: DB 조회 결과 검증
        assertThat(posts).extracting(Post::getTitle)
                .containsExactly("JPA 테스트");
    }
}
```

### 내부 실행 흐름

1. `@DataJpaTest`가 JPA 관련 Bean과 테스트 트랜잭션을 준비한다.
2. `persist()`로 테스트 엔티티를 영속 상태로 만든다.
3. `flush()`로 변경 내용을 DB에 반영하고 `clear()`로 영속성 컨텍스트를 비운다.
4. 실제 `PostRepository`의 조회 메서드가 실행되어 DB에서 데이터를 조회한다.
5. `extracting()`으로 제목을 추출해 기대 결과와 비교한다.
6. 테스트가 끝나면 기본 설정에 따라 트랜잭션이 Rollback된다.

### 핵심 문법과 사용 시점

| 문법 | 언제 쓰는가? |
|---|---|
| `@DataJpaTest` | Repository 쿼리와 JPA 매핑을 검증할 때 |
| `@Autowired` | 실제 Repository와 테스트용 JPA 객체를 주입할 때 |
| `persist()` | 테스트에 필요한 엔티티를 저장할 때 |
| `flush()` | SQL 반영 시점과 DB 제약 조건 등을 확인할 때 |
| `clear()` | 영속성 컨텍스트의 기존 객체를 비워 DB 재조회를 확인할 때 |
| `extracting()` | 조회한 객체 목록에서 특정 속성만 검증할 때 |
| `containsExactly()` | 목록의 값과 순서가 모두 기대값과 같아야 할 때 |

`flush()`와 `clear()`는 모든 테스트에 필수는 아니다. **저장한 엔티티의 영속성 컨텍스트 상태에 의존하지 않고 조회를 검증하고 싶을 때** 사용한다.

### 어떤 DB로 테스트할까?

| 상황 | 선택 기준 |
|---|---|
| 단순 JPA 저장·조회, 빠른 피드백 | H2 등 임베디드 테스트 DB |
| PostgreSQL 전용 SQL, 함수, 타입, DB 제약 | PostgreSQL 등 실제 사용 DB와 같은 종류 |
| 팀·CI에서도 동일한 DB 환경이 필요 | Testcontainers 등으로 테스트 DB 실행 |

`@DataJpaTest`는 임베디드 DB가 클래스패스에 있으면 이를 사용하도록 설정될 수 있다. PostgreSQL 연결 설정을 유지하려면 다음 옵션을 사용한다.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class PostRepositoryTest {
    // 별도의 테스트 DB 연결 설정 필요
}
```

이 옵션만으로 DB가 생성되거나 연결되는 것은 아니다. **테스트용 데이터소스 설정**이 별도로 필요하다.

---

## 5. 테스트를 작성하는 실제 순서

테스트를 작성할 때는 다음 순서를 기준으로 판단한다.

| 순서 | 질문 | 예시 |
|---|---|---|
| 1. 요구사항 선택 | 무엇이 맞아야 하는가? | "재고가 3개 줄어야 한다" |
| 2. 테스트 경계 결정 | 어느 계층의 책임인가? | 계산 규칙이므로 Service |
| 3. 의존성 구분 | 실제로 실행하지 않아도 되는 것은? | ProductRepository는 Mock |
| 4. Given 준비 | 어떤 데이터와 반환값이 필요한가? | 초기 재고 10, 차감 수량 3 |
| 5. When 실행 | 검증할 동작은 무엇인가? | `stockService.decrease(1L, 3)` |
| 6. Then 검증 | 무엇을 확인해야 성공인가? | 최종 재고 7 |
| 7. 실패 조건 선택 | 어떤 경계 조건이 중요한가? | 초기 재고보다 큰 수량 차감 |

### Given / When / Then에서 주의할 점

- **Given:** 테스트의 사전 조건만 준비한다. `given()`은 Mockito 문법이며, Given 단계마다 반드시 사용해야 하는 것은 아니다.
- **When:** 검증할 실제 동작을 실행한다. Controller에서는 `perform()`, Service에서는 메서드 직접 호출, Repository에서는 쿼리 메서드 호출이 해당한다.
- **Then:** 요구사항을 직접 증명하는 값·상태·예외를 검증한다. 호출 횟수만 확인하고 실제 결과를 놓치지 않는다.

### 공통 문법 빠르게 찾기

| 문법 | 목적 |
|---|---|
| `@Test` | 테스트 메서드 지정 |
| `@BeforeEach` | 각 테스트 실행 전 반복되는 준비 작업 |
| `@DisplayName` | 테스트 목적을 읽기 쉬운 이름으로 표시 |
| `given(mock.method()).willReturn(value)` | Mock 반환값 설정 |
| `any(Type.class)` | Mockito에서 특정 타입의 인자를 매칭 |
| `verify(mock).method()` | Mock 호출 검증 |
| `assertThat(actual)` | 실제 값·상태 검증 |
| `assertThatThrownBy(() -> ...)` | 예외 발생 검증 |

`any()`는 실제 값을 검증하는 수단이 아니라 **Mock의 인자를 유연하게 매칭하는 수단**이다. 값 자체가 중요한 경우에는 구체적인 인자 또는 별도의 인자 검증을 사용한다.

---

## 6. 실무 실수와 판단 기준

| 흔한 실수 | 왜 문제가 되는가? | 어떻게 판단할까? |
|---|---|---|
| Controller 테스트에 실제 DB까지 연결 | 검증 범위가 커져 실패 원인이 불분명해짐 | 요청·응답만 확인하면 Service를 Mock으로 대체 |
| Service 단위 테스트마다 `@SpringBootTest` 사용 | 단순 로직 검증에도 Spring 컨텍스트를 로딩 | 스프링 기능이 필요 없다면 Mockito 단위 테스트 |
| Repository를 `@Mock`으로 만들어 쿼리 테스트 | 실제 생성 SQL과 매핑을 검증할 수 없음 | `@DataJpaTest`로 실제 JPA 실행 |
| `given()`만 작성하고 검증 생략 | 설정한 가짜 응답만 있고 요구사항 증명이 없음 | 반드시 의미 있는 `Then` 검증 추가 |
| 모든 테스트에 `verify()` 추가 | 구현 순서와 호출 횟수에 과도하게 결합 | 호출 자체가 요구사항일 때만 사용 |
| H2 테스트 통과를 PostgreSQL 동작 보장으로 해석 | DB별 SQL 문법과 동작 차이가 있음 | DB 의존 기능은 동일 DB 환경에서 검증 |

## 7. 최종 선택 기준

- **요청을 제대로 받는가?** → Controller (`@WebMvcTest`, `MockMvc`)
- **규칙과 계산이 맞는가?** → Service (`@Mock`, `@InjectMocks`, `assertThat`)
- **저장·조회가 실제로 되는가?** → Repository (`@DataJpaTest`)
- **여러 계층이 연결되어 동작하는가?** → 통합 테스트 (`@SpringBootTest`, 필요 시 `@AutoConfigureMockMvc`)

### 확인 질문

"재고 부족 시 주문을 거부하는 규칙"과 "주문 데이터를 DB에 저장하는 동작"을 검증해야 한다면, 각각 어떤 테스트를 선택하고 어떤 의존성을 Mock으로 대체해야 할까?

**오늘 배운 것 한 줄 요약:** 테스트 기술은 문법이 아니라 **검증하려는 책임과 실제로 실행해야 할 범위**에 따라 선택한다.
