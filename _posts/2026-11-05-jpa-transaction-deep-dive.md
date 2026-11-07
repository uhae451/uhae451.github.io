---
title: "JPA 트랜잭션 해부"
date: 2026-11-05 09:00:00 +0900
categories: ["Spring", "Transaction"]
tags: ["spring", "jpa", "transactional", "transaction-propagation", "proxy"]
description: "Spring과 JPA 트랜잭션의 내부 동작, 프록시와 영속성 컨텍스트의 관계 및 실무에서 주의할 동작을 정리한다."
toc: true
---

> `@Transactional`만 붙이면 트랜잭션은 알아서 처리됩니다. 그런데 어느 날 이런 일을 겪습니다.
>
> - 같은 클래스 안에서 `@Transactional` 메서드를 불렀는데 **트랜잭션이 안 걸린다**
> - `save()`를 안 불렀는데 **UPDATE 쿼리가 나간다**
> - 예외를 try-catch로 잡았는데 **`UnexpectedRollbackException`** 이 터진다
> - 트래픽이 몰리면 **커넥션 풀이 바닥난다**
>
> 이 글은 위 질문들을 풀 수 있도록, `@Transactional`이 **호출되는 순간부터 끝나는 순간까지** 무슨 일이 일어나는지 쉬운 말로 따라갑니다.

---

## 0. 이 글에서 쓸 예제

회원과 상품을 조회하고, 재고를 줄이고, 주문을 저장하는 흔한 서비스입니다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MemberRepository memberRepository;
    private final ItemRepository itemRepository;
    private final OrderRepository orderRepository;

    @Transactional
    public Long placeOrder(Long memberId, Long itemId, int count) {
        Member member = memberRepository.findById(memberId).orElseThrow();
        Item item = itemRepository.findById(itemId).orElseThrow();

        item.removeStock(count);                    // 재고 감소 (save() 호출 없음!)
        Order order = Order.create(member, item, count);
        orderRepository.save(order);                // 주문 저장

        return order.getId();
    }
}
```

---

## 1. 먼저 큰 그림: 등장인물 4명

| 등장인물 | 한 줄 역할 | 비유 |
|---|---|---|
| **프록시** | 내 서비스 앞에서 호출을 가로채는 대리인 | 건물 입구의 **경비원** |
| **트랜잭션 매니저** | 트랜잭션을 시작/커밋/롤백하는 담당자 | **총무** |
| **영속성 컨텍스트** (EntityManager) | 조회한 엔티티와 변경 사항을 모아두는 메모장 | **임시 메모장** |
| **JDBC 커넥션** | DB와 실제로 연결된 통로, 진짜 트랜잭션의 주인 | **전화선** |

>  JPA의 트랜잭션도 결국은 **JDBC 커넥션 하나의 트랜잭션**입니다.
> `autoCommit=false`로 시작해서 `commit()` 또는 `rollback()`으로 끝납니다.
> 영속성 컨텍스트는 그 사이에 변경 사항을 **모아뒀다가, 커밋 직전에 SQL로 한꺼번에** 보냅니다.

---

## 2. 한 번의 호출, 7단계

`orderService.placeOrder()`가 호출됐을 때의 흐름입니다.

```
1. 프록시가 호출을 가로챈다
2. "트랜잭션이 이미 있나?" 확인 → 없으니 새로 시작
3. 커넥션을 빌리고 autoCommit을 끈다  (DB 입장에서 트랜잭션 시작)
4. 내 메서드(placeOrder)가 실행된다  ← 변경을 영속성 컨텍스트에서 관리
5. 예외 없이 끝나면 커밋을 요청한다
6. 플러시: 변경 내용이 SQL로 변환되어 DB로 간다 → 커밋
7. 커넥션을 풀에 반납한다
```

핵심은 4번과 6번입니다.

> **4번에서 변경을 모으고, 6번의 flush에서 SQL을 보낼 수 있다.** 조회 SQL, IDENTITY INSERT, 명시적 flush 및 조회 전 자동 flush는 메서드 실행 중에도 발생한다.

### 로그로 확인하기

설정을 켜면 이 흐름을 직접 볼 수 있습니다. **꼭 한 번 직접 돌려 보세요.**

```yaml
logging:
  level:
    org.springframework.orm.jpa.JpaTransactionManager: DEBUG
    org.hibernate.SQL: DEBUG
