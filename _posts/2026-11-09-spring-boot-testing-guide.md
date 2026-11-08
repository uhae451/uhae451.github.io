---
title: "Spring Boot 계층별 테스트 - Controller, Service, Repository"
date: 2026-11-09 02:00:00 +0900
categories: [Spring, Testing]
tags: [spring-boot, junit, mockito, mockmvc, webmvctest, datajpatest, assertj]
description: "Controller·Service·Repository 테스트의 선택 기준, 실행 흐름, 실전 예제와 핵심 검증 문법을 정리한다."
toc: true
---

Spring Boot 테스트에서 먼저 결정할 것은 **어디까지 실제로 실행하고, 어떤 의존성을 대체할 것인지**이다. 같은 회원 조회라도 Controller는 요청·응답을, Service는 비즈니스 판단을, Repository는 실제 JPA 조회 결과를 검증한다.

> 예제는 Java 17+, JUnit 5, Mockito, AssertJ를 기준으로 한다. `@MockitoBean`은 Spring Framework 6.2+/Spring Boot 3.4+에서 사용한다. 아래 코드는 핵심을 보여주기 위해 `package`와 대부분의 `import`를 생략했으며, 예제별 도메인 객체는 서로 독립적이다. `@WebMvcTest`, `@DataJpaTest`의 import는 Spring Boot 3과 4에서 다를 수 있다.

## 1. 어떤 테스트를 선택할까?

| 검증 내용 | 방식 | 실제 실행 | 대체하는 대상 |
|---|---|---|---|
| URL 매핑, 요청 DTO, JSON 응답, 입력 검증 | `@WebMvcTest` + `MockMvc` | Controller와 MVC 처리 | Service (`@MockitoBean`) |
| 계산, 분기, 예외, 상태 변경 | JUnit + Mockito | Service | Repository (`@Mock`) |
| 저장·조회·수정·삭제, JPQL·파생 쿼리 | `@DataJpaTest` | JPA + DB | Repository를 Mock 처리하지 않음 |
| Controller부터 DB까지 연결 검증 | `@SpringBootTest` 등 | 여러 계층 | 목적에 따라 결정 |

**테스트 작성 순서:** ① 검증할 책임 정하기 → ② 테스트할 객체와 Mock 대상 분리 → ③ Given(조건) → ④ When(실행) → ⑤ Then(결과·상호작용 검증).

> `// when`은 Given/When/Then에서 **실행 단계**를 뜻한다. Mockito의 `when(mock.method()).thenReturn(...)`은 실제 실행 단계가 아니라 **Given 단계에서 Mock의 동작을 설정하는 문법**이다.

---

## 2. Controller 테스트

**목적:** 실제 서버와 DB 없이 Spring MVC 요청 처리 흐름을 검증한다. `MockMvc.perform()`은 요청 매핑·역직렬화·검증·응답 직렬화 등을 실행하지만, Mock으로 대체한 Service의 내부 로직은 실행하지 않는다.

### 2-1. 테스트 대상

```java
record MemberCreateRequest(@NotBlank String name) {}
record MemberResponse(Long id, String name) {}

interface MemberService {
    MemberResponse create(MemberCreateRequest request);
    MemberResponse find(Long id);
    void changeName(Long id, String name);
}

@RestController
@RequestMapping("/members")
@RequiredArgsConstructor
class MemberController {
    private final MemberService memberService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    MemberResponse create(@Valid @RequestBody MemberCreateRequest request) {
        return memberService.create(request);
    }

    @GetMapping("/{id}")
    MemberResponse find(@PathVariable Long id) {
        return memberService.find(id);
    }

    @PatchMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    void changeName(@PathVariable Long id, @RequestParam String name) {
        memberService.changeName(id, name);
    }
}
```

예외 처리까지 검증하려면 예외와 MVC 예외 처리기가 필요하다. 예제에서는 다음 구현을 사용한다.

```java
class MemberNotFoundException extends RuntimeException {
    MemberNotFoundException(Long id) { super("회원 없음: " + id); }
}

@RestControllerAdvice
class MemberErrorHandler {
    @ExceptionHandler(MemberNotFoundException.class)
    ResponseEntity<Map<String, String>> handle(MemberNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(Map.of("code", "MEMBER_NOT_FOUND"));
    }
}
```


### 2-2. 테스트 예제

