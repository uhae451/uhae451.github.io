---
title: "Bean Scope와 Bean Lifecycle 정리"
date: 2026-10-22 09:00:00 +0900
categories: ["Spring", "Core"]
tags: ["spring", "bean-scope", "bean-lifecycle", "singleton", "prototype"]
description: "Spring Bean의 스코프와 객체의 생성·초기화·소멸 생명주기 및 적용 기준을 정리한다."
toc: true
---

Spring에서 Bean은 단순히 객체를 등록하고 사용하는 것에서 끝나지 않는다. Bean마다 **객체가 유지되는 범위**가 존재하며, 생성된 Bean은 일정한 생명주기를 가진다.

이를 각각 **Bean Scope**와 **Bean Lifecycle**이라고 한다.

- Bean Scope: Bean 객체가 생성되고 유지되는 범위
- Bean Lifecycle: Bean 객체가 생성된 이후 소멸될 때까지의 과정

두 개념은 서로 연결되어 있으며, Bean의 Scope에 따라 Lifecycle도 달라질 수 있다.

---

## 1. Bean Scope란?

Bean Scope는 **Spring Container가 Bean 객체를 생성하고 유지하는 범위**를 의미한다.

같은 Bean을 여러 번 요청했을 때 하나의 객체를 계속 공유할 수도 있고, 요청할 때마다 새로운 객체를 생성할 수도 있다.

Spring에서는 대표적으로 다음과 같은 Scope를 제공한다.

| Scope | 특징 |
|---|---|
| `singleton` | 하나의 Bean 객체를 생성하고 공유 |
| `prototype` | Bean을 요청할 때마다 새로운 객체 생성 |
| `request` | HTTP 요청마다 새로운 Bean 생성 |
| `session` | HTTP Session마다 새로운 Bean 생성 |

일반적인 Spring 애플리케이션에서 가장 많이 사용하는 것은 `singleton`이며, 별도의 Scope를 지정하지 않으면 기본적으로 Singleton Scope가 적용된다.

---

## 2. Singleton Scope

Spring Bean의 기본 Scope는 **Singleton**이다.

예를 들어 다음과 같이 Bean을 등록하면 별도로 Scope를 지정하지 않았기 때문에 Singleton Bean이 된다.

```java
@Service
public class UserService {
}
```

Singleton Bean은 **컨테이너별·Bean 정의별로** 하나의 객체를 공유한다. 같은 클래스라도 서로 다른 Bean으로 등록하면 여러 객체가 생길 수 있다.

예를 들어 `OrderService`와 `PaymentService`가 모두 `UserService`를 주입받는다고 가정한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final UserService userService;
}
```

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final UserService userService;
}
```

이 경우 `OrderService`와 `PaymentService`를 위해 각각 새로운 `UserService`가 만들어지는 것이 아니다.

Spring Container가 관리하고 있는 하나의 `UserService` Bean이 두 Service에 주입된다.

| Bean | 주입받는 `UserService` |
|---|---|
| `OrderService` | 동일한 `UserService` Bean |
| `PaymentService` | 동일한 `UserService` Bean |

따라서 Singleton의 핵심은 **하나의 Bean 객체를 여러 곳에서 공유한다는 것**이다.

---

## 3. Singleton을 기본으로 사용하는 이유

Service나 Repository와 같은 Spring Bean은 일반적으로 객체 내부에 사용자별 데이터를 저장하기보다 특정 기능이나 로직을 수행하는 역할을 한다.

예를 들어 다음과 같은 `OrderService`가 있다고 가정한다.

```java
@Service
public class OrderService {

    public void order(Long userId) {
        // 주문 처리
    }
}
```

사용자가 주문할 때마다 `OrderService` 객체 자체를 새롭게 생성할 필요는 없다.

동일한 객체의 `order()` 메서드를 여러 요청에서 사용할 수 있기 때문이다.

Singleton을 사용하면 다음과 같은 장점이 있다.

- 불필요한 객체 생성을 줄일 수 있다.
- 메모리를 효율적으로 사용할 수 있다.
- 동일한 기능을 제공하는 객체를 여러 곳에서 공유할 수 있다.

