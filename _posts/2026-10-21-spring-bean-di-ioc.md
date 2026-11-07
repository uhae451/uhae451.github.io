---
title: "Spring Bean, DI, IoC의 관계"
date: 2026-10-21 09:00:00 +0900
categories: ["Spring", "Core"]
tags: ["spring", "bean", "di", "ioc", "spring-container"]
description: "Spring의 Bean, DI, IoC가 어떻게 연결되어 객체 생성과 의존관계를 관리하는지 정리한다."
toc: true
---

Spring을 공부하다 보면 Bean, DI, IoC라는 개념이 각각 따로 등장한다. 하지만 실제로 Spring이 객체를 관리하는 과정에서는 세 개가 서로 연결되어 하나의 구조처럼 동작한다.

각 개념의 역할을 먼저 살펴본 뒤, 마지막에 세 개가 어떻게 연결되는지 정리한다.

## 1. Bean

Bean은 **Spring Container가 생성하고 관리하는 객체**다.

일반적인 Java에서는 객체가 필요할 때 개발자가 직접 `new` 키워드를 사용해 생성한다.

```java
OrderService orderService = new OrderService();
```

반면 Spring에서는 특정 객체를 Spring이 관리하도록 등록할 수 있다.

예를 들어 다음과 같이 `@Service`를 사용하면 `OrderService`는 Spring이 관리하는 Bean이 된다.

```java
@Service
public class OrderService {
}
```

Spring은 애플리케이션 실행 과정에서 해당 클래스를 찾아 객체를 생성하고 Spring Container 내부에서 관리한다.

대표적으로 다음과 같은 어노테이션이 붙은 클래스들이 Bean 등록 대상이 될 수 있다.

| 어노테이션 | 주로 사용하는 계층 |
|---|---|
| `@Component` | 일반적인 컴포넌트 |
| `@Controller` | Controller 계층 |
| `@Service` | Service 계층 |
| `@Repository` | Repository 계층 |

따라서 Bean의 핵심은 다음과 같다.

**Bean은 개발자가 직접 생성하고 관리하는 객체가 아니라 Spring이 생성하고 관리하는 객체다.**

---

## 2. DI

DI는 **Dependency Injection**, 즉 의존성 주입을 의미한다.

어떤 객체가 다른 객체를 필요로 하는 관계를 의존 관계라고 한다.

예를 들어 `OrderService`가 주문 데이터를 조회하기 위해 `OrderRepository`를 사용한다면 `OrderService`는 `OrderRepository`에 의존한다.

DI를 사용하지 않는다면 다음과 같이 직접 객체를 생성할 수 있다.

```java
public class OrderService {

    private final OrderRepository orderRepository
        = new OrderRepository();
}
```

이 경우 `OrderService`가 `OrderRepository`의 생성까지 직접 담당한다.

Spring에서는 필요한 객체를 직접 생성하지 않고 외부에서 전달받는 방식을 사용할 수 있다.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

여기서 `OrderService`는 `OrderRepository`를 직접 생성하지 않는다.

대신 생성자를 통해 `OrderRepository`를 전달받는다.

이러한 방식을 **생성자 주입**이라고 한다.

그리고 `OrderRepository`가 다음과 같이 Bean으로 등록되어 있다면,

```java
@Repository
public class OrderRepository {
}
```

Spring은 `OrderRepository` Bean을 찾아 `OrderService`를 생성할 때 자동으로 전달한다.

개념적으로는 Spring이 다음과 같은 역할을 대신 수행하는 것이다.

```java
OrderRepository orderRepository = new OrderRepository();

OrderService orderService =
    new OrderService(orderRepository);
```

따라서 DI의 핵심은 다음과 같다.

**객체가 필요한 의존 객체를 직접 생성하지 않고 외부에서 전달받는 방식이다.**

Spring에서는 Spring Container가 Bean을 찾아 필요한 곳에 주입한다.

---

