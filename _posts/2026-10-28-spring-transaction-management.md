---
title: "@Transactional과 트랜잭션 관리"
date: 2026-10-28 09:00:00 +0900
categories: ["Spring", "Transaction"]
tags: ["spring", "transaction", "transactional", "rollback", "aop"]
description: "Spring의 @Transactional이 트랜잭션을 관리하는 방식과 커밋·롤백, readOnly 및 프록시 관련 주의사항을 정리한다."
toc: true
---

Spring에서 데이터베이스 작업을 처리할 때 여러 작업을 하나의 단위로 묶어야 하는 경우가 있다.

예를 들어 주문을 생성하면서 주문 정보와 결제 정보를 함께 저장한다고 가정한다.

```java
orderRepository.save(order);
paymentRepository.save(payment);
```

두 작업이 모두 성공하면 문제가 없지만, 주문 저장은 성공하고 결제 저장에서 오류가 발생하면 데이터가 불완전한 상태로 남을 수 있다.

이러한 문제를 방지하기 위해 사용하는 개념이 **트랜잭션(Transaction)** 이다.

---

## 1. 트랜잭션이란?

트랜잭션은 **여러 데이터베이스 작업을 하나의 작업 단위로 묶어 처리하는 것**을 의미한다.

예를 들어 다음 두 작업이 하나의 주문 처리 과정이라고 가정한다.

```text
주문 저장
결제 정보 저장
```

두 작업이 모두 성공하면 변경 내용을 최종적으로 데이터베이스에 반영한다.

이를 **Commit**이라고 한다.

반대로 작업 중 하나라도 실패하면 이전에 수행했던 작업까지 취소하여 트랜잭션 시작 전 상태로 되돌린다.

이를 **Rollback**이라고 한다.

| 결과 | 처리 |
|---|---|
| 모든 작업 성공 | Commit |
| 작업 중 오류 발생 | Rollback |

따라서 트랜잭션의 핵심은 **하나의 작업 단위에 포함된 데이터 변경을 모두 성공시키거나 모두 실패시키는 것**이다.

---

## 2. Commit과 Rollback

### Commit

트랜잭션 내의 작업이 모두 정상적으로 완료되면 변경 내용을 데이터베이스에 최종 반영한다.

```text
트랜잭션 시작

주문 저장 성공
결제 저장 성공

Commit

데이터베이스 반영
```

### Rollback

트랜잭션 처리 중 문제가 발생하면 트랜잭션 안에서 수행했던 작업을 취소한다.

```text
트랜잭션 시작

주문 저장 성공
결제 저장 실패

Rollback

주문 저장도 취소
```

Rollback을 통해 일부 데이터만 저장되는 것을 방지하고 데이터의 일관성을 유지할 수 있다.

---

## 3. Spring의 `@Transactional`

Spring에서는 `@Transactional`을 사용하여 트랜잭션을 선언적으로 적용할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;

    @Transactional
    public void createOrder() {

        orderRepository.save(order);
        paymentRepository.save(payment);
    }
}
```

`createOrder()`가 트랜잭션 범위가 된다.

주요 동작은 다음과 같다.

| 순서 | 처리 |
|---|---|
| 1 | 트랜잭션 시작 |
| 2 | `orderRepository.save()` 실행 |
| 3 | `paymentRepository.save()` 실행 |
| 4 | 정상적으로 종료되면 Commit |
| 5 | Rollback 대상 예외가 발생하면 Rollback |

개발자가 직접 Connection을 가져와 `commit()`이나 `rollback()`을 호출하지 않아도 Spring이 트랜잭션 처리를 담당한다.

---

## 4. `@Transactional`과 Spring AOP

Spring의 `@Transactional`은 **Proxy 기반의 AOP 구조를 이용하여 동작한다.**

예를 들어 외부에서 다음 메서드를 호출한다고 가정한다.

```java
orderService.createOrder();
```

Spring에서는 실제 `OrderService` 객체를 바로 호출하는 것이 아니라 트랜잭션 기능이 적용된 Proxy가 먼저 호출을 받는다.

| 순서 | 동작 |
|---|---|
| 1 | 외부에서 `createOrder()` 호출 |
| 2 | Proxy가 호출을 받음 |
| 3 | 트랜잭션 시작 |
| 4 | 실제 `OrderService.createOrder()` 실행 |
| 5 | 정상 종료 시 Commit |
| 6 | Rollback 대상 예외 발생 시 Rollback |

즉, 개발자가 작성한 비즈니스 로직에 트랜잭션 코드를 직접 추가하지 않아도 Proxy가 메서드 실행 전후에 필요한 트랜잭션 처리를 추가한다.

---

## 5. Rollback은 언제 발생하는가?

Spring의 기본 트랜잭션 설정에서는 일반적으로 **`RuntimeException`과 `Error`가 발생했을 때 Rollback**한다.

```java
@Transactional
public void createOrder() {

    orderRepository.save(order);

    throw new RuntimeException("주문 처리 실패");
}
```

`RuntimeException`이 발생하면서 메서드가 정상적으로 종료되지 않았기 때문에 트랜잭션은 Rollback된다.

따라서 앞에서 실행한 `orderRepository.save(order)` 역시 최종적으로 반영되지 않는다.

반면 Checked Exception은 기본 Rollback 대상이 아니다.

필요한 경우 `rollbackFor`를 이용해 Rollback 대상을 직접 지정할 수 있다.

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws Exception {
}
```

