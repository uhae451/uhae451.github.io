---
title: "JPA 모델링의 판단 기준 — 엔티티, 값 타입, 상속을 어떻게 나눌까"
date: 2026-11-01 09:00:00 +0900
categories: ["Courses","JPA기본"]
tags: ["Courses","JPA기본"]
description: "엔티티와 값 타입의 구분, 상속 매핑 선택, DB 제약과 스키마 변경의 판단 기준을 정리한다."
toc: true
---

`@Entity`, `@Embedded`, `@Inheritance`를 각각 익히면 문법은 이해할 수 있습니다. 하지만 실제 설계에서는 같은 요구사항을 여러 방식으로 표현할 수 있어 선택이 더 어렵습니다.

도서관의 주소를 별도 엔티티로 만들지, 자료의 종류를 상속으로 표현할지, 공통 필드를 부모 클래스로 올릴지는 어노테이션 이름만으로 결정되지 않습니다. **독립적으로 추적할 대상인지, 함께 바뀌는 값인지, 다형성이 필요한지**를 먼저 판단해야 합니다.

> 기준: Java 17, Jakarta Persistence 3.1, Hibernate 6 계열. 예제는 매핑의 핵심 부분이며 import, 일부 접근자와 생성자는 생략했습니다. 이 글의 코드와 DDL은 실행 결과가 아닌 설계 예시입니다.

## 1. 테이블을 만들기 전에 식별해야 할 대상을 정한다

책의 제목이 바뀌어도 같은 책으로 추적해야 합니다. 반면 주소는 도시와 도로명 등이 같으면 같은 값으로 취급할 수 있습니다. 이 차이를 먼저 구분합니다.

| 질문 | 엔티티 쪽에 가까운 경우 | 값 타입 쪽에 가까운 경우 |
|---|---|---|
| 값이 바뀌어도 같은 대상으로 추적해야 하나? | 대출, 회원, 도서 | 주소, 기간, 금액 |
| 별도 조회·권한·수정 API가 필요한가? | 대출 이력 상세 조회 | 회원의 현재 주소 교체 |
| 다른 객체가 특정 대상을 참조해야 하나? | 특정 대출에 대한 연체 처리 | 같은 주소 값을 복사해 사용 |

주소도 요구사항에 따라 엔티티가 될 수 있습니다. 회원의 현재 주소라면 값으로 충분하지만, 배송지 ID로 기본 배송지를 선택하고 개별 수정해야 한다면 `SavedAddress`라는 엔티티가 자연스럽습니다.

따라서 “주소는 무조건 값 타입”이 아니라 **그 주소를 서비스에서 어떻게 추적하는가**가 기준입니다.

## 2. 엔티티의 ID와 업무상 유일성은 다르다

```java
@Entity
@Table(name = "book", uniqueConstraints = {
    @UniqueConstraint(name = "uk_book_catalog_code", columnNames = "catalog_code")
})
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "catalog_code", nullable = false, length = 40)
    private String catalogCode;

    @Column(nullable = false, length = 200)
    private String title;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private BookStatus status;

    protected Book() {}
}
```

여기서 `id`는 내부 식별자이고 `catalogCode`는 업무상 중복되면 안 되는 자료 코드입니다. 코드는 변경될 수 있어도 내부 ID를 유지하면 다른 테이블의 참조를 안정적으로 관리할 수 있습니다.

ISBN을 무조건 유일 키로 쓰지 않은 이유도 있습니다. 도서관에서 같은 판본을 여러 권 보유한다면 ISBN은 같아도 실물 도서는 다릅니다. 이 경우 서지 정보와 실물 도서를 분리하거나, 적어도 실물별 등록 번호를 따로 두어야 합니다. **제약 조건은 필드 이름이 아니라 업무에서 무엇을 한 건으로 보는지에 맞춰야 합니다.**

위 매핑은 단순화를 위해 `Book`을 등록 자료 한 건으로 취급합니다. 실제 프로젝트에서 서지와 실물을 구분한다면 모델과 유일성 제약도 함께 바뀝니다.

### nullable과 유효성 검증은 역할이 다르다

