---
title: "Spring MVC 컨트롤러 테스트 - MockMvc와 Mockito 실전 패턴"
date: 2026-11-08 20:31:00 +0900
categories: [Spring, Testing]
tags: [spring-boot, mockmvc, mockito, junit5, webmvctest, controller-test, validation]
description: "Spring Boot 4에서 MockMvc와 Mockito로 HTTP 요청·응답, Multipart, Bean Validation, 예외 처리, Pageable·커서 전달을 검증하는 컨트롤러 테스트 패턴 정리"
toc: true
---

컨트롤러 테스트는 비즈니스 로직의 실행 결과가 아니라 **HTTP 요청을 받는 경계가 올바르게 동작하는지** 확인한다. 유저·채널·메시지 API에 공통으로 적용되는 패턴을 기준으로 정리한다.

> **기준 환경**: Java, JUnit Jupiter, Mockito, Spring Boot 4.1.x, Spring MVC. 예시 URI는 `/api/users`, `/api/channels`, `/api/messages`이며, 클래스 수준의 `@RequestMapping`이 해당 URI로 설정되어 있다고 가정한다. `MessageDto`, `PageResponse` 등 생성자 구조가 제시되지 않은 타입은 임의의 필드를 가정하지 않는다.
{: .prompt-info }

## 1. 컨트롤러 테스트의 범위

| 검증 대상 | 테스트 내용 |
|---|---|
| 요청 매핑 | HTTP Method와 URL이 메서드에 연결되는가? |
| 파라미터 바인딩 | `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestPart`를 올바르게 변환하는가? |
| 유효성 검증 | 제약 조건을 위반한 요청을 Service 호출 전에 차단하는가? |
| Service 연동 | 정확한 인자로 해당 메서드를 호출하는가? |
| 응답 | HTTP 상태, JSON 필드, 빈 응답 본문이 올바른가? |
| 예외 변환 | Service 예외 및 MVC 예외를 의도한 HTTP 상태로 처리하는가? |

실제 DB에 데이터가 생성·수정·삭제되었는지는 이 테스트의 책임이 아니다. 해당 동작은 Service 테스트 또는 통합 테스트에서 검증한다.

## 2. 테스트 환경 구성

### 2-1. `@WebMvcTest`와 `@MockitoBean`

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

@WebMvcTest(ChannelController.class)
class ChannelControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private ChannelService channelService;
}
```

- `@WebMvcTest`: MVC 관련 Bean을 중심으로 테스트 컨텍스트 구성. 컨트롤러, 컨트롤러 어드바이스, 변환기 등이 대상이다.
- `MockMvc`: 실제 HTTP 서버를 띄우지 않고 HTTP 요청과 응답을 검증한다.
- `@MockitoBean`: 테스트 대상 컨트롤러의 Service 의존성을 Mock으로 대체한다.
- Spring Boot 4의 `@WebMvcTest` import는 `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`이다. 이전 Spring Boot 3.x의 패키지와 다르다.

`@SpringBootTest`는 전체 애플리케이션 컨텍스트가 필요한 경우에 사용한다. **Spring Boot 4에서 `@SpringBootTest`와 MockMvc를 함께 사용하려면 `@AutoConfigureMockMvc`도 필요하다.**

`@WebMvcTest`는 전역 `@RestControllerAdvice`도 검색 대상에 포함하지만, 테스트 구성에서 해당 Bean을 제외했다면 예외 응답이 실제 애플리케이션과 달라질 수 있다. Spring Security가 적용되어 있으면 인증·인가 필터 역시 테스트 결과에 영향을 준다.

### 2-2. 가장 많이 사용하는 MockMvc 구문

```java
mockMvc.perform(post("/api/channels/public")
        .contentType(MediaType.APPLICATION_JSON)
        .content(json))
    .andExpect(status().isCreated())
    .andExpect(jsonPath("$.name").value("channelName"));
```

| 구문 | 의미 |
|---|---|
| `perform(get/post/patch/delete(...))` | 가상 HTTP 요청 실행 |
| `contentType(APPLICATION_JSON)` | 요청 본문의 Content-Type 지정 |
| `content(json)` | 요청 본문 지정 |
| `andExpect(status().isOk())` | HTTP 200 검증 |
| `andExpect(jsonPath("$.name").value(...))` | 응답 JSON의 특정 값 검증 |
| `andExpect(content().string(""))` | 응답 본문이 비어 있는지 검증 |

UUID는 Java에서 `UUID` 타입이지만 일반적인 JSON 응답에서는 문자열로 직렬화된다. 따라서 JSON 검증에서는 `channelId.toString()`을 사용한다.

## 3. Mockito: 조건 설정과 호출 검증

### 3-1. 반환값과 예외 설정

```java
// 정상 결과를 반환하는 상황
when(channelService.update(eq(channelId), any(ChannelUpdateRequest.class)))
    .thenReturn(channelDto);