## 3. IoC

IoC는 **Inversion of Control**, 즉 제어의 역전을 의미한다.

일반적인 Java 프로그램에서는 객체의 생성과 연결을 개발자가 직접 제어한다.

```java
OrderRepository repository = new OrderRepository();
OrderService service = new OrderService(repository);
```

어떤 객체를 생성할지, 언제 생성할지, 어떤 객체를 전달할지를 개발자가 직접 결정한다.

Spring을 사용하면 이 역할을 Spring Container가 담당한다.

개발자는 클래스를 정의하고 필요한 의존성을 선언하며, 실제 객체 생성과 관리 및 연결은 Spring이 담당한다.

| 구분 | 일반적인 Java | Spring |
|---|---|---|
| 객체 생성 | 개발자가 직접 생성 | Spring이 생성 |
| 객체 관리 | 개발자가 관리 | Spring Container가 관리 |
| 의존 관계 연결 | 개발자가 직접 연결 | Spring이 DI 수행 |
| 객체 생명주기 | 개발자가 관리 | Spring이 관리 |

즉, 객체 관리에 대한 제어권이

**개발자 → Spring**

으로 이동한다.

이 때문에 제어가 역전되었다는 의미에서 IoC라고 부른다.

따라서 IoC의 핵심은 다음과 같다.

**객체의 생성과 관리에 대한 제어권을 개발자가 직접 가지는 것이 아니라 Spring에 맡기는 구조다.**

---

## 4. @RequiredArgsConstructor
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
}
```
이렇게 하면 Lombok이 컴파일 시점에 아래 생성자를 자동으로 만들어줍니다.
```java
public OrderService(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}
```
그래서 직접 생성자와 this.orderRepository = orderRepository;를 작성하지 않아도 됩니다.

이 코드는 Bean 등록 + 생성자 DI + Lombok을 통한 생성자 코드 생략까지 한 번에 표현한 형태라고 보면 됩니다.
## 5. Spring Container

Bean, DI, IoC를 실제로 동작시키는 중심에는 Spring Container가 있다.

Spring에서는 대표적으로 `ApplicationContext`가 Spring Container 역할을 담당한다.

Spring Container는 다음과 같은 작업을 수행한다.

| 역할 | 설명 |
|---|---|
| Bean 생성 | 등록 대상 객체를 생성 |
| Bean 관리 | 생성된 객체를 Container 내부에서 관리 |
| 의존 관계 확인 | Bean이 어떤 Bean을 필요로 하는지 확인 |
| DI 수행 | 필요한 Bean을 찾아 주입 |
| 생명주기 관리 | Bean의 생성부터 소멸까지 관리 |

즉, IoC라는 원칙을 실제 Spring 애플리케이션에서 구현하는 주체가 Spring Container라고 볼 수 있다.

---

## 6. Bean, DI, IoC는 어떻게 연결되는가?

다음과 같은 코드가 있다고 가정한다.

```java
@Repository
public class OrderRepository {
}
```

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

이 코드에는 Bean, DI, IoC가 모두 포함되어 있다.

먼저 `@Repository`와 `@Service`를 통해 `OrderRepository`와 `OrderService`는 Spring이 관리하는 **Bean**이 된다.

`OrderService`는 `OrderRepository`가 필요하지만 직접 다음과 같이 생성하지 않는다.

```java
new OrderRepository();
```

대신 생성자로 전달받는다.

```java
public OrderService(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}
```

Spring Container는 등록된 `OrderRepository` Bean을 찾아 `OrderService`를 생성할 때 전달한다.

이것이 **DI**다.

그리고 `OrderRepository`와 `OrderService`를 누가 생성하고, 관리하고, 연결할지를 개발자가 직접 결정하지 않는다.

Spring Container가 이 역할을 담당한다.

이 전체적인 제어 구조가 **IoC**다.

정리하면 다음과 같다.

| 개념 | 역할 |
|---|---|
| **IoC** | 객체 생성과 관리의 제어권을 Spring이 가지는 원칙 |
| **Bean** | Spring Container가 생성하고 관리하는 객체 |
| **DI** | Bean이 필요로 하는 다른 Bean을 주입하는 방식 |
| **Spring Container** | Bean 생성, 관리, DI를 실제로 수행하는 주체 |

세 개의 관계를 하나의 과정으로 표현하면 다음과 같다.

1. Spring이 객체의 생성과 관리에 대한 제어권을 가진다. → **IoC**
2. Spring이 관리할 객체를 생성하고 Container에 등록한다. → **Bean**
3. Bean이 다른 Bean을 필요로 하면 Spring이 해당 객체를 전달한다. → **DI**

따라서 Bean, DI, IoC는 서로 같은 개념은 아니지만 Spring의 객체 관리 구조에서는 긴밀하게 연결되어 동작한다.

가장 간단하게 정리하면 다음과 같다.

**IoC는 Spring이 객체를 관리하는 원칙이고, Bean은 그 관리 대상이며, DI는 Bean들을 서로 연결하는 방법이다.**

## 7. 같은 타입의 Bean이 여러 개일 때 `@Primary`와 `@Qualifier`

DI를 사용하면 Spring은 필요한 타입의 Bean을 찾아 자동으로 주입한다.

예를 들어 다음과 같이 `PaymentService`를 주입받는다고 가정한다.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring Container에 `PaymentService` 타입의 Bean이 하나만 존재한다면 해당 Bean을 문제없이 주입할 수 있다.

하지만 같은 타입의 Bean이 여러 개 존재하면 문제가 발생한다.

```java
public interface PaymentService {
    void pay();
}
```

```java
@Service
public class KakaoPaymentService implements PaymentService {

