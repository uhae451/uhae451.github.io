---
title: "테스트 코드 작성"
date: 2026-10-27 09:00:00 +0900
categories: ["Spring", "Testing"]
tags: ["spring", "junit5", "mockito", "assertj", "mockmvc"]
description: "JUnit 5, AssertJ, Mockito, @DataJpaTest, @WebMvcTest를 이용해 계층별 테스트 코드를 작성하는 방법을 정리한다."
toc: true
---

## 0. 공부하면서 잡은 큰 그림

테스트 코드를 쓰다 보니 결국 **"스프링의 어느 부분을 띄우고, 어느 부분을 가짜로 바꿀까"** 만 달라졌다. 이렇게 정리하니 어노테이션이 덜 헷갈렸다.

| 단계 | 테스트 대상 | 쓰는 도구 |
|---|---|---|
| 1 | 자바 클래스 (스프링 없음) | `@Test` + AssertJ |
| 2 | Repository | `@DataJpaTest` |
| 3 | Service | Mockito |
| 4 | Controller | `@WebMvcTest` + `MockMvc` |

---

## 1. 테스트 코드가 뭔지부터

테스트 코드는 **"이 코드는 이렇게 동작해야 한다"를 코드로 적어 두고 자동으로 확인하는 코드**다.

```
직접 확인 : 서버 실행 → Postman으로 요청 → 응답 눈으로 확인 → 수정 → 또 반복
테스트 코드 : ./gradlew test 한 번 → 전부 자동으로 확인
```

테스트는 전부 이 3단계 구조로 쓴다는 걸 먼저 알게 됐다. 이걸 알고 나니 뒤의 코드가 다 똑같이 읽혔다.

```java
@Test
void 테스트_이름() {
  // given : 준비 (테스트에 쓸 객체·데이터)
  // when  : 실행 (확인하려는 동작 딱 하나)
  // then  : 검증 (결과가 예상과 같은지)
}
```

---

## 2. 연습용으로 만든 앱

> **Todo 앱**
> - 할 일을 만든다 (제목이 비어 있으면 안 된다)
> - 할 일을 조회한다 (없으면 404)
> - 할 일을 완료 처리한다

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Todo {

  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;
  private boolean done;           // 완료 여부

  public Todo(String title) {
    if (title == null || title.isBlank()) {
      throw new IllegalArgumentException("제목은 비어 있을 수 없습니다.");
    }
    this.title = title;
  }

  public void complete() {
    this.done = true;
  }
}
```

---

## 3. 1단계 - 스프링 없이 자바 클래스 테스트

처음에는 테스트를 하려면 스프링을 띄워야 하는 줄 알았는데, **그냥 객체를 만들어서 확인하면 됐다.** 가장 단순하고 가장 빠르다.

`Todo`의 규칙 3개를 테스트로 옮겼다.

1. 새로 만들면 완료 상태가 아니다
2. `complete()`를 호출하면 완료 상태가 된다
3. 제목이 비어 있으면 예외가 발생한다

```java
class TodoTest {

  @Test
  @DisplayName("새로 만든 할 일은 아직 완료되지 않았다")
  void newTodoIsNotDone() {
    Todo todo = new Todo("스프링 공부");                    // given

    assertThat(todo.getTitle()).isEqualTo("스프링 공부");      // then
    assertThat(todo.isDone()).isFalse();
  }

  @Test
  @DisplayName("complete() 를 호출하면 완료 상태가 된다")
  void complete() {
    Todo todo = new Todo("스프링 공부");                    // given

    todo.complete();                                       // when

    assertThat(todo.isDone()).isTrue();                    // then
  }

  @Test
  @DisplayName("제목이 비어 있으면 예외가 발생한다")
  void blankTitle() {
    assertThatThrownBy(() -> new Todo(" "))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessage("제목은 비어 있을 수 없습니다.");
  }
}
```

### 여기서 정리한 것

| | 뜻 |
|---|---|
| `@Test` | 이 메서드는 테스트라는 표시. 없으면 실행되지 않는다 |
| `@DisplayName("...")` | 결과에 보일 한글 이름 |
| `assertThat(실제값)` | AssertJ의 검증 시작. 뒤에 `.isEqualTo(기대값)` 처럼 이어 쓴다 |
| `.isEqualTo(x)` / `.isTrue()` / `.isFalse()` | 실제값이 x와 같다 / true다 / false다 |
| `assertThatThrownBy(() -> 코드)` | 이 코드를 실행하면 예외가 발생한다 |

`assertThat(실제값).isEqualTo(기대값)`이 영어 문장처럼 읽혀서 JUnit의 `assertEquals`보다 읽기 편했다.

### 일부러 틀리게 해서 실패시켜 봤다

테스트가 정말 잡아주는지 궁금해서 `complete()`를 `this.done = false;`로 바꾸고 돌려 봤다.

```
TodoTest > complete() 를 호출하면 완료 상태가 된다 FAILED
    org.opentest4j.AssertionFailedError at TodoTest.java:27