```java
@WebMvcTest(MemberController.class)
@Import(MemberErrorHandler.class)
class MemberControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean MemberService memberService;

    @Test
    @DisplayName("회원 생성 성공 - 요청값 전달과 JSON 응답")
    void create_success() throws Exception {
        // given: Service Mock의 반환값 설정
        when(memberService.create(any(MemberCreateRequest.class)))
                .thenReturn(new MemberResponse(1L, "kim"));

        // when & then: MVC 요청 후 응답 및 전달 인자 검증
        mockMvc.perform(post("/members")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"kim\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("kim"));

        verify(memberService).create(argThat(req -> req.name().equals("kim")));
    }

    @Test
    @DisplayName("필수값 누락 - Service 호출 전 요청 차단")
    void create_invalid() throws Exception {
        mockMvc.perform(post("/members")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"\"}"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(memberService);
    }

    @Test
    @DisplayName("조회 성공 - 경로 변수와 응답 검증")
    void find_success() throws Exception {
        given(memberService.find(1L)).willReturn(new MemberResponse(1L, "lee"));

        mockMvc.perform(get("/members/{id}", 1L))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("lee"));
    }

    @Test
    @DisplayName("없는 회원 - Service 예외가 HTTP 응답으로 변환됨")
    void find_notFound() throws Exception {
        when(memberService.find(99L)).thenThrow(new MemberNotFoundException(99L));

        mockMvc.perform(get("/members/{id}", 99L))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.code").value("MEMBER_NOT_FOUND"));
    }

    @Test
    @DisplayName("이름 변경 - 반환값 없는 Service 호출 검증")
    void changeName_success() throws Exception {
        mockMvc.perform(patch("/members/{id}", 1L).param("name", "park"))
                .andExpect(status().isNoContent());

        verify(memberService, times(1)).changeName(1L, "park");
    }
}
```

**첫 번째 테스트의 내부 흐름:**

1. `@WebMvcTest`가 MVC 관련 Bean과 `MemberController`를 구성한다.
2. `@MockitoBean`이 Controller에 주입할 `MemberService`를 Mock으로 등록한다.
3. `when(...).thenReturn(...)`으로 미래의 Service 호출 결과를 예약한다. **아직 Service 메서드를 실행한 것은 아니다.**
4. `perform()`이 JSON 요청을 전달한다. MVC가 요청 DTO를 만들고 `@Valid`를 검증한다.
5. Controller가 Mock Service를 호출하고, 설정한 DTO를 응답으로 직렬화한다.
6. `andExpect()`로 응답을, `verify()`로 Service에 전달된 인자를 검증한다.

**Controller 테스트 핵심 문법**

| 문법 | 언제 쓰는가? |
|---|---|
| `@WebMvcTest(Controller.class)` | 웹 계층만 테스트할 때 |
| `@MockitoBean` | Spring 테스트 컨텍스트 안의 의존 Bean을 Mock으로 바꿀 때 |
| `post()`, `get()`, `patch()`, `delete()` | 요청 HTTP 메서드를 정할 때 |
| `.contentType(...)`, `.content(...)` | `@RequestBody`로 JSON을 보낼 때 |
| `.param("name", "kim")` | `@RequestParam` 값을 전달할 때 |
| `.andExpect(status()...)` | 요청 처리 결과를 검증할 때 |
| `.andExpect(jsonPath("$.name").value(...))` | JSON 필드 값을 검증할 때 |
| `.andDo(print())` | 실패 원인 파악을 위해 요청·응답 내용을 출력할 때 |
| `verifyNoInteractions(mock)` | 검증 실패 등으로 의존 객체가 전혀 호출되지 않아야 할 때 |
| `verify(mock, never()).method()` | 특정 메서드만 호출되지 않아야 할 때 |

#### multipart 요청은 언제 다르게 작성할까?

`@RequestPart`를 받는 API는 `.content()` 대신 `MockMultipartFile`을 요청 파트로 전달한다. 아래 코드는 Controller에 `@PostMapping(consumes = MULTIPART_FORM_DATA_VALUE)`과 `@RequestPart("memberCreateRequest")` 파라미터가 있는 **별도의 multipart API**를 가정한 요청 예제다.

```java
MockMultipartFile requestPart = new MockMultipartFile(
        "memberCreateRequest", "", "application/json",
        """{"name":"kim"}""".getBytes(StandardCharsets.UTF_8));

mockMvc.perform(multipart("/members").file(requestPart))
        .andExpect(status().isCreated());
```