`@Column(nullable = false, length = 200)`은 매핑과 스키마의 의도를 나타냅니다. 이것만으로 사용자에게 적절한 검증 오류 응답까지 만들어지지는 않습니다.

- 입력 DTO의 `@NotBlank`, `@Size`는 요청 경계에서 잘못된 값을 설명하기 좋습니다.
- 엔티티의 변경 메서드는 업무 규칙을 유지합니다.
- DB의 NOT NULL·UNIQUE·FK는 다른 경로의 쓰기와 동시 요청까지 포함한 최종 제약입니다.

중복 여부를 먼저 SELECT한 뒤 INSERT하는 방식만으로는 동시 요청의 중복을 막을 수 없습니다. DB 유일성 제약과 위반 처리까지 있어야 합니다. 계층별 검증은 단순한 중복 코드가 아니라 막는 실패 상황이 다릅니다.

### enum은 저장 표현을 변경 계약으로 본다

`ORDINAL`은 선언 순서를 저장하므로 항목 삽입이나 순서 변경에 취약합니다. 명시적인 `STRING` 매핑은 이 문제를 줄이지만, enum 이름을 바꾸면 저장된 문자열과의 호환성이 깨질 수 있습니다.

외부 코드와 독립된 저장 값이 필요하면 `AttributeConverter`로 명시적인 코드를 매핑하는 대안도 있습니다. 이 경우 `@Enumerated`와 함께 사용하는 방식으로 설계하지 않고, 변환 규칙과 미지원 값 처리 정책을 정합니다.

판단 기준은 “문자열이면 안전하다”가 아니라 **기존 데이터가 남아 있는 상태에서 Java 코드가 바뀌어도 해석 가능한가**입니다.

## 3. 값 타입은 컬럼을 묶는 것보다 변경을 통제하는 데 의미가 있다

회원의 주소를 세 개의 문자열로 관리하면 주소 검증과 표시 로직이 여러 곳에 흩어집니다. `Address`로 묶으면 주소라는 개념에 행동을 모을 수 있습니다.

```java
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;

    protected Address() {}

    public Address(String city, String street, String zipcode) {
        this.city = Objects.requireNonNull(city);
        this.street = Objects.requireNonNull(street);
        this.zipcode = Objects.requireNonNull(zipcode);
    }

    public Address withCity(String city) {
        return new Address(city, street, zipcode);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address other)) return false;
        return Objects.equals(city, other.city)
            && Objects.equals(street, other.street)
            && Objects.equals(zipcode, other.zipcode);
    }

    @Override
    public int hashCode() {
        return Objects.hash(city, street, zipcode);
    }
}
```

`java.util.Objects`를 사용합니다. setter 없이 교체 메서드가 새 값을 반환하도록 만들어, 애플리케이션 코드에서는 값을 수정할 수 없게 했습니다. 실제 주소 검증은 서비스가 지원하는 국가와 형식에 맞춰 추가해야 합니다.

```java
// Reader의 일부
@Embedded
private Address address;

public void moveTo(Address address) {
    this.address = Objects.requireNonNull(address);
}
```

`@Embedded`는 기본적으로 소유 엔티티의 테이블에 컬럼을 펼쳐 저장합니다. Address 클래스를 추가했다고 address 테이블이 생기는 것은 아닙니다.

### 왜 setter를 없애는가?

두 회원에게 같은 가변 `Address` 인스턴스를 넣은 뒤 한 회원의 주소 내부를 수정하면, 자바 메모리에서는 다른 회원이 바라보는 주소도 바뀝니다. 두 회원이 모두 관리 중이면 의도하지 않은 변경까지 저장될 수 있습니다.

주소 변경을 `reader.moveTo(new Address(...))`로 표현하면 수정 범위가 분명해집니다. 단순히 setter 사용을 금지하는 취향이 아니라, 공유 참조로 생기는 변경 전파를 차단하려는 설계입니다.

값 비교를 제공하는 것도 같은 이유입니다. `Set<Address>`에 넣거나 기존 주소를 찾아 제거할 때 객체 주소가 아닌 값의 동일성으로 판단해야 합니다. 컬렉션에 들어간 뒤 동등성 기준이 바뀌지 않도록 값 타입을 불변으로 다루는 것이 중요합니다.

