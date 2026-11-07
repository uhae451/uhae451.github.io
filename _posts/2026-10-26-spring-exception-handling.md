---
title: "Spring 예외 처리"
date: 2026-10-26 09:00:00 +0900
categories: ["Spring", "Exception"]
tags: ["spring", "exception-handler", "rest-controller-advice", "error-response", "error-code"]
description: "Spring의 예외 처리 흐름과 @RestControllerAdvice, 사용자 정의 예외 및 공통 오류 응답 설계를 정리한다."
toc: true
---

Spring 애플리케이션에서는 요청 처리 중 다양한 예외가 발생할 수 있다.

예를 들어 주문 조회 API에서 존재하지 않는 주문을 조회하거나, 주문 생성 중 재고가 부족할 수 있다.

이때 단순히 Java 예외를 발생시키는 것으로 끝내기보다, **예외를 적절한 HTTP 상태 코드와 응답 형태로 변환해서 클라이언트에게 전달하는 구조**가 필요하다.

예를 들어 존재하지 않는 주문을 조회했다고 가정한다.

```http
GET /orders/999
```

이 경우 서버에서는 다음과 같은 응답을 반환하는 것이 자연스럽다.

```http
HTTP/1.1 404 Not Found
```

```json
{
  "code": "ORDER_NOT_FOUND",
  "message": "주문을 찾을 수 없습니다."
}
```

즉, Spring 예외 처리의 핵심은 다음과 같다.

```text
Java Exception
      ↓
Spring 예외 처리
      ↓
HTTP 상태 코드
      ↓
Error Response
```

---

## 1. 예외는 어디서 발생하는가

예외는 애플리케이션의 여러 계층에서 발생할 수 있다.

| 위치 | 예외 예시 |
|---|---|
| Controller | 잘못된 요청 파라미터 |
| Service | 비즈니스 규칙 위반 |
| Repository | 데이터 접근 오류 |
| Validation | 요청 DTO 검증 실패 |

예를 들어 주문 조회 Service가 다음과 같다고 가정한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    @Transactional(readOnly = true)
    public OrderResponse findOrder(Long id) {

        Order order = orderRepository.findById(id)
                .orElseThrow();

        return OrderResponse.from(order);
    }
}
```

`findById(id)` 결과가 없으면 `orElseThrow()`에서 예외가 발생한다.

하지만 이 상태로 두면:

```text
왜 실패했는지
어떤 HTTP 상태 코드를 반환해야 하는지
클라이언트가 어떤 형식으로 받아야 하는지
```

가 명확하지 않다.

그래서 실무에서는 의미 있는 Custom Exception을 사용하는 경우가 많다.

---

## 2. Custom Exception

주문이 존재하지 않을 때 사용할 예외를 따로 만들 수 있다.

```java
public class OrderNotFoundException extends RuntimeException {

    public OrderNotFoundException(Long id) {
        super("주문을 찾을 수 없습니다. id = " + id);
    }
}
```

Service에서는 다음과 같이 사용한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    @Transactional(readOnly = true)
    public OrderResponse findOrder(Long id) {

        Order order = orderRepository.findById(id)
                .orElseThrow(() ->
                        new OrderNotFoundException(id)
                );

        return OrderResponse.from(order);
    }
}
```

이렇게 하면 단순한 `NoSuchElementException`보다 예외의 의미가 명확해진다.

```text
OrderNotFoundException
```

만 봐도 어떤 문제가 발생했는지 알 수 있다.

---

## 3. 왜 `RuntimeException`을 상속하는가

실무에서 비즈니스 예외는 `RuntimeException`을 상속해서 만드는 경우가 많다.

```java
public class OrderNotFoundException
        extends RuntimeException {
}
```

이유는 여러 가지가 있지만 Spring에서는 특히 트랜잭션과 관련이 있다.

기본적으로 `@Transactional`은 `RuntimeException`이 발생하면 롤백한다.

```java
@Transactional
public void createOrder() {
    // RuntimeException 발생
}
```