`MockMultipartFile`은 실제 파일만 표현하는 것이 아니라 JSON 파트에도 사용할 수 있다. **일반 `@RequestBody` API에 multipart 요청을 보내면 안 된다.**

---

## 3. Service 테스트

**목적:** 비즈니스 로직은 실제로 실행하고 Repository만 Mock으로 대체한다. Spring 컨텍스트가 필요하지 않으므로 일반적으로 `@SpringBootTest` 없이 Mockito를 사용한다.

### 3-1. 테스트 대상

```java
class Product {
    private int stock;
    Product(int stock) { this.stock = stock; }
    int getStock() { return stock; }

    void decrease(int quantity) {
        if (stock < quantity) throw new IllegalStateException("재고 부족");
        stock -= quantity;
    }
}

record Order(Long productId, int quantity) {}

interface ProductRepository { Optional<Product> findById(Long id); }
interface OrderRepository {
    Order save(Order order);
    void deleteById(Long id);
}

@RequiredArgsConstructor
class OrderService {
    private final ProductRepository products;
    private final OrderRepository orders;

    Order create(Long productId, int quantity) {
        if (quantity <= 0) throw new IllegalArgumentException("수량 오류");
        Product product = products.findById(productId).orElseThrow(NoSuchElementException::new);
        product.decrease(quantity);
        return orders.save(new Order(productId, quantity));
    }

    void cancel(Long orderId) { orders.deleteById(orderId); }
}
```

### 3-2. 테스트 예제

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock ProductRepository products;
    @Mock OrderRepository orders;
    @InjectMocks OrderService orderService;
    @Captor ArgumentCaptor<Order> orderCaptor;

    @Test
    @DisplayName("주문 성공 - 결과, 재고, 저장 인자 검증")
    void create_success() {
        // given
        Product product = new Product(10);
        given(products.findById(1L)).willReturn(Optional.of(product));
        when(orders.save(any(Order.class))).thenAnswer(inv -> inv.getArgument(0));

        // when: 실제 Service 실행
        Order result = orderService.create(1L, 3);

        // then: 결과 + 상태 + 협력 객체에 전달한 값
        assertThat(result.quantity()).isEqualTo(3);
        assertThat(product.getStock()).isEqualTo(7);
        verify(orders).save(orderCaptor.capture());
        assertThat(orderCaptor.getValue().productId()).isEqualTo(1L);
    }

    @ParameterizedTest
    @ValueSource(ints = {0, -1})
    @DisplayName("잘못된 주문 수량은 DB 조회 전에 거부")
    void create_invalidQuantity(int quantity) {
        assertThatThrownBy(() -> orderService.create(1L, quantity))
                .isInstanceOf(IllegalArgumentException.class);
        verifyNoInteractions(products, orders);
    }

    @Test
    @DisplayName("존재하지 않는 상품은 예외 발생")
    void create_missingProduct() {
        when(products.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> orderService.create(99L, 1))
                .isInstanceOf(NoSuchElementException.class);
        verify(orders, never()).save(any(Order.class));
    }

    @Test
    @DisplayName("재고 부족이면 저장하지 않고 상태 유지")
    void create_outOfStock() {
        Product product = new Product(2);
        given(products.findById(1L)).willReturn(Optional.of(product));

        assertThatThrownBy(() -> orderService.create(1L, 3))
                .isInstanceOf(IllegalStateException.class)
                .hasMessage("재고 부족");
        assertThat(product.getStock()).isEqualTo(2);
        verify(orders, never()).save(any(Order.class));
    }

    @Test
    @DisplayName("void 메서드 호출과 예외 전파")
    void cancel_failure() {
        doThrow(new IllegalStateException("삭제 실패"))
                .when(orders).deleteById(5L);

        assertThatThrownBy(() -> orderService.cancel(5L))
                .isInstanceOf(IllegalStateException.class);
        verify(orders).deleteById(5L);
    }
}
```

**성공 테스트의 실행 흐름:** `@Mock`으로 두 Repository를 만들고 `@InjectMocks`로 **진짜 Service**에 주입한다. `given()`과 `when()`으로 Repository의 반환값을 준비한 다음 `create()`를 실행한다. Service의 수량 검사와 재고 차감은 실제 실행되지만, `save()`는 DB에 접근하지 않고 인자로 받은 주문을 그대로 돌려준다. 마지막으로 반환값, 재고 상태, 저장 인자를 따로 검증한다.

**Mockito 문법 선택 기준**

| 문법 | 언제 사용하는가? |
|---|---|
| `@Mock` + `@InjectMocks` | Spring 없이 Service만 단위 테스트할 때 |
| `when(mock.call()).thenReturn(value)` | Mock의 반환값을 고정할 때 |
| `given(mock.call()).willReturn(value)` | 위와 동일하며 BDD 스타일로 작성할 때 |
| `thenAnswer(inv -> ...)` | 호출 인자에 따라 반환값을 계산해야 할 때 |
| `when(mock.call()).thenThrow(ex)` | 반환형 메서드에서 예외를 모의할 때 |
| `doThrow(ex).when(mock).voidCall()` | `void` 메서드가 예외를 던지도록 할 때 |
| `any(Type.class)`, `eq(value)`, `isNull()` | Mock 인자를 유연하게 매칭할 때 |
| `verify(mock)`, `times(n)`, `never()` | 의존 객체 호출 여부·횟수가 요구사항일 때 |
| `@Captor` + `.capture()` | Mock에 전달된 객체의 필드를 검증할 때 |
| `assertThat()`, `assertThatThrownBy()` | 반환값·상태·예외를 검증할 때 |

> `any()`는 인자를 **검증**하는 것이 아니라 **매칭**한다. 실제 전달 내용이 중요하면 `argThat(...)` 또는 `ArgumentCaptor`를 사용한다. 반환값/상태 검증이 충분한데 모든 호출을 `verify()`할 필요는 없다.

---

## 4. Repository 테스트

**목적:** JPA 매핑과 쿼리가 실제 DB에서 올바르게 동작하는지 검증한다. Repository는 Mock으로 대체하면 검증할 대상 자체가 사라진다.

### 4-1. 테스트 대상

```java
@Entity
class Article {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    protected Article() {}
    Article(String title) { this.title = title; }
    Long getId() { return id; }
    String getTitle() { return title; }
    void changeTitle(String title) { this.title = title; }
}