이 경우 `Exception` 계열의 예외도 Rollback 대상으로 지정한다.

---

## 6. 예외를 내부에서 처리할 때 주의점

트랜잭션 메서드 내부에서 발생한 예외를 `catch`하고 정상적으로 메서드를 종료시키면 트랜잭션 처리 결과가 의도와 달라질 수 있다.

```java
@Transactional
public void createOrder() {

    orderRepository.save(order);

    try {
        paymentRepository.save(payment);
    } catch (RuntimeException e) {
        System.out.println("결제 저장 실패");
    }
}
```

`paymentRepository.save()`에서 예외가 발생했지만 메서드 내부에서 처리하여 외부로 전달하지 않았다.

바깥 Service의 Proxy는 정상 반환을 보고 Commit을 시도한다. 그러나 위 예제의 Spring Data Repository도 트랜잭션에 참여하므로, 그 경계를 통과한 RuntimeException이 이미 rollback-only를 설정했다면 Commit되지 않고 `UnexpectedRollbackException`이 발생한다. 반면 트랜잭션 경계를 통과하지 않은 일반 로직의 예외를 내부에서 잡고 rollback-only도 설정되지 않았다면 Commit될 수 있다.

Rollback이 필요한 예외라면 예외를 다시 외부로 전달해야 한다.

```java
@Transactional
public void createOrder() {

    try {
        orderRepository.save(order);
        paymentRepository.save(payment);
    } catch (RuntimeException e) {
        throw e;
    }
}
```

따라서 트랜잭션에서는 **예외를 어디에서 처리하고 외부로 전달하는지**도 중요하다.

---

## 7. `@Transactional`은 어디에 적용하는가?

