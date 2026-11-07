---
title: "Bean과 의존성 주입(DI) 정리"
date: 2026-10-20 09:00:00 +0900
categories: ["Spring", "Core"]
tags: ["spring", "bean", "dependency-injection", "di", "constructor-injection"]
description: "Spring Bean의 개념과 의존성 주입 방식, 객체를 외부에서 주입받도록 설계하는 이유를 정리한다."
toc: true
---

## 1. Bean이란?

Spring에서 **Bean은 Spring Container가 생성하고 관리하는 객체**를 의미한다.

일반 Java에서는 객체가 필요할 때 직접 `new`를 사용해 생성한다.

```java
UserService userService = new UserService();
```

Spring에서는 객체를 Bean으로 등록하면 객체의 생성과 관리 책임을 Spring Container가 담당한다.

```java
@Service
public class UserService {
}
```

`@Service`가 선언된 `UserService`는 Spring의 컴포넌트 스캔 대상이 되고, Spring Container에 Bean으로 등록된다.

대표적인 Bean 등록 관련 애노테이션은 다음과 같다.

| 애노테이션 | 용도 |
|---|---|
| `@Component` | 일반적인 Spring Bean |
| `@Service` | 비즈니스 로직 계층 |
| `@Repository` | 데이터 접근 계층 |
| `@Controller` | MVC Controller |
| `@RestController` | REST API Controller |
| `@Bean` | 개발자가 직접 Bean 생성 방법을 정의할 때 |

---

## 2. Bean을 사용하는 이유

객체를 직접 생성하면 해당 객체가 필요로 하는 의존성도 직접 생성하고 전달해야 한다.

```java
UserRepository userRepository = new UserRepository();
UserService userService = new UserService(userRepository);
```

객체 수와 의존 관계가 많아질수록 객체 생성 코드도 복잡해진다.

예를 들어 다음과 같은 구조가 있다고 가정한다.

| 계층 | 클래스 | 의존 대상 |
|---|---|---|
| Controller | `UserController` | `UserService` |
| Service | `UserService` | `UserRepository` |
| Repository | `UserRepository` | 없음 |

직접 객체를 관리하면 각 객체의 생성 순서와 생성자 인자를 개발자가 직접 맞춰야 한다.

Spring에서는 필요한 객체들을 Bean으로 등록하고, 클래스에서는 필요한 의존성만 선언한다.

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

`UserService`는 `UserRepository`를 직접 생성하지 않는다.

```java
new UserRepository();
```

대신 생성자에 필요한 타입을 선언한다.

```java
public UserService(UserRepository userRepository)
```

Spring이 해당 타입의 Bean을 찾아서 전달한다.

---

## 3. 의존성 주입(DI)

DI는 **Dependency Injection, 의존성 주입**을 의미한다.

객체 내부에서 필요한 객체를 직접 생성하지 않고 외부에서 전달받는 구조이다.

### 직접 객체 생성

```java
public class UserService {

    private final UserRepository userRepository
            = new UserRepository();
}
```

이 방식에서는 `UserService`가 `UserRepository`의 생성까지 직접 담당한다.

### 의존성 주입

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

`UserService`는 `UserRepository`가 필요하다는 사실만 선언한다.

실제 객체를 생성하고 전달하는 작업은 Spring Container가 담당한다.

| 구분 | 직접 객체 관리 | Spring Bean 사용 |
|---|---|---|
| 객체 생성 | 개발자가 직접 수행 | Spring이 수행 |
| 생성 순서 관리 | 개발자가 관리 | Spring이 관리 |
| 생성자 인자 전달 | 개발자가 직접 전달 | Spring이 자동 주입 |
| 객체 생명주기 | 개발자가 직접 관리 | Spring Container가 관리 |
| 의존 관계 연결 | 개발자가 직접 구성 | Spring이 자동 구성 |

Spring도 내부적으로는 객체를 생성해야 하므로 결국 `new`를 사용한다.

차이는 객체 생성 자체가 아니라 **객체 생성과 의존 관계 조립의 책임이 누구에게 있는가**이다.

---

## 4. Spring Boot 실행 시 Bean 생성 과정

Spring Boot 애플리케이션은 보통 다음 구조에서 시작한다.

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`SpringApplication.run()`이 실행되면 Spring Container 역할을 하는 `ApplicationContext`가 생성되고, Bean 등록과 생성 과정이 진행된다.

