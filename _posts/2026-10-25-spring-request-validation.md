---
title: "Spring 요청 데이터 검증"
date: 2026-10-25 09:00:00 +0900
categories: ["Spring", "Validation"]
tags: ["spring", "bean-validation", "valid", "validation", "request-dto"]
description: "Spring에서 Bean Validation과 @Valid를 사용해 요청 데이터를 검증하고 오류를 처리하는 흐름을 정리한다."
toc: true
---

Spring에서는 클라이언트가 전달한 요청 데이터를 Controller에서 바로 사용하기 전에 검증할 수 있다.

실무에서는 단순히 `null` 여부만 확인하는 수준이 아니라, **입력 형식 검증과 비즈니스 검증을 분리하고, 검증 실패 응답 형식까지 일관되게 관리하는 구조**가 중요하다.

예를 들어 주문 생성 API가 다음과 같다고 가정한다.

```http
POST /orders
Content-Type: application/json
```

```json
{
  "productId": 10,
  "quantity": 2,
  "deliveryAddress": {
    "zipCode": "06236",
    "address1": "서울특별시 강남구 테헤란로 123",
    "address2": "101동 1201호"
  }
}
```

주문 생성 요청에는 다음과 같은 검증 조건이 필요할 수 있다.

| 필드                | 검증 조건    |
| ----------------- | -------- |
| `productId`       | 필수       |
| `quantity`        | 필수, 1 이상 |
| `deliveryAddress` | 필수       |
| `zipCode`         | 필수       |
| `address1`        | 필수       |
| `address2`        | 선택       |

---

검증 구현체를 사용하려면 Spring Boot 프로젝트에 `spring-boot-starter-validation` 의존성을 추가한다. 이 글의 예제는 `jakarta.validation` 패키지를 사용한다.

## 1. Bean Validation

Bean Validation은 Java 객체에 제약 조건을 선언하고, 해당 조건을 만족하는지 검증할 수 있도록 제공하는 표준이다.

Spring에서는 일반적으로 Request DTO에 검증 어노테이션을 선언하고 Controller에서 `@Valid`를 사용한다.

예를 들어 주문 생성 DTO를 다음과 같이 정의할 수 있다.

```java
public record CreateOrderRequest(

        @NotNull(message = "상품 ID는 필수입니다.")
        Long productId,

        @NotNull(message = "수량은 필수입니다.")
        @Positive(message = "수량은 1 이상이어야 합니다.")
        Integer quantity,

        @NotNull(message = "배송지 정보는 필수입니다.")
        @Valid
        DeliveryAddressRequest deliveryAddress
) {
}
```

배송지 DTO는 별도로 분리한다.

```java
public record DeliveryAddressRequest(

        @NotBlank(message = "우편번호는 필수입니다.")
        String zipCode,

        @NotBlank(message = "기본 주소는 필수입니다.")
        @Size(max = 100, message = "기본 주소는 100자 이하여야 합니다.")
        String address1,

        @Size(max = 100, message = "상세 주소는 100자 이하여야 합니다.")
        String address2
) {
}
```

이 구조를 사용하면 DTO 자체에 요청 데이터가 만족해야 하는 조건을 명확하게 표현할 수 있다.

---

## 2. `@Valid`

DTO에 검증 어노테이션을 선언한 뒤에는 Controller에서 검증을 실행해야 한다.

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    // POST /orders
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request
    ) {
        OrderResponse response = orderService.createOrder(request);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(response);
    }
}
```

여기서:

```java
@Valid @RequestBody CreateOrderRequest request
```

는 두 가지 작업과 연결된다.

| 어노테이션          | 역할                                   |
| -------------- | ------------------------------------ |
| `@RequestBody` | HTTP Request Body의 JSON을 Java 객체로 변환 |
| `@Valid`       | 변환된 객체에 선언된 검증 조건 실행                 |

즉, JSON을 DTO로 변환한 뒤 검증을 수행한다.

---

## 3. 요청 검증 처리 순서

정상적인 요청이 들어온다고 가정한다.

```json
{
  "productId": 10,
  "quantity": 2,
  "deliveryAddress": {
    "zipCode": "06236",
    "address1": "서울특별시 강남구 테헤란로 123",
    "address2": "101동 1201호"
  }
}
```

처리 과정은 다음과 같다.

| 순서 | 처리                                     |
| -- | -------------------------------------- |
| 1  | 클라이언트가 JSON 요청 전송                      |
| 2  | Spring이 JSON을 `CreateOrderRequest`로 변환 |
| 3  | `@Valid`가 DTO의 제약 조건 검사                |
| 4  | 중첩 DTO의 `@Valid` 검증 수행                 |
| 5  | 모든 검증을 통과하면 Controller 메서드 실행          |
| 6  | Controller가 Service 호출                 |

구조적으로 보면 다음과 같다.

```text
1. HTTP Request