일반적으로 트랜잭션은 **Service 계층의 비즈니스 로직 단위**에 적용한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentRepository paymentRepository;

    @Transactional
    public void createOrder() {

        orderRepository.save(order);
        paymentRepository.save(payment);
    }
}
```

이 코드에서 중요한 것은 Repository 하나의 작업이 아니라:

```text
주문 저장 + 결제 저장
```

전체가 하나의 비즈니스 작업이라는 점이다.

따라서 두 작업을 하나의 트랜잭션으로 묶는다.

트랜잭션 경계는 기술적인 Repository 호출 하나하나보다는 **하나의 비즈니스 작업이 어디에서 시작되고 끝나는지**를 기준으로 설정하는 것이 일반적이다.

---

## 8. 트랜잭션 범위

트랜잭션은 필요한 범위에서 사용하되 지나치게 오래 유지되지 않도록 주의해야 한다.

예를 들어:

```java
@Transactional
public void createOrder() {

    orderRepository.save(order);

    externalPaymentApi.call();

    sendEmail();

    uploadFile();
}
```

DB 작업뿐만 아니라 외부 API 호출, 이메일 전송, 파일 업로드처럼 시간이 오래 걸릴 수 있는 작업까지 하나의 트랜잭션 안에 포함되어 있다.

이 경우 트랜잭션이 오랫동안 유지되면서 DB Connection 등의 리소스를 오래 점유할 수 있다.

따라서 트랜잭션은 일반적으로:

> 필요한 데이터베이스 작업을 하나의 일관된 작업 단위로 묶되, 트랜잭션 범위는 가능한 명확하고 짧게 유지한다.

는 기준으로 설계하는 것이 중요하다.

---

## 9. 조회 작업과 `readOnly = true`

데이터를 변경하지 않고 조회만 수행하는 메서드에서는 다음과 같이 사용할 수 있다.

```java
@Transactional(readOnly = true)
public Order findOrder(Long id) {

    return orderRepository.findById(id)
            .orElseThrow();
}
```

`readOnly = true`는 해당 트랜잭션이 **조회 전용이라는 의도를 나타낸다.**

단순히 데이터를 수정하지 않겠다는 표시만을 의미하는 것은 아니다.

또한 여러 조회를 하나의 트랜잭션 범위에서 처리하거나 영속성 컨텍스트를 유지해야 하는 경우에도 사용할 수 있다.

예를 들어:

```java
@Transactional(readOnly = true)
public OrderDetail getOrderDetail(Long orderId) {

    Order order = orderRepository.findById(orderId)
            .orElseThrow();

    User user = userRepository.findById(order.getUserId())
            .orElseThrow();

    return new OrderDetail(order, user);
}
```

여러 조회 작업을 하나의 조회 비즈니스 로직 범위로 관리할 수 있다.

---

## 10. `readOnly = true`는 쓰기를 완전히 차단하는 기능이 아니다

주의해야 할 점은 다음 코드가:

```java
@Transactional(readOnly = true)
```

데이터베이스 변경을 절대 불가능하게 만드는 보안 기능은 아니라는 것이다.

`readOnly`는 기본적으로 Spring과 JPA Provider 등에:

> 이 트랜잭션은 조회를 목적으로 사용한다.

는 의도와 최적화 정보를 전달하는 설정으로 보는 것이 적절하다.

따라서 다음과 같이 구분할 수 있다.

| 설정 | 목적 |
|---|---|
| `@Transactional` | 데이터 변경을 포함한 일반적인 트랜잭션 |
| `@Transactional(readOnly = true)` | 조회 중심의 트랜잭션 |

---

## 11. 클래스 전체에 `readOnly` 적용하기

Service 클래스의 대부분이 조회 기능이라면 클래스 전체에 조회용 트랜잭션을 지정할 수도 있다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderService {

    private final OrderRepository orderRepository;

    public Order findOrder(Long id) {
        return orderRepository.findById(id)
                .orElseThrow();
    }

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}
```

클래스 수준에서는:

```java
@Transactional(readOnly = true)
```

를 적용하여 기본적으로 조회용 트랜잭션을 사용한다.

데이터를 변경하는 `createOrder()`에는 다시:

```java
@Transactional
```

을 선언한다.

이 경우 메서드 수준의 설정이 클래스 수준의 설정보다 우선 적용된다.

---

## 12. Self Invocation 문제

`@Transactional`은 Proxy 기반으로 동작하기 때문에 같은 클래스 내부에서 트랜잭션 메서드를 직접 호출할 때 주의해야 한다.

```java
@Service
public class OrderService {

    public void order() {
        payment();
    }

    @Transactional
    public void payment() {
        // DB 작업
    }
}
```

외부에서는 다음 메서드를 호출한다.

```java
orderService.order();
```

호출 구조는 개념적으로 다음과 같다.

| 단계 | 호출 |
|---|---|
| 1 | 외부에서 `order()` 호출 |
| 2 | Proxy를 통해 실제 `order()` 실행 |
| 3 | `order()` 내부에서 `payment()` 직접 호출 |

문제는:

```java
payment();
```

호출이 다시 Proxy를 거치지 않는다는 점이다.

같은 객체 내부에서 직접 메서드가 호출되기 때문에 `payment()`에 선언된 `@Transactional`이 기대한 방식으로 적용되지 않을 수 있다.

이를 **Self Invocation 문제**라고 한다.

따라서 다음 내용을 기억할 필요가 있다.

> Spring의 기본적인 Proxy 기반 `@Transactional`은 Proxy를 거쳐 메서드가 호출되어야 적용된다.