3 tests completed, 1 failed
```

**어느 테스트의 몇 번째 줄이 깨졌는지** 바로 알려준다. 코드를 고치다 실수했을 때 이걸로 알 수 있다는 점이 테스트를 쓰는 이유라는 걸 체감했다.

### 1단계에서 기억할 것

- 테스트 하나에서 **확인하는 건 하나**만 (실패했을 때 원인을 바로 안다)
- 정상 케이스만이 아니라 **잘못된 입력(빈 제목)** 도 테스트한다

---

## 4. 2단계 - Repository 테스트 (`@DataJpaTest`)

기본 `save`, `findById`는 이미 검증된 코드라서 따로 테스트하지 않아도 된다. **내가 직접 만든 쿼리 메서드**만 확인하면 된다고 이해했다.

```java
public interface TodoRepository extends JpaRepository<Todo, Long> {
  List<Todo> findByDone(boolean done);      // 완료 여부로 조회
}
```

```java
@DataJpaTest
class TodoRepositoryTest {

  @Autowired
  TodoRepository todoRepository;

  @Test
  @DisplayName("완료 여부로 할 일을 골라낸다")
  void findByDone() {
    Todo a = new Todo("운동");                 // given : 미완료 1개, 완료 1개 저장
    Todo b = new Todo("독서");
    b.complete();
    todoRepository.save(a);
    todoRepository.save(b);

    List<Todo> doneList = todoRepository.findByDone(true);        // when

    assertThat(doneList).extracting(Todo::getTitle)               // then
        .containsExactly("독서");
  }
}
```

### 여기서 정리한 것

| | 뜻 |
|---|---|
| `@DataJpaTest` | **JPA 관련 부분만** 띄운다. 메모리 DB(H2)를 쓰고, **테스트가 끝나면 데이터를 자동으로 되돌린다(롤백)** |
| `@Autowired` | 스프링이 만든 객체를 가져온다 |
| `.extracting(Todo::getTitle)` | 리스트에서 제목만 뽑아서 검사 |
| `.containsExactly("독서")` | 정확히 이 값들만, 이 순서로 들어 있다 |

### 완료/미완료를 둘 다 저장한 이유

완료된 것 하나만 저장하고 `findByDone(true)`를 하면, 쿼리가 조건을 무시하고 전부 가져와도 통과한다. 미완료도 같이 넣어야 **"골라낸다"** 가 검증된다.

### 2단계에서 기억할 것

- Repository 테스트는 **진짜 DB(H2)에 쿼리를 보내서** 확인한다 → Mock으로는 검증할 수 없다
- **걸러져야 하는 데이터도 함께** 넣는다

---

## 5. 3단계 - Service 테스트 (Mockito)

Service는 Repository를 쓴다. 로직만 확인하고 싶은데 DB까지 쓰기는 싫으니 **Repository를 가짜(Mock)로 바꾼다.**

```java
@Service
@RequiredArgsConstructor
public class TodoService {

  private final TodoRepository todoRepository;

  public Todo create(String title) {
    return todoRepository.save(new Todo(title));
  }

  public Todo get(Long id) {
    return todoRepository.findById(id).orElseThrow(() -> new TodoNotFoundException(id));
  }

  @Transactional
  public Todo complete(Long id) {
    Todo todo = get(id);
    todo.complete();
    return todo;
  }
}
```

가짜가 뭔지 이렇게 이해했다.

```
진짜 Repository : DB에 실제로 저장하고 조회한다
가짜 Repository : "findById(1L)이 호출되면 이걸 돌려줘" 라고 내가 정한 대로만 행동한다
```

"DB에 이 데이터가 있다고 치자"를 코드 한 줄로 만들 수 있어서 빠르고, 원하는 상황을 마음대로 만들 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class TodoServiceTest {

  @Mock
  TodoRepository todoRepository;        // 가짜 Repository

  @InjectMocks
  TodoService todoService;              // 가짜를 넣어서 만든 진짜 Service

  @Test
  @DisplayName("할 일을 만들면 저장하고 그 결과를 돌려준다")
  void create() {
    when(todoRepository.save(any(Todo.class))).thenAnswer(inv -> inv.getArgument(0));   // given

    Todo result = todoService.create("운동");                                           // when

    assertThat(result.getTitle()).isEqualTo("운동");                                     // then
    verify(todoRepository).save(any(Todo.class));
  }

  @Test
  @DisplayName("없는 id 를 조회하면 TodoNotFoundException")
  void getNotFound() {
    when(todoRepository.findById(99L)).thenReturn(Optional.empty());

    assertThatThrownBy(() -> todoService.get(99L))
        .isInstanceOf(TodoNotFoundException.class);
  }

  @Test
  @DisplayName("완료 처리하면 done 이 true 가 된다")
  void complete() {
    when(todoRepository.findById(1L)).thenReturn(Optional.of(new Todo("운동")));

    Todo result = todoService.complete(1L);

    assertThat(result.isDone()).isTrue();
  }
}
```