interface ArticleRepository extends JpaRepository<Article, Long> {
    List<Article> findByTitleContainingOrderByIdAsc(String keyword);
    Page<Article> findByTitleContaining(String keyword, Pageable pageable);
}
```

### 4-2. 테스트 예제

```java
@DataJpaTest
class ArticleRepositoryTest {

    @Autowired ArticleRepository articles;
    @Autowired TestEntityManager em;

    @Test
    @DisplayName("저장한 엔티티를 ID로 다시 조회")
    void saveAndFind() {
        Article saved = articles.save(new Article("JPA 기본"));
        em.flush();
        em.clear();

        Article found = articles.findById(saved.getId()).orElseThrow();
        assertThat(found.getTitle()).isEqualTo("JPA 기본");
    }

    @Test
    @DisplayName("제목 포함 검색은 조건에 맞는 결과만 반환")
    void findByTitle() {
        articles.save(new Article("JPA 기본"));
        articles.save(new Article("Spring MVC"));
        em.flush();
        em.clear();

        List<Article> result = articles.findByTitleContainingOrderByIdAsc("JPA");
        assertThat(result).extracting(Article::getTitle)
                .containsExactly("JPA 기본");
        assertThat(articles.findByTitleContainingOrderByIdAsc("없는값")).isEmpty();
    }

    @Test
    @DisplayName("변경 감지로 수정한 값이 DB에 반영")
    void update() {
        Article saved = articles.save(new Article("수정 전"));
        em.flush();
        em.clear();

        Article found = articles.findById(saved.getId()).orElseThrow();
        found.changeTitle("수정 후");
        em.flush();
        em.clear();

        assertThat(articles.findById(saved.getId()).orElseThrow().getTitle())
                .isEqualTo("수정 후");
    }

    @Test
    @DisplayName("삭제 후 존재하지 않음")
    void delete() {
        Article saved = articles.save(new Article("삭제 대상"));
        articles.deleteById(saved.getId());
        em.flush();
        em.clear();

        assertThat(articles.existsById(saved.getId())).isFalse();
    }