이 경우 트랜잭션은 롤백된다.

반면 checked exception은 기본 설정에서는 자동 롤백되지 않을 수 있다.

따라서 Spring 기반 애플리케이션에서는 비즈니스 예외를 Runtime Exception 계열로 설계하는 경우가 많다.

---

## 4. `@ExceptionHandler`

특정 Controller에서 발생하는 예외를 처리하려면 `@ExceptionHandler`를 사용할 수 있다.

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public OrderResponse findOrder(
            @PathVariable Long id
    ) {
        return orderService.findOrder(id);
    }

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse>
    handleOrderNotFound(
            OrderNotFoundException e
    ) {

        ErrorResponse response =
                new ErrorResponse(
                        "ORDER_NOT_FOUND",
                        e.getMessage()
                );

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(response);
    }
}
```

이 경우 `OrderController` 내부에서 `OrderNotFoundException`이 발생하면 해당 메서드가 처리한다.

처리 흐름은 다음과 같다.

```text
OrderController
      ↓
OrderService
      ↓
OrderNotFoundException 발생
      ↓
@ExceptionHandler
      ↓
ErrorResponse 생성
      ↓
HTTP 404 반환
```

---

## 5. Controller마다 예외 처리를 작성하면 생기는 문제

Controller가 여러 개 생기면 같은 예외 처리 코드가 반복될 수 있다.

예를 들어:

```text
OrderController
ProductController
MemberController
PaymentController
```

각 Controller마다 다음과 같은 코드를 작성해야 할 수 있다.

```java
@ExceptionHandler(...)
```

이렇게 되면 예외 처리 로직이 여러 곳에 흩어진다.

그래서 실무에서는 전역 예외 처리를 사용한다.

---

## 6. `@RestControllerAdvice`

`@RestControllerAdvice`는 여러 Controller에서 발생하는 예외를 한곳에서 처리할 수 있도록 해준다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
}
```

주문 조회 예외를 처리하면 다음과 같다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse>
    handleOrderNotFound(
            OrderNotFoundException e
    ) {

        ErrorResponse response =
                new ErrorResponse(
                        "ORDER_NOT_FOUND",
                        e.getMessage()
                );

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(response);
    }
}
```

이제 `OrderController`에 별도의 예외 처리 코드를 작성하지 않아도 된다.

전체 구조는 다음과 같다.

```text
Controller
    ↓
Service
    ↓
Exception
    ↓
GlobalExceptionHandler
    ↓
ErrorResponse
    ↓
HTTP Response
```

---

## 7. ErrorResponse

실무에서는 예외마다 제각각 다른 JSON을 반환하기보다 공통 에러 응답 구조를 사용하는 것이 좋다.

예를 들어:

```java
public record ErrorResponse(
        String code,
        String message
) {
}
```

주문이 존재하지 않을 경우:

```json
{
  "code": "ORDER_NOT_FOUND",
  "message": "주문을 찾을 수 없습니다."
}
```

재고가 부족한 경우:

```json
{
  "code": "INSUFFICIENT_STOCK",
  "message": "재고가 부족합니다."
}
```

이렇게 응답 형식을 통일하면 클라이언트도 처리하기 쉬워진다.

---

## 8. HTTP 상태 코드와 예외 매핑

예외를 처리할 때는 적절한 HTTP 상태 코드를 선택하는 것이 중요하다.

대표적인 상태 코드는 다음과 같다.

| 상태 코드 | 의미 | 사용 예시 |
|---|---|---|
| `400 Bad Request` | 잘못된 요청 | Validation 실패 |
| `401 Unauthorized` | 인증 필요 | 로그인되지 않은 사용자 |
| `403 Forbidden` | 접근 권한 없음 | 다른 사용자의 주문 수정 |
| `404 Not Found` | 리소스 없음 | 주문 조회 실패 |
| `409 Conflict` | 현재 상태와 충돌 | 이미 취소된 주문 재취소 |
| `500 Internal Server Error` | 서버 내부 오류 | 예상하지 못한 시스템 오류 |

예외와 상태 코드는 다음처럼 매핑할 수 있다.

| Exception | HTTP Status |
|---|---|
| `OrderNotFoundException` | `404 Not Found` |
| `InsufficientStockException` | `409 Conflict` |
| `MethodArgumentNotValidException` | `400 Bad Request` |
| 예상하지 못한 Exception | `500 Internal Server Error` |

---

## 9. 비즈니스 예외 처리

주문 생성 중 재고가 부족한 경우를 생각해보자.

Service:

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;

    @Transactional
    public OrderResponse createOrder(
            CreateOrderRequest request
    ) {

        Product product = productRepository
                .findById(request.productId())
                .orElseThrow(() ->
                        new ProductNotFoundException(
                                request.productId()
                        )
                );

        if (product.getStock() < request.quantity()) {
            throw new InsufficientStockException();
        }

        Order order = Order.create(
                product,
                request.quantity()
        );

        Order savedOrder =
                orderRepository.save(order);

        return OrderResponse.from(savedOrder);
    }
}
```

