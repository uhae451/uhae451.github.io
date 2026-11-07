---
title: "디자인 패턴"
date: 2026-10-18 09:00:00 +0900
categories: ["Design", "Patterns"]
tags: ["design-pattern", "singleton", "factory", "strategy", "java"]
description: "싱글톤, 팩토리, 전략 등 주요 디자인 패턴의 구조와 사용 목적을 Java 예제 중심으로 정리한다."
toc: true
---

## 싱글톤 패턴 (Singleton Pattern)

클래스의 인스턴스를 딱 하나만 만들어서 어디서든 그 하나를 공유해서 쓰는 패턴. 설정 객체, DB 커넥션 풀, 로거처럼 애플리케이션 전체에서 상태를 하나로 유지해야 하는 경우에 쓴다.

```java
public class AppConfig {
    private static AppConfig instance;
    private String env;

    private AppConfig() {
        this.env = "production";
    }

    public static AppConfig getInstance() {
        if (instance == null) {
            instance = new AppConfig();
        }
        return instance;
    }

    public String getEnv() {
        return env;
    }
}
```

```java
AppConfig config1 = AppConfig.getInstance();
AppConfig config2 = AppConfig.getInstance();

System.out.println(config1 == config2); // true, 같은 객체
```

생성자를 `private`으로 막아서 외부에서 `new`를 못 하게 하고, `getInstance()`를 통해서만 접근하게 만드는 게 핵심. 멀티스레드 환경에서는 동시에 여러 인스턴스가 생성될 수 있어서 `synchronized`나 `Enum`, 정적 초기화 방식으로 스레드 안전성을 챙겨야 한다.

**주로 쓰는 곳**: 설정값 관리, 로깅, DB 커넥션 풀, 캐시 매니저처럼 애플리케이션 전역에서 상태를 공유해야 하는 곳.

---

## 팩토리 패턴 (Factory Pattern)

객체 생성 로직을 별도의 클래스로 분리해서, 사용하는 쪽에서 `new`로 직접 구체 클래스를 생성하지 않게 만드는 패턴.

```java
public interface Animal {
    void sound();
}

public class Dog implements Animal {
    public void sound() {
        System.out.println("멍멍");
    }
}

public class Cat implements Animal {
    public void sound() {
        System.out.println("야옹");
    }
}

public class AnimalFactory {
    public static Animal create(String type) {
        if (type.equals("dog")) {
            return new Dog();
        } else if (type.equals("cat")) {
            return new Cat();
        }
        throw new IllegalArgumentException("없는 동물: " + type);
    }
}
```

```java
Animal animal = AnimalFactory.create("dog");
animal.sound(); // 멍멍
```

사용하는 쪽은 `Dog`, `Cat` 같은 구체 클래스 이름을 몰라도 되고 `Animal` 인터페이스만 알면 된다. 이 예제는 단순 팩토리(Simple Factory)이며, 상속으로 생성 메서드를 재정의하는 GoF Factory Method와는 구분한다. 새로운 동물이 추가돼도 팩토리 클래스 내부만 수정하면 되고, 이 코드를 사용하는 다른 곳은 건드릴 필요가 없다.

**주로 쓰는 곳**: 구현체가 여러 개고 조건에 따라 다른 구현체를 생성해야 할 때, 나중에 구현체가 교체될 가능성이 있을 때.

---

## 전략 패턴 (Strategy Pattern)

동작(알고리즘)을 인터페이스로 뽑아내서, 런타임에 어떤 전략을 쓸지 갈아 끼울 수 있게 만드는 패턴.

```java
public interface PaymentStrategy {
    void pay(int amount);
}

public class CardPayment implements PaymentStrategy {
    public void pay(int amount) {
        System.out.println("카드로 " + amount + "원 결제");
    }
}

public class KakaoPayPayment implements PaymentStrategy {
    public void pay(int amount) {
        System.out.println("카카오페이로 " + amount + "원 결제");
    }
}

public class PaymentService {
    private PaymentStrategy strategy;

    public PaymentService(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void pay(int amount) {
        strategy.pay(amount);
    }
}
```

```java
PaymentService service = new PaymentService(new CardPayment());
service.pay(1000); // 카드로 1000원 결제

PaymentService service2 = new PaymentService(new KakaoPayPayment());
service2.pay(2000); // 카카오페이로 2000원 결제
```

`PaymentService`는 어떤 결제수단이 들어오든 신경 쓰지 않고 `PaymentStrategy` 인터페이스에만 의존한다. 결제수단이 늘어나도 클래스 하나만 추가하면 되고 기존 로직은 안 건드려도 된다.

**주로 쓰는 곳**: 정렬 기준, 할인 정책, 결제수단처럼 같은 동작이지만 방식이 여러 가지로 갈리고 런타임에 선택되는 경우.

---

## 옵저버 패턴 (Observer Pattern)

어떤 객체(주체)의 상태가 바뀌었을 때, 그 변화를 구독하고 있는 객체들(옵저버)한테 자동으로 알림을 보내는 패턴.