2. JSON

3. CreateOrderRequest

4. Bean Validation

5. 검증 성공

6. Controller

7. Service
```

---

## 4. 검증 실패

다음과 같은 잘못된 요청이 들어온다고 가정한다.

```json
{
  "productId": null,
  "quantity": 0,
  "deliveryAddress": {
    "zipCode": "",
    "address1": "",
    "address2": "101동 1201호"
  }
}
```

DTO의 검증 조건은 다음과 같다.

```java
@NotNull
Long productId;

@NotNull
@Positive
Integer quantity;
```

배송지에는 다음 조건이 있다.

```java
@NotBlank
String zipCode;

@NotBlank
String address1;
```

따라서 다음 문제가 발생한다.

| 필드          | 값      | 실패 조건       |
| ----------- | ------ | ----------- |
| `productId` | `null` | `@NotNull`  |
| `quantity`  | `0`    | `@Positive` |
| `zipCode`   | `""`   | `@NotBlank` |
| `address1`  | `""`   | `@NotBlank` |

이 경우 Controller 메서드는 정상적으로 실행되지 않는다.

```java
orderService.createOrder(request);
```

까지 도달하지 않는다.

---

## 5. 주요 검증 어노테이션

실무에서 자주 사용하는 검증 어노테이션은 다음과 같다.

| 어노테이션             | 검증 조건                          |
| ----------------- | ------------------------------ |
| `@NotNull`        | `null` 금지                      |
| `@NotEmpty`       | `null`, 빈 문자열, 빈 Collection 금지 |
| `@NotBlank`       | `null`, 빈 문자열, 공백 문자열 금지       |
| `@Size`           | 문자열, 배열, Collection 등의 크기 검사   |
| `@Min`            | 최소 숫자값                         |
| `@Max`            | 최대 숫자값                         |
| `@Positive`       | 0보다 큰 값                        |
| `@PositiveOrZero` | 0 이상의 값                        |
| `@Negative`       | 0보다 작은 값                       |
| `@Email`          | 이메일 형식 검사                      |
| `@Pattern`        | 정규식 검사                         |

---

## 6. `@NotNull`, `@NotEmpty`, `@NotBlank`

문자열 검증에서 자주 혼동되는 부분이다.

### `@NotNull`

```java
@NotNull
String name;
```

`null`만 금지한다.

```text
null  → 실패
""    → 성공
"   " → 성공
```

---

### `@NotEmpty`

```java
@NotEmpty
String name;
```

`null`과 빈 문자열을 금지한다.

```text
null  → 실패
""    → 실패
"   " → 성공
```

---

### `@NotBlank`

```java
@NotBlank
String name;
```

문자열의 `null`, 빈 문자열, 공백 문자열을 모두 금지한다.

```text
null    → 실패
""      → 실패
"   "   → 실패
"order" → 성공
```

이름, 주소, 제목처럼 실제 문자열 입력값에는 `@NotBlank`가 자주 사용된다.

---

## 7. 숫자 검증

주문 수량은 필수이고 1 이상이어야 한다고 가정한다.

```java
@NotNull(message = "수량은 필수입니다.")
@Positive(message = "수량은 1 이상이어야 합니다.")
Integer quantity;
```

결과는 다음과 같다.

|      값 | 결과 |
| -- | - |
| `null` | 실패 |
|    `0` | 실패 |
|   `-1` | 실패 |
|    `1` | 성공 |
|   `10` | 성공 |

여기서 중요한 점은 `@Positive`만으로는 필수값을 완전히 표현하지 못한다는 것이다.

따라서 필수값이라면 일반적으로:

```java
@NotNull
@Positive
Integer quantity;
```

처럼 조합해서 사용한다.

---

## 8. 중첩 DTO 검증

실무에서는 DTO 안에 또 다른 DTO가 포함되는 경우가 많다.

예를 들어:

```java
public record CreateOrderRequest(

        @NotNull
        Long productId,

        @NotNull
        @Positive
        Integer quantity,

        @NotNull
        @Valid
        DeliveryAddressRequest deliveryAddress
) {
}
```

여기서 중요한 것은:

```java
@Valid
DeliveryAddressRequest deliveryAddress
```

이다.

`@Valid`가 없으면 내부 객체의 검증 어노테이션이 실행되지 않을 수 있다.

배송지 DTO:

```java
public record DeliveryAddressRequest(

        @NotBlank
        String zipCode,

        @NotBlank
        String address1,

        String address2
) {
}
```

---

## 9. 생성 DTO와 수정 DTO 분리

실무에서는 하나의 DTO를 생성과 수정에 모두 사용하는 것보다, 요청 목적에 따라 DTO를 분리하는 경우가 많다.

예를 들어 주문 생성 DTO는 다음과 같다.

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

반면 수량 수정 요청에서는 `productId`가 필요하지 않을 수 있다.

```java
public record UpdateOrderQuantityRequest(

        @NotNull
        @Positive
        Integer quantity
) {
}
```

Controller도 각각 분리한다.

```java
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody CreateOrderRequest request
) {
    OrderResponse response = orderService.createOrder(request);

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(response);
}
```

```java
@PatchMapping("/{id}/quantity")
public ResponseEntity<OrderResponse> updateQuantity(
        @PathVariable Long id,
        @Valid @RequestBody UpdateOrderQuantityRequest request
) {
    OrderResponse response = orderService.updateQuantity(id, request);

    return ResponseEntity.ok(response);
}
```

요청 목적마다 필요한 데이터와 검증 조건이 다르기 때문에 DTO를 분리하면 구조가 명확해진다.

---

## 10. Request DTO 검증과 비즈니스 검증 분리

실무에서 특히 중요한 부분이다.

다음 조건은 Request DTO에서 검증하기 적합하다.

```text
productId가 null인가?
quantity가 1 이상인가?
address1이 비어 있는가?
문자열 길이가 제한을 초과하는가?
```

이러한 검증은 요청 데이터 자체만 보고 판단할 수 있다.

반면 다음 조건은 DTO만으로 판단할 수 없다.

```text
해당 상품이 실제 존재하는가?
주문 가능한 상품인가?
재고가 충분한가?
이미 취소된 주문인가?
현재 사용자가 이 주문을 수정할 권한이 있는가?
```

이러한 조건은 Service에서 처리한다.

예를 들어:

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
                        new ProductNotFoundException(request.productId())
                );

        if (product.getStock() < request.quantity()) {
            throw new InsufficientStockException();
        }

        Order order = Order.create(
                product,
                request.quantity()
        );

        Order savedOrder = orderRepository.save(order);

        return OrderResponse.from(savedOrder);
    }
}
```