재고 부족 예외:

```java
public class InsufficientStockException
        extends RuntimeException {

    public InsufficientStockException() {
        super("재고가 부족합니다.");
    }
}
```

전역 예외 처리:

```java
@ExceptionHandler(InsufficientStockException.class)
public ResponseEntity<ErrorResponse>
handleInsufficientStock(
        InsufficientStockException e
) {

    ErrorResponse response =
            new ErrorResponse(
                    "INSUFFICIENT_STOCK",
                    e.getMessage()
            );

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(response);
}
```

이 경우 클라이언트는 다음과 같은 응답을 받을 수 있다.

```http
HTTP/1.1 409 Conflict
```

```json
{
  "code": "INSUFFICIENT_STOCK",
  "message": "재고가 부족합니다."
}
```

---

## 10. Validation 예외 처리

Request DTO 검증에 실패하면 일반적으로 `MethodArgumentNotValidException`이 발생한다.

예를 들어:

```java
public record CreateOrderRequest(

        @NotNull
        Long productId,

        @NotNull
        @Positive
        Integer quantity
) {
}
```

Controller:

```java
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody CreateOrderRequest request
) {
    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(orderService.createOrder(request));
}
```

잘못된 요청:

```json
{
  "productId": null,
  "quantity": 0
}
```

이 경우 `MethodArgumentNotValidException`이 발생한다.

전역 예외 처리에서 다음처럼 처리할 수 있다.

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ValidationErrorResponse>
handleValidationException(
        MethodArgumentNotValidException e
) {

    List<FieldErrorResponse> errors =
            e.getBindingResult()
                    .getFieldErrors()
                    .stream()
                    .map(error ->
                            new FieldErrorResponse(
                                    error.getField(),
                                    error.getDefaultMessage()
                            )
                    )
                    .toList();

    ValidationErrorResponse response =
            new ValidationErrorResponse(
                    "INVALID_REQUEST",
                    "요청 값이 올바르지 않습니다.",
                    errors
            );

    return ResponseEntity
            .badRequest()
            .body(response);
}
```

검증 실패 응답은 다음과 같이 만들 수 있다.

```json
{
  "code": "INVALID_REQUEST",
  "message": "요청 값이 올바르지 않습니다.",
  "errors": [
    {
      "field": "productId",
      "message": "상품 ID는 필수입니다."
    },
    {
      "field": "quantity",
      "message": "수량은 1 이상이어야 합니다."
    }
  ]
}
```

---

## 11. 예상하지 못한 예외 처리

개발자가 미리 정의하지 않은 예외가 발생할 수도 있다.

이런 경우 최종적으로 `Exception`을 처리하는 Handler를 둘 수 있다. 단, JSON 파싱 실패·타입 변환 실패·지원하지 않는 HTTP 메서드 같은 Spring MVC 예외까지 500으로 바꾸면 안 된다. 아래 전체 예제처럼 `ResponseEntityExceptionHandler`를 상속하여 기본 상태 코드를 보존한다.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse>
handleException(
        Exception e
) {

    ErrorResponse response =
            new ErrorResponse(
                    "INTERNAL_SERVER_ERROR",
                    "서버 내부 오류가 발생했습니다."
            );

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(response);
}
```

