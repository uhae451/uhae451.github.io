---
title: "SOLID"
date: 2026-10-17 09:00:00 +0900
categories: ["Design", "Principles"]
tags: ["solid", "oop", "srp", "ocp", "dip"]
description: "객체지향 설계의 SOLID 다섯 원칙을 코드 예제와 함께 정리하고 각 원칙을 적용할 때의 판단 기준을 살펴본다."
toc: true
---

개발 공부를 다시 시작하면서 예전에 "그냥 외웠던" 개념들을 다시 붙잡고 있습니다. 그중에서도 SOLID는 진짜 애매하게 알고 있었던 대표적인 녀석이었어요. "단일 책임 원칙 = 클래스는 하나의 책임만 가져야 한다" 이렇게 문장은 외우는데, 막상 "그럼 이 코드에서 책임이 몇 개인가요?"라고 물어보면 말문이 막히더라고요.

그래서 이번 포스팅은 정의를 나열하는 글이 아니라, 나쁜 코드 -> 왜 나쁜지 -> 어떻게 고치는지 -> 그게 실무(스프링)에서 어떤 모습으로 나타나는지까지 끝까지 따라가보는 글로 써봤습니다. 면접에서 "SOLID 원칙 설명해주세요"라는 질문을 받았을 때, 정의만 읊는 게 아니라 코드 레벨에서 근거를 댈 수 있는 수준을 목표로 정리했습니다.

## 목차

1. SRP — 단일 책임 원칙
2. OCP — 개방-폐쇄 원칙
3. LSP — 리스코프 치환 원칙
4. ISP — 인터페이스 분리 원칙
5. DIP — 의존관계 역전 원칙
6. 다섯 원칙은 어떻게 서로 연결되는가
7. 면접 대비 정리

---

## 1. SRP (Single Responsibility Principle) — 단일 책임 원칙

### 정의부터 다시 보기

가장 많이 오해하는 원칙이 SRP입니다. "클래스는 하나의 기능만 해야 한다"고 잘못 외우는 경우가 많은데, 로버트 마틴이 실제로 정의한 문장은 이겁니다.

> "클래스를 변경하는 이유는 오직 하나여야 한다." (A class should have only one reason to change)

"기능 하나"가 아니라 "변경 이유 하나"라는 게 핵심입니다. 이 차이를 이해하지 못하면 SRP를 적용할 때마다 클래스를 무작정 잘게 쪼개는 실수를 하게 됩니다.

### 위반 코드부터 보겠습니다

```java
public class Employee {

    private String name;
    private double baseSalary;

    public Employee(String name, double baseSalary) {
        this.name = name;
        this.baseSalary = baseSalary;
    }

    // 1) 급여 계산 로직 — "비즈니스 정책"이 바뀌면 수정
    public double calculatePay() {
        return baseSalary * 1.1; // 세금 등 정책 반영
    }

    // 2) DB 저장 로직 — "저장 기술"이 바뀌면 수정
    public void save() {
        String sql = "INSERT INTO employee (name, salary) VALUES ('"
                + name + "', " + baseSalary + ")";
        // JDBC 커넥션 열고 실행...
        System.out.println("Executing: " + sql);
    }

    // 3) 리포트 출력 로직 — "출력 포맷"이 바뀌면 수정
    public String generateReport() {
        return "[Employee Report] " + name + " - " + calculatePay();
    }
}
```

언뜻 보면 "직원 관련 기능이니까 한 클래스에 있는 게 자연스럽지 않나?" 싶을 수 있는데, **이 클래스가 변경되는 이유를 나열**해보면 문제가 보입니다.

- 회계팀이 급여 정책을 바꾸면 → `calculatePay()` 수정
- DBA가 DB를 MySQL에서 PostgreSQL로 바꾸면 → `save()` 수정
- 기획팀이 리포트 양식을 바꾸면 → `generateReport()` 수정

**서로 다른 세 팀, 서로 다른 세 가지 이유**가 하나의 클래스를 건드립니다. 이게 SRP 위반입니다. 문제는 여기서 그치지 않습니다.

- 리포트 포맷을 고치다가 실수로 급여 계산 로직에 영향을 줄 수 있음 (책임이 섞여있으니 결합도가 높음)
- `Employee` 클래스를 테스트하려면 DB 연결까지 필요함 (단위 테스트가 무거워짐)
- 여러 개발자가 동시에 이 클래스를 수정하면 충돌(merge conflict)이 잦아짐

### 리팩토링