## 4. 값 타입 컬렉션을 언제 엔티티로 바꿀까?

회원의 선호 장르처럼 작은 값 목록에는 `@ElementCollection`이 간단합니다.

```java
@ElementCollection
@CollectionTable(name = "reader_genre",
    joinColumns = @JoinColumn(name = "reader_id"))
@Column(name = "genre", nullable = false)
private Set<String> favoriteGenres = new HashSet<>();
```

반면 주소 변경 이력에 다음 요구사항이 생겼다고 합시다.

- 이력 한 건을 ID로 조회한다.
- 변경 일시와 변경 주체를 보관한다.
- 과거 기록을 다른 업무에서 참조한다.
- 특정 한 건만 정정한다.

이제 주소 값 외에 추적할 대상이 생겼습니다. `AddressHistory` 엔티티가 `Address` 값을 포함하도록 설계하는 편이 적합합니다.

```java
@Entity
public class AddressHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    private Address address;

    private LocalDateTime changedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "reader_id", nullable = false)
    private Reader reader;

    protected AddressHistory() {}
}
```

값 컬렉션 변경 시 발생하는 SQL은 List/Set, 순서 컬럼, 값 구성과 구현체에 따라 달라집니다. “항상 전체 삭제 후 재등록”이나 “모든 값 컬렉션은 중복을 허용하지 않는다”로 일반화하면 안 됩니다. 다만 별도 엔티티 ID가 없다는 특성 때문에 개별 항목의 추적과 갱신이 복잡해질 수 있습니다.

이력 엔티티로 바꾸는 이유도 무조건 성능이 좋아서가 아닙니다. **독립 식별과 변경 정책이 필요해졌기 때문**이고, 성능은 생성 SQL과 실제 데이터 크기로 확인합니다.

## 5. 공통 필드가 있다는 이유만으로 엔티티 상속을 쓰지 않는다

책과 DVD에 `title`이 있다고 해서 바로 `Media` 엔티티를 부모로 만들 필요는 없습니다. 먼저 서비스가 `Media`라는 대상을 공통으로 다루는지 봅니다.

예를 들어 대출이 책과 DVD를 같은 규칙으로 참조하고, 전체 자료 검색을 부모 타입으로 수행해야 한다면 다형적 엔티티 모델이 의미가 있습니다. 반면 두 서비스가 독립적이고 단지 필드가 겹치는 정도라면 상속은 조회와 저장 구조에 불필요한 결합을 만들 수 있습니다.

### 엔티티 상속, MappedSuperclass, Embedded의 차이

| 방법 | 해결하려는 문제 | 부모·값 자체의 조회 |
|---|---|---|
| 엔티티 상속 | 공통 식별 체계 아래 여러 하위 타입을 다형적으로 다룸 | 부모 엔티티를 조회 가능 |
| `@MappedSuperclass` | 공통 매핑 정보를 상속해서 재사용 | 부모는 엔티티 조회 대상이 아님 |
| `@Embedded` | 함께 의미를 갖는 속성과 행동을 값으로 묶음 | 독립 엔티티로 조회하지 않음 |

`createdAt`, `updatedAt`을 재사용하기 위해 엔티티 상속 테이블을 만들 필요는 없습니다.

```java
@MappedSuperclass
public abstract class BaseEntity {
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

이 선언은 매핑만 물려줍니다. 시간을 자동으로 채워 주지는 않습니다. 생성·수정 시각을 언제 기록할지는 콜백이나 Spring Data Auditing 등 별도 정책이 필요합니다.

## 6. 상속 전략은 조회 패턴과 DB 제약으로 선택한다

다형성이 필요하다고 결정했다면 그다음에 테이블 전략을 선택합니다.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Media {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    protected Media() {}
}

@Entity
public class Book extends Media {
    private String isbn;
    protected Book() {}
}
```

이 코드는 앞의 독립 `Book`과는 다른 모델링 대안입니다.

