---
title: "JPA 기초 정리"
date: 2026-10-31 09:00:00 +0900
categories: ["JPA", "Core"]
tags: ["jpa", "hibernate", "entity", "entity-manager", "jpql"]
description: "JPA를 사용할 때 알아야 할 핵심 개념과 엔티티, 영속성 컨텍스트, 매핑 및 조회 방법을 정리한다."
toc: true
---

JPA는 Java 애플리케이션에서 관계형 데이터베이스를 객체 중심으로 다룰 수 있도록 제공하는 표준 기술이다.

핵심은 **Java 객체와 데이터베이스 테이블을 매핑하고, Entity를 기준으로 데이터를 저장하고 조회할 수 있도록 해주는 것**이다.

예를 들어 주문 데이터를 조회할 때 다음과 같이 작성할 수 있다.

```java
Order order = orderRepository.findById(1L)
        .orElseThrow();
```

개발자는 `Order`라는 Java 객체를 중심으로 코드를 작성하고, JPA는 해당 작업에 필요한 데이터베이스 접근을 처리한다.

---

## 1. JPA란?

JPA는 `Java Persistence API`의 약자다.

Java 객체와 관계형 데이터베이스 사이의 데이터를 연결하고 관리하기 위해 정의된 표준이다.

JPA 자체는 실제 동작을 구현한 라이브러리가 아니라 **규칙과 인터페이스를 정의한 표준**이다.

따라서 실제로 JPA를 사용하려면 JPA 표준을 구현한 구현체가 필요하다.

대표적인 JPA 구현체가 Hibernate다.

Spring Boot에서는 Spring Data JPA를 함께 사용하는 경우가 많다.

| 구성 요소           | 역할                                   |
| --------------- | ------------------------------------ |
| Spring Data JPA | JPA를 편리하게 사용할 수 있도록 Repository 기능 제공 |
| JPA             | 객체와 관계형 데이터베이스 매핑을 위한 Java 표준        |
| Hibernate       | JPA 표준을 실제로 구현한 대표적인 구현체             |
| Database        | 실제 데이터 저장                            |

개발자가 자주 사용하는 `JpaRepository`는 Spring Data JPA가 제공하고, 실제 JPA 기능은 Hibernate 같은 구현체가 처리한다.

---

## 2. ORM

JPA를 이해하려면 ORM 개념을 알아야 한다.

ORM은 `Object Relational Mapping`의 약자다.

즉, **Java 객체와 관계형 데이터베이스 테이블을 서로 매핑하는 기술**이다.

Java 객체와 데이터베이스의 관계를 정리하면 다음과 같다.

| Java 객체          | Database          |
| ---------------- | ----------------- |
| `Order` 클래스      | `orders` 테이블      |
| `id` 필드          | `id` 컬럼           |
| `productName` 필드 | `product_name` 컬럼 |
| `quantity` 필드    | `quantity` 컬럼     |

예를 들어 다음과 같은 객체가 있다고 가정한다.

```java
public class Order {

    private Long id;
    private String productName;
    private Integer quantity;
}
```

데이터베이스에는 다음과 같은 데이터가 존재할 수 있다.

| id | product_name | quantity |
| -: | ------------ | -------: |
|  1 | Keyboard     |        2 |

JPA는 Entity에 선언된 매핑 정보를 이용해 Java 객체와 데이터베이스 데이터를 연결한다.

---

## 3. Entity

JPA에서 데이터베이스 테이블과 매핑되어 관리되는 Java 객체를 **Entity**라고 한다.

주문 Entity를 다음과 같이 작성할 수 있다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name")
    private String productName;

    private Integer quantity;

    public Order(String productName, Integer quantity) {
        this.productName = productName;
        this.quantity = quantity;
    }
}
```

Entity를 작성할 때 자주 사용하는 어노테이션을 하나씩 살펴보면 다음과 같다.

### `@Entity`

```java
@Entity
public class Order {}
```

`@Entity`를 선언하면 JPA가 해당 클래스를 **관리할 Entity로 인식**한다.

즉, 일반 Java 클래스에 `@Entity`를 선언하면 JPA가 데이터베이스와 매핑되는 객체로 관리할 수 있다.

---

### `@NoArgsConstructor`

JPA Entity에는 **매개변수가 없는 기본 생성자**가 필요하다.

Lombok의 `@NoArgsConstructor`를 사용하면 기본 생성자를 자동으로 생성할 수 있다.

```java
@NoArgsConstructor
public class Order {}
```

이는 개념적으로 다음 생성자를 만드는 것과 같다.

```java
public Order() {}
```

Entity에서는 보통 다음과 같이 사용한다.

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
```