```

```
Creating new transaction with name [...OrderService.placeOrder]       // 2,3
Opened new EntityManager [SessionImpl(...)] for JPA transaction
select ... from member where member_id=?                              // 4 조회
select ... from item where item_id=?
Initiating transaction commit                                         // 5
Committing JPA transaction on EntityManager [...]
insert into orders (...) values (...)                                 // 6 플러시
update item set name=?, price=?, stock=? where item_id=?
Closing JPA EntityManager [...] after transaction                     // 7
```

로그에서 눈여겨볼 점이 두 가지 있습니다.

1. **`insert`와 `update`가 맨 마지막(커밋 시점)에 나갑니다.** 코드에서는 재고 감소(update)가 먼저인데도요. → 쓰기 SQL은 모아뒀다가 한꺼번에 보내기 때문입니다.
2. **INSERT가 UPDATE보다 먼저 나갑니다.** Hibernate는 플러시할 때 정해진 순서(보통 INSERT → UPDATE → DELETE)로 SQL을 실행하기 때문입니다.

---

## 3. 1막 - 프록시: 호출을 가로채는 경비원

### 3-1. 프록시란?

이 예제에서는 컨트롤러에 **`OrderService`를 상속한 CGLIB 프록시**가 주입됩니다. 설정에 따라 인터페이스 기반 JDK 동적 프록시를 사용할 수도 있습니다.

```java
System.out.println(orderService.getClass());
// class com.example.shop.OrderService$$SpringCGLIB$$0   ← 프록시!
```

호출 순서는 이렇습니다.

```
컨트롤러 → [프록시] → 진짜 OrderService
              ↑
        여기서 트랜잭션 시작/종료를 처리
```

프록시가 "메서드 실행 전에는 트랜잭션 시작, 실행 후에는 커밋 또는 롤백"을 대신 해주기 때문에, **우리 코드에는 트랜잭션 관련 코드가 없어도** 됩니다. (이게 AOP입니다.)

프록시 내부 동작을 아주 단순화하면 이렇습니다.

```java
// 이런 코드를 프록시가 대신 실행한다고 생각하면 됩니다
tx = 트랜잭션 시작();
try {
    진짜 메서드 실행();     // placeOrder()
    tx.커밋();
} catch (예외) {
    롤백 여부 판단 → 롤백 또는 커밋;
    throw 예외;
}
```

### 3-2. 함정 ① 자기 호출은 프록시를 안 거친다

```java
@Service
public class OrderService {

    public void placeOrders(List<OrderRequest> requests) {
        for (OrderRequest req : requests) {
            placeOrder(req);   // this.placeOrder() → 프록시를 안 거침!
        }
    }

    @Transactional
    public void placeOrder(OrderRequest req) { ... }
}
```

**왜 안 될까?**
- 밖에서 `placeOrders()`를 부르면 프록시를 거쳐 진짜 객체로 들어갑니다. (이 메서드엔 `@Transactional`이 없으니 그냥 통과)
- 진짜 객체 **안에서** `placeOrder()`를 부르면 그건 `this`, 즉 **진짜 객체끼리의 호출**입니다. 프록시(경비원)는 이 호출을 볼 방법이 없습니다.
- 결국 `placeOrder()`는 트랜잭션 없이 실행됩니다.

**해결: 트랜잭션이 필요한 메서드를 다른 빈으로 분리**

```java
@Service
@RequiredArgsConstructor
public class OrderBatchService {