| 단계 | 처리 내용 |
|---|---|
| 1 | Spring Boot 애플리케이션 실행 |
| 2 | `ApplicationContext` 생성 |
| 3 | Component Scan 수행 |
| 4 | Bean 등록 대상 탐색 |
| 5 | Bean 정의 등록 |
| 6 | Bean 간 의존 관계 확인 |
| 7 | Bean 객체 생성 |
| 8 | 필요한 Bean 의존성 주입 |
| 9 | 애플리케이션 실행 준비 완료 |

예를 들어 다음과 같은 클래스가 있다고 가정한다.

```java
@Repository
public class UserRepository {
}
```

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

```java
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

각 Bean의 의존 관계는 다음과 같다.

| Bean | 필요한 의존성 |
|---|---|
| `UserRepository` | 없음 |
| `UserService` | `UserRepository` |
| `UserController` | `UserService` |

Spring은 이 관계를 바탕으로 필요한 객체를 순서에 맞게 준비한다.

개념적으로 다음과 같은 객체 생성 작업을 Spring이 대신 수행한다고 이해할 수 있다.

```java
UserRepository repository = new UserRepository();

UserService service =
        new UserService(repository);

UserController controller =
        new UserController(service);
```

---

## 5. `@Autowired`

`@Autowired`는 **Spring Container에서 조건에 맞는 Bean을 찾아 의존성을 주입하도록 지정하는 애노테이션**이다.

대표적으로 필드에 직접 사용할 수 있다.

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
}
```

Spring은 `UserRepository` 타입의 Bean을 찾아 `userRepository` 필드에 주입한다.

이 방식을 **필드 주입(Field Injection)**이라고 한다.

필드 주입에서는 생성자와 다음 코드가 필요하지 않다.

```java
this.userRepository = userRepository;
```

다만 일반적인 애플리케이션 코드에서는 필드 주입보다 생성자 주입이 더 권장된다.

---

## 6. 생성자 주입

생성자 주입은 생성자를 통해 필요한 Bean을 전달받는 방식이다.

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

생성자가 하나만 존재하는 경우 Spring은 해당 생성자를 의존성 주입 대상으로 자동 인식한다.

따라서 다음과 같이 `@Autowired`를 생략할 수 있다.

```java
public UserService(UserRepository userRepository) {
    this.userRepository = userRepository;
}
```

Spring은 생성자 파라미터의 타입을 기준으로 필요한 Bean을 찾는다.

| 확인 대상 | 처리 내용 |
|---|---|
| 생성 대상 | `UserService` |
| 생성자 파라미터 | `UserRepository` |
| 검색 대상 | `UserRepository` 타입의 Bean |
| 주입 위치 | `UserService` 생성자 |
| 결과 | `UserRepository`를 가진 `UserService` Bean 생성 |

---

## 7. `final`과 `this`

다음 코드에서 `final`과 `this`는 Spring 기능이 아니라 Java 문법이다.

```java
private final UserRepository userRepository;

public UserService(UserRepository userRepository) {
    this.userRepository = userRepository;
}
```

### `final`

```java
private final UserRepository userRepository;
```

`final`이 붙은 필드는 한 번 초기화된 이후 다른 객체로 다시 할당할 수 없다.

생성자에서 한 번 주입받은 의존성을 이후에 변경하지 않도록 할 수 있기 때문에 생성자 주입과 잘 맞는다.

### `this`

```java
this.userRepository = userRepository;
```

왼쪽의 `this.userRepository`는 현재 객체의 필드이고, 오른쪽의 `userRepository`는 생성자 파라미터이다.

생성자로 전달받은 객체를 클래스의 필드에 저장하는 Java 문법이다.

| 구분 | 항목 |
|---|---|
| Spring 기능 | Bean, DI, `@Autowired` |
| Java 기능 | `final`, `this`, 생성자 |

---

## 8. Spring의 의존성 주입 방식

Spring에서 대표적으로 사용하는 의존성 주입 방식은 생성자 주입, 필드 주입, Setter 주입이다.

### 8.1 생성자 주입

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

필수 의존성을 표현하기 적합하며 일반적으로 가장 권장된다.

| 특징 | 설명 |
|---|---|
| `final` 사용 | 가능 |
| 의존성 확인 | 생성자를 보면 바로 확인 가능 |
| 주입 시점 | 객체 생성 시점 |
| 필수 의존성 표현 | 적합 |
| 테스트 | 객체를 직접 생성하기 쉬움 |
| 일반적인 권장도 | 높음 |

---