### 여기서 정리한 것

| | 뜻 |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | 이 테스트에서 Mockito를 쓴다는 선언 |
| `@Mock` | 가짜 객체 만들기 |
| `@InjectMocks` | 가짜들을 생성자에 넣어서 테스트 대상(진짜 Service)을 만든다 |
| `when(가짜.메서드()).thenReturn(값)` | 가짜의 이 메서드가 호출되면 이 값을 돌려준다 |
| `thenAnswer(inv -> inv.getArgument(0))` | 받은 인자를 그대로 돌려준다. `save()`가 저장한 객체를 돌려주는 동작을 흉내 낸다 |
| `verify(가짜).메서드()` | 이 메서드가 실제로 호출됐는지 확인 |

### 알게 된 점 : Service 테스트에서는 `@Transactional`이 동작하지 않는다

`complete()`에 `@Transactional`이 있지만 이 테스트는 스프링을 띄우지 않아서 트랜잭션이 적용되지 않는다. 이 테스트가 확인하는 건 **로직이 맞는가**까지다.

### 3단계에서 기억할 것

- Service 테스트는 **스프링도 DB도 없이** 빠르게 돈다
- 정상 하나만이 아니라 **"없을 때 어떻게 되나"** 같은 실패 경로도 테스트한다
- 흐름은 `when(...)`으로 **상황을 만들고** → `assertThat`/`assertThatThrownBy`로 **결과를 확인**

---

## 6. 4단계 - Controller 테스트 (`@WebMvcTest` + `MockMvc`)

Controller는 **요청과 응답의 모양**을 담당한다. 그래서 확인할 것도 URL, 상태 코드, 응답 JSON, 입력값 검증이다. 로직은 Service 테스트에서 이미 확인했으니 **Service는 가짜로 바꾼다.**

```java
@RestController
@RequestMapping("/api/todos")
@RequiredArgsConstructor
public class TodoController {

  private final TodoService todoService;

  public record CreateRequest(@NotBlank String title) {}

  public record TodoResponse(Long id, String title, boolean done) {
    static TodoResponse from(Todo t) {
      return new TodoResponse(t.getId(), t.getTitle(), t.isDone());
    }
  }

  @PostMapping
  @ResponseStatus(HttpStatus.CREATED)
  public TodoResponse create(@Valid @RequestBody CreateRequest request) {
    return TodoResponse.from(todoService.create(request.title()));
  }

  @GetMapping("/{id}")
  public TodoResponse get(@PathVariable Long id) {
    return TodoResponse.from(todoService.get(id));
  }
}
```

없는 할 일은 예외 클래스에 `@ResponseStatus`를 붙여서 404로 응답하게 했다.

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class TodoNotFoundException extends RuntimeException {
  public TodoNotFoundException(Long id) {
    super("할 일을 찾을 수 없습니다. id=" + id);
  }
}
```

```java
@WebMvcTest(TodoController.class)
class TodoControllerTest {

  @Autowired
  MockMvc mockMvc;                  // 가짜 HTTP 요청을 보내는 도구

  @MockitoBean
  TodoService todoService;          // 가짜 Service

