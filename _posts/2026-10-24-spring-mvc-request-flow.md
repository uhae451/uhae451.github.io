---
title: "Spring 요청 처리 흐름"
date: 2026-10-24 09:00:00 +0900
categories: ["Spring", "MVC"]
tags: ["spring", "spring-mvc", "dispatcher-servlet", "controller", "http"]
description: "Spring MVC에서 HTTP 요청이 DispatcherServlet을 거쳐 컨트롤러에 전달되고 응답되는 과정을 정리한다."
toc: true
---

Spring MVC에서는 클라이언트의 HTTP 요청이 바로 Controller로 전달되지 않는다.

먼저 `DispatcherServlet`이 요청을 받고, `HandlerMapping`을 통해 어떤 Controller 메서드가 요청을 처리할지 찾는다.

이 글에서는 **HTTP 요청이 Controller까지 도달하고, Controller가 요청 데이터를 받는 과정**을 중심으로 정리한다.

Servlet Filter(예: Spring Security)는 `DispatcherServlet`보다 먼저 실행될 수 있다. 여기서 “먼저”는 MVC 내부의 컨트롤러 호출 흐름을 기준으로 한다.

Spring MVC 내부에는 `HandlerAdapter` 등 추가적인 구성 요소도 존재하지만, 여기서는 전체 요청 흐름을 이해하는 데 필요한 핵심 구성 요소만 중심으로 단순화해서 설명한다.

---

## 1. Spring MVC란?

Spring MVC는 **HTTP 요청을 적절한 Controller에 연결하고 처리 결과를 HTTP 응답으로 반환하는 Spring의 웹 MVC 구조**다.

예를 들어 클라이언트가 다음 요청을 보낸다고 가정한다.

```text
GET /orders/1
```

Controller에 도달하기까지의 기본 흐름은 다음과 같다.

```text
1. 클라이언트 요청

2. DispatcherServlet

3. HandlerMapping

4. Controller
```

중요한 점은 **Controller가 HTTP 요청을 가장 먼저 받는 것이 아니라는 것**이다.

---

## 2. DispatcherServlet

`DispatcherServlet`은 Spring MVC에서 **HTTP 요청을 가장 먼저 받는 Front Controller**다.

클라이언트가 다음 요청을 보내면:

```text
GET /orders/1
```

먼저 `DispatcherServlet`이 요청을 받는다.

하지만 `DispatcherServlet`이 주문 조회 같은 비즈니스 로직을 직접 처리하지는 않는다.

주요 역할은 다음과 같다.

| 역할 | 설명 |
|---|---|
| 요청 수신 | HTTP 요청을 가장 먼저 받음 |
| Handler 탐색 | 요청을 처리할 Controller 메서드를 찾도록 연결 |
| Controller 실행 | 찾아낸 Controller 메서드가 실행되도록 처리 |
| 응답 처리 | Controller의 처리 결과를 HTTP 응답으로 연결 |

즉, `DispatcherServlet`은 **Spring MVC 요청 처리의 중심 역할**을 담당한다.

---

## 3. HandlerMapping

`DispatcherServlet`이 요청을 받으면 다음으로 **어떤 Controller 메서드를 실행해야 하는지** 결정해야 한다.

이 역할을 `HandlerMapping`이 담당한다.

예를 들어 다음과 같은 요청 매핑이 등록되어 있다고 가정한다.

```java
@RequestMapping("/orders")
```

```java
@GetMapping("/{id}")
```

두 설정을 합치면 다음 요청을 처리하는 매핑이 된다.

```text
GET /orders/{id}
```

실제로 다음 요청이 들어오면:

```text
GET /orders/1
```

`HandlerMapping`은 요청의 HTTP Method와 URL을 확인한다.

| 확인 정보 | 값 |
|---|---|
| HTTP Method | `GET` |
| 요청 경로 | `/orders/1` |

그리고 등록된 매핑 중:

```text
GET /orders/{id}
```

와 일치하는 Controller 메서드를 찾는다.

예를 들어 해당 메서드가:

```java
OrderController.findOrder()
```

라면 이 메서드를 요청 처리 대상으로 결정한다.