// Service에서 예외가 발생하는 상황
when(channelService.update(eq(channelId), any(ChannelUpdateRequest.class)))
    .thenThrow(new NoSuchElementException("채널이 존재하지 않습니다."));
```

- `when(...)`: Mock 메서드 호출의 조건을 설정한다.
- `thenReturn(...)`: 조건이 맞으면 지정한 값을 반환한다.
- `thenThrow(...)`: 조건이 맞으면 지정한 예외를 발생시킨다.
- `eq(channelId)`: 해당 값과 같은 인자만 허용한다.
- `any(ChannelUpdateRequest.class)`: 지정한 타입의 **null이 아닌** 객체를 허용한다.
- `isNull()`: `null` 인자를 명시적으로 매칭한다.

하나의 메서드 호출에서 Mockito의 인자 매처를 사용한다면 다른 인자도 `eq()`, `any()` 같은 매처로 작성해야 한다.

### 3-2. `void` 메서드는 `doThrow()` 사용

```java
// delete()는 반환값이 없는 void 메서드
// 성공 시 기본 Mock 동작은 아무것도 하지 않으므로 설정 불필요
verify(channelService).delete(channelId);

// 실패 상황만 명시적으로 설정
doThrow(new NoSuchElementException("채널이 존재하지 않습니다."))
    .when(channelService).delete(channelId);
```

`when(channelService.delete(...))`는 `void` 표현식을 전달하려 하므로 사용할 수 없다.

### 3-3. `verify()`, `argThat()`, `ArgumentCaptor`

```java
// 특정 인자로 호출되었는가?
verify(channelService).delete(channelId);

// 유효하지 않은 요청이므로 Service가 전혀 호출되지 않았는가?
verifyNoInteractions(channelService);

