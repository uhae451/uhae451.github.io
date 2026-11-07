---
title: "AOP"
date: 2026-10-23 09:00:00 +0900
categories: ["Spring", "AOP"]
tags: ["spring", "aop", "aspect", "proxy", "cross-cutting-concern"]
description: "Spring AOP가 공통 관심사를 분리하는 방식과 프록시 기반 동작, 주요 어노테이션을 예제로 정리한다."
toc: true
---

Spring AOP는 여러 클래스나 메서드에서 반복적으로 사용되는 공통 로직을 비즈니스 로직과 분리하기 위한 기능이다.

AOP는 **Aspect-Oriented Programming**, 즉 관점 지향 프로그래밍을 의미한다.

핵심은 다음과 같다.

> 여러 곳에서 반복되는 공통 기능을 별도로 분리하고, 필요한 위치에 공통으로 적용한다.

---

## 1. AOP가 필요한 이유

예를 들어 주문 처리와 결제 처리에서 공통으로 로그를 남기고 실행 시간을 측정한다고 가정한다.

```java
@Service
public class OrderService {

    public void createOrder() {
        System.out.println("로그 기록");

        // 주문 생성 로직

        System.out.println("실행 시간 측정");
    }
}
```

```java
@Service
public class PaymentService {

    public void pay() {
        System.out.println("로그 기록");

        // 결제 처리 로직

        System.out.println("실행 시간 측정");
    }
}
```

이런 구조에서는 Service마다 같은 코드가 반복된다.

공통 로직이 많아질수록 다음과 같은 문제가 발생한다.

- 중복 코드 증가
- 비즈니스 로직과 부가 기능이 섞임
- 공통 기능을 수정할 때 여러 클래스를 함께 수정해야 함
- 코드의 핵심 목적을 파악하기 어려워짐

AOP를 사용하면 이런 공통 기능을 별도의 영역으로 분리할 수 있다.

---

## 2. 핵심 관심사와 공통 관심사

AOP에서는 로직을 크게 두 종류로 구분할 수 있다.

| 구분 | 의미 | 예시 |
|---|---|---|
| 핵심 관심사 | 해당 기능의 본래 목적 | 주문 생성, 결제, 회원가입 |
| 공통 관심사 | 여러 기능에서 공통으로 필요한 부가 기능 | 로그, 실행 시간 측정, 권한 검사, 트랜잭션 |

예를 들어 다음 메서드의 핵심 목적은 주문 생성이다.

```java
public void createOrder() {
    // 주문 생성
}
```

반면 로그 기록이나 실행 시간 측정은 주문 자체의 핵심 기능은 아니다.

이런 부가 기능을 별도로 분리하는 것이 AOP의 핵심이다.

---

## 3. `@Aspect`

Spring에서 AOP 로직을 정의할 때 대표적으로 `@Aspect`를 사용한다.

```java
@Aspect
@Component
public class LoggingAspect {
}
```

각 어노테이션의 의미는 다음과 같다.

| 어노테이션 | 역할 |
|---|---|
| `@Aspect` | 해당 클래스가 AOP 기능을 담당한다는 의미 |
| `@Component` | Spring Bean으로 등록 |

AOP 클래스 역시 Spring이 관리해야 하므로 Bean으로 등록되어야 한다.

---

## 4. Advice

Advice는 공통 로직을 **언제 실행할지** 정의한다.

Spring AOP에서 자주 사용하는 Advice는 다음과 같다.

| 어노테이션 | 실행 시점 |
|---|---|
| `@Before` | 대상 메서드 실행 전 |
| `@After` | 대상 메서드 실행 후 |
| `@AfterReturning` | 메서드가 정상적으로 반환된 후 |
| `@AfterThrowing` | 메서드 실행 중 예외가 발생한 후 |
| `@Around` | 메서드 실행 전후 전체 제어 |

처음에는 `@Before`, `@After`, `@Around`를 중심으로 이해하면 된다.

---

## 5. `@Before`