---

## 13. 트랜잭션의 ACID

데이터베이스 트랜잭션에서는 ACID라는 네 가지 특성을 자주 사용한다.

| 특성 | 의미 | 설명 |
|---|---|---|
| Atomicity | 원자성 | 모든 작업이 성공하거나 모두 실패해야 함 |
| Consistency | 일관성 | 트랜잭션 전후 데이터 규칙이 유지되어야 함 |
| Isolation | 격리성 | 동시에 실행되는 트랜잭션이 적절하게 격리되어야 함 |
| Durability | 지속성 | Commit된 결과는 지속적으로 보존되어야 함 |

### Atomicity

하나의 트랜잭션에 포함된 작업은 전체가 하나의 단위로 처리되어야 한다.

```text
주문 저장 성공
결제 저장 성공
```

또는:

```text
주문 저장 취소
결제 저장 취소
```

와 같이 처리되어야 한다.

일부 작업만 반영되는 상태를 방지하는 특성이다.

### Consistency

트랜잭션이 수행되기 전과 후에 데이터베이스가 정의된 규칙을 만족하는 상태를 유지해야 한다.

### Isolation

여러 트랜잭션이 동시에 실행되더라도 서로의 작업이 부적절하게 영향을 주지 않도록 격리해야 한다.

### Durability

정상적으로 Commit된 데이터는 이후 시스템 문제가 발생하더라도 보존되어야 한다.

---

## 14. 주문 처리 예제로 보는 트랜잭션

상품 주문 과정에서 재고 감소와 주문 저장이 함께 수행된다고 가정한다.

```java
@Transactional
public void order(Long productId) {

    Product product =
            productRepository.findById(productId)
                    .orElseThrow();

    product.decreaseStock();

    Order order = new Order(product);

    orderRepository.save(order);
}
```

비즈니스 로직은 다음 작업으로 구성된다.

| 순서 | 작업 |
|---|---|
| 1 | 상품 조회 |
| 2 | 상품 재고 감소 |
| 3 | 주문 객체 생성 |
| 4 | 주문 저장 |

재고 감소가 정상적으로 처리되었지만 주문 저장 과정에서 예외가 발생했다고 가정한다.

트랜잭션이 없다면 재고만 감소하고 주문은 존재하지 않는 문제가 발생할 수 있다.

트랜잭션으로 하나의 작업 단위를 구성하면:

```text
재고 감소
주문 저장 실패
Rollback
재고 감소도 취소
```

와 같이 처리할 수 있다.

따라서 트랜잭션은 여러 데이터 변경이 하나의 비즈니스 작업으로 연결되어 있을 때 데이터의 일관성을 유지하는 데 중요한 역할을 한다.

---

## 정리

트랜잭션은 **여러 데이터베이스 작업을 하나의 작업 단위로 묶어 처리하는 개념**이다.

모든 작업이 정상적으로 완료되면 Commit하고, Rollback 대상 예외가 발생하면 작업을 취소한다.

Spring에서는 `@Transactional`을 통해 이러한 트랜잭션을 선언적으로 적용할 수 있으며, 내부적으로 Proxy 기반 AOP 구조를 이용해 트랜잭션 시작과 종료를 처리한다.

핵심 개념을 정리하면 다음과 같다.

| 개념 | 설명 |
|---|---|
| Transaction | 여러 DB 작업을 하나의 단위로 처리 |
| Commit | 정상 처리된 변경 내용을 최종 반영 |
| Rollback | 문제가 발생한 작업을 이전 상태로 복구 |
| `@Transactional` | Spring에서 트랜잭션을 선언적으로 적용 |
| `readOnly = true` | 조회 전용 트랜잭션임을 명시 |
| Proxy | 실제 메서드 실행 전후에 트랜잭션을 관리 |
| Self Invocation | 내부 메서드 호출로 Proxy를 거치지 않는 문제 |
| ACID | 트랜잭션이 가져야 하는 주요 특성 |

`@Transactional`을 사용할 때는 단순히 어노테이션을 붙이는 것보다 **어떤 비즈니스 작업들을 하나의 트랜잭션으로 묶어야 하는지**, **어디에서 트랜잭션이 시작되고 끝나야 하는지**를 결정하는 것이 중요하다.