// DTO 내부 값까지 확인
verify(channelService).update(
    eq(channelId),
    argThat(request -> "newName".equals(request.name()))
);
```

`when()`은 **동작을 준비**하고 `verify()`는 **실제 호출을 확인**한다. 응답만 검증하면 Service에 잘못된 인자를 전달해도 테스트가 통과할 수 있으므로 중요한 요청 값은 함께 확인한다.

## 4. JSON 요청: 생성과 수정

### 4-1. PUBLIC 채널 생성 성공 (201)

```java
@Test
@DisplayName("PUBLIC 채널 생성 성공")
void public_channel_create_success_case() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();
    ChannelDto channelDto = new ChannelDto(
        channelId, ChannelType.PUBLIC, "channelName",
        "channelDesc", List.of(), null
    );

    String json = """
        {
          "name": "channelName",
          "description": "channelDesc"
        }
        """;

    when(channelService.createPublic(any())).thenReturn(channelDto);

    // when & then
    mockMvc.perform(post("/api/channels/public")
            .contentType(MediaType.APPLICATION_JSON)
            .content(json))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.id").value(channelId.toString()))
        .andExpect(jsonPath("$.type").value("PUBLIC"))
        .andExpect(jsonPath("$.name").value("channelName"));

    verify(channelService).createPublic(any());
}
```

`ChannelDto`는 실제 프로젝트에서 사용한 생성자 형태에 맞췄다. `jsonPath()`는 Java 객체가 아니라 **직렬화된 JSON**을 검사하므로 UUID는 문자열, Enum은 기본 직렬화 설정에서 이름 문자열로 비교한다.

### 4-2. PUBLIC 채널 수정 실패: 존재하지 않는 채널 (404)

```java
@Test
@DisplayName("PUBLIC 채널 수정 실패 - 채널 없음")
void public_channel_update_fail_not_found() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();
    String json = """
        {
          "name": "newName",
          "description": "newDesc"
        }
        """;

    when(channelService.update(eq(channelId), any(ChannelUpdateRequest.class)))
        .thenThrow(new NoSuchElementException("채널이 존재하지 않습니다."));

    // when & then
    mockMvc.perform(patch("/api/channels/{channelId}", channelId)
            .contentType(MediaType.APPLICATION_JSON)
            .content(json))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.code").value("NO_CONTENTS"));

    verify(channelService).update(eq(channelId), any(ChannelUpdateRequest.class));
}
```

위 코드는 전역 예외 처리기가 `NoSuchElementException`을 404와 `NO_CONTENTS`로 변환한다는 전제다. 실제 Service에서 커스텀 예외를 사용한다면 해당 예외로 변경해야 한다.

### 4-3. `@Size`를 적용한 수정 요청 검증 (400)

```java
public record ChannelUpdateRequest(
    @Size(max = 100) String name,
    @Size(max = 200) String description
) {}
```

| 입력 | `@Size(max = 100)` 결과 |
|---|---|
| `null` | 통과 |
| `""` | 통과 |
| 길이 100 | 통과 |
| 길이 101 | 실패 |

`@Size`는 길이를 검사하며, `@NotBlank`처럼 빈 문자열을 금지하지 않는다. PATCH에서 `null`을 미수정 값으로 취급한다면 Service의 부분 수정 정책과 함께 설계해야 한다.

```java
@Test
@DisplayName("PUBLIC 채널 수정 실패 - 이름 길이 초과")
void public_channel_update_fail_invalid_name() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();
    String json = """
        {
          "name": "%s",
          "description": "newDesc"
        }
        """.formatted("a".repeat(101));

    // when & then
    mockMvc.perform(patch("/api/channels/{channelId}", channelId)
            .contentType(MediaType.APPLICATION_JSON)
            .content(json))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.code").value("VALID_EXCEPTION"))
        .andExpect(jsonPath("$.details.name").exists());

    verifyNoInteractions(channelService);
}
```

`VALID_EXCEPTION`은 제시된 `GlobalExceptionHandler`의 실제 응답 코드다. 에러 코드 명칭을 `VALIDATION_FAILED`로 변경했다면 테스트도 함께 변경한다.

## 5. Multipart 요청: JSON Part와 첨부파일

`@RequestBody`는 요청 본문을 하나의 객체로 변환하지만, `@RequestPart`는 Multipart 요청의 **이름 붙은 Part**를 변환한다. JSON Part는 `MockMultipartFile`로 구성한다.

### 5-1. 유저 생성 성공: JSON Part 전달 (201)

```java
@Test
@DisplayName("유저 생성 성공")
void user_create_success_case() throws Exception {
    // given
    UUID userId = UUID.randomUUID();
    UserDto userDto = new UserDto(
        userId, "username", "user@naver.com", null, false
    );

    String json = """
        {
          "username": "username",
          "email": "user@naver.com",
          "password": "password"
        }
        """;

    MockMultipartFile request = new MockMultipartFile(
        "userCreateRequest", "", MediaType.APPLICATION_JSON_VALUE,
        json.getBytes(StandardCharsets.UTF_8)
    );

    when(userService.create(any(UserCreateRequest.class), isNull()))
        .thenReturn(userDto);

    // when & then
    mockMvc.perform(multipart("/api/users").file(request))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.id").value(userId.toString()))
        .andExpect(jsonPath("$.username").value("username"))
        .andExpect(jsonPath("$.email").value("user@naver.com"));
}
```

여기서 `isNull()`은 첨부 프로필이 없는 요청이므로 Service의 두 번째 인자로 `null`이 전달된다는 구현에 맞춘 것이다. `any()`를 무조건 사용하는 것보다 실제 계약을 드러낸다.

### 5-2. 필수 JSON Part 누락 (400)

```java
@Test
@DisplayName("유저 생성 실패 - 필수 Part 누락")
void user_create_fail_missing_request() throws Exception {
    mockMvc.perform(multipart("/api/users"))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.exceptionType")
            .value("MissingServletRequestPartException"));

    verifyNoInteractions(userService);
}
```

이 테스트는 `@Valid` 실패가 아니다. `@RequestPart("userCreateRequest")`에 필요한 Part **자체가 없기 때문에**, DTO를 만들기 전에 `MissingServletRequestPartException`이 발생한다. 400 응답을 얻으려면 이 예외를 400으로 매핑한 처리기가 필요하다(7절 참조).

### 5-3. 메시지 생성: 첨부파일 전달 방식

메시지 생성에서는 JSON Part `messageCreateRequest`와 선택적인 파일 목록 `attachments`를 함께 받는다.

```java
UUID authorId = UUID.randomUUID();
UUID channelId = UUID.randomUUID();
String json = """
    {
      "authorId": "%s",
      "channelId": "%s",
      "content": "안녕하세요"
    }
    """.formatted(authorId, channelId);