```java
// 1) 순수 도메인 객체 — 상태와 핵심 규칙만 가짐
public class Employee {
    private final String name;
    private final double baseSalary;

    public Employee(String name, double baseSalary) {
        this.name = name;
        this.baseSalary = baseSalary;
    }

    public String getName() { return name; }
    public double getBaseSalary() { return baseSalary; }
}

// 2) 급여 계산 책임만 담당
public class PayCalculator {
    public double calculate(Employee employee) {
        return employee.getBaseSalary() * 1.1;
    }
}

// 3) 저장 책임만 담당
public class EmployeeRepository {
    public void save(Employee employee) {
        String sql = "INSERT INTO employee (name, salary) VALUES (?, ?)";
        // JDBC/JPA 실행
        System.out.println("Executing: " + sql);
    }
}

// 4) 출력 책임만 담당
public class EmployeeReportGenerator {
    private final PayCalculator payCalculator;

    public EmployeeReportGenerator(PayCalculator payCalculator) {
        this.payCalculator = payCalculator;
    }

    public String generate(Employee employee) {
        return "[Employee Report] " + employee.getName()
                + " - " + payCalculator.calculate(employee);
    }
}
```
이제 급여 정책이 바뀌어도 `PayCalculator`만 건드리면 됩니다. `EmployeeRepository`와 `EmployeeReportGenerator`는 전혀 영향받지 않아요. **각 클래스가 "변경되는 이유" 하나씩만 가지게 된 것**이죠.

![](https://velog.velcdn.com/images/uhae451/post/a28a70e2-88ac-420a-a6c7-0a110ca5a7ca/image.png)
### 면접 포인트

- **"SRP는 기능이 하나여야 한다는 뜻인가요?"** → 아니요, "변경되는 이유가 하나"여야 한다는 뜻입니다. 기능이 여러 개여도 그게 하나의 책임(같은 이유로 변경됨)에 속한다면 문제없습니다.
- **책임을 나누는 기준**은 "누가, 왜 이 코드를 바꾸게 되는가"를 물어보는 것입니다. 이걸 흔히 **액터(actor)** 기준이라고 부릅니다. (로버트 마틴이 이후에 "Clean Architecture"에서 이 개념을 더 명확히 함)

---

## 2. OCP (Open-Closed Principle) — 개방-폐쇄 원칙

### 정의

> **"확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다."**

새로운 요구사항이 생겼을 때 **기존 코드를 수정하지 않고 코드를 추가하는 것만으로 대응**할 수 있어야 한다는 뜻입니다. 말은 간단한데 실제로 지키기 가장 까다로운 원칙이기도 합니다.

### 위반 코드

할인 정책을 처리하는 코드를 예로 들어보겠습니다.

```java
public class DiscountPolicy {

    public int calculateDiscount(String customerType, int price) {
        if (customerType.equals("VIP")) {
            return price * 20 / 100;
        } else if (customerType.equals("REGULAR")) {
            return price * 10 / 100;
        } else if (customerType.equals("NEW")) {
            return price * 5 / 100;
        }
        return 0;
    }
}
```

여기에 "STUDENT" 할인 정책이 추가된다면? `else if`를 또 추가해야 합니다. **새로운 요구사항이 생길 때마다 기존 메서드를 뜯어고쳐야 하는 구조**인 거죠. 이게 왜 위험하냐면,

- 기존 로직(VIP, REGULAR)은 이미 검증되어 운영 중인 코드인데, 새 조건을 추가하다가 실수로 건드릴 위험이 있음
- if-else 체인이 길어질수록 가독성과 유지보수성이 급격히 떨어짐
- 이 메서드를 테스트하는 테스트 코드도 조건이 늘어날 때마다 계속 수정해야 함

### 리팩토링 — 다형성으로 확장 지점을 만들기

```java
public interface DiscountPolicy {
    int calculate(int price);
}

public class VipDiscountPolicy implements DiscountPolicy {
    @Override
    public int calculate(int price) {
        return price * 20 / 100;
    }
}

public class RegularDiscountPolicy implements DiscountPolicy {
    @Override
    public int calculate(int price) {
        return price * 10 / 100;
    }
}

// 새로운 정책 추가 = 기존 클래스는 건드리지 않고 새 클래스만 추가
public class StudentDiscountPolicy implements DiscountPolicy {
    @Override
    public int calculate(int price) {
        return price * 15 / 100;
    }
}
```

```java
public class OrderService {
    private final DiscountPolicy discountPolicy;

    // 어떤 정책을 쓸지는 외부에서 주입받는다 (DIP와 자연스럽게 연결됨)
    public OrderService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public int calculateFinalPrice(int price) {
        return price - discountPolicy.calculate(price);
    }
}
```

이제 학생 할인이 추가되어도 `OrderService`나 기존 `VipDiscountPolicy`는 **단 한 줄도 수정하지 않습니다.** `StudentDiscountPolicy`라는 새 클래스만 추가(확장)하면 끝입니다. 이게 "확장에는 열려있고 변경에는 닫혀있다"는 말의 실체입니다.
![](https://velog.velcdn.com/images/uhae451/post/0b7ea1d3-4d65-48a7-9533-55def0816c0b/image.png)
### 실무 연결

스프링에서 `List<DiscountPolicy>`를 통째로 주입받아 전략 맵으로 구성하는 패턴, 혹은 `@Primary`/`@Qualifier`로 구현체를 교체하는 패턴이 전부 OCP를 지키기 위한 장치입니다. 스프링의 DI 컨테이너 자체가 "구현은 갈아끼우고 코드는 안 건드린다"는 철학 위에 서 있다고 봐도 됩니다.

---

## 3. LSP (Liskov Substitution Principle) — 리스코프 치환 원칙

### 정의

> **"자식 타입은 언제나 부모 타입으로 교체할 수 있어야 한다."**

즉, `Parent p = new Child();`로 바꿔 껴도 프로그램의 동작이 이상해지면 안 된다는 뜻입니다. 상속 관계에서 **"IS-A"가 실제로 성립하는가**를 검증하는 원칙이라고 보면 됩니다.

### 고전적인 위반 예제 — 정사각형/직사각형 문제

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    // 정사각형은 가로=세로이므로 둘 다 같이 바꿔버림
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;
    }

    @Override
    public void setHeight(int height) {
        this.width = height;
        this.height = height;
    }
}
```

수학적으로는 "정사각형은 직사각형이다"가 맞는 말입니다. 하지만 코드에서는 문제가 생깁니다.

```java
public void resize(Rectangle rectangle) {
    rectangle.setWidth(5);
    rectangle.setHeight(10);
    // Rectangle이라면 가로 5, 세로 10 → 넓이 50을 기대
    assert rectangle.getArea() == 50;
}