이 경우 다음과 같은 생성자가 만들어진다.

```java
protected Order() {}
```

JPA Entity의 기본 생성자는 `public` 또는 `protected`여야 한다.

보통 `protected`를 사용하는 이유는 JPA가 Entity를 생성할 수 있도록 하면서, 외부 코드에서 다음과 같은 빈 객체 생성을 제한하기 위해서다.

```java
new Order();
```

대신 실제 객체 생성에는 필요한 값을 받는 생성자를 사용한다.

```java
Order order = new Order("Keyboard", 2);
```

---

### `@Id`

모든 Entity에는 식별자가 필요하다.

```java
@Id
private Long id;
```

`@Id`가 선언된 필드는 Entity를 구분하는 기준이 된다.

데이터베이스에서는 일반적으로 Primary Key와 연결된다.

예를 들어:

| id | product_name | quantity |
| -: | ------------ | -------: |
|  1 | Keyboard     |        2 |
|  2 | Mouse        |        1 |

각 주문은 `id` 값으로 구분된다.

---

### `@GeneratedValue`

식별자를 자동으로 생성하도록 설정할 때 사용한다.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

`GenerationType.IDENTITY`를 사용하면 데이터베이스가 ID 값을 생성하도록 맡긴다.

따라서 새로운 Entity를 만들 때 ID를 직접 전달하지 않아도 된다.

```java
Order order = new Order("Keyboard", 2);
```

저장 전에는 `id`가 `null`일 수 있다.

```text
id = null
productName = "Keyboard"
quantity = 2
```

저장 이후에는 데이터베이스에서 생성된 값이 Entity에 반영될 수 있다.

```text
id = 1
productName = "Keyboard"
quantity = 2
```

---

### `@Table`

Entity와 매핑할 테이블 이름을 직접 지정할 때 사용한다.

```java
@Entity
@Table(name = "orders")
public class Order {}
```

이 경우 `Order` Entity는 `orders` 테이블과 매핑된다.

---

### `@Column`

Entity의 필드와 데이터베이스 컬럼 이름이 다른 경우 사용할 수 있다.

```java
@Column(name = "product_name")
private String productName;
```

매핑 관계는 다음과 같다.

| Entity        | Database       |
| ------------- | -------------- |
| `Order`       | `orders`       |
| `id`          | `id`           |
| `productName` | `product_name` |
| `quantity`    | `quantity`     |

필드명과 컬럼명이 자동 매핑 가능한 경우 `@Column`을 반드시 작성할 필요는 없다.

---

## 4. Repository

Entity를 만들었다면 해당 Entity를 저장하고 조회할 기능이 필요하다.

Spring Data JPA에서는 `JpaRepository`를 사용할 수 있다.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {}
```

여기서:

```java
JpaRepository<Order, Long>
```

에는 두 가지 타입 정보가 들어간다.

| 타입      | 의미                     |
| ------- | ---------------------- |
| `Order` | Repository가 관리할 Entity |
| `Long`  | Entity의 ID 타입          |

따라서 `OrderRepository`는 `Order` Entity를 저장하고 조회하는 Repository다.

---

## 5. 기본 CRUD

`JpaRepository`를 상속하면 기본적인 CRUD 기능을 사용할 수 있다.

### 저장

```java
orderRepository.save(order);
```

### ID 조회

```java
orderRepository.findById(id);
```

### 전체 조회

```java
orderRepository.findAll();
```

### 삭제

```java
orderRepository.delete(order);
```

### 존재 여부 확인

```java
orderRepository.existsById(id);
```

대표적인 메서드는 다음과 같다.

| 메서드            | 역할             |
| -------------- | -------------- |
| `save()`       | Entity 저장      |
| `findById()`   | ID 기준 단건 조회    |
| `findAll()`    | 전체 Entity 조회   |
| `delete()`     | Entity 삭제      |
| `existsById()` | ID 기준 존재 여부 확인 |

---

## 6. Entity 저장

주문 Entity를 생성하고 저장한다고 가정한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;

    @Transactional
    public void createOrder() {

        Order order = new Order("Keyboard", 2);

        orderRepository.save(order);
    }
}
```