```java
public interface Subscriber {
    void update(String video);
}

public class SubscriberA implements Subscriber {
    public void update(String video) {
        System.out.println("구독자A: " + video + " 알림 받음");
    }
}

import java.util.ArrayList;
import java.util.List;

public class YoutubeChannel {
    private List<Subscriber> subscribers = new ArrayList<>();

    public void subscribe(Subscriber subscriber) {
        subscribers.add(subscriber);
    }

    public void unsubscribe(Subscriber subscriber) {
        subscribers.remove(subscriber);
    }

    public void upload(String video) {
        System.out.println("[채널] " + video + " 업로드!");
        for (Subscriber subscriber : subscribers) {
            subscriber.update(video);
        }
    }
}
```

```java
YoutubeChannel channel = new YoutubeChannel();
Subscriber subA = new SubscriberA();

channel.subscribe(subA);
channel.upload("옵저버 패턴 강의");
// [채널] 옵저버 패턴 강의 업로드!
// 구독자A: 옵저버 패턴 강의 알림 받음
```

주체는 구독자가 몇 명이고 어떤 타입인지 몰라도 되고, `Subscriber` 인터페이스만 알면 된다. 구독/해지가 런타임에 자유롭게 이루어질 수 있다는 것도 특징.

**주로 쓰는 곳**: 이벤트 리스너, GUI 버튼 클릭 처리, 스프링의 `ApplicationEvent`, 실시간 알림 시스템처럼 상태 변화를 여러 곳에 전파해야 하는 경우.

---

## 이터레이터 패턴 (Iterator Pattern)

컬렉션 내부 구조(배열인지, 리스트인지, 트리인지)를 몰라도 순회할 수 있게, 순회 방법 자체를 객체로 캡슐화하는 패턴.

```java
import java.util.Iterator;
import java.util.NoSuchElementException;

public class NameCollection implements Iterable<String> {
    private String[] names;

    public NameCollection(String[] names) {
        this.names = names;
    }

    @Override
    public Iterator<String> iterator() {
        return new NameIterator();
    }

    private class NameIterator implements Iterator<String> {
        private int index = 0;

        @Override
        public boolean hasNext() {
            return index < names.length;
        }

        @Override
        public String next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            return names[index++];
        }
    }
}
```

```java
NameCollection collection = new NameCollection(new String[]{"현", "지훈", "민수"});

for (String name : collection) {
    System.out.println(name);
}
// 현
// 지훈
// 민수
```

자바의 `for-each`는 `Iterable`에 대해 `Iterator`를 사용한다. `List`와 `Set`은 직접 순회할 수 있지만, `Map`은 `entrySet()`, `keySet()`, `values()`를 통해 순회한다. 배열의 `for-each`는 인덱스 기반으로 처리된다. 직접 커스텀 컬렉션을 만들 때 이 인터페이스를 구현하면 `for-each`를 그대로 쓸 수 있다.

**주로 쓰는 곳**: 커스텀 자료구조를 만들었을 때 외부에서 내부 구현을 몰라도 순회할 수 있게 하고 싶을 때.

---

## 노출모듈 패턴 (Revealing Module Pattern)

내부 상태와 구현 로직은 감추고, 외부에는 꼭 필요한 기능만 공개하는 패턴. 원래는 클로저를 이용하는 자바스크립트 쪽 패턴에 가깝지만, 핵심 아이디어인 "내부는 숨기고 필요한 것만 노출한다"는 자바에서는 `private` 필드/메서드와 `public` 메서드를 구분하는 방식으로 그대로 구현할 수 있다.

```java
public class Counter {
    private int count = 0; // 외부에서 직접 접근 불가

    private void validate() {
        if (count == 0) {
            throw new IllegalStateException("count는 음수가 될 수 없음");
        }
    }

    public void increase() {
        count++;
    }

    public void decrease() {
        validate(); // 변경 전에 검사해 예외 발생 시에도 상태 유지
        count--;
    }

    public int getCount() {
        return count;
    }
}
```

```java
Counter counter = new Counter();
counter.increase();
counter.increase();
counter.decrease();

System.out.println(counter.getCount()); // 1
// counter.count -> 컴파일 에러, 외부에서 직접 접근 불가
// counter.validate() -> 컴파일 에러, private 메서드라 호출 불가
```

`count` 필드와 `validate()` 메서드는 `private`이라 클래스 밖에서 절대 접근할 수 없고, 오직 `increase()`, `decrease()`, `getCount()`처럼 공개하기로 정한 메서드를 통해서만 상태를 조작할 수 있다. 내부 검증 로직이 아무리 복잡해져도 외부 코드는 전혀 몰라도 되고, 나중에 내부 구현을 바꿔도 공개된 메서드 시그니처만 유지하면 사용하는 쪽 코드는 안 깨진다.