MockMultipartFile request = new MockMultipartFile(
    "messageCreateRequest", "", MediaType.APPLICATION_JSON_VALUE,
    json.getBytes(StandardCharsets.UTF_8)
);

mockMvc.perform(multipart("/api/messages").file(request))
    .andExpect(status().isCreated());

verify(messageService).create(
    any(MessageCreateRequest.class),
    argThat(List::isEmpty)
);
```

이 예시는 **첨부파일이 없는 경우의 요청 바인딩 및 빈 리스트 전달**을 확인한다. Mock Service의 기본 응답은 `null`이므로 응답 본문은 검증하지 않는다. 응답 JSON까지 테스트하려면 실제 구조에 맞는 `MessageDto` 픽스처를 반환하도록 설정한다.

첨부파일이 있다면 같은 Multipart 요청에 파일 Part를 추가한다. 각 파일의 Part 이름은 컨트롤러의 파라미터 이름인 `attachments`로 맞춘다.

```java
MockMultipartFile attachment = new MockMultipartFile(
    "attachments", "image.png", "image/png", imageBytes
);

mockMvc.perform(multipart("/api/messages")
        .file(request)
        .file(attachment));
```

`imageBytes`는 유효한 이미지 바이트 배열로 준비해야 한다. 파일의 확장자·MIME 타입·실제 바이트를 검사하는 `validateImageFile()` 구현에 따라 성공 조건이 달라지므로, 임의의 바이트를 정상 이미지로 가정하지 않는다.

## 6. DELETE 요청: 반환값 없는 Service

### 6-1. 삭제 성공 (204)

```java
@Test
@DisplayName("채널 삭제 성공")
void channel_delete_success_case() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();

    // when & then
    mockMvc.perform(delete("/api/channels/{channelId}", channelId))
        .andExpect(status().isNoContent())
        .andExpect(content().string(""));

    verify(channelService).delete(channelId);
}
```

성공 시 Mock의 `void` 메서드는 기본적으로 아무 동작도 하지 않는다. 별도 `doNothing()` 설정이 필요 없다.

### 6-2. 삭제 실패 (404)

```java
@Test
@DisplayName("메시지 삭제 실패 - 메시지 없음")
void message_delete_fail_not_found() throws Exception {
    // given
    UUID messageId = UUID.randomUUID();

    doThrow(new NoSuchElementException("메시지가 존재하지 않습니다."))
        .when(messageService).delete(messageId);

    // when & then
    mockMvc.perform(delete("/api/messages/{messageId}", messageId))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.code").value("NO_CONTENTS"));

    verify(messageService).delete(messageId);
}
```

채널·메시지 삭제는 경로와 Service만 다를 뿐 같은 테스트 패턴을 사용한다. 같은 코드를 두 API에 대해 장황하게 반복할 필요는 없다.

## 7. 전역 예외 처리와 HTTP 상태

컨트롤러가 예외를 발생시켰다는 사실과 **최종 HTTP 상태 코드**는 별개다. 전역 예외 처리기의 설정이 실제 응답을 결정한다.

### 7-1. `@Valid`와 필수 요청값 검사 구분

| 실패 상황 | 발생 단계 | 대표 예외 | 권장 상태 |
|---|---|---|---|
| 필수 Multipart Part 누락 | 요청 Part 바인딩 | `MissingServletRequestPartException` | 400 |
| 필수 Query Parameter 누락 | 요청 파라미터 바인딩 | `MissingServletRequestParameterException` | 400 |
| JSON 형식 또는 UUID 역직렬화 실패 | 요청 본문 변환 | `HttpMessageNotReadableException` | 400 |
| DTO의 `@Size`, `@NotBlank` 등 위반 | Bean Validation | `MethodArgumentNotValidException` | 400 |
| 조회 대상 없음 | Service 비즈니스 로직 | 프로젝트가 지정한 예외 | 404 등 |
| 예상하지 못한 서버 오류 | 요청 처리 중 | `Exception` 등 | 500 |

**주의:** `@Valid`는 DTO 인스턴스가 생성되고 Bean Validation이 실행될 때만 동작한다. DTO 필드에 실제 검증 제약이 없으면 `@Valid`만으로 빈 문자열·빈 리스트를 거부하지 않는다.

### 7-2. 400을 기대했는데 500이 나온 이유

다음처럼 범용 핸들러가 있으면, 별도 처리가 없는 Spring MVC 예외까지 500으로 변환될 수 있다.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
    return build(HttpStatus.INTERNAL_SERVER_ERROR,
        "INTERNAL_ERROR", e, new HashMap<>());
}
```

