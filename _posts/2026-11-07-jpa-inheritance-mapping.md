---
title: "JPA 상속 매핑 — JOINED, SINGLE_TABLE, TABLE_PER_CLASS와 @MappedSuperclass"
date: 2026-11-07 16:00:00 +0900
categories: [JPA, Core]
tags: [jpa, hibernate, inheritance, mapped-superclass, orm]
description: "JPA 상속 매핑의 세 가지 전략인 JOINED, SINGLE_TABLE, TABLE_PER_CLASS의 테이블 구조와 장단점, 선택 기준 및 @MappedSuperclass와의 차이를 정리한다."
toc: true
---

관계형 데이터베이스에는 객체지향 언어의 상속 개념이 없다. 따라서 `Media`를 상속받는 `Book`, `Dvd`, `Magazine`을 테이블로 표현하려면 **상속 매핑 전략**을 선택해야 한다.

## 1. 문제: 객체 상속을 테이블로 어떻게 표현할까?

도서관에는 책만 있는 것이 아니다. 대출 가능한 자료를 `Media`로 정의하고 `Book`, `Dvd`, `Magazine`이 이를 상속한다고 가정한다.

- 공통 속성: `Media.title`
- 고유 속성: `Book.isbn`, `Dvd.runningTime`

JPA는 세 가지 상속 매핑 전략을 지원하며, `@Inheritance(strategy = ...)`로 선택한다.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "dtype")
public abstract class Media {
    @Id @GeneratedValue
    private Long id;
    private String title;
}

@Entity
@DiscriminatorValue("BOOK")
public class Book extends Media {
    private String isbn;
}

@Entity
@DiscriminatorValue("DVD")
public class Dvd extends Media {
    private int runningTime;
}
```

![Media, Book, Dvd 상속 구조](https://velog.velcdn.com/images/uhae451/post/26830efe-0dea-42ef-96cb-a0ec642958aa/image.png)

### 전략 1. JOINED (조인 전략)

공통 컬럼은 `media`, 자식 고유 컬럼은 `book`, `dvd` 테이블에 나눠 저장한다. 자식 테이블의 기본 키(PK)는 부모 테이블의 기본 키를 참조하는 외래 키(FK)이기도 하다.

![JOINED 전략의 테이블 구조](https://velog.velcdn.com/images/uhae451/post/01fc9f25-eb90-4182-8e3e-3bedf0f2151b/image.png)

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "dtype") // 선택 사항
public abstract class Media {
    @Id @GeneratedValue
    private Long id;
    private String title;
}
```

- **장점:** 테이블이 정규화되고 자식 고유 컬럼에 `NOT NULL` 제약을 적용하기 쉽다. 중복 데이터가 적다.
- **단점:** 자식 엔티티 저장 시 일반적으로 부모·자식 테이블에 각각 `INSERT`가 발생한다. 조회할 때도 조인이 필요하다.
- **구분 컬럼:** `JOINED`에서는 `@DiscriminatorColumn`이 필수가 아니다. 명시하면 부모 테이블에 타입을 구분하는 컬럼을 둘 수 있다.

### 전략 2. SINGLE_TABLE (단일 테이블 전략)

모든 하위 타입의 컬럼을 `media` 테이블 하나에 저장하고, 구분 컬럼으로 엔티티 타입을 식별한다. **상속 전략을 생략했을 때의 JPA 기본값**이다.