여기서 중요한 점은 내부 예외 메시지를 그대로 클라이언트에게 노출하지 않는 것이다.

예를 들어:

```text
SQL 오류
DB 연결 정보
파일 경로
내부 클래스명
Stack Trace
```

같은 정보는 외부에 노출하지 않는 것이 좋다.

클라이언트에는 일반화된 메시지를 반환하고, 실제 예외 내용은 서버 로그에서 확인하도록 구성한다.

---

## 12. 예외 메시지와 에러 코드 분리

실무에서는 메시지만 사용하는 것보다 별도의 에러 코드를 사용하는 것이 좋다.

예를 들어:

```json
{
  "code": "ORDER_NOT_FOUND",
  "message": "주문을 찾을 수 없습니다."
}
```

여기서:

```text
code
```

는 프로그램이 처리하기 위한 값이고,

```text
message
```

는 사람이 읽기 위한 값이라고 볼 수 있다.

클라이언트는 문자열 메시지 대신:

```text
ORDER_NOT_FOUND
```

같은 고정된 코드를 기준으로 동작할 수 있다.

---

## 13. ErrorCode로 예외 정보 관리

예외가 많아지면 각 Exception Handler에 상태 코드와 에러 코드를 직접 작성하는 방식도 중복이 생긴다.

이 경우 Enum을 사용해 에러 정보를 관리할 수 있다.

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    ORDER_NOT_FOUND(
            HttpStatus.NOT_FOUND,
            "ORDER_NOT_FOUND",
            "주문을 찾을 수 없습니다."
    ),

    PRODUCT_NOT_FOUND(
            HttpStatus.NOT_FOUND,
            "PRODUCT_NOT_FOUND",
            "상품을 찾을 수 없습니다."
    ),

    INSUFFICIENT_STOCK(
            HttpStatus.CONFLICT,
            "INSUFFICIENT_STOCK",
            "재고가 부족합니다."
    );

    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

Custom Exception에서는 ErrorCode를 가지고 있도록 만들 수 있다.

```java
@Getter
public class BusinessException
        extends RuntimeException {

    private final ErrorCode errorCode;

    public BusinessException(
            ErrorCode errorCode
    ) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

주문 예외:

```java
public class OrderNotFoundException
        extends BusinessException {

    public OrderNotFoundException() {
        super(ErrorCode.ORDER_NOT_FOUND);
    }
}
```

이 구조를 사용하면 예외가 많아져도 상태 코드와 에러 코드를 한곳에서 관리할 수 있다.

---

## 14. 공통 비즈니스 예외 처리

`BusinessException`을 기준으로 처리하면 여러 Custom Exception을 하나의 Handler에서 처리할 수 있다.

```java
@ExceptionHandler(BusinessException.class)
public ResponseEntity<ErrorResponse>
handleBusinessException(
        BusinessException e
) {

    ErrorCode errorCode = e.getErrorCode();

    ErrorResponse response =
            new ErrorResponse(
                    errorCode.getCode(),
                    errorCode.getMessage()
            );

    return ResponseEntity
            .status(errorCode.getStatus())
            .body(response);
}
```

이제:

```text
OrderNotFoundException
ProductNotFoundException
InsufficientStockException
```

같은 비즈니스 예외를 같은 Handler에서 처리할 수 있다.

전체 구조는 다음과 같다.

```text
BusinessException
   ├─ OrderNotFoundException
   ├─ ProductNotFoundException
   └─ InsufficientStockException
```

그리고:

```text
BusinessException 발생
        ↓
GlobalExceptionHandler
        ↓
ErrorCode 확인
        ↓