    @Override
    public void pay() {
        System.out.println("카카오 결제");
    }
}
```

```java
@Service
public class TossPaymentService implements PaymentService {

    @Override
    public void pay() {
        System.out.println("토스 결제");
    }
}
```

현재 Spring Container에는 `PaymentService` 타입으로 사용할 수 있는 Bean이 두 개 존재한다.

| Bean | 타입 |
|---|---|
| `kakaoPaymentService` | `PaymentService` |
| `tossPaymentService` | `PaymentService` |

이 상태에서 다음과 같이 `PaymentService`만 요청하면 Spring은 어떤 Bean을 주입해야 하는지 판단할 수 없다.

```java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

이처럼 같은 타입의 Bean이 여러 개 존재할 때 어떤 Bean을 사용할지 결정하기 위해 `@Primary`와 `@Qualifier`를 사용할 수 있다.

---

### 7.1 `@Primary`

`@Primary`는 같은 타입의 Bean이 여러 개 있을 때 **기본적으로 우선해서 사용할 Bean을 지정하는 어노테이션**이다.

```java
@Primary
@Service
public class KakaoPaymentService implements PaymentService {

    @Override
    public void pay() {
        System.out.println("카카오 결제");
    }
}
```

```java
@Service
public class TossPaymentService implements PaymentService {

    @Override
    public void pay() {
        System.out.println("토스 결제");
    }
}
```

이 상태에서 별도의 Bean을 지정하지 않고 `PaymentService`를 주입받으면 `@Primary`가 지정된 `KakaoPaymentService`가 주입된다.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

즉, `@Primary`는 여러 구현체 중 **기본값을 지정하는 역할**을 한다.

---

### 7.2`@Qualifier`

`@Qualifier`는 여러 Bean 중에서 **특정 Bean을 명확하게 지정하여 주입받을 때 사용하는 어노테이션**이다.