역할을 정리하면 다음과 같다.

| 검증 종류   | 예시                  | 처리 위치       |
| ------- | ------------------- | ----------- |
| 입력값 검증  | null, 공백, 길이, 숫자 범위 | Request DTO |
| 비즈니스 검증 | 상품 존재 여부, 재고, 상태    | Service     |

---

## 11. 검증 실패 응답 통일

실무에서는 Bean Validation의 기본 오류 응답을 그대로 사용하기보다 공통 형식으로 가공하는 경우가 많다.

예를 들어 다음과 같은 응답 구조를 사용할 수 있다.

```json
{
  "code": "INVALID_REQUEST",
  "message": "요청 값이 올바르지 않습니다.",
  "errors": [
    {
      "field": "quantity",
      "message": "수량은 1 이상이어야 합니다."
    },
    {
      "field": "deliveryAddress.address1",
      "message": "기본 주소는 필수입니다."
    }
  ]
}
```

이를 위한 DTO를 다음과 같이 만들 수 있다.

```java
public record ValidationErrorResponse(
        String code,
        String message,
        List<FieldErrorResponse> errors
) {
}
```

```java
public record FieldErrorResponse(
        String field,
        String message
) {
}
```

---

## 12. 전역 검증 예외 처리

`@Valid` 검증에 실패하면 일반적으로 `MethodArgumentNotValidException`이 발생할 수 있다.