  @Test
  @DisplayName("할 일을 만들면 201 과 만들어진 내용을 돌려준다")
  void create() throws Exception {
    when(todoService.create("운동")).thenReturn(new Todo("운동"));                  // given

    mockMvc.perform(post("/api/todos")                                           // when
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"title": "운동"}
                """))
        .andExpect(status().isCreated())                                         // then
        .andExpect(jsonPath("$.title").value("운동"))
        .andExpect(jsonPath("$.done").value(false));
  }

  @Test
  @DisplayName("제목이 비어 있으면 400")
  void createBlank() throws Exception {
    mockMvc.perform(post("/api/todos")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"title": ""}
                """))
        .andExpect(status().isBadRequest());
  }

  @Test
  @DisplayName("없는 할 일을 조회하면 404")
  void getNotFound() throws Exception {
    when(todoService.get(99L)).thenThrow(new TodoNotFoundException(99L));

    mockMvc.perform(get("/api/todos/{id}", 99L))
        .andExpect(status().isNotFound());
  }
}
```

### 여기서 정리한 것

| | 뜻 |
|---|---|
| `@WebMvcTest(TodoController.class)` | **웹(Controller) 부분만** 띄운다. 서버를 실제로 켜지 않아서 빠르다 |
| `MockMvc` | HTTP 요청을 흉내 내는 도구. 포트 없이 컨트롤러를 호출한다 |
| `@MockitoBean` | 스프링이 관리하는 Bean을 가짜로 교체. Controller가 쓰는 Service를 가짜로 만든다 |
| `perform(post("/api/todos"))` | POST 요청을 보낸다 |
| `.contentType(...)`, `.content(...)` | 요청 헤더와 본문(JSON). `"""`는 여러 줄 문자열 |
| `status().isCreated()` | 상태 코드가 201인지 검증 |
| `jsonPath("$.title").value("운동")` | 응답 JSON의 `title` 값이 "운동"인지. `$`는 JSON의 시작 |

### 4단계에서 기억할 것

- Controller 테스트는 **요청 → 응답의 모양**만 확인한다. 로직은 확인하지 않는다
- 성공뿐 아니라 **실패 응답(400, 404)** 도 테스트한다

---

## 7. 한눈에 정리

| | 1단계 | 2단계 | 3단계 | 4단계 |
|---|---|---|---|---|
| **대상** | 자바 클래스 | Repository | Service | Controller |
| **어노테이션** | `@Test` | `@DataJpaTest` | `@ExtendWith(MockitoExtension.class)` | `@WebMvcTest` |
| **스프링 사용** | ❌ | 일부 | ❌ | 일부 |
| **가짜(Mock)** | 없음 | 없음 | `@Mock` Repository | `@MockitoBean` Service |
| **확인하는 것** | 객체의 규칙 | 쿼리 결과 | 로직·예외 | 상태 코드·JSON |

어떤 걸 써야 할지 헷갈릴 때 쓸 기준도 만들어 뒀다.

```
스프링이 필요 없나?                  → 그냥 @Test (1단계) 또는 Mockito (3단계)
DB 쿼리를 확인하고 싶나?             → @DataJpaTest
HTTP 요청/응답을 확인하고 싶나?       → @WebMvcTest + MockMvc
전체가 연결돼서 동작하는지 보고 싶나?  → @SpringBootTest (느리니까 꼭 필요할 때만)
```

---

## 8. 앞으로 조심할 점

공부하면서 "이렇게 쓰면 안 되겠구나" 싶었던 것들이다.

| 조심할 것 | 이렇게 한다 |
|---|---|
| 테스트 하나에 여러 가지를 한꺼번에 검증 | 한 테스트에는 한 가지만 |
| 정상 케이스만 테스트 | 빈 값, 없는 데이터 같은 실패 케이스도 함께 |
| `assertThat(result).isNotNull()`만 쓰기 | 값을 직접 비교 (`isEqualTo`) |
| 모든 테스트에 `@SpringBootTest` | 필요한 부분만 띄우는 슬라이스 테스트 |
| 상태 코드만 확인 | 응답 본문(JSON)도 함께 확인 |

---

## 9. 실행 결과

```bash
./gradlew test
```

| 테스트 클래스 | 테스트 수 | 결과 |
|---|---|---|
| `TodoTest` | 3 | ✅ |
| `TodoRepositoryTest` | 1 | ✅ |
| `TodoServiceTest` | 3 | ✅ |
| `TodoControllerTest` | 3 | ✅ |

> Spring Boot 4에서는 테스트 모듈이 분리되어 import 경로가 달라졌다.
> ```java
> import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;     // Boot 3 : ...test.autoconfigure.orm.jpa.DataJpaTest
> import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;        // Boot 3 : ...test.autoconfigure.web.servlet.WebMvcTest
> import org.springframework.test.context.bean.override.mockito.MockitoBean;  // Spring 6.2+ (@MockBean 의 후속)
> ```

---

## 10. 오늘 배운 것 정리

- 테스트는 **Given(준비) → When(실행) → Then(검증)** 의 반복이다.
- 어디를 테스트하느냐에 따라 **스프링의 어느 부분을 띄우고 무엇을 가짜로 바꿀지**만 달라진다.
  - 자바 클래스 → 그냥 `@Test`
  - Repository → `@DataJpaTest`
  - Service → Mockito (`@Mock`, `@InjectMocks`)
  - Controller → `@WebMvcTest` + `MockMvc` + `@MockitoBean`
- 성공만이 아니라 **실패 케이스**도 테스트한다.
- 각 계층은 **자기 일만** 확인한다. Controller 테스트에서 로직을 검증하지 않는다.

### 다음에 공부할 것

- `@ParameterizedTest`로 같은 테스트를 여러 입력값으로 반복하기
- 에러 코드를 담은 공통 예외 응답 검증
- `@SpringBootTest` 통합 테스트
- 시간·동시성이 걸린 로직 테스트