**주로 쓰는 곳**: 유틸리티 클래스나 도메인 객체를 만들 때 내부 상태를 직접 건드리지 못하게 막고, 정해진 메서드로만 조작하게 강제하고 싶을 때. 캡슐화의 기본 원칙 자체가 이 패턴의 목적과 맞닿아 있다.

---

## MVC 패턴 (Model-View-Controller)

애플리케이션을 데이터(Model), 화면(View), 사용자 입력 처리(Controller) 세 영역으로 나누는 패턴. 스프링 MVC가 대표적으로 이 구조를 따른다.

```java
// Model
public class User {
    private String name;
    private int age;
    // getter, setter
}

// Controller
@Controller
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/users/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.findById(id);
        model.addAttribute("user", user); // View에 데이터 전달
        return "user-detail"; // View 이름 반환
    }
}
```

```html
<!-- View (user-detail.html) -->
<p th:text="${user.name}"></p>
<p th:text="${user.age}"></p>
```

Controller가 사용자 요청을 받아서 Model(비즈니스 로직/데이터)을 다루고, 그 결과를 View한테 넘겨서 화면을 그리게 한다. Controller는 Model에 작업을 요청하고 View는 표현을 담당하도록 역할을 나누는 게 핵심.

**주로 쓰는 곳**: 웹 애플리케이션 백엔드 구조 전반(스프링 MVC, 익스프레스 등). 지금 만들고 있는 StudySlot도 이 구조를 따르고 있다.

---

## MVP 패턴 (Model-View-Presenter)

MVC에서 View와 Model의 결합을 더 끊어내기 위해 등장한 패턴. View는 사용자 입력을 그대로 Presenter한테 위임하고, Presenter가 Model을 조작한 뒤 View를 직접 갱신한다. View와 Presenter가 인터페이스로 연결되기 때문에 View를 테스트 코드로 대체하기 쉽다.

```java
// View 인터페이스
public interface UserView {
    void showUserName(String name);
}

// Model
public class UserModel {
    public String getUserName() {
        return "리부트현";
    }
}

// Presenter
public class UserPresenter {
    private UserView view;
    private UserModel model;

    public UserPresenter(UserView view, UserModel model) {
        this.view = view;
        this.model = model;
    }

    public void loadUser() {
        String name = model.getUserName();
        view.showUserName(name); // Presenter가 View를 직접 갱신
    }
}

// View 구현체 (Android Activity 등이 여기 해당)
public class UserActivity implements UserView {
    public void showUserName(String name) {
        System.out.println("화면에 표시: " + name);
    }
}
```

```java
UserActivity view = new UserActivity();
UserModel model = new UserModel();
UserPresenter presenter = new UserPresenter(view, model);

presenter.loadUser(); // 화면에 표시: 리부트현
```

MVC와 다르게 View가 Model을 직접 참조하지 않고, 모든 흐름이 Presenter를 거친다. View는 `UserView` 인터페이스라서 테스트할 때 가짜 View 구현체를 넣어서 Presenter 로직만 따로 검증하기 좋다.

**주로 쓰는 곳**: 안드로이드 앱 개발(특히 Jetpack이 나오기 전 구조)에서 많이 쓰였고, View 로직과 비즈니스 로직을 확실히 분리하고 싶은 경우.

---

## MVVM 패턴 (Model-View-ViewModel)

View와 ViewModel을 데이터 바인딩으로 연결해서, View가 상태 변화를 감지해 자동으로 갱신되게 만드는 패턴. Presenter처럼 View를 직접 조작하는 코드가 없고, ViewModel은 View의 존재 자체를 몰라도 된다.

```java
// ViewModel (Android LiveData 예시)
public class UserViewModel extends ViewModel {
    private MutableLiveData<String> userName = new MutableLiveData<>();

    public LiveData<String> getUserName() {
        return userName;
    }

    public void loadUser() {
        userName.setValue("리부트현"); // 값만 바꿔줌, View를 직접 안 건드림
    }
}
```

```java
// View (Activity)
public class UserActivity extends AppCompatActivity {
    private UserViewModel viewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewModel = new ViewModelProvider(this).get(UserViewModel.class);

        // 데이터 바인딩: userName이 바뀔 때마다 자동으로 텍스트뷰 갱신
        viewModel.getUserName().observe(this, name -> {
            textView.setText(name);
        });

        viewModel.loadUser();
    }
}
```

ViewModel은 `MutableLiveData`의 값만 바꾸고, View는 그 값을 구독(`observe`)하고 있다가 변화가 생기면 알아서 화면을 갱신한다. ViewModel이 View를 참조하지 않기 때문에 MVP보다 결합도가 더 낮고, 상태를 갱신하는 코드를 일일이 안 써도 된다는 게 장점.

**주로 쓰는 곳**: 안드로이드(LiveData, DataBinding, Jetpack Compose), Vue.js, Angular처럼 양방향 바인딩이나 반응형 상태 관리를 지원하는 프레임워크 기반 프론트엔드 개발.

## 참고 문서

- [Java Iterator API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Iterator.html)