resize(new Rectangle()); // 통과
resize(new Square());    // 실패! 넓이가 100이 나옴
```

`Rectangle`을 매개변수로 받는 메서드에 `Square`를 넣었더니 **동작이 완전히 달라집니다.** 이게 LSP 위반입니다. `Square`가 `Rectangle`을 상속받는 순간, `setWidth`와 `setHeight`가 서로 독립적이라는 `Rectangle`의 암묵적 계약(contract)을 깨버린 거죠.

### 해결 — 상속 관계 자체를 재설계

```java
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() { return width * height; }
}

public class Square implements Shape {
    private final int side;

    public Square(int side) { this.side = side; }

    @Override
    public int getArea() { return side * side; }
}
```

**"상속이 아니라 다른 방식으로 관계를 표현할 수 있는지"**를 먼저 고민해야 합니다. `Square`가 `Rectangle`의 행동(setWidth/setHeight가 독립적으로 동작)을 그대로 보장할 수 없다면, 상속이 아니라 같은 인터페이스를 구현하는 형제 관계로 바꾸는 게 맞습니다.

![](https://velog.velcdn.com/images/uhae451/post/4b97ee34-8d79-4f98-bf22-c8b74662e0b2/image.png)

### 자바에서 LSP를 깨는 흔한 실수들

- 부모 계약이 보장한 동작을 자식이 `UnsupportedOperationException`으로 거부하는 경우. 단, Java `List.add()`처럼 계약 자체가 선택적 연산으로 명시한 경우는 예외를 던진다는 이유만으로 LSP 위반이라고 할 수 없습니다.
- 자식이 부모보다 사전조건을 강화하거나, 부모가 보장한 사후조건을 약화하는 경우
- `equals()`, `hashCode()`를 자식에서 재정의하면서 대칭성(symmetry)을 깨는 경우

---

## 4. ISP (Interface Segregation Principle) — 인터페이스 분리 원칙

### 정의

> **"클라이언트는 자신이 사용하지 않는 메서드에 의존하도록 강요받아서는 안 된다."**

하나의 거대한 인터페이스보다, **작고 구체적인 여러 개의 인터페이스**가 낫다는 원칙입니다.

### 위반 코드

```java
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class HumanWorker implements Worker {
    @Override public void work() { System.out.println("일합니다"); }
    @Override public void eat() { System.out.println("밥 먹습니다"); }
    @Override public void sleep() { System.out.println("잡니다"); }
}