실제 테스트에서는 필수 `userCreateRequest`가 없어서 `MissingServletRequestPartException`이 발생했지만, 범용 핸들러에 의해 최종 응답이 500이 되었다. 이때 **테스트 기대값을 500으로 바꾸지 않고 예외 매핑을 수정**해야 한다.

기존 `GlobalExceptionHandler`에 필요한 MVC 예외를 명시적으로 매핑할 수 있다.

```java
@ExceptionHandler(MissingServletRequestPartException.class)
public ResponseEntity<ErrorResponse> handleMissingPart(
    MissingServletRequestPartException e) {
    return build(HttpStatus.BAD_REQUEST,
        "MISSING_REQUEST_PART", e, new HashMap<>());
}

@ExceptionHandler(MissingServletRequestParameterException.class)
public ResponseEntity<ErrorResponse> handleMissingParameter(
    MissingServletRequestParameterException e) {
    return build(HttpStatus.BAD_REQUEST,
        "INVALID_REQUEST", e, new HashMap<>());
}

@ExceptionHandler(HttpMessageNotReadableException.class)
public ResponseEntity<ErrorResponse> handleNotReadable(
    HttpMessageNotReadableException e) {
    return build(HttpStatus.BAD_REQUEST,
        "INVALID_REQUEST", e, new HashMap<>());
}
```

이 메서드들은 앞서 정의한 `build(HttpStatus, String, Exception, Map<String, Object>)` 헬퍼를 사용하는 예시다. `MISSING_REQUEST_PART`는 새로 제안하는 에러 코드이므로 프로젝트의 에러 코드 정책에 맞춰 통일해야 한다. `ResponseEntityExceptionHandler`를 상속한다면 같은 예외에 `@ExceptionHandler`를 중복 등록하지 말고 제공되는 `handle...` 메서드를 재정의한다.

## 8. `@RequestParam`, Pageable, 커서 조회

채널 메시지 조회는 다음 값을 Service에 전달한다.

```java
@GetMapping
public ResponseEntity<PageResponse<MessageDto>> findAllInChannel(
    @RequestParam("channelId") UUID channelId,
    @PageableDefault(size = 50, sort = "createdAt", direction = Sort.Direction.DESC)
    Pageable pageable,
    @RequestParam(value = "cursor", required = false) Instant cursor) {

    return ResponseEntity.ok(
        messageService.findAllByChannelId(channelId, cursor, pageable)
    );
}
```

### 8-1. 기본 Pageable 조건 검증

```java
@Test
@DisplayName("채널 메시지 조회 - 기본 페이징 설정")
void message_find_all_default_pageable() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();

    // when & then
    mockMvc.perform(get("/api/messages")
            .param("channelId", channelId.toString()))
        .andExpect(status().isOk());

    ArgumentCaptor<Pageable> captor = ArgumentCaptor.forClass(Pageable.class);
    verify(messageService).findAllByChannelId(
        eq(channelId), isNull(), captor.capture());

    Pageable pageable = captor.getValue();
    assertEquals(0, pageable.getPageNumber());
    assertEquals(50, pageable.getPageSize());
    assertEquals(Sort.Direction.DESC,
        pageable.getSort().getOrderFor("createdAt").getDirection());
}
```

이 테스트는 **입력 바인딩과 Service 인자 전달만 검증**한다. Service Mock이 반환하는 값은 기본적으로 `null`이므로 응답 본문 검증은 제외했다. 목록의 JSON 구조·건수까지 확인하려면 실제 `PageResponse<MessageDto>` 픽스처를 구성해 별도 테스트한다.

- `any(Pageable.class)`: Pageable이 전달되었다는 사실만 확인한다.
- `captor.capture()`: 실제 전달된 Pageable을 가져와 크기·정렬 조건까지 확인한다.
- 기본 요청에서는 `page = 0`, `size = 50`, `createdAt DESC`, `cursor = null`이다.

### 8-2. 커서 전달 검증

```java
@Test
@DisplayName("채널 메시지 조회 - 커서 전달")
void message_find_all_with_cursor() throws Exception {
    // given
    UUID channelId = UUID.randomUUID();
    Instant cursor = Instant.parse("2026-11-08T10:00:00Z");

    // when & then
    mockMvc.perform(get("/api/messages")
            .param("channelId", channelId.toString())
            .param("cursor", cursor.toString()))
        .andExpect(status().isOk());

    verify(messageService).findAllByChannelId(
        eq(channelId), eq(cursor), any(Pageable.class));
}
```