이러한 이유로 Spring Bean의 기본 Scope는 Singleton이다.

---

## 4. Singleton Bean 사용 시 주의점

Singleton Bean은 하나의 객체를 여러 요청에서 공유하기 때문에 **객체 내부에 변경 가능한 상태를 저장할 때 주의해야 한다.**

예를 들어 다음과 같은 코드는 문제가 발생할 수 있다.

```java
@Service
public class OrderService {

    private String userName;

    public void order(String userName) {
        this.userName = userName;
    }
}
```

`OrderService` Bean이 하나뿐이기 때문에 여러 사용자가 동시에 이 객체를 사용할 수 있다.

사용자 A가 다음 값을 저장했다고 가정한다.

```text
userName = "Kim"
```

그 이후 사용자 B가 같은 Bean을 사용하면:

```text
userName = "Lee"
```

동일한 객체의 필드가 변경된다.

따라서 Singleton Bean에는 일반적으로 요청별 데이터를 필드에 저장하지 않고, 필요한 값은 메서드의 파라미터나 지역 변수로 처리하는 **Stateless한 구조**를 사용하는 것이 적절하다.

```java
@Service
public class OrderService {

    public void order(String userName) {
        System.out.println(userName);
    }
}
```

---

## 5. Prototype Scope

Prototype Scope는 Bean을 요청할 때마다 **새로운 Bean 객체를 생성하는 방식**이다.

다음과 같이 `@Scope("prototype")`을 지정할 수 있다.

```java
@Component
@Scope("prototype")
public class OrderProcessor {
}
```

Spring Container에서 Bean을 두 번 요청하면 각각 다른 객체가 반환된다.

```java
OrderProcessor processor1 = applicationContext.getBean(OrderProcessor.class);

OrderProcessor processor2 = applicationContext.getBean(OrderProcessor.class);
```

두 객체를 비교하면:

```java
processor1 == processor2
```

결과는 `false`가 된다.

반면 Singleton Bean은 동일한 방식으로 여러 번 가져와도 같은 객체를 반환한다.

> Singleton에 Prototype을 생성자 주입하면 Singleton 생성 시 받은 객체를 계속 사용한다. 사용 시마다 새 객체가 필요하면 `ObjectProvider.getObject()` 등으로 매번 컨테이너에 요청해야 한다.

### Singleton과 Prototype 비교

| 구분 | Singleton | Prototype |
|---|---|---|
| 객체 생성 | 기본적으로 Bean당 하나 | 요청할 때마다 새로운 객체 |
| 객체 공유 | 공유 | 공유하지 않음 |
| Spring 기본 Scope | O | X |
| 주 사용 목적 | Service, Repository 등 일반적인 Bean | 매번 독립적인 객체가 필요한 경우 |

---

## 6. Bean Lifecycle이란?

Bean Lifecycle은 **Spring Bean이 생성된 이후 사용되고 최종적으로 소멸될 때까지의 과정**을 의미한다.

Singleton Bean을 기준으로 주요 단계는 다음과 같다.

| 순서 | 단계 | 설명 |
|---|---|---|
| 1 | 객체 생성 | Spring이 Bean 객체 생성 |
| 2 | 의존성 주입 | Bean이 필요로 하는 다른 Bean 주입 |
| 3 | 초기화 | Bean 사용에 필요한 초기화 작업 수행 |
| 4 | 사용 | 애플리케이션에서 Bean 사용 |
| 5 | 소멸 | Container 종료 시 Bean 정리 |

Spring은 Bean 객체만 생성하는 것이 아니라 이러한 생명주기도 관리한다.

---

## 7. `@PostConstruct`

`@PostConstruct`는 **Bean 객체가 생성되고 의존성 주입까지 완료된 이후 한 번 실행할 초기화 메서드**를 지정할 때 사용한다.

```java
@Component
public class ExternalClient {

    @PostConstruct
    public void init() {
        System.out.println("초기화");
    }
}
```