이를 `@RestControllerAdvice`에서 처리하면 모든 Controller의 검증 실패 응답을 한곳에서 통일할 수 있다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse>
    handleValidationException(
            MethodArgumentNotValidException e
    ) {

        List<FieldErrorResponse> errors = e.getBindingResult()
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
}
```

이렇게 하면 Controller마다 검증 오류 처리 코드를 작성하지 않아도 된다.

전체 구조는 다음과 같다.

```text
1. HTTP Request

2. Request DTO 변환

3. @Valid 검증

4. 검증 실패

5. MethodArgumentNotValidException

6. @RestControllerAdvice

7. ValidationErrorResponse

8. HTTP 400 Bad Request
```

---

## 13. 전체 실무 구조

주문 생성 API를 기준으로 전체 검증 흐름을 정리한다.

### Request DTO

```java
public record CreateOrderRequest(

        @NotNull(message = "상품 ID는 필수입니다.")
        Long productId,

        @NotNull(message = "수량은 필수입니다.")
        @Positive(message = "수량은 1 이상이어야 합니다.")
        Integer quantity,

        @NotNull(message = "배송지 정보는 필수입니다.")
        @Valid
        DeliveryAddressRequest deliveryAddress
) {
}
```

### Controller

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    // POST /orders
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request
    ) {
        OrderResponse response =
                orderService.createOrder(request);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(response);
    }
}
```

### Service

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
                        new ProductNotFoundException(request.productId())
                );

        if (product.getStock() < request.quantity()) {
            throw new InsufficientStockException();
        }

        Order order = Order.create(
                product,
                request.quantity()
        );

        Order savedOrder = orderRepository.save(order);

        return OrderResponse.from(savedOrder);
    }
}
```

### 전역 검증 예외 처리

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse>
    handleValidationException(
            MethodArgumentNotValidException e
    ) {

        List<FieldErrorResponse> errors = e.getBindingResult()
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
}
```

---

## 정리

Spring에서 요청 데이터 검증은 단순히 `@Valid` 하나를 사용하는 것으로 끝나지 않는다.

실무에서는 다음 역할을 분리해서 구성하는 것이 중요하다.

| 구성 요소                   | 역할                      |
| ----------------------- | ----------------------- |
| Request DTO             | 요청 값의 형식과 범위 검증         |
| `@Valid`                | Bean Validation 실행      |
| 중첩 DTO의 `@Valid`        | 내부 객체까지 검증              |
| Service                 | DB 조회가 필요한 비즈니스 검증      |
| 생성/수정 DTO 분리            | 요청 목적별 검증 조건 분리         |
| `@RestControllerAdvice` | 검증 실패 응답 통일             |
| Error Response DTO      | 클라이언트가 처리하기 쉬운 오류 형식 제공 |

전체 흐름은 다음과 같다.

[요청 수신]
HTTP Request
JSON → Request DTO

[입력값 검증]
Bean Validation
- 검증 실패: GlobalExceptionHandler에서 처리 후 400 Error Response
- 검증 성공: Controller로 요청 전달

[비즈니스 처리]
Controller
Service
비즈니스 검증

[데이터 처리]
Repository
Database

핵심은 **요청 데이터 자체의 형식 검증은 Request DTO에서 처리하고, 데이터베이스 조회나 도메인 상태가 필요한 검증은 Service에서 처리하는 것**이다.

여기에 검증 실패 응답까지 공통 형식으로 관리하면 실제 프로젝트에서도 유지보수하기 좋은 검증 구조를 만들 수 있다.