    private final OrderService orderService;   // 프록시가 주입됨

    public void placeOrders(List<OrderRequest> requests) {
        requests.forEach(orderService::placeOrder);   // 매번 프록시를 거침
    }
}
```

> 같은 클래스 안에서 꼭 코드 블록 단위로 걸어야 한다면 `TransactionTemplate`을 쓰는 방법도 있습니다.

### 3-3. 프록시가 적용되지 않는 경우

- **`private` 메서드**: 프록시를 통한 외부 호출 대상이 아니므로 적용되지 않습니다.
- **CGLIB의 `final` 클래스/메서드**: 상속·오버라이드할 수 없어 적용할 수 없습니다. 클래스가 final이면 프록시 생성 자체가 실패할 수 있습니다. JDK 동적 프록시는 인터페이스 호출을 가로채므로 구분해야 합니다.
- **`@PostConstruct` 안에서의 호출**: 이 시점에는 프록시가 아직 만들어지지 않았습니다.

> `protected`, package-private 메서드는 Spring 6.0부터 CGLIB 프록시에서 적용됩니다. (그 이전에는 `public`만 지원)

---

## 4. 2막 - 트랜잭션은 "스레드"에 붙는다

### 4-1. 트랜잭션 매니저가 하는 일

프록시는 JPA를 모릅니다. 그냥 "트랜잭션 시작해줘"라고 **트랜잭션 매니저**에게 부탁할 뿐입니다. JPA를 쓰면 `JpaTransactionManager`가 이 일을 합니다. 하는 일은 이렇습니다.

1. `EntityManager`(영속성 컨텍스트)를 새로 만든다.
2. 커넥션을 풀에서 빌려오고 `autoCommit`을 끈다. → **DB 트랜잭션 시작**
3. 만든 `EntityManager`와 커넥션을 **현재 스레드에 묶어둔다.** (`ThreadLocal`)

### 4-2. 왜 "스레드에 묶는" 걸까?

우리 코드 어디에도 `EntityManager`를 파라미터로 넘기지 않았는데, `memberRepository`, `itemRepository`, `orderRepository`는 **전부 같은 영속성 컨텍스트**를 씁니다. 이게 가능한 이유가 스레드에 묶어두었기 때문입니다.

```
[스레드 하나]
   트랜잭션 매니저가 EntityManager를 스레드 보관함에 넣어둠
        ↓
   memberRepository → 같은 스레드의 보관함에서 꺼내 씀
   itemRepository   → 같은 보관함에서 꺼내 씀
   orderRepository  → 같은 보관함에서 꺼내 씀
```

> 매니저와 리포지토리는 서로를 직접 알지 못합니다. **같은 스레드의 보관함**을 통해 만날 뿐입니다.

리포지토리 메서드에도 `@Transactional`이 붙어 있지만, 이미 스레드에 트랜잭션이 있으니 **새로 만들지 않고 기존 트랜잭션에 참여**합니다. (로그의 `Participating in existing transaction`)

### 4-3. 함정 ② 스레드가 바뀌면 트랜잭션도 끊긴다

보관함이 `ThreadLocal`이니, **다른 스레드에서는 보이지 않습니다.**

```java
@Transactional
public void process() {
    CompletableFuture.runAsync(() -> {
        // 다른 스레드! 이 트랜잭션에 참여하지 못함
        repository.save(...);
    });
}
```

`@Async`, `CompletableFuture`, 병렬 스트림 등으로 다른 스레드에서 DB 작업을 하면 **별개의 트랜잭션** 이거나 트랜잭션 없이 실행됩니다. 롤백도 같이 되지 않습니다.

### 4-4. 함정 ③ 트랜잭션이 길면 커넥션을 오래 잡고 있다

커넥션 획득 시점은 Hibernate·DataSource 설정에 따라 지연될 수 있지만, DB 작업으로 커넥션을 획득한 뒤에는 일반적으로 트랜잭션 종료까지 점유합니다.

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);
    externalPaymentApi.call();   // 외부 API 응답 3초 대기 → 그동안 커넥션 점유
    sendEmail();                 // 이메일 전송도 마찬가지
}
```