중요한 점은 `@PostConstruct`가 해당 Bean의 메서드를 사용할 때마다 실행되는 것이 아니라는 것이다.

기본 Singleton Bean이라면 애플리케이션이 시작되는 과정에서 Bean이 생성되고 의존성 주입이 완료된 이후 한 번 실행된다.

| 단계 | `ExternalClient` 상태 |
|---|---|
| 객체 생성 | 객체가 만들어짐 |
| DI | 필요한 의존성이 주입됨 |
| `@PostConstruct` | 초기화 메서드 실행 |
| 사용 가능 | 다른 Bean에서 정상적으로 사용 |

따라서 사용자가 API를 호출해야 `@PostConstruct`가 실행되는 것은 아니다.

일반적인 Singleton Bean이라면 Spring 애플리케이션이 시작되면서 Bean을 준비하는 과정에서 실행될 수 있다.

---

## 8. `@PostConstruct`를 사용하는 경우

`@PostConstruct`는 Bean을 실제로 사용하기 전에 **한 번 준비해야 하는 작업**이 있을 때 사용할 수 있다.

대표적인 예시는 다음과 같다.

- 초기 설정값 검증
- 외부 API Client 초기화
- 필요한 리소스 준비
- 초기 데이터 로딩
- 자주 사용하는 데이터를 메모리에 미리 적재

예를 들어 카테고리 데이터가 자주 조회되지만 거의 변경되지 않는다고 가정한다.

매 요청마다 다음 코드를 실행하면:

```java
categoryRepository.findAll();
```

요청마다 DB 조회가 발생한다.

필요하다면 Bean이 초기화될 때 한 번 데이터를 조회해서 메모리에 저장할 수도 있다.

```java
@Component
@RequiredArgsConstructor
public class CategoryCache {

    private final CategoryRepository categoryRepository;

    private List<Category> categories;

    @PostConstruct
    public void init() {
        categories = categoryRepository.findAll();
    }

    public List<Category> getCategories() {
        return categories;
    }
}
```

`categories`의 `List` 자체가 캐시라는 의미는 아니다.

`List`는 단순한 Java 자료구조지만, 원본인 DB를 반복해서 조회하지 않기 위해 데이터를 메모리에 저장하고 재사용하고 있으므로 이 예제에서는 **간단한 인메모리 캐시 역할**을 한다.

다만 DB의 카테고리 데이터가 변경된다면 메모리에 저장된 데이터도 적절하게 갱신해야 한다.

실제 캐시 시스템에서는 Spring Cache, Caffeine, Redis 등의 별도 캐시 기술을 사용할 수도 있다.

---

## 9. `@PreDestroy`

`@PreDestroy`는 **Bean이 소멸되기 전에 실행할 작업**을 지정한다.

```java
@Component
public class ExternalClient {

    @PreDestroy
    public void close() {
        System.out.println("리소스 정리");
    }
}
```

Bean을 사용하면서 외부 리소스를 사용했다면 Bean이 제거되기 전에 이를 정리해야 할 수 있다.

예를 들면 다음과 같은 작업이다.

- 네트워크 연결 종료
- 외부 리소스 반환
- 사용 중인 연결 정리
- 임시 리소스 제거

Singleton Bean의 경우 Spring Container가 정상적으로 종료될 때 소멸 과정도 관리할 수 있다.

---

## 10. Bean Lifecycle 전체 흐름

다음과 같은 Bean이 있다고 가정한다.

```java
@Component
public class ExternalClient {

    public ExternalClient() {
        System.out.println("객체 생성");
    }

    @PostConstruct
    public void init() {
        System.out.println("초기화");
    }

    public void request() {
        System.out.println("사용");
    }

    @PreDestroy
    public void close() {
        System.out.println("소멸");
    }
}
```

Singleton Bean의 주요 Lifecycle은 다음과 같이 정리할 수 있다.