`@Before`는 대상 메서드가 실행되기 전에 공통 로직을 실행한다.

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service..*(..))")
    public void logBefore() {
        System.out.println("Service 메서드 실행 전");
    }
}
```

대상 메서드가 호출되면 개념적으로 다음 순서로 동작한다.

| 순서 | 동작 |
|---|---|
| 1 | `@Before` 로직 실행 |
| 2 | 실제 대상 메서드 실행 |

로그 기록이나 권한 확인처럼 메서드 실행 전에 처리해야 하는 기능에 사용할 수 있다.

---

## 6. `@After`

`@After`는 대상 메서드의 실행이 끝난 뒤 동작한다.

```java
@After("execution(* com.example.service..*(..))")
public void logAfter() {
    System.out.println("Service 메서드 실행 후");
}
```

개념적으로는 다음 순서다.

| 순서 | 동작 |
|---|---|
| 1 | 실제 대상 메서드 실행 |
| 2 | `@After` 로직 실행 |

---

## 7. `@Around`

`@Around`는 대상 메서드의 실행 전과 후를 모두 제어할 수 있다.

실행 시간 측정과 같은 기능을 구현할 때 자주 사용된다.

```java
@Aspect
@Component
public class TimeTraceAspect {

    @Around("execution(* com.example.service..*(..))")
    public Object trace(ProceedingJoinPoint joinPoint)
            throws Throwable {

        long start = System.currentTimeMillis();

        Object result = joinPoint.proceed();

        long end = System.currentTimeMillis();

        System.out.println(
            "실행 시간: " + (end - start) + "ms"
        );

        return result;
    }
}
```

여기서 가장 중요한 부분은 다음 코드다.

```java
joinPoint.proceed();
```

`joinPoint.proceed()`는 **원래 실행하려던 대상 메서드를 실제로 호출하는 역할**을 한다.

전체 흐름은 다음과 같다.

| 순서 | 동작 |
|---|---|
| 1 | AOP 로직 시작 |
| 2 | 실행 시간 측정 시작 |
| 3 | `joinPoint.proceed()` 실행 |
| 4 | 실제 대상 메서드 실행 |
| 5 | 대상 메서드 종료 |
| 6 | 실행 시간 측정 종료 |
| 7 | 결과 반환 |

이처럼 `@Around`는 메서드 호출 자체를 감싸는 형태로 동작한다.

---

## 8. Pointcut

Pointcut은 **AOP를 어느 위치에 적용할지 지정하는 조건**이다.

예를 들어 다음 표현식이 있다.

```java
execution(* com.example.service..*(..))
```

이는 `com.example.service` 패키지와 그 하위 영역의 메서드를 대상으로 삼는 Pointcut 표현식이다.

AOP에서는 크게 두 가지를 결정해야 한다.

| 개념 | 역할 |
|---|---|
| Advice | 언제 어떤 공통 로직을 실행할지 결정 |
| Pointcut | 어느 메서드에 적용할지 결정 |

예를 들어:

```java
@Before("execution(* com.example.service..*(..))")
public void logging() {
    System.out.println("실행");
}
```

여기서:

- `@Before`는 실행 시점을 결정하고
- `execution(...)`은 적용 범위를 결정한다.

---

## 9. Spring AOP의 Proxy 구조

Spring AOP는 보통 **Proxy 객체를 이용해서 동작한다.**

원래 구조가 다음과 같다고 가정한다.

```text
Controller
    |
OrderService
```

AOP가 적용되면 개념적으로 다음과 같은 구조가 된다.

```text
Controller
    |
OrderService Proxy
    |
AOP 공통 로직
    |
실제 OrderService
```

Controller는 `OrderService`를 호출한다고 생각하지만 실제로는 Proxy가 먼저 호출을 받는다.

Proxy는 필요한 공통 로직을 실행한 뒤 실제 객체를 호출한다.

예를 들어 실행 시간 측정 AOP라면 다음과 같은 구조가 된다.

| 순서 | 동작 |
|---|---|
| 1 | Controller가 Service 호출 |
| 2 | Proxy가 호출을 먼저 받음 |
| 3 | 실행 시간 측정 시작 |
| 4 | 실제 Service 메서드 실행 |
| 5 | 실행 시간 측정 종료 |
| 6 | 결과 반환 |

---

## 10. Proxy를 사용하는 이유

AOP의 핵심 목적은 기존 비즈니스 로직을 직접 수정하지 않고 공통 기능을 추가하는 것이다.

원래 메서드가 다음과 같다고 가정한다.

```java
public void createOrder() {
    // 주문 로직
}
```

이 코드는 그대로 유지하면서 외부에서는 다음처럼 동작하게 만들 수 있다.

| 단계 | 실행 내용 |
|---|---|
| 1 | 로그 또는 실행 시간 측정 시작 |
| 2 | 주문 로직 실행 |
| 3 | 로그 또는 실행 시간 측정 종료 |

이를 가능하게 만드는 중간 객체가 Proxy다.

---

## 11. AOP와 `@Transactional`

Spring에서 AOP를 이해해야 하는 대표적인 이유 중 하나가 `@Transactional`이다.

예를 들어 다음 코드가 있다고 가정한다.

```java
@Transactional
public void createOrder() {

    orderRepository.save();
    paymentRepository.save();
}
```

개발자는 직접 다음과 같은 코드를 작성하지 않는다.

```text
트랜잭션 시작
commit
rollback
```

하지만 Spring이 이를 자동으로 처리한다.

개념적으로는 Proxy가 대상 메서드 실행 전후에 트랜잭션 로직을 적용한다.

| 상황 | 처리 |
|---|---|
| 메서드 실행 전 | 트랜잭션 시작 |
| 정상 종료 | Commit |
| 기본 설정에서 `RuntimeException` 또는 `Error` 발생 | Rollback |

Checked Exception은 기본적으로 롤백 대상이 아니며 `rollbackFor` 등으로 지정할 수 있다.

즉, `@Transactional` 역시 Spring의 Proxy 기반 AOP 구조와 밀접하게 연결되어 있다.

---

## 12. Self Invocation 문제

Spring AOP는 Proxy를 기반으로 동작하기 때문에 같은 클래스 내부에서 메서드를 직접 호출할 경우 주의해야 한다.

예를 들어 다음과 같은 코드가 있다.

```java
@Service
public class OrderService {