| 전략 | 테이블 구성 | 검토할 비용 |
|---|---|---|
| `JOINED` | 공통 부모 테이블 + 하위 타입 테이블 | 저장·조회에 여러 테이블이 참여 |
| `SINGLE_TABLE` | 모든 타입을 한 테이블에 저장 | 넓은 행, 타입별 제약 표현, 인덱스 설계 |
| `TABLE_PER_CLASS` | 구체 타입마다 공통 필드를 포함한 테이블 | 부모 조회의 UNION, 공통 참조와 ID 전략 |

`JOINED`는 하위 타입 전용 테이블에 필수 컬럼을 두기 좋습니다. 대신 부모 타입 검색이 자주 일어나고 하위 타입이 많으면 조인 비용을 검토해야 합니다.

`SINGLE_TABLE`은 조인을 줄일 수 있지만 “무조건 빠르다”는 결론은 이릅니다. 테이블 폭, 인덱스, 데이터 분포와 실제 실행 계획이 영향을 줍니다. 하위 타입 전용 필드는 다른 타입 행에서는 비어 있어야 하므로 단순한 NOT NULL을 걸기 어렵고, 필요하면 타입을 고려한 CHECK 제약 등을 검토합니다.

`TABLE_PER_CLASS`는 타입별 조회가 독립적일 때 고려할 여지가 있지만, 부모 타입으로 통합 조회하거나 다른 테이블에서 모든 자료를 하나의 FK로 참조하려면 복잡해집니다. Hibernate에서 이 전략과 IDENTITY를 함께 쓰는 제약도 있어 ID 전략까지 검토해야 합니다.

### 전략 변경은 어노테이션 한 줄로 끝나지 않는다

Java 코드에서 전략을 바꾸는 작업은 작아 보입니다. 하지만 운영 데이터는 그렇지 않습니다. 테이블 분리·통합, 기존 FK 이동, 인덱스 재설계, 마이그레이션 중 구버전 애플리케이션과의 호환성이 필요할 수 있습니다.

그래서 “일단 JOINED로 시작하고 느리면 바꾸자”를 보편 규칙으로 삼기 어렵습니다. 초기에도 대표 조회 몇 가지와 데이터 증가 방향을 정리해 선택 근거를 남기는 편이 좋습니다.

## 7. 매핑과 스키마 변경을 함께 검토한다

로컬 실습에서 `ddl-auto=create`는 편리하지만 운영에서는 버전이 있는 마이그레이션 스크립트로 변경을 관리하는 것이 검토와 복구에 유리합니다. `validate`는 매핑 불일치 탐지에 도움이 되지만 모든 제약이나 인덱스, 데이터 호환성을 검증하는 도구는 아닙니다.

`update`도 “컬럼 추가만 하므로 안전하다”고 볼 수 없습니다. 구현체와 스키마 차이에 따라 변경 SQL이 달라지고, 데이터 변환이나 배포 순서를 대신 설계해 주지 않습니다.

새 매핑을 검토할 때는 다음 내용을 짝지어 확인합니다.

| 객체 모델에서의 변경 | 함께 검토할 DB·운영 항목 |
|---|---|
| nullable 필드를 필수로 바꿈 | 기존 NULL 데이터 보정과 제약 적용 순서 |
| enum 이름 변경 | 기존 저장 값 변환, 구버전과의 호환 |
| 값 타입을 엔티티로 분리 | 새 ID, FK, 데이터 이관 |
| 상속 전략 변경 | 테이블 재구성, 조회 계획, 배포 절차 |

설계가 전문적으로 보이는 지점은 어노테이션의 수가 아니라, 이 변경이 데이터에 미치는 영향을 설명할 수 있다는 데 있습니다.

**확인 질문:** 주소 변경 이력에 ID 기반 상세 조회와 개별 정정 기능이 생겼다면, `List<Address>`를 유지할지 엔티티로 바꿀지 어떤 근거로 판단할까요?

참고: [Jakarta Persistence 3.1의 엔티티·값 타입·상속 매핑 정의](https://jakarta.ee/specifications/persistence/3.1/jakarta-persistence-spec-3.1), [Hibernate 6.6 매핑 가이드](https://docs.hibernate.org/orm/6.6/userguide/html_single/).

**오늘 배운 것 한 줄 요약:** JPA 모델링은 식별과 변경의 요구를 먼저 나누고, 조회 비용과 데이터 제약을 만족하는 매핑을 선택하는 일이다.