예를 들어 `TossPaymentService`를 사용하고 싶다면 다음과 같이 작성할 수 있다.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(
            @Qualifier("tossPaymentService")
            PaymentService paymentService
    ) {
        this.paymentService = paymentService;
    }
}
```

Spring은 `PaymentService` 타입의 Bean 중 `tossPaymentService`라는 이름을 가진 Bean을 찾아 주입한다.

`@Service`와 같은 컴포넌트 어노테이션을 사용할 경우 기본 Bean 이름은 클래스명의 첫 글자를 소문자로 변경한 형태로 생성된다.

```java
TossPaymentService
```

기본 Bean 이름은 다음과 같다.

```text
tossPaymentService
```

따라서 다음과 같이 지정할 수 있다.

```java
@Qualifier("tossPaymentService")
```

---

### 7.3`@Primary`와 `@Qualifier`의 차이

두 어노테이션 모두 같은 타입의 Bean이 여러 개 존재할 때 사용하는 기능이지만 역할에는 차이가 있다.

| 구분 | `@Primary` | `@Qualifier` |
|---|---|---|
| 목적 | 기본 Bean 지정 | 특정 Bean 지정 |
| 사용 위치 | 주로 Bean 정의 부분 | 주로 의존성을 주입받는 부분 |
| 의미 | 별도 지정이 없으면 이 Bean 사용 | 해당 위치에서는 이 Bean 사용 |
| 사용 상황 | 대부분 동일한 구현체를 사용할 때 | 특정 위치에서 다른 구현체가 필요할 때 |

정리하면 다음과 같다.

**`@Primary`는 기본값을 지정하고, `@Qualifier`는 사용할 Bean을 직접 지정한다.**

---

### 7.4`@Primary`와 `@Qualifier`를 함께 사용하는 경우

다음과 같이 `KakaoPaymentService`가 기본 Bean으로 지정되어 있다고 가정한다.

```java
@Primary
@Service
public class KakaoPaymentService implements PaymentService {
}
```

```java
@Service
public class TossPaymentService implements PaymentService {
}
```

별도의 지정 없이 주입받으면 `KakaoPaymentService`가 사용된다.

```java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

반면 특정 위치에서 `TossPaymentService`가 필요하다면 `@Qualifier`를 사용할 수 있다.

```java
public OrderService(
        @Qualifier("tossPaymentService")
        PaymentService paymentService
) {
    this.paymentService = paymentService;
}
```

이 경우 `@Primary`가 지정되어 있더라도 `@Qualifier`로 명확하게 지정한 `TossPaymentService`가 주입된다.

따라서 두 어노테이션을 함께 사용할 경우 다음과 같이 이해할 수 있다.

| 어노테이션 | 의미 |
|---|---|
| `@Primary` | 별도 지정이 없을 때 사용할 기본 Bean |
| `@Qualifier` | 해당 주입 위치에서 사용할 Bean을 명시적으로 지정 |

---

### 7.5 Bean과 DI 관점에서 정리

`@Primary`와 `@Qualifier`는 별도의 독립적인 개념이라기보다 **DI 과정에서 발생할 수 있는 Bean 선택 문제를 해결하는 방법**이다.

Spring이 DI를 수행할 때 먼저 필요한 타입의 Bean을 찾는다.

Bean이 하나라면 해당 Bean을 바로 주입할 수 있지만, 같은 타입의 Bean이 여러 개 존재하면 어떤 Bean을 사용할지 추가적인 기준이 필요하다.

이때 `@Primary` 또는 `@Qualifier`를 사용하여 주입할 Bean을 결정할 수 있다.

| 상황 | Spring의 처리 |
|---|---|
| 같은 타입 Bean이 1개 | 해당 Bean 주입 |
| 같은 타입 Bean이 여러 개 | 어떤 Bean을 사용할지 결정 필요 |
| `@Primary` 존재 | `@Primary` Bean 우선 사용 |
| `@Qualifier` 지정 | 지정된 Bean 사용 |

결국 `@Primary`와 `@Qualifier`는 Spring의 Bean과 DI 구조 안에서 **여러 Bean 중 실제로 주입할 객체를 선택하는 역할**을 한다.
