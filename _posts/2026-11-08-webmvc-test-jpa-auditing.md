---
title: "@WebMvcTest에서 JPA Auditing 설정 오류 해결"
date: 2026-11-07 23:00:00 +0900
categories: ["Troubleshooting"]
tags: ["spring-boot-test", "webmvctest", "jpa-auditing", "slice-test"]
description: "@WebMvcTest 실행 중 JPA metamodel 오류가 발생한 원인과 JPA Auditing 설정을 테스트 종류별로 분리한 과정을 정리한다."
toc: true
---

컨트롤러 슬라이스 테스트를 실행하자 테스트 메서드에 들어가기 전에 애플리케이션 컨텍스트 로딩이 실패했습니다. 요청 형식이나 MockMvc 검증이 아니라, MVC 테스트에 JPA Auditing 설정이 함께 들어온 것이 원인이었습니다.

---

## 1. 문제 상황

테스트는 `UserController`만 대상으로 했습니다.

```java
@WebMvcTest(UserController.class)
@ActiveProfiles("test")
class UserControllerTest {
  // MockMvc와 서비스 Mock
}
```

실행 로그의 핵심 오류는 다음과 같았습니다.

```text
Error creating bean with name 'jpaAuditingHandler'
JPA metamodel must not be empty
```

`@WebMvcTest`는 Controller와 MVC 인프라를 중심으로 테스트 컨텍스트를 구성합니다. JPA 엔티티 메타모델이나 데이터베이스는 로드하지 않습니다. 그런데 `@EnableJpaAuditing`이 메인 애플리케이션 클래스에 붙어 있으면 MVC 테스트에도 감사 설정이 따라옵니다.

감사 처리기는 `@CreatedDate`, `@LastModifiedDate`를 처리하려고 JPA 매핑 정보를 필요로 합니다. MVC 슬라이스에는 그 정보가 없어 다음 흐름으로 실패합니다.

```text
@EnableJpaAuditing 적용
→ jpaAuditingHandler 생성
→ jpaMappingContext 조회
→ JPA metamodel 없음
→ 컨텍스트 로딩 실패
```

이 오류는 컨트롤러 메서드나 테스트 요청이 실행되기 전에 발생합니다.

## 2. 해결: 감사 설정 분리

감사 기능을 삭제하면 일반 애플리케이션에서 엔티티를 저장할 때 `createdAt`이 채워지지 않을 수 있습니다. 따라서 기능은 유지하고, 애플리케이션 시작 클래스에서 별도 설정으로 옮겼습니다.

```java
// DiscodeitApplication.java
@SpringBootApplication
public class DiscodeitApplication {
  // 애플리케이션 시작 코드
}
```

```java
// config/JpaAuditConfig.java
@Configuration
@EnableJpaAuditing
@Profile("!webmvc-test & !test")
public class JpaAuditConfig {
}
```

일반 애플리케이션 실행에서는 감사 기능이 켜지고 `test` 프로파일에서는 이 설정이 제외됩니다. Repository 슬라이스 테스트는 JPA가 필요하므로 테스트 클래스에서 감사를 활성화합니다.

```java
@DataJpaTest
@ActiveProfiles("test")
@EnableJpaAuditing
class UserRepositoryTest {
}
```

컨트롤러 테스트는 서비스 계층을 Mock으로 대체합니다.

```java
@WebMvcTest(UserController.class)
@ActiveProfiles("test")
class UserControllerTest {
  @MockitoBean
  UserService userService;
}
```

따라서 컨트롤러 테스트는 실제 엔티티를 DB에 저장하지 않습니다. 이 테스트에서 Auditing을 꺼도 `createdAt`이 필요한 저장 동작이 실행되지 않습니다. 반대로 실제 DB 저장을 확인하는 Repository 테스트와 일반 애플리케이션에서는 Auditing이 필요합니다.

## 3. 함께 확인한 테스트 코드

`@WebMvcTest`에는 테스트 클래스가 아니라 테스트할 Controller를 지정해야 합니다.

```java
@WebMvcTest(UserController.class)
```

multipart 회원 생성 요청에서 JSON 파트는 파일 업로드가 아니라 `@RequestPart` 데이터를 전달하기 위한 테스트 객체입니다. 프로필 파일 파트는 요청에 넣지 않으면 컨트롤러가 `null`로 받습니다.

```java
MockMultipartFile requestPart = new MockMultipartFile(
    "userCreateRequest",
    "",
    MediaType.APPLICATION_JSON_VALUE,
    json.getBytes(StandardCharsets.UTF_8)
);

mockMvc.perform(multipart("/api/users").file(requestPart))
    .andExpect(status().isCreated())
    .andExpect(jsonPath("$.username").value("username"));
```

`MockMultipartFile`이라는 이름 때문에 프로필 파일로 오해하기 쉬웠지만, 여기서는 JSON 요청 파트를 표현합니다. 선택인 프로필 파일은 추가하지 않았습니다.

## 4. 정리

- `@WebMvcTest`는 웹 계층만 구성하므로 JPA 메타모델이 없습니다.
- 메인 애플리케이션 클래스의 `@EnableJpaAuditing`은 웹 테스트 컨텍스트에도 영향을 줄 수 있습니다.
- Auditing 기능을 없애지 않고 설정을 분리해 테스트 종류에 따라 적용합니다.
- 컨트롤러 테스트는 서비스 Mock을 사용하므로 실제 엔티티 저장이나 생성 시각 기록을 확인하지 않습니다.

이번 로그에서 확인한 실패 원인은 JPA metamodel 부재였습니다. 설정 변경 뒤 테스트 전체가 통과하는지는 별도로 실행해 확인해야 합니다.

**오늘 배운 것 한 줄 요약:** `@WebMvcTest`에는 JPA Auditing이 필요하지 않으므로, 감사 설정을 분리해 JPA가 필요한 실행 환경에서만 활성화합니다.