HTTP Status + ErrorResponse
```

의 형태로 처리할 수 있다.

---

## 15. 전체 실무 구조

아래 Advice는 `org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler`를 상속한다. MVC 기본 오류는 부모가 적절한 상태 코드로 처리하며, 응답 본문까지 같은 DTO로 통일하려면 `handleExceptionInternal()` 등 확장 지점을 재정의한다. 10절의 검증 응답을 적용할 때도 상속한 핸들러와 중복되는 `@ExceptionHandler`를 추가하지 말고 `handleMethodArgumentNotValid()`를 재정의한다.

예외 처리 구조를 하나로 정리하면 다음과 같다.

### ErrorCode

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    ORDER_NOT_FOUND(
            HttpStatus.NOT_FOUND,
            "ORDER_NOT_FOUND",
            "주문을 찾을 수 없습니다."
    ),

    INSUFFICIENT_STOCK(
            HttpStatus.CONFLICT,
            "INSUFFICIENT_STOCK",
            "재고가 부족합니다."
    );

    private final HttpStatus status;
    private final String code;
    private final String message;
}
```

### BusinessException

```java
@Getter
public class BusinessException
        extends RuntimeException {

    private final ErrorCode errorCode;

    public BusinessException(
            ErrorCode errorCode
    ) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }
}
```

### Custom Exception

```java
public class OrderNotFoundException
        extends BusinessException {

    public OrderNotFoundException() {
        super(ErrorCode.ORDER_NOT_FOUND);
    }
}
```

### Service

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    @Transactional(readOnly = true)
    public OrderResponse findOrder(Long id) {

        Order order = orderRepository
                .findById(id)
                .orElseThrow(
                        OrderNotFoundException::new
                );

        return OrderResponse.from(order);
    }
}
```

### ErrorResponse

```java
public record ErrorResponse(
        String code,
        String message
) {
}
```

### GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse>
    handleBusinessException(
            BusinessException e
    ) {

        ErrorCode errorCode =
                e.getErrorCode();

        ErrorResponse response =
                new ErrorResponse(
                        errorCode.getCode(),
                        errorCode.getMessage()
                );

        return ResponseEntity
                .status(errorCode.getStatus())
                .body(response);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse>
    handleException(
            Exception e
    ) {

        ErrorResponse response =
                new ErrorResponse(
                        "INTERNAL_SERVER_ERROR",
                        "서버 내부 오류가 발생했습니다."
                );

        return ResponseEntity
                .status(
                        HttpStatus.INTERNAL_SERVER_ERROR
                )
                .body(response);
    }
}
```

---

## 정리

Spring 예외 처리의 핵심은 발생한 Java 예외를 그대로 클라이언트에게 전달하는 것이 아니라, **적절한 HTTP 상태 코드와 일관된 에러 응답으로 변환하는 것**이다.

각 구성 요소의 역할은 다음과 같다.

| 구성 요소 | 역할 |
|---|---|
| Custom Exception | 발생한 문제의 의미 표현 |
| `@ExceptionHandler` | 특정 예외 처리 |
| `@RestControllerAdvice` | 여러 Controller의 예외를 전역 처리 |
| ErrorResponse | 공통 에러 응답 구조 |
| ErrorCode | HTTP 상태 코드와 에러 정보 관리 |
| BusinessException | 비즈니스 예외의 공통 부모 |
| Validation Handler | 입력값 검증 실패 처리 |

전체적인 흐름은 다음과 같다.

```text
HTTP Request
      ↓
Controller
      ↓
Service
      ↓
Exception 발생
      ↓
GlobalExceptionHandler
      ↓
Exception → ErrorCode
      ↓
ErrorResponse 생성
      ↓
HTTP Status 결정
      ↓
클라이언트 응답
```

실무에서는 **비즈니스 예외, Validation 예외, 예상하지 못한 시스템 예외를 구분해서 처리하고, 모든 에러 응답을 일관된 형식으로 관리하는 구조**가 중요하다.

## 참고 문서

- [Spring MVC 오류 응답과 ResponseEntityExceptionHandler](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