### 8.2 필드 주입

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
}
```

Spring이 필드에 직접 Bean을 주입한다.

| 특징 | 설명 |
|---|---|
| 코드 길이 | 짧음 |
| `final` 사용 | 어려움 |
| 의존성 확인 | 필드를 직접 확인해야 함 |
| 객체 직접 생성 | 의존성을 외부에서 넣기 불편함 |
| 일반적인 권장도 | 낮음 |

---

### 8.3 Setter 주입

```java
@Service
public class UserService {

    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Setter 메서드를 통해 Bean을 전달받는 방식이다.

| 특징 | 설명 |
|---|---|
| 주입 시점 | 객체 생성 이후 가능 |
| 의존성 변경 | 가능 |
| 선택적 의존성 | 활용 가능 |
| `final` 사용 | 어려움 |
| 일반적인 사용 | 제한적 |

세 방식을 비교하면 다음과 같다.

| 방식 | `final` | 필수 의존성 표현 | 일반적인 사용 |
|---|---:|---:|---|
| 생성자 주입 | O | 적합 | 권장 |
| 필드 주입 | X | 불명확 | 제한적 |
| Setter 주입 | X | 선택적 의존성에 적합 | 필요 시 사용 |

---

## 9. Lombok `@RequiredArgsConstructor`

생성자 주입을 사용하면 의존성이 많아질수록 생성자 코드도 길어진다.

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Lombok의 `@RequiredArgsConstructor`를 사용하면 `final` 필드를 포함하는 생성자를 자동으로 만들어준다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
}
```

컴파일 과정에서 개념적으로 다음 생성자가 만들어진다.

```java
public UserService(UserRepository userRepository) {
    this.userRepository = userRepository;
}
```

Spring 입장에서는 일반적인 생성자 주입과 동일하게 처리된다.

| 처리 단계 | 내용 |
|---|---|
| 1 | `@RequiredArgsConstructor`가 생성자 생성 |
| 2 | 생성자에 `UserRepository` 파라미터 포함 |
| 3 | Spring이 생성자 확인 |
| 4 | `UserRepository` Bean 검색 |
| 5 | 생성자에 Bean 전달 |
| 6 | `UserService` Bean 생성 |

Spring Boot 프로젝트에서는 다음 형태를 자주 사용한다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
}
```

---

## 10. 전체 개념 정리

Spring의 Bean과 DI 관련 개념을 정리하면 다음과 같다.

| 개념 | 의미 |
|---|---|
| Bean | Spring Container가 생성하고 관리하는 객체 |
| Spring Container | Bean의 생성, 관리, 의존성 연결을 담당하는 컨테이너 |
| `ApplicationContext` | 대표적인 Spring Container |
| Component Scan | Bean 등록 대상 클래스를 탐색하는 과정 |
| DI | 객체가 필요로 하는 다른 객체를 외부에서 주입하는 방식 |
| `@Autowired` | Spring에게 Bean 주입을 요청하는 애노테이션 |
| 생성자 주입 | 생성자를 통해 의존성을 전달받는 방식 |
| 필드 주입 | 필드에 직접 Bean을 주입하는 방식 |
| Setter 주입 | Setter 메서드를 통해 Bean을 주입하는 방식 |
| `@RequiredArgsConstructor` | `final` 필드 등을 대상으로 생성자를 자동 생성하는 Lombok 애노테이션 |

## 핵심 요약

객체를 직접 관리하는 경우에는 다음과 같은 작업을 개발자가 직접 해야 한다.

| 개발자가 관리해야 하는 항목 |
|---|
| 객체 생성 |
| 생성 순서 |
| 생성자 인자 |
| 객체 간 의존 관계 |
| 객체 생명주기 |

Spring에서는 이 역할을 Spring Container가 담당한다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
}
```

이 코드에서 개발자가 표현하는 것은 `UserService`가 `UserRepository`를 필요로 한다는 사실이다.

`UserRepository` 객체의 생성, 검색, 전달, Bean 관리는 Spring Container가 처리한다.

따라서 핵심은 다음과 같이 정리할 수 있다.

| 항목 | 정리 |
|---|---|
| Bean | Spring이 관리하는 객체 |
| DI | 필요한 Bean을 다른 Bean에 주입하는 것 |
| `@Autowired` | Bean 자동 주입을 지정하는 방법 |
| 생성자 주입 | 일반적으로 권장되는 의존성 주입 방식 |
| `@RequiredArgsConstructor` | 생성자 주입 코드를 간소화하는 Lombok 기능 |