HTTP 요청의 `cursor`는 문자열이지만 Service의 파라미터는 `Instant`다. 테스트는 바인딩 후의 값이 같다는 것을 확인한다. 프로젝트의 날짜·시간 변환 설정에 따라 `@DateTimeFormat` 설정이 필요할 수 있다.

### 8-3. 필수 채널 ID 누락 (400)

```java
@Test
@DisplayName("채널 메시지 조회 실패 - channelId 누락")
void message_find_all_missing_channel_id() throws Exception {
    mockMvc.perform(get("/api/messages"))
        .andExpect(status().isBadRequest());

    verifyNoInteractions(messageService);
}
```

`MissingServletRequestParameterException`을 400으로 처리하는 예외 매핑이 있어야 한다.

## 9. PRIVATE 채널 생성 시 리스트 검증

```java
public record PrivateChannelCreateRequest(
    @NotEmpty List<UUID> participantIds
) {
    public PrivateChannelCreateRequest {
        if (participantIds != null) {
            participantIds = List.copyOf(participantIds);
        }
    }
}
```

- `@NotEmpty`: `null`과 빈 리스트 모두 검증 실패로 처리한다.
- `List.copyOf()`: 전달받은 리스트를 방어적으로 복사하지만 **빈 리스트를 거부하지는 않는다**.
- `List.copyOf(null)`은 `NullPointerException`을 발생시키므로 검증 전에 실행되는 생성자에서 `null`을 바로 복사하지 않도록 주의한다.

```java
@Test
@DisplayName("PRIVATE 채널 생성 실패 - 참여자 목록 비어 있음")
void private_channel_create_empty_participants() throws Exception {
    mockMvc.perform(post("/api/channels/private")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                { "participantIds": [] }
                """))
        .andExpect(status().isBadRequest());

    verifyNoInteractions(channelService);
}
```

## 10. API별 최소 테스트 체크리스트

| API | 성공 | 우선 검증할 실패 |
|---|---|---|
| 유저 생성 (`multipart`) | 201, DTO JSON | 필수 Part 누락 400, DTO 필드 검증 400, 중복 사용자 예외 |
| PUBLIC 채널 생성 | 201, ID·이름 | DTO 제약 위반 400, Service 예외 |
| PRIVATE 채널 생성 | 201, 참여자 전달 | 빈 참여자 목록 400, UUID 형식 오류 400 |
| 채널 수정 | 200, 변경 필드 | 이름 101자/설명 201자 400, 대상 없음 404 |
| 채널 삭제 | 204, 본문 없음 | 대상 없음 404 |
| 메시지 생성 (`multipart`) | 201, 첨부파일 변환 | 필수 Part 누락 400, 파일 검증 실패, 대상 없음 |
| 메시지 수정 | 200, 변경 내용 | `MessageUpdateRequest`의 실제 제약 위반, 대상 없음 404 |
| 채널 메시지 목록 | 200, 기본 Pageable·cursor | channelId 누락 400, 대상 없음 404(정책에 따라) |
| 메시지 삭제 | 204, 본문 없음 | 대상 없음 404 |

**실패 상태 코드는 컨트롤러 코드만으로 확정되지 않는다.** Service가 던지는 실제 예외, DTO 제약 조건, `GlobalExceptionHandler`의 HTTP 매핑을 함께 확인해야 한다. 특히 빈 조회 결과는 보통 404가 아니라 200과 빈 목록으로 반환한다.

## 11. 정리

1. 컨트롤러 테스트는 **HTTP 경계와 Service 호출 계약**을 검증한다.
2. 정상 응답은 `status()`와 `jsonPath()`로, Service 인자는 `verify()`·`argThat()`·`ArgumentCaptor`로 확인한다.
3. `@Valid`는 객체 필드 검증이며, 필수 `@RequestPart` 누락은 별도의 MVC 예외다.
4. 반환값이 있는 Mock 메서드는 `when().thenReturn()/thenThrow()`, `void` 메서드는 `doThrow().when()`을 사용한다.
5. 전역 범용 예외 처리기가 4xx 요청 오류를 500으로 바꾸지 않도록 예외 매핑을 확인한다.

### 참고 문서

- [Spring Boot 공식 문서 — Auto-configured Spring MVC Tests](https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html)
- [Spring Boot 4.0 Migration Guide — Testing Features](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Framework 공식 문서 — Error Responses](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