// 로봇은 밥을 먹지도, 자지도 않는다
public class RobotWorker implements Worker {
    @Override public void work() { System.out.println("작업을 수행합니다"); }

    @Override
    public void eat() {
        throw new UnsupportedOperationException("로봇은 먹지 않습니다");
    }

    @Override
    public void sleep() {
        throw new UnsupportedOperationException("로봇은 자지 않습니다");
    }
}
```

`RobotWorker`가 `eat()`, `sleep()`을 억지로 구현하면서 예외를 던지고 있습니다. 이건 사실 **위에서 다룬 LSP 위반이기도 합니다.** `Worker` 타입으로 다루다가 `eat()`을 호출하면 프로그램이 깨지니까요. 그리고 이 문제의 근본 원인이 바로 ISP 위반입니다. `Worker` 인터페이스가 너무 많은 책임을 한꺼번에 강요하고 있는 거죠.

### 리팩토링

```java
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class HumanWorker implements Workable, Eatable, Sleepable {
    @Override public void work() { System.out.println("일합니다"); }
    @Override public void eat() { System.out.println("밥 먹습니다"); }
    @Override public void sleep() { System.out.println("잡니다"); }
}

// 필요한 인터페이스만 구현 — 억지 구현이 사라짐
public class RobotWorker implements Workable {
    @Override public void work() { System.out.println("작업을 수행합니다"); }
}
```

이제 `RobotWorker`는 자신이 실제로 할 수 있는 행동만 구현합니다. 클라이언트 코드도 `Eatable`이 필요하면 `Eatable`만 의존하면 되니, **불필요한 의존이 사라지고 결합도가 낮아집니다.**

### 실무에서 자주 보는 ISP 사례

스프링 데이터의 `Repository` 계층 구조가 대표적인 ISP 적용 사례입니다. `CrudRepository`, `PagingAndSortingRepository`처럼 필요한 기능 단위로 인터페이스를 쪼개놓아서, 페이징이 필요 없는 리포지토리는 `CrudRepository`만 상속받으면 되도록 설계되어 있습니다.

---

## 5. DIP (Dependency Inversion Principle) — 의존관계 역전 원칙

### 정의

> **"고수준 모듈은 저수준 모듈에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다."**
> **"추상화는 세부사항에 의존해서는 안 되고, 세부사항이 추상화에 의존해야 한다."**

다섯 원칙 중 가장 추상적으로 느껴지지만, 사실 스프링을 쓰는 개발자라면 매일 체감하고 있는 원칙입니다. **"역전(inversion)"**이라는 단어가 붙은 이유부터 봐야 합니다.

### 위반 코드

```java
// 저수준 모듈: 구체적인 구현체
public class MySqlUserRepository {
    public void save(String user) {
        System.out.println("MySQL에 저장: " + user);
    }
}

// 고수준 모듈: 비즈니스 로직
public class UserService {
    // 고수준 모듈이 저수준 모듈을 직접 new 해서 의존!
    private final MySqlUserRepository userRepository = new MySqlUserRepository();

    public void register(String user) {
        userRepository.save(user);
    }
}
```

`UserService`(비즈니스 로직, 고수준)가 `MySqlUserRepository`(구체적인 DB 기술, 저수준)를 **직접 알고 있고, 직접 생성**합니다. 이게 왜 문제일까요?

- DB를 MongoDB로 바꾸면 `UserService` 내부 코드를 직접 고쳐야 함 (OCP도 함께 위반)
- `UserService`를 테스트하려면 실제 MySQL 연결이 필요함 (테스트가 느리고 불안정해짐)
- **"비즈니스 로직"이라는 중요한 정책이, "DB가 MySQL이다"라는 세부 구현 사항에 종속되어 버림** — 원래는 반대여야 하는데 말이죠. 비즈니스 정책이 더 중요하고 안정적이어야 하는데, 오히려 자주 바뀌는 세부 기술에 발목 잡히는 상황입니다.

### 리팩토링 — 추상화(인터페이스)를 매개로 연결

```java
// 추상화 — 고수준과 저수준이 함께 바라보는 지점
public interface UserRepository {
    void save(String user);
}

// 저수준 모듈이 추상화를 구현 (구현체가 인터페이스에 의존)
public class MySqlUserRepository implements UserRepository {
    @Override
    public void save(String user) {
        System.out.println("MySQL에 저장: " + user);
    }
}