| 순서 | 동작 |
|---|---|
| 1 | Spring이 `ExternalClient` 객체를 생성 |
| 2 | 필요한 의존성을 주입 |
| 3 | `@PostConstruct` 메서드 실행 |
| 4 | 애플리케이션에서 Bean 사용 |
| 5 | Spring Container 종료 |
| 6 | `@PreDestroy` 메서드 실행 |

즉, `@PostConstruct`는 **Bean 사용 전 초기화**, `@PreDestroy`는 **Bean 소멸 전 정리 작업**에 해당한다.

---

## 11. Prototype Bean의 Lifecycle

Prototype Scope는 Singleton과 Lifecycle 관리 범위에 차이가 있다.

Spring은 Prototype Bean을 요청받으면 새로운 객체를 생성하고, 의존성을 주입하고, 초기화까지 수행한 뒤 해당 객체를 반환한다.

하지만 객체를 반환한 이후에는 Spring Container가 Prototype Bean의 전체 생명주기를 계속 관리하지 않는다.

따라서 Prototype Bean은 일반적으로 Spring이 소멸 콜백까지 자동으로 관리하지 않는다.

| Lifecycle 단계 | Singleton | Prototype |
|---|---|---|
| 객체 생성 | Spring 관리 | Spring 관리 |
| DI | Spring 관리 | Spring 관리 |
| 초기화 | Spring 관리 | Spring 관리 |
| Bean 사용 | Container가 객체 유지 | 사용하는 코드에서 관리 |
| 소멸 처리 | Spring 관리 | 직접 관리 필요 |

즉, Prototype은 **생성 및 초기화까지는 Spring이 담당하지만 이후 관리 책임은 객체를 사용하는 쪽에 있다**는 점이 중요하다.

---

## 12. Bean Scope와 Lifecycle의 관계

Scope와 Lifecycle은 서로 연결된 개념이다.

**Scope는 Bean이 어느 범위에서 유지되는지를 결정하고, Lifecycle은 그 Bean이 생성되어 소멸할 때까지 어떤 과정을 거치는지를 정의한다.**

Singleton의 경우 Spring Container가 하나의 객체를 생성한 뒤 계속 공유한다.

| 시점 | 상태 |
|---|---|
| 애플리케이션 시작 | Singleton Bean 생성 |
| Bean 생성 직후 | DI 및 초기화 |
| 애플리케이션 실행 중 | 동일한 Bean 공유 |
| 애플리케이션 종료 | Bean 소멸 처리 |

반면 Prototype은 Bean을 요청할 때마다 Lifecycle이 새롭게 시작된다.

| Bean 요청 | 결과 |
|---|---|
| 첫 번째 요청 | 새로운 객체 생성 및 초기화 |
| 두 번째 요청 | 또 다른 객체 생성 및 초기화 |
| 세 번째 요청 | 또 다른 객체 생성 및 초기화 |

따라서 Scope에 따라 Bean 객체가 생성되는 시점과 횟수, 관리되는 기간에 차이가 발생한다.

---

## 정리

Bean Scope는 **Bean 객체가 얼마나 오래, 어떤 범위에서 유지될지를 결정하는 개념**이고, Bean Lifecycle은 **그 Bean이 생성된 이후 소멸될 때까지의 과정을 의미한다.**

Spring Bean은 기본적으로 Singleton Scope를 사용하기 때문에 하나의 객체를 생성하고 여러 곳에서 공유한다.

Singleton Bean은 공유되는 객체이므로 요청별 상태를 Bean 필드에 저장하지 않는 Stateless한 설계가 중요하다.

또한 Spring은 Bean을 생성한 뒤 DI를 수행하고 초기화 과정을 거쳐 사용할 수 있는 상태로 만든다.

이 과정에서 `@PostConstruct`를 사용하면 **DI 완료 이후 실행할 초기화 작업**을 정의할 수 있으며, `@PreDestroy`를 사용하면 **Bean 소멸 전에 수행할 정리 작업**을 정의할 수 있다.

결국 Bean Scope와 Lifecycle은 Spring Container가 **Bean을 언제 만들고, 얼마나 유지하고, 언제 정리하는지**를 이해하기 위한 핵심 개념이다.