저장 과정은 다음과 같다.

| 순서 | 처리                               |
| -- | -------------------------------- |
| 1  | `Order` Entity 생성                |
| 2  | `orderRepository.save(order)` 호출 |
| 3  | JPA가 Entity의 매핑 정보 확인            |
| 4  | Hibernate가 저장 작업 수행              |
| 5  | 데이터베이스에 주문 데이터 저장                |

개발자는 Entity를 생성한 뒤 Repository에 전달하면 된다.

---

## 7. Entity 조회

ID가 `1`인 주문을 조회한다고 가정한다.

```java
Order order = orderRepository
        .findById(1L)
        .orElseThrow();
```

조회 과정은 다음과 같다.

| 순서 | 처리                        |
| -- | ------------------------- |
| 1  | `findById(1L)` 호출         |
| 2  | Repository가 조회 요청 처리      |
| 3  | JPA가 Entity 매핑 정보 확인      |
| 4  | 데이터베이스에서 주문 데이터 조회        |
| 5  | 조회 결과를 `Order` Entity로 반환 |

데이터베이스에 다음 데이터가 존재한다고 가정한다.

| id | product_name | quantity |
| -: | ------------ | -------: |
|  1 | Keyboard     |        2 |

Java에서는 다음과 같은 Entity로 다룰 수 있다.

```text
Order

id = 1
productName = "Keyboard"
quantity = 2
```

---

## 8. `findById()`와 Optional

`findById()`의 반환 타입은 `Order`가 아니라 `Optional<Order>`다.

```java
Optional<Order> order = orderRepository.findById(1L);
```

요청한 ID에 해당하는 Entity가 존재하지 않을 수도 있기 때문이다.

보통 다음과 같이 사용한다.

```java
Order order = orderRepository
        .findById(id)
        .orElseThrow();
```

실제 애플리케이션에서는 의미 있는 예외를 지정할 수 있다.

```java
Order order = orderRepository
        .findById(id)
        .orElseThrow(
                OrderNotFoundException::new
        );
```

이렇게 하면 주문이 존재하지 않는 상황을 명확한 예외로 표현할 수 있다.

---

## 9. Query Method

Spring Data JPA에서는 Repository의 메서드 이름을 기준으로 조회 조건을 만들 수 있다.

예를 들어 주문 상태가 있다고 가정한다.

```java
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

Repository에는 다음 메서드를 선언할 수 있다.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByStatus(OrderStatus status);
}
```

사용할 때는 다음과 같다.

```java
List<Order> orders = orderRepository.findByStatus(OrderStatus.PAID);
```

Spring Data JPA는 `findByStatus`라는 메서드 이름을 분석해 `status`를 기준으로 조회하는 기능을 만든다.

이러한 방식을 **Query Method**라고 한다.

대표적인 형태는 다음과 같다.

| 메서드                       | 의미               |
| ------------------------- | ---------------- |
| `findByStatus(status)`    | 상태 기준 조회         |
| `findByProductName(name)` | 상품명 기준 조회        |
| `existsByStatus(status)`  | 해당 상태의 데이터 존재 여부 |
| `countByStatus(status)`   | 해당 상태의 데이터 개수    |

---

## 10. JPA와 Spring Data JPA

JPA와 Spring Data JPA는 같은 개념이 아니다.

### JPA

JPA는 객체와 관계형 데이터베이스를 매핑하고 관리하기 위한 Java 표준이다.

주요 개념은 다음과 같다.

| 개념                  | 역할              |
| ------------------- | --------------- |
| Entity              | DB 테이블과 매핑되는 객체 |
| EntityManager       | Entity 저장·조회·관리 |
| Persistence Context | Entity를 관리하는 공간 |
| Entity Lifecycle    | Entity의 상태 변화   |

### Spring Data JPA

Spring Data JPA는 JPA를 Spring에서 더 편리하게 사용할 수 있도록 제공하는 기술이다.