커넥션 풀은 보통 10개 안팎이라, **느린 작업이 트랜잭션 안에 있으면** 요청이 몰릴 때 풀이 금방 바닥납니다.

> **원칙: 외부 API 호출, 파일 업로드, 이메일 전송 같은 느린 작업은 트랜잭션 밖으로 빼고, 트랜잭션은 DB 작업만 짧게 유지합니다.**

---

## 5. 3막 - 영속성 컨텍스트: `save()` 안 불렀는데 UPDATE가 나간 이유

처음에 던진 질문이었죠.

```java
item.removeStock(count);   // save()를 안 불렀는데...?
```

### 5-1. 변경 감지 (Dirty Checking)

영속성 컨텍스트는 엔티티를 **조회한 시점의 모습(스냅샷)** 을 같이 기억해 둡니다. 그리고 **커밋 직전(플러시)에** 현재 모습과 스냅샷을 비교합니다.

```
조회 시점: item { stock = 10 }   ← 스냅샷 저장
메서드 안: item.removeStock(3)    ← 자바 객체만 stock = 7 로 바뀜 (DB는 그대로)
커밋 직전: 스냅샷(10) ≠ 현재(7)  → "바뀌었네?" → UPDATE 쿼리 생성!
```

그래서 **조회한 엔티티의 값을 바꾸기만 해도** UPDATE가 나갑니다. `save()`는 필요 없습니다.

> 반대로 `save()`를 호출해도 이미 영속 상태인 엔티티라면 별도로 하는 일은 없습니다.
> `save()`가 필요한 건 **새로 만든 엔티티를 저장(INSERT)** 할 때입니다.

### 5-2. 쓰기 지연

`orderRepository.save(order)`를 호출해도 INSERT가 **그 자리에서 바로 나가지 않을 수 있습니다.** 영속성 컨텍스트의 "쓰기 지연 저장소"에 쌓아뒀다가 커밋 때 한꺼번에 보냅니다.

> 단, ID 생성 전략이 `IDENTITY`면 ID를 얻어야 하므로 `save()` 시점에 INSERT가 **즉시** 나갑니다.

### 5-3. `readOnly = true`는 무엇을 바꿀까?

```java
@Transactional(readOnly = true)
public Order findOrder(Long id) { ... }
```

- 플러시 모드를 **MANUAL**로 바꿔서, **커밋 때 자동 플러시(변경 감지)를 하지 않습니다.** → 불필요한 비교 작업 생략
- DB 드라이버에 읽기 전용 힌트를 전달해 최적화에 쓰일 수 있습니다. (DB에 따라 다름)

> **쓰기를 "완전히 차단"하는 보안 기능이 아닙니다.** 조회 전용이라는 의도 + 성능 최적화로 이해하세요.
> 그래서 조회 메서드에는 `readOnly = true`를 붙이는 것이 권장됩니다.

---

## 6. 4막 - 커밋과 롤백: 예외가 나면 어떻게 될까?

### 6-1. 기본 롤백 규칙

| 예외 종류 | 기본 동작 |
|---|---|
| `RuntimeException` (Unchecked) | **롤백** |
| `Error` | **롤백** |
| `Exception` (Checked) | **커밋!** (롤백 안 됨) |

```java
@Transactional
public void createOrder() throws IOException {
    orderRepository.save(order);
    throw new IOException("파일 오류");   // 체크 예외 → 롤백 안 되고 커밋됨
}
```

체크 예외에도 롤백하려면 명시해야 합니다.

```java
@Transactional(rollbackFor = Exception.class)
```