    public void order() {
        payment();
    }

    @Transactional
    public void payment() {
    }
}
```

`order()` 안에서 다음과 같이 호출한다.

```java
payment();
```

이 호출은 외부 Proxy를 거치지 않고 같은 객체 내부에서 직접 수행될 수 있다.

이 경우 `payment()`에 붙은 `@Transactional` 같은 AOP 기능이 기대한 대로 적용되지 않을 수 있다.

이를 **Self Invocation 문제**라고 한다.

초기 학습 단계에서는 다음 내용만 기억하면 충분하다.

> Spring AOP는 기본적으로 Proxy를 거쳐 호출될 때 적용된다.

---

## 13. AOP의 주요 활용 사례

AOP는 여러 기능에 공통으로 적용해야 하는 부가 로직을 분리할 때 사용한다.

대표적인 활용 사례는 다음과 같다.

| 활용 영역 | 예시 |
|---|---|
| 로깅 | 메서드 실행 기록 |
| 성능 측정 | 메서드 실행 시간 측정 |
| 트랜잭션 | Commit / Rollback 처리 |
| 권한 검사 | 특정 기능 실행 전 권한 확인 |
| 예외 기록 | 예외 발생 시 로그 기록 |

특히 실행 시간 측정은 성능 문제를 확인할 때 간단한 계측 방법으로 사용할 수 있다.

```java
@Around("execution(* com.example.service..*(..))")
public Object measure(
        ProceedingJoinPoint joinPoint
) throws Throwable {

    long start = System.currentTimeMillis();

    try {
        return joinPoint.proceed();
    } finally {
        long end = System.currentTimeMillis();

        System.out.println(
            joinPoint.getSignature()
            + " 실행 시간: "
            + (end - start)
            + "ms"
        );
    }
}
```

이런 방식으로 특정 Service 메서드가 얼마나 오래 걸리는지 확인할 수 있다.

다만 AOP 실행 시간 측정은 성능 문제의 원인을 모두 알려주는 도구는 아니다.

예를 들어 어떤 메서드가 800ms 걸린다는 사실은 알 수 있지만, 그중 DB 쿼리가 700ms인지 외부 API가 700ms인지까지는 추가적인 분석이 필요하다.

따라서 성능 분석에서는 AOP를 특정 구간의 실행 시간을 확인하는 **계측 수단 중 하나**로 활용할 수 있다.

---

## 정리

AOP는 **핵심 비즈니스 로직과 여러 곳에서 반복되는 공통 로직을 분리하는 프로그래밍 방식**이다.

Spring AOP에서는 Proxy 객체가 실제 Bean 앞에서 호출을 먼저 받아 공통 로직을 실행한 뒤 실제 메서드를 호출한다.

핵심 개념은 다음과 같다.

| 개념 | 의미 |
|---|---|
| Aspect | 공통 기능을 정의한 영역 |
| Advice | 공통 로직을 언제 실행할지 정의 |
| Pointcut | 공통 로직을 어디에 적용할지 정의 |
| Proxy | 실제 객체 대신 호출을 받아 AOP를 수행하는 객체 |
| `joinPoint.proceed()` | 실제 대상 메서드를 실행 |

결국 Spring AOP는 **Proxy를 이용해 실제 메서드 호출 전후에 공통 로직을 적용하는 구조**라고 정리할 수 있다.