![SINGLE_TABLE 전략의 테이블 구조](https://velog.velcdn.com/images/uhae451/post/f1ed5ba0-443c-4c81-89d9-94a963c031d2/image.png)

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype") // Hibernate는 생략해도 기본 DTYPE 사용
public abstract class Media {
    @Id @GeneratedValue
    private Long id;
    private String title;
}
```

- **장점:** 조회할 때 상속 관계를 위한 조인이 필요 없어 쿼리가 단순하다.
- **단점:** 다른 타입에서 사용하지 않는 자식 전용 컬럼에는 `NULL`이 발생한다. 하위 타입과 컬럼이 늘어나면 테이블이 커질 수 있다.
- **제약조건:** 자식 전용 컬럼에 단순한 `NOT NULL` 제약을 걸기 어렵다. 필요하면 구분 컬럼을 활용한 조건부 `CHECK` 등을 고려한다.
- **구분 컬럼:** Hibernate는 `@DiscriminatorColumn`을 생략해도 기본적으로 `DTYPE`을 사용한다.

### 전략 3. TABLE_PER_CLASS (구현 클래스마다 테이블)

부모 테이블을 만들지 않고 `book`, `dvd` 테이블마다 공통 속성까지 중복 저장한다. 여기서는 `Media`가 추상 클래스이므로 별도의 부모 테이블이 생성되지 않는다.

![TABLE_PER_CLASS 전략의 테이블 구조](https://velog.velcdn.com/images/uhae451/post/4809dcb5-0d3c-4fe5-abb2-dbd9c51d7d38/image.png)

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Media {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    private String title;
}
```

- **장점:** 구체적인 하위 타입 하나만 조회하거나 저장할 때 해당 테이블만 다루면 된다.
- **단점:** 부모 타입으로 다형성 조회를 수행하면 여러 자식 테이블을 `UNION`으로 결합할 수 있어 비용이 증가한다.
- **연관관계:** 부모 타입을 참조하는 외래 키 제약을 데이터베이스에서 표현하기 어렵다.
- **식별자:** Hibernate에서는 여러 구체 클래스에 걸쳐 식별자를 일관되게 생성해야 하므로 `IDENTITY` 방식과 호환되지 않는다. 지원되는 데이터베이스라면 `SEQUENCE` 등을 사용한다.

이러한 제약 때문에 일반적인 도메인 상속 구조에서는 우선순위가 낮다.

### 어떤 전략을 선택할까?

| 상황 | 고려할 전략 |
|---|---|
| 테이블 정규화와 무결성 제약이 중요하고 상속 구조가 복잡하다 | `JOINED` |
| 하위 타입이 적고 조회 단순성과 성능이 중요하다 | `SINGLE_TABLE` |
| 구체 타입별 테이블 분리가 반드시 필요하다 | `TABLE_PER_CLASS`를 제약 검토 후 사용 |

`JOINED`와 `SINGLE_TABLE` 중 하나를 **도메인 구조, 데이터 무결성, 조회 패턴**에 따라 결정한다. `JOINED`를 무조건 기본 선택으로 볼 필요는 없다.

또한 상속 전략 변경은 어노테이션만 바꾸면 끝나는 작업이 아니다. 실제로는 **기존 테이블과 데이터 구조를 변경하고 마이그레이션**해야 하므로 초기 설계 단계에서 선택 기준을 검토하는 것이 중요하다.

## 2. @MappedSuperclass: 엔티티 상속이 아니라 공통 매핑 정보 재사용

모든 엔티티에 `createdAt`, `updatedAt` 같은 공통 필드를 선언할 때 사용할 수 있다.

![MappedSuperclass를 활용한 공통 필드 매핑](https://velog.velcdn.com/images/uhae451/post/d768a98c-4271-4f51-bbb6-7ed38a48fc33/image.png)

```java
@MappedSuperclass
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Entity
public class Reader extends BaseEntity {
    @Id @GeneratedValue
    private Long id;
}
```

- `BaseEntity`는 **엔티티가 아니며 별도의 테이블도 생성되지 않는다.**
- 상속받는 `Reader` 테이블에 `createdAt`, `updatedAt` 컬럼이 매핑된다.
- `em.find(BaseEntity.class, ...)` 또는 `select b from BaseEntity b`처럼 부모 타입을 엔티티로 조회할 수 없다.
- 공통 매핑 정보 재사용을 목적으로 하므로 보통 `abstract` 클래스로 선언한다.

`@Inheritance`는 **의미 있는 부모 엔티티 타입과 다형성**을 데이터베이스에 매핑하는 기능이고, `@MappedSuperclass`는 **필드와 매핑 설정만 재사용**하는 기능이다.

## 3. 정리

- **상속 전략의 JPA 기본값은 `SINGLE_TABLE`**이다. `JOINED`는 정규화와 제약조건, `SINGLE_TABLE`은 단순한 조회 구조가 강점이다.
- `TABLE_PER_CLASS`는 다형성 조회에 `UNION`이 필요할 수 있어 사용 전에 제약을 검토한다.
- **공통 필드 재사용**만 필요하다면 `@MappedSuperclass`를 사용한다.
- 상속 전략은 변경 시 스키마·데이터 마이그레이션이 필요하므로 처음부터 조회 패턴과 무결성 요구를 고려한다.