### 6-2. 함정 ④ 예외를 잡아버리면 프록시는 모른다

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);
    try {
        validatePayment(payment);         // 일반 검증 메서드에서 예외 발생 (별도 트랜잭션 경계 없음)
    } catch (RuntimeException e) {
        log.error("결제 실패");             // 잡고 끝냄
    }
}
```

프록시 입장에서는 메서드가 **예외 없이 정상 종료**된 것으로 보입니다. rollback-only가 설정되지 않았다면 **커밋**합니다. 결제 검증은 실패했는데 주문만 저장되는 불일치가 생깁니다. 반대로 Repository나 다른 트랜잭션 Bean에서 전파된 예외가 이미 rollback-only를 설정했다면 아래 6-3처럼 롤백됩니다.

> 롤백이 필요한 예외라면 **잡았더라도 다시 던져야** 합니다.

### 6-3. 함정 ⑤ `UnexpectedRollbackException`

더 헷갈리는 경우입니다. 트랜잭션이 **중첩된 호출**일 때 일어납니다.

```java
@Transactional
public void outer() {
    try {
        innerService.inner();    // 주입받은 다른 빈의 프록시를 통한 호출
    } catch (RuntimeException e) {
        log.warn("inner 실패, 계속 진행");   // 잡았으니 괜찮겠지?
    }
    // 여기서 정상 종료 → 커밋 시도 → UnexpectedRollbackException
}

// 별도 InnerService Bean에 선언된 메서드
@Transactional
public void inner() {
    throw new RuntimeException();
}
```

**왜 터질까?**

1. `outer()`가 시작한 트랜잭션에 `inner()`가 **참여**합니다. (전파 속성 `REQUIRED`의 기본 동작: 이미 있으면 같이 쓴다.)
2. 두 메서드는 **같은 하나의 트랜잭션**입니다.
3. `inner()`에서 예외가 나는 순간, 이 트랜잭션에 **"롤백만 가능(rollback-only)" 표시**가 붙습니다.
4. `outer()`가 예외를 잡고 정상 종료해서 커밋을 시도하지만, 이미 롤백 표시가 있어서 **"커밋하려 했는데 롤백됐어요"** 라며 예외를 던집니다.

> 비유: 같은 배를 탄 사람 중 한 명이 "이 배 침몰"이라고 깃발을 꽂았으면, 다른 사람이 아무리 "괜찮아요"라고 해도 배는 침몰합니다.

**해결 방법**
- `inner()`의 실패를 `outer()`에서 정말 무시하고 싶다면, `inner()`를 `REQUIRES_NEW`로 분리해 **별도 트랜잭션**으로 만듭니다. (단, 아래 주의 참고)
- 또는 예외를 잡지 않고 그대로 전파합니다.

---

## 7. 전파 속성 (Propagation) 핵심만

| 속성 | 의미 |
|---|---|
| **REQUIRED** (기본값) | 이미 있으면 **참여**, 없으면 새로 만든다 |
| **REQUIRES_NEW** | 항상 **새 트랜잭션**. 기존 트랜잭션은 잠시 멈춘다 |
| SUPPORTS | 있으면 참여, 없으면 트랜잭션 없이 실행 |
| MANDATORY | 반드시 이미 있어야 함. 없으면 예외 |
| NOT_SUPPORTED | 트랜잭션 없이 실행 (있으면 잠시 멈춤) |
| NEVER | 트랜잭션이 있으면 예외 |
| NESTED | 세이브포인트 기반. 매니저·드라이버 지원 필요 |

`JpaTransactionManager`는 기본적으로 NESTED를 허용하지 않으며, JDBC 세이브포인트를 허용해도 JPA 영속성 컨텍스트의 엔티티 상태까지 되돌리는 기능은 아니다.

실무에서는 **REQUIRED(기본)** 와 **REQUIRES_NEW** 두 가지를 가장 많이 씁니다.

### 함정 ⑥ REQUIRES_NEW와 커넥션 풀 고갈

`REQUIRES_NEW`는 **새 커넥션이 하나 더** 필요합니다. 기존 트랜잭션은 멈추기만 할 뿐 **커넥션을 반납하지 않습니다.**

```
커넥션 풀 크기 = 10