    @Test
    @DisplayName("페이지 크기와 전체 개수 검증")
    void paging() {
        articles.save(new Article("JPA 1"));
        articles.save(new Article("JPA 2"));
        articles.save(new Article("JPA 3"));
        em.flush();
        em.clear();

        Page<Article> page = articles.findByTitleContaining(
                "JPA", PageRequest.of(0, 2, Sort.by("id")));

        assertThat(page.getContent()).hasSize(2);
        assertThat(page.getTotalElements()).isEqualTo(3);
    }
}
```

**저장·조회 테스트의 실행 흐름:** `@DataJpaTest`로 JPA와 테스트 DB를 구성한 뒤 실제 Repository에 `save()`를 호출한다. `flush()`로 SQL 반영을 요청하고 `clear()`로 1차 캐시를 비운다. 이후 `findById()`는 DB에서 다시 조회하므로, 메모리의 기존 객체만 검사하는 실수를 줄일 수 있다.

| 문법 | 사용 이유 |
|---|---|
| `@DataJpaTest` | JPA 관련 Bean과 테스트 트랜잭션 구성 (기본적으로 테스트 후 롤백) |
| `TestEntityManager` | 테스트에서 `persist`, `flush`, `clear` 등을 제어할 때 |
| `flush()` / `clear()` | SQL 반영 후 영속성 컨텍스트를 비워 재조회할 때 |
| `findBy...()` | 파생 쿼리의 필터링·정렬 결과를 검증할 때 |
| `PageRequest.of()` | 페이지 번호·크기·정렬 조건을 전달할 때 |
| `extracting()` / `containsExactly()` | 엔티티 목록에서 특정 속성과 순서를 검증할 때 |

`@DataJpaTest`는 환경에 따라 H2 같은 임베디드 DB를 사용하도록 구성될 수 있다. **PostgreSQL 고유 동작이나 SQL 문법을 검증한다면 실제 PostgreSQL을 사용**해야 한다. 해당 환경에서는 다음 설정으로 DB 자동 대체를 막을 수 있지만, DB 연결 설정과 실행 중인 DB는 별도로 필요하다.

```java
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
```

---

## 5. 자주 쓰는 검증·테스트 문법 모음

| 문법 | 목적 | 예시 |
|---|---|---|
| `@Test` | 단일 테스트 | `@Test void test() {}` |
| `@DisplayName` | 테스트 이름 표시 | `@DisplayName("조회 실패")` |
| `@BeforeEach` | 각 테스트 전 준비 | Mock 외 테스트 데이터 초기화 |
| `@ParameterizedTest` + `@ValueSource` | 여러 입력 반복 검증 | `@ValueSource(ints = {0, -1})` |
| `assertThat(x).isEqualTo(y)` | 값 비교 | 재고·ID 검증 |
| `assertThat(list).hasSize(n)` | 목록 크기 검증 | 조회 건수 |
| `assertThat(list).isEmpty()` | 결과 없음 | 검색 결과 없음 |
| `assertThatThrownBy(() -> ...)` | 예외 검증 | 재고 부족·조회 실패 |
| `verifyNoInteractions(mock)` | Mock 전체 호출 없음 | 입력 검증 실패 |
| `verify(mock, never()).save(any())` | 특정 호출 없음 | 실패 시 저장 금지 |
| `andDo(print())` | MVC 요청·응답 디버깅 | MockMvc 테스트 실패 분석 |

### 자주 하는 실수

- **Controller 테스트에서 Service 내부까지 검증:** Mock을 사용했다면 Service 구현은 실행되지 않는다. 별도 Service 테스트로 분리한다.
- **Service 테스트에서 Repository를 실제 연결:** 순수 단위 테스트가 목적이라면 `@Mock`으로 대체한다.
- **Repository 테스트에서 `@Mock` 사용:** JPA 쿼리를 검증할 수 없다. `@DataJpaTest`에서 실제 Repository를 사용한다.
- **`when()`을 테스트의 When 단계로 착각:** Mockito 설정은 Given에 해당한다.
- **성공 여부만 검증:** 핵심 실패 조건에서는 예외와 **저장·외부 호출 차단**까지 확인한다.
- **호출 횟수만 검증:** 비즈니스 결과가 중요하면 `assertThat()`으로 결과나 상태도 확인한다.

**선택 기준:** 요청/응답이 궁금하면 Controller, 비즈니스 규칙이 궁금하면 Service, 실제 쿼리가 궁금하면 Repository 테스트를 작성한다. 계층이 연결된 전체 흐름을 확인하려면 별도 통합 테스트를 추가한다.

**오늘 배운 것 한 줄 요약:** 테스트 대상은 실제로 실행하고, 범위 밖 의존성만 필요에 따라 Mock으로 대체하며, 반환값·상태·호출 여부 중 요구사항에 맞는 항목을 검증한다.