대표적인 기능이 `JpaRepository`다.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {}
```

각 구성 요소의 역할을 정리하면 다음과 같다.

| 구성 요소           | 역할                       |
| --------------- | ------------------------ |
| Spring Data JPA | Repository 추상화와 편의 기능 제공 |
| JPA             | 객체와 관계형 데이터베이스 관리 표준     |
| Hibernate       | JPA 표준의 대표적인 구현체         |
| Database        | 실제 데이터 저장                |

---

## 11. Entity와 DTO의 차이

Entity와 DTO는 역할이 다르다.

Entity는 데이터베이스와 연결되고 JPA가 관리하는 객체다.

```java
@Entity
public class Order {}
```

DTO는 데이터를 전달하기 위한 객체다.

```java
public record OrderResponse(Long id, String productName, Integer quantity) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(order.getId(), order.getProductName(), order.getQuantity());
    }
}
```

차이를 정리하면 다음과 같다.

| 구분        | Entity        | DTO      |
| --------- | ------------- | -------- |
| 주요 목적     | 데이터베이스 데이터 관리 | 데이터 전달   |
| JPA 관리 여부 | 관리 대상         | 관리 대상 아님 |
| DB 테이블 매핑 | 있음            | 없음       |
| API 요청/응답 | 직접 사용은 지양     | 주로 사용    |

조회한 Entity를 API 응답으로 전달할 때는 다음과 같이 DTO로 변환할 수 있다.

```java
Order order = orderRepository.findById(id).orElseThrow();

OrderResponse response = OrderResponse.from(order);
```

---

## 12. JPA의 기본 처리 구조

Spring Data JPA를 사용하는 애플리케이션의 각 구성 요소는 다음 역할을 담당한다.

| 구성 요소           | 담당 역할                |
| --------------- | -------------------- |
| Service         | 비즈니스 로직 수행           |
| Repository      | Entity 저장·조회 요청      |
| Spring Data JPA | Repository 기능 제공     |
| JPA             | Entity 관리와 ORM 표준 제공 |
| Hibernate       | JPA 기능 실제 구현         |
| Database        | 데이터 저장               |

예를 들어 다음 코드를 실행한다고 가정한다.

```java
orderRepository.findById(1L);
```

처리 과정은 다음과 같다.

| 순서 | 처리                               |
| -- | -------------------------------- |
| 1  | Service가 `OrderRepository` 호출    |
| 2  | Repository가 조회 작업 요청             |
| 3  | JPA와 Hibernate가 Entity 매핑 정보를 확인 |
| 4  | 데이터베이스에서 주문 데이터 조회               |
| 5  | 조회 결과를 `Order` Entity로 반환        |

개발자는 주로 Entity와 Repository를 중심으로 데이터를 다루게 된다.

---

## 정리

JPA는 **Java 객체와 관계형 데이터베이스를 매핑하고 관리하기 위한 Java 표준**이다.

Entity와 데이터베이스의 기본 매핑 관계는 다음과 같다.

| Java Entity   | Database       |
| ------------- | -------------- |
| `Order`       | `orders` 테이블   |
| `id`          | Primary Key    |
| `productName` | `product_name` |
| `quantity`    | `quantity`     |

Entity에서 자주 사용하는 설정은 다음과 같다.

| 어노테이션                | 역할                        |
| -------------------- | ------------------------- |
| `@Entity`            | JPA가 관리할 Entity 지정        |
| `@NoArgsConstructor` | JPA Entity에 필요한 기본 생성자 생성 |
| `@Id`                | Entity 식별자 지정             |
| `@GeneratedValue`    | 식별자 생성 전략 설정              |
| `@Table`             | 테이블 매핑 설정                 |
| `@Column`            | 컬럼 매핑 설정                  |

Spring Data JPA에서는 `JpaRepository`를 이용해 기본적인 저장과 조회 기능을 사용할 수 있다.

| 메서드          | 역할        |
| ------------ | --------- |
| `save()`     | Entity 저장 |
| `findById()` | ID 기준 조회  |
| `findAll()`  | 전체 조회     |
| `delete()`   | Entity 삭제 |

JPA를 처음 공부할 때는 **Entity가 무엇인지, Entity와 테이블이 어떻게 매핑되는지, Repository를 통해 어떻게 저장하고 조회하는지**를 먼저 이해하는 것이 중요하다.
