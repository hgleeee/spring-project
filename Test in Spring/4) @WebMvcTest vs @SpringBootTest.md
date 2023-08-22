# SpringBoot 테스트 시 @WebMvcTest와 @SpringBootTest의 차이

## 개요
- Junit 테스트 시 @WebMvcTest와 @SpringBootTest를 대표적으로 사용하는데, 이 두 가지 Test 어노테이션에는 차이가 존재한다.
- 두 가지 어노테이션의 차이를 살펴보자.

### ※ Mock이란?
- 실제 객체를 만들어서 테스트하기가 어려운 경우에, 가짜 객체를 만들어서 테스트하는 기술이다.

### ※ MockMvc란?
- MVC에 관련된 Mock 가짜 객체를 가리킨다.
- 웹 어플리케이션을 애플리케이션 서버에 배포하지 않고, 테스트용 MVC 환경을 만들어서 요청 및 전송, 응답 기능을 제공해주는 객체이다.
- 대부분의 어플리케이션 기능을 테스트하기 위해서는 MockMvc 객체를 만들어서 테스트하게 되는데, MockMvc를 @Autowired로 주입받아서 사용할 수 있다.
  - 이렇게 MockMvc를 @Autowired로 주입 받을 때 두 어노테이션의 차이가 존재한다.
- 또한, 빈의 등록 범위에도 차이가 존재한다.

## 1. Mock 주입 시 차이
### @SpringBootTest
```java
@SpringBootTest
class SpringBootTest {

    @Autowired
    MockMvc mockMvc; // 주입 X
}
```

- 이렇게 @SpringBootTest만 선언하고, MockMvc를 @Autowired로 주입받고자 하면, 주입이 되지 않아 오류가 발생한다.
  - 주입이 되지 않는 이유는, @SpringBootTest는 MockMvc를 빈으로 등록시키지 않기 때문이다.
  - 따라서, MockMvc를 빈으로 등록하는 과정 필요하다.
- 이러한 과정은 @AutoConfigureMockMvc 어노테이션을 선언하면 된다.

```java
@SpringBootTest
@AutoConfigureMockMvc
class SpringBootTest {

    @Autowired
    MockMvc mockMvc; // 주입 O
}
```
- 결과적으로, @SpringBootTest에서 MockMvc 객체를 사용하려면, @AutoConfigureMockMvc 어노테이션을 함께 선언해주어야 한다.

### @WebMvcTest
```java
@WebMvcTest
class SpringBootTest {

    @Autowired
    MockMvc mockMvc; // 주입 O
}
```
- @WebMvcTest는 MockMvc를 빈으로 등록하기 때문에, @WebMvcTest만 선언해줘도, MockMvc 객체가 주입되게 된다.

## 2. Bean 등록 범위 차이
### @SpringBootTest
```java
@SpringBootTest
class SpringBootTest {

    @Autowired
    MockMvc mockMvc; // 주입 O

    @Autowired
    UserController userController; // 주입 O

    @Autowired
    UserRepository userRepository; // 주입 O

    @Autowired
    UserService userService; // 주입 O
}
```
- @SpringBootTest에서는 프로젝트의 컨트롤러, 리포지토리, 서비스가 @Autowired로 다 주입된다.

### @WebMvcTest
```java
@WebMvcTest
class SpringBootTest {

		@Autowired
		MockMvc mockMvc; // 주입 O

		@Autowired
		UserController userController; // 주입 O

		@Autowired
		UserRepository userRepository; // 주입 X

		@Autowired
		UserService userService; // 주입 X
}
```
- @WebMvcTest에서는 Web Layer 관련 빈들만 등록하기 때문에 컨트롤러는 주입이 정상적으로 되지만, @Component로 등록된 리포지토리와 서비스는 주입이 되지 않는다.
- 따라서, @WebMvcTest에서 리포지토리와 서비스를 사용하기 위해서는 @MockBean을 사용하여 리포지토리와 서비스를 Mock 객체에 빈으로 등록해줘야 한다.

### ※ Web Layer에 속한 항목들
```java
Security, Filter, Interceptor, request/response Handling, Controller
```
```java
@WebMvcTest
class SpringBootTest {

    @Autowired
    MockMvc mockMvc; // 주입 O

    @Autowired
    UserController userController; // 주입 O

    @MockBean
    UserRepository userRepository; // 주입 O

    @MockBean
    UserService userService; // 주입 O
}
```

## @SpringBootTest와 @WebMvcTest 차이 결론
### @SpringBootTest
- MockMvc 객체를 빈으로 등록하지 않기 때문에 @AutoConfigureMockMvc로 빈으로 등록해야한다.
- 프로젝트에 있는 스프링 빈을 모두 등록해서 테스트에 필요한 의존성을 추가해준다.

#### 장점
```
1) 프로젝트에 있는 모든 스프링 빈을 등록하므로, 테스트에 필요한 객체를 주입받아서 쉽게 사용 가능하다.
2) 실제 환경과 가장 유사하게 테스트가 가능하다.
```

#### 단점
```
1) 모든 스프링 빈을 등록할 때 프로젝트의 전체 컨텍스트를 로드해서 빈을 주입하기 때문에 속도가 느리다.
2) 테스트 단위가 크기 때문에 디버깅이 어려울 수 있다.
```
- 이러한 이유로, 단위 테스트와 같은 기능 테스트가 아닌, 전체적인 프로그램 작동이 제대로 이루어지는지 검증하는 통합 테스트 시에 많이 사용한다!

### @WebMvcTest
- MockMvc 객체를 빈으로 등록해서 @Autowired로 MockMvc 주입이 가능하다.
- Web Layer 관련 빈들만 등록하기 때문에, @Component로 등록한 빈은 @MockBean으로 등록해야 한다.

#### 장점
```
1) Web Layer 관련 빈만 로드하기 때문에, 속도가 @SpringBootTest보다 빠르다.
2) 통합 테스트에서 테스트가 어려운 작은 단위 테스트들을 @WebMvcTest로 진행할 수 있다.
```

#### 단점
```
1) Mock 객체를 사용하기 때문에 실제 환경에서는 다른 오류가 발생할 수 있다.
```
- 이러한 이유로 컨트롤러 테스트, 단위 테스트 시 많이 사용한다.