// 고수준 모듈도 추상화에만 의존
public class UserService {
    private final UserRepository userRepository;

    // 구현체는 외부에서 주입 — 어떤 구현체인지 UserService는 모른다
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public void register(String user) {
        userRepository.save(user);
    }
}
```

여기서 "역전"이라는 단어가 실감 납니다. 원래는 `UserService → MySqlUserRepository`로 향하던 의존 화살표가, 이제는 **둘 다 `UserRepository` 인터페이스를 바라보는 구조**로 바뀌었습니다. 심지어 `MySqlUserRepository`가 `UserRepository`를 구현(implements)하니까, 오히려 **저수준 모듈이 고수준 모듈이 정의한 추상화에 의존하는 모양**이 됩니다. 화살표 방향이 뒤집혔죠.

![](https://velog.velcdn.com/images/uhae451/post/75540ba8-c6ba-4637-b2b2-02909f08f8fd/image.png)
### 스프링과의 연결 — 여기가 진짜 핵심입니다

방금 만든 `UserService(UserRepository userRepository)` 생성자 주입 구조, 이게 바로 **스프링 DI(Dependency Injection)의 본질**입니다.

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    // 스프링 컨테이너가 UserRepository의 구현체(Bean)를 찾아서 주입
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

@Repository
public class MySqlUserRepository implements UserRepository { ... }
```

**DIP는 "원칙"이고, DI(의존성 주입)는 그 원칙을 지키기 위한 "구현 기법"입니다.** 그리고 스프링의 IoC 컨테이너는 그 DI를 자동화해주는 "프레임워크"입니다. 이 셋을 헷갈리지 않고 구분해서 설명할 수 있으면 면접에서 꽤 좋은 인상을 줄 수 있습니다.

- **DIP**: 고수준 모듈이 추상화에 의존해야 한다는 설계 원칙
- **DI**: 그 의존성을 외부에서 주입해주는 기법 (생성자 주입, 필드 주입, 세터 주입)
- **IoC**: 객체 생성과 생명주기의 제어권을 개발자가 아닌 프레임워크(컨테이너)가 갖는 것

---

## 6. 다섯 원칙은 어떻게 서로 연결되는가

SOLID를 공부하면서 제일 좋았던 부분은, 원칙들이 따로 노는 게 아니라 **서로를 지탱해준다**는 걸 깨달았을 때였습니다.

- **OCP를 지키려면 다형성이 필요**하고, 다형성이 제대로 동작하려면 **자식 타입이 부모 타입을 완전히 대체할 수 있어야(LSP)** 합니다. LSP가 깨지면 OCP로 확장한 코드가 런타임에 터집니다.
- **DIP는 OCP를 실현하는 구체적인 방법**입니다. "추상화에 의존하라"는 DIP를 지켜야, 새 구현체를 갈아끼우는 것만으로 확장(OCP)이 가능해집니다.
- **ISP를 지키지 않으면 LSP 위반이 따라오기 쉽습니다.** 인터페이스가 너무 크면, 그 인터페이스를 구현하는 클래스가 일부 메서드를 억지로 구현하면서 예외를 던지거나 빈 구현을 남기게 되고, 이게 곧 치환 가능성을 깨뜨립니다.
- **SRP는 나머지 네 원칙의 전제조건**에 가깝습니다. 클래스의 책임이 명확히 나뉘어 있어야, 확장 지점(OCP)을 어디에 둘지, 인터페이스를 어떻게 쪼갤지(ISP)도 명확해집니다.