즉, `HandlerMapping`은 **요청과 등록된 매핑 정보를 비교해서 실행할 Controller 메서드를 결정하는 역할**을 한다.

---

## 4. Controller

Controller는 **HTTP 요청에서 필요한 데이터를 받아 Service에 처리를 요청하고 결과를 반환하는 계층**이다.

주문 관련 Controller를 다음과 같이 작성할 수 있다.

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    // GET /orders/{id}
    @GetMapping("/{id}")
    public OrderResponse findOrder(@PathVariable Long id) {
        return orderService.findOrder(id);
    }

    // GET /orders?status=PAID
    @GetMapping
    public List<OrderResponse> findOrders(@RequestParam String status) {
        return orderService.findOrders(status);
    }

    // POST /orders
    @PostMapping
    public void createOrder(@RequestBody CreateOrderRequest request) {
        orderService.createOrder(request);
    }
}
```

Controller는 직접 DB를 조회하거나 모든 비즈니스 로직을 처리하지 않는다.

실제 작업은 Service에 위임한다.

```java
orderService.findOrder(id);
```

```java
orderService.findOrders(status);
```

```java
orderService.createOrder(request);
```

Controller의 역할은 다음과 같이 정리할 수 있다.

| 역할 | 설명 |
|---|---|
| 요청 매핑 | 어떤 HTTP 요청을 처리할지 지정 |
| 요청 데이터 수신 | URL, Query Parameter, Body 등의 데이터를 받음 |
| Service 호출 | 실제 비즈니스 로직을 Service에 위임 |
| 응답 반환 | 처리 결과를 클라이언트에 반환 |

---

## 5. `@RestController`

REST API Controller에는 주로 `@RestController`를 사용한다.

```java
@RestController
public class OrderController {
}
```

`@RestController`는 개념적으로 다음 두 어노테이션을 함께 사용하는 것과 비슷하다.

```java
@Controller
@ResponseBody
```

`@Controller`는 해당 클래스를 Controller로 사용하도록 하고, `@ResponseBody`는 메서드의 반환값을 HTTP Response Body에 담아 반환하도록 한다.

따라서 Spring Boot에서 REST API를 만들 때 주로 `@RestController`를 사용한다.

---

## 6. 요청 매핑

Controller가 어떤 HTTP 요청을 처리할지는 Mapping 어노테이션을 통해 지정한다.

### `@RequestMapping`

클래스 수준에서 기본 URL을 지정할 수 있다.

```java
@RequestMapping("/orders")
```

이 Controller의 기본 경로는:

```text
/orders
```

가 된다.

메서드에 추가적인 Mapping을 지정하면 클래스의 경로와 합쳐진다.

예를 들어:

```java
@GetMapping("/{id}")
```

가 함께 있으면 최종 요청은:

```text
GET /orders/{id}
```

가 된다.

### HTTP Method별 Mapping

| 어노테이션 | HTTP Method | 주 용도 |
|---|---|---|
| `@GetMapping` | GET | 조회 |
| `@PostMapping` | POST | 생성 |
| `@PutMapping` | PUT | 전체 수정 |
| `@PatchMapping` | PATCH | 일부 수정 |
| `@DeleteMapping` | DELETE | 삭제 |

---

## 7. Controller에서 요청 데이터 받기

HTTP 요청의 데이터가 어디에 들어있는지에 따라 사용하는 어노테이션이 달라진다.

### `@PathVariable`

URL 경로에 포함된 값을 받을 때 사용한다.

요청:

```text
GET /orders/1
```

Controller:

```java
@GetMapping("/{id}")
public OrderResponse findOrder(@PathVariable Long id) {
    return orderService.findOrder(id);
}
```

`{id}` 위치의 `1`이 다음 변수에 전달된다.

```java
Long id
```

개념적으로는:

```java
findOrder(1L);
```

와 같이 호출되는 것이다.

---

### `@RequestParam`

Query Parameter를 받을 때 사용한다.

요청:

```text
GET /orders?status=PAID
```

Controller:

```java
@GetMapping
public List<OrderResponse> findOrders(@RequestParam String status) {
    return orderService.findOrders(status);
}
```

`status=PAID`의 값이:

```java
String status
```

에 전달된다.

따라서 `status`의 값은:

```text
PAID
```

가 된다.

---

### `@RequestBody`

HTTP Request Body의 데이터를 Java 객체로 받을 때 사용한다.

요청:

```text
POST /orders
Content-Type: application/json
```

Request Body:

```json
{
  "productId": 10,
  "quantity": 2
}
```

요청을 받을 DTO를 정의한다.

```java
public record CreateOrderRequest(Long productId,Integer quantity) {
}
```

Controller에서는 다음과 같이 받는다.

```java
@PostMapping
public void createOrder(@RequestBody CreateOrderRequest request) {
    orderService.createOrder(request);
}
```

Spring은 JSON 데이터를 `CreateOrderRequest` 객체로 변환해서 전달한다.

개념적으로는:

```java
new CreateOrderRequest(10L,2);
```

와 같은 객체가 생성되어 Controller에 전달된다고 볼 수 있다.

JSON을 Java 객체로 변환하는 구체적인 과정은 이후 `HttpMessageConverter`에서 별도로 다룬다.

---

## 8. 요청 데이터 전달 방식 정리

세 가지 방식의 차이를 정리하면 다음과 같다.

| 데이터 위치 | 요청 예시 | 사용 어노테이션 |
|---|---|---|
| URL Path | `GET /orders/1` | `@PathVariable` |
| Query Parameter | `GET /orders?status=PAID` | `@RequestParam` |
| Request Body | `POST /orders` + JSON | `@RequestBody` |

`@PathVariable`은 특정 리소스를 식별할 때 주로 사용한다.

```text
/orders/1
/orders/10
/orders/100
```

`@RequestParam`은 검색이나 필터 조건 등에 주로 사용한다.

```text
/orders?status=PAID
/orders?status=CANCELED
```

`@RequestBody`는 생성이나 수정 요청에서 JSON과 같은 데이터를 전달할 때 주로 사용한다.

---

## 9. 전체 요청 흐름

마지막으로 다음 요청을 기준으로 Controller까지의 흐름을 정리한다.

```text
GET /orders/1
```

등록된 Controller 메서드는 다음과 같다.

```java
@GetMapping("/{id}")
public OrderResponse findOrder(@PathVariable Long id) {
    return orderService.findOrder(id);
}
```

처리 과정은 다음과 같다.

| 순서 | 처리 |
|---|---|
| 1 | 클라이언트가 `GET /orders/1` 요청 |
| 2 | `DispatcherServlet`이 요청을 받음 |
| 3 | `HandlerMapping`이 HTTP Method와 URL을 확인 |
| 4 | `OrderController.findOrder()`를 처리 대상으로 선택      |
| 5 | URL의 `1`을 `id`에 전달 |
| 6 | `OrderController.findOrder(1L)` 실행 |
| 7 | Controller가 `OrderService.findOrder(1L)` 호출 |

이후 과정은 Service → Repository → Database를 거쳐 처리된다.

---

## 정리

Spring MVC에서는 HTTP 요청이 Controller로 바로 전달되지 않는다.

`DispatcherServlet`이 먼저 요청을 받고, `HandlerMapping`이 해당 요청을 처리할 Controller 메서드를 찾아준다.

Controller에서는 요청 데이터의 위치에 따라 다음 어노테이션을 사용한다.

| 어노테이션 | 역할 |
|---|---|
| `@PathVariable` | URL 경로 값 전달 |
| `@RequestParam` | Query Parameter 전달 |
| `@RequestBody` | Request Body 데이터 전달 |

전체 흐름을 간단히 정리하면 다음과 같다.

```text
1. 클라이언트

2. DispatcherServlet

3. HandlerMapping

4. Controller

5. Service
```

즉, **`DispatcherServlet`은 요청 처리 흐름을 관리하고, `HandlerMapping`은 요청을 처리할 Controller 메서드를 찾으며, Controller는 전달받은 요청 데이터를 이용해 Service에 실제 처리를 요청한다.**