요청 10개 동시 진입 → 각각 바깥 트랜잭션이 커넥션 1개씩 사용 (10개 모두 점유)
  → 각자 REQUIRES_NEW 메서드 호출 → 새 커넥션 필요
  → 풀이 비어 있음 → 모두 서로 기다림 (사실상 데드락)
```

> `REQUIRES_NEW`를 쓸 때는 **커넥션 풀 크기**와 **동시 요청 수**를 같이 고려하고, 꼭 필요한 곳(감사 로그 등)에만 쓰세요.

---

## 8. 한눈에 정리

| 질문 | 답 |
|---|---|
| `@Transactional`은 어떻게 동작? | **프록시**가 메서드 앞뒤로 트랜잭션 시작/종료를 대신 처리 |
| 같은 클래스 내부 호출은 왜 안 걸려? | `this` 호출이라 **프록시를 거치지 않기** 때문 |
| `save()` 없이 UPDATE가 나가는 이유는? | **변경 감지**: 커밋 직전 스냅샷과 비교해 UPDATE 생성 |
| 왜 리포지토리들이 같은 트랜잭션을 쓰나? | 트랜잭션 자원이 **ThreadLocal(스레드)** 에 묶여 있어서 |
| 체크 예외는 롤백되나? | **기본은 커밋.** `rollbackFor`로 지정 필요 |
| 예외를 try-catch로 잡으면? | rollback-only가 없으면 커밋 가능. 이미 표시됐으면 롤백 |
| `UnexpectedRollbackException`은? | 참여한 트랜잭션이 **rollback-only 표시**를 남겨서 |
| 커넥션은 언제 잡고 반납? | 획득 시점은 설정에 따라 다르며, 보통 트랜잭션 종료 시 반납 |
| `readOnly = true`는? | 플러시(변경 감지) 생략 + 읽기 힌트. 쓰기 차단 기능은 **아님** |

---

## 9. 면접 대비 질문과 모범 답안

### Q1. `@Transactional`은 내부적으로 어떻게 동작하나요?

**A.** Spring AOP의 **프록시 기반**으로 동작합니다. 애플리케이션이 뜰 때 `@Transactional`이 붙은 빈 대신 프록시 객체가 컨테이너에 등록되고, 메서드를 호출하면 프록시가 먼저 호출을 가로채 트랜잭션 매니저를 통해 **트랜잭션을 시작**합니다. 이후 실제 메서드를 실행하고, 정상 종료하면 **커밋**, 롤백 대상 예외가 발생하면 **롤백**합니다. 개발자가 트랜잭션 코드를 직접 작성하지 않아도 되는 이유입니다.

### Q2. 같은 클래스 내부에서 `@Transactional` 메서드를 호출하면 왜 트랜잭션이 적용되지 않나요?

**A.** 프록시를 거치지 않기 때문입니다. 외부에서 호출할 때는 프록시를 거쳐 트랜잭션이 적용되지만, 객체 내부에서 `this.method()`로 호출하면 프록시가 아닌 **실제 객체를 직접 호출**하므로 트랜잭션 로직이 끼어들 수 없습니다. 해결하려면 트랜잭션이 필요한 메서드를 **별도의 빈으로 분리**하거나 `TransactionTemplate`을 사용합니다.

### Q3. `save()`를 호출하지 않았는데 UPDATE 쿼리가 나가는 이유는?

**A.** JPA의 **변경 감지(Dirty Checking)** 때문입니다. 영속성 컨텍스트는 엔티티를 조회할 때 **스냅샷**을 저장해 두고, 트랜잭션 커밋 직전 플러시 시점에 현재 상태와 스냅샷을 비교해 변경된 엔티티에 대한 UPDATE 쿼리를 자동으로 생성합니다. 그래서 영속 상태의 엔티티는 값을 바꾸기만 해도 반영됩니다.

### Q4. 체크 예외가 발생하면 롤백되나요?

**A.** 기본적으로 롤백되지 않고 **커밋**됩니다. Spring은 `RuntimeException`과 `Error`만 기본 롤백 대상으로 합니다. 체크 예외도 롤백하려면 `@Transactional(rollbackFor = Exception.class)`처럼 명시해야 합니다.

### Q5. 예외를 try-catch로 잡았는데 `UnexpectedRollbackException`이 발생하는 이유는?

**A.** 전파 속성 `REQUIRED`에서는 안쪽 메서드가 바깥 트랜잭션에 **참여**하기 때문에 둘은 하나의 트랜잭션입니다. 안쪽에서 예외가 발생하면 해당 트랜잭션에 **rollback-only 표시**가 붙는데, 바깥 메서드가 예외를 잡고 정상 종료해 커밋을 시도하면 이미 롤백 표시가 있어서 `UnexpectedRollbackException`이 발생합니다. 안쪽 실패를 독립적으로 처리하고 싶다면 `REQUIRES_NEW`로 별도 트랜잭션을 만들거나 예외를 전파해야 합니다.

### Q6. `readOnly = true`는 어떤 효과가 있나요?

**A.** 해당 트랜잭션이 조회 전용임을 나타내며, Hibernate의 플러시 모드를 `MANUAL`로 바꿔 **커밋 시 변경 감지와 플러시를 생략**합니다. 또한 JDBC 드라이버나 DB에 읽기 전용 힌트를 전달해 최적화가 가능합니다. 다만 쓰기를 강제로 막는 보안 기능은 아니므로 의도와 성능 최적화 용도로 이해해야 합니다.

### Q7. `REQUIRES_NEW`를 쓸 때 주의할 점은?

**A.** 기존 트랜잭션은 멈출 뿐 **커넥션을 반납하지 않고**, `REQUIRES_NEW`는 새 커넥션을 추가로 요청합니다. 그래서 동시 요청이 많으면 모든 스레드가 바깥 트랜잭션의 커넥션을 잡은 채로 안쪽 트랜잭션의 커넥션을 기다리다 **커넥션 풀이 고갈되어 사실상 데드락**에 빠질 수 있습니다. 꼭 필요한 곳에만 쓰고 커넥션 풀 크기를 함께 고려해야 합니다.

### Q8. 트랜잭션 안에서 외부 API를 호출하면 안 되는 이유는?

**A.** 커넥션을 획득한 이후에는 일반적으로 트랜잭션 **종료될 때까지 점유**합니다. 외부 API 응답이 느리면 그동안 커넥션을 쥐고 있어 풀이 고갈되고, 다른 요청이 커넥션을 얻지 못해 장애로 이어질 수 있습니다. 또한 외부 호출은 롤백으로 되돌릴 수 없어 데이터 정합성 문제도 생깁니다. 느린 작업은 트랜잭션 밖으로 분리하는 것이 좋습니다.

---

## 마치며

`@Transactional` 하나가 **프록시 → 트랜잭션 매니저 → 스레드 → 영속성 컨텍스트 → 커넥션**이 맞물려 돌아가는 구조라는 걸 알면, "왜 안 되지?" 싶던 문제들이 대부분 이 흐름 안에서 설명됩니다.

- 트랜잭션이 안 걸린다 → **프록시를 안 거쳤나?**
- 롤백이 안 된다 → **어떤 예외인지, 예외를 잡았는지?**
- 풀이 바닥난다 → **트랜잭션이 너무 길거나 REQUIRES_NEW를 남발했나?**

## 참고 문서

- [Spring 트랜잭션 전파](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html)