<svg id="solid-principles-diagram" viewBox="0 0 700 300" xmlns="http://www.w3.org/2000/svg">
  <style>
    #solid-principles-diagram .node { fill: #ffffff; stroke: #4a5568; stroke-width: 1.5; }
    #solid-principles-diagram .center { fill: #edf2f7; stroke: #2d3748; stroke-width: 2; }
    #solid-principles-diagram .title { font: bold 12px sans-serif; fill: #1a202c; }
    #solid-principles-diagram .small { font: 10px sans-serif; fill: #718096; }
    #solid-principles-diagram .conn { stroke: #a0aec0; stroke-width: 1.3; fill: none; }
  </style>
  <circle cx="350" cy="150" r="55" class="center"/>
  <text x="350" y="145" text-anchor="middle" class="title">SRP</text>
  <text x="350" y="163" text-anchor="middle" class="small">모든 원칙의 기반</text>

  <rect x="30" y="30" width="120" height="45" rx="6" class="node"/>
  <text x="65" y="57" class="title">OCP</text>
  <line x1="150" y1="60" x2="300" y2="130" class="conn"/>

  <rect x="30" y="220" width="120" height="45" rx="6" class="node"/>
  <text x="65" y="247" class="title">LSP</text>
  <line x1="150" y1="245" x2="300" y2="170" class="conn"/>

  <rect x="550" y="30" width="120" height="45" rx="6" class="node"/>
  <text x="580" y="57" class="title">DIP</text>
  <line x1="550" y1="60" x2="400" y2="130" class="conn"/>

  <rect x="550" y="220" width="120" height="45" rx="6" class="node"/>
  <text x="580" y="247" class="title">ISP</text>
  <line x1="550" y1="245" x2="400" y2="170" class="conn"/>

  <line x1="150" y1="52" x2="550" y2="52" class="conn" stroke-dasharray="3,3"/>
  <text x="300" y="45" text-anchor="middle" class="small">OCP는 DIP로 실현됨</text>

  <line x1="150" y1="242" x2="550" y2="242" class="conn" stroke-dasharray="3,3"/>
  <text x="300" y="270" text-anchor="middle" class="small">ISP 위반 → LSP 위반으로 이어짐</text>
</svg>

---

## 7. 정리 — 면접 대비 한 장 요약

| 원칙 | 한 줄 정의 | 위반 시 나타나는 증상 | 자바/스프링에서의 실체 |
|---|---|---|---|
| **SRP** | 변경 이유는 하나여야 한다 | 하나 고치면 다른 기능이 깨짐 | Service/Repository/Controller 계층 분리 |
| **OCP** | 확장엔 열려있고 변경엔 닫혀있어야 한다 | if-else/switch가 계속 늘어남 | 인터페이스 + 다형성, 전략 패턴 |
| **LSP** | 자식은 부모를 완전히 대체할 수 있어야 한다 | 다형성으로 처리하면 런타임 예외 발생 | 오버라이딩 시 계약(contract) 준수 |
| **ISP** | 안 쓰는 메서드에 의존하면 안 된다 | 억지 구현, `UnsupportedOperationException` | Spring Data의 세분화된 Repository 인터페이스 |
| **DIP** | 고수준/저수준 모두 추상화에 의존해야 한다 | 구현체 교체 시 상위 로직까지 수정 | 스프링 DI, 생성자 주입 |

### 면접에서 나올 법한 꼬리 질문에 대비하기

- **"SOLID를 다 지키면 무조건 좋은 설계인가요?"** → 아닙니다. 원칙을 과도하게 적용하면 클래스와 인터페이스 수가 불필요하게 늘어나 오히려 복잡도가 올라갑니다(오버 엔지니어링). 요구사항 변경 가능성이 낮은 부분까지 억지로 확장 가능하게 만들 필요는 없습니다. **"이 부분이 자주 바뀌는가"를 먼저 판단하고 원칙을 적용하는 게 실전 감각**입니다.
- **"DIP와 DI의 차이를 설명해보세요"** → DIP는 설계 원칙, DI는 그 원칙을 구현하는 기법입니다. DIP 없이도 DI(단순히 생성자로 객체를 주입받는 것)는 할 수 있지만, 진짜 DIP를 지키려면 반드시 추상화(인터페이스)를 매개로 주입해야 합니다.
- **"실무 코드에서 SOLID 위반을 발견하고 리팩토링해본 경험이 있나요?"** → 이 질문에 대비해서, 지금 만들고 있는 개인 프로젝트(StudySlot)에서도 의도적으로 SRP/DIP를 적용해본 지점을 예시로 준비해두면 좋을 것 같습니다. 예를 들어 `AuthController`와 `JwtTokenProvider`를 분리한 것도 결국 SRP 적용 사례로 설명할 수 있거든요.

---

이번 글을 쓰면서 느낀 건, SOLID는 "암기 대상"이 아니라 **"코드를 보는 눈"**에 가깝다는 점이었습니다. 정의를 외우는 것보다, 지금 짜고 있는 코드에서 "이 클래스가 변경되는 이유가 몇 개지?", "이 인터페이스를 구현하는데 억지로 구현하는 메서드는 없나?"를 스스로 물어보는 습관이 훨씬 중요한 것 같습니다.

다음 글에서는 SOLID 원칙이 실제로 디자인 패턴(전략 패턴, 데코레이터 패턴 등)과 어떻게 연결되는지 조금 더 깊게 다뤄보겠습니다.
