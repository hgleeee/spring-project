# @Valid와 @Validated를 이용한 유효성 검증의 동작 원리 및 사용법
## 0. 개요
- Spring으로 개발을 하다 보면 DTO 또는 객체를 검증해야 하는 경우가 있다.
- 이를 별도의 검증 클래스로 만들어 사용할 수 있지만 간단한 검증의 경우에는 JSR 표준을 이용해 간결하게 처리할 수 있다. 

## 1. @Valid와 @Validated
### @Valid의 개념 및 사용법
- @Valid는 JSR-303 표준 스펙(자바 진영 스펙)으로써 빈 검증기(Bean Validator)를 이용해 객체의 제약 조건을 검증하도록 지시하는 어노테이션이다.
  - JSR 표준의 빈 검증 기술의 특징은 객체의 필드에 달린 어노테이션으로 편리하게 검증을 한다는 것이다.
- Spring에서는 일종의 어댑터인 LocalValidatorFactoryBean이 제약 조건 검증을 처리한다.
- 이를 이용하려면 LocalValidatorFactoryBean을 빈으로 등록해야 하는데, SpringBoot에서는 아래의 의존성만 추가해주면 해당 기능들이 자동 설정된다.

```php
// https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-validation
implementation group: 'org.springframework.boot', name: 'spring-boot-starter-validation'
```

- 예를 들어 @NotNull 어노테이션은 필드의 값이 null이 아님을 확인하도록 하며 @Min은 해당 값의 최솟값을 지정할 수 있도록 한다.

```java
@Getter
@RequiredArgsConstructor
public class AddUserRequest {

	@Email
	private final String email;

	@NotBlank
	private final String pw;

	@NotNull
	private final UserRole userRole;

	@Min(12)
	private final int age;

}
```

- 그리고 다음과 같이 컨트롤러의 메소드에 @Valid를 붙여주면 유효성 검증이 진행된다.

```java
@PostMapping("/user/add") 
public ResponseEntity<Void> addUser(@RequestBody @Valid AddUserRequest addUserRequest) {
      ...
}
```

### @Valid의 동작 원리
- 모든 요청은 프론트 컨트롤러인 디스패처 서블릿을 통해 컨트롤러로 전달된다.
- 전달 과정에서는 컨트롤러 메소드의 객체를 만들어주는 ArgumentResolver가 동작하는데, @Valid 역시 ArgumentResolver에 의해 처리가 된다.
- 대표적으로 @RequestBody는 Json 메세지를 객체로 변환해주는 작업이 ArgumentResolver의 구현체인 RequestResponseBodyMethodProcessor가 처리하며, 이 내부에서 @Valid로 시작하는 어노테이션이 있을 경우에 유효성 검사를 진행한다.
  - 이러한 이유로 @Valid가 아니라 커스톰 어노테이션인 @Validxxxxx여도 동작한다.
  - 만약 @ModelAttribute를 사용중이라면 ModelAttributeMethodProcessor에 의해 @Valid가 처리된다.
- 그리고 검증에 오류가 있다면 MethodArgumentNotValidException 예외가 발생하게 되고, 디스패처 서블릿에 기본으로 등록된 예외 리졸버(Exception Resolver)인 DefaultHandlerExceptionResolver에 의해 400 BadRequest 에러가 발생한다.

```bash
org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public org.springframework.http.ResponseEntity<java.lang.Void> com.example.testing.validator.UserController.addUser(com.example.testing.validator.AddUserRequest) with 2 errors: [Field error in object 'addUserRequest' on field 'email': rejected value [asdfad]; codes [Email.addUserRequest.email,Email.email,Email.java.lang.String,Email]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [addUserRequest.email,email]; arguments []; default message [email],[Ljavax.validation.constraints.Pattern$Flag;@18c5ad90,.*]; default message [올바른 형식의 이메일 주소여야 합니다]] [Field error in object 'addUserRequest' on field 'age': rejected value [5]; codes [Min.addUserRequest.age,Min.age,Min.int,Min]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [addUserRequest.age,age]; arguments []; default message [age],12]; default message [12 이상이어야 합니다]] 
	at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.resolveArgument(RequestResponseBodyMethodProcessor.java:141) ~[spring-webmvc-5.3.15.jar:5.3.15]
	at org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.resolveArgument(HandlerMethodArgumentResolverComposite.java:122) ~[spring-web-5.3.15.jar:5.3.15]
```

- 이러한 이유로 @Valid는 기본적으로 컨트롤러에서만 동작하며 기본적으로 다른 계층에서는 검증이 되지 않는다.
- 다른 계층에서 파라미터를 검증하기 위해서는 @Validated와 결합되어야 하는데, 아래에서 @Validated와 함께 자세히 살펴보도록 하자.

### @Validated의 개념 및 사용법
- 입력 파라미터의 유효성 검증은 컨트롤러에서 최대한 처리하고 넘겨주는 것이 좋다.
  - 하지만 개발을 하다보면 불가피하게 다른 곳에서 파라미터를 검증해야 할 수 있다.
- Spring에서는 이를 위해 AOP 기반으로 메소드의 요청을 가로채서 유효성 검증을 진행해주는 @Validated를 제공하고 있다.
  - @Validated는 JSR 표준 기술이 아니며 Spring 프레임워크에서 제공하는 어노테이션 및 기능이다.
- 다음과 같이 클래스에 @Validated를 붙여주고, 유효성을 검증할 메소드의 파라미터에 @Valid를 붙여주면 유효성 검증이 진행된다.

```java
@Service
@Validated
public class UserService {

	public void addUser(@Valid AddUserRequest addUserRequest) {
		...
	}
}
```

- 유효성 검증에 실패하면 에러가 발생하는데, 로그를 확인해보면 이전의 MethodArgumentNotValidException 예외가 아닌 ConstraintViolationException 예외가 발생했다.
  - 이는 앞서 잠깐 설명한대로 동작 원리가 다르기 때문인데, 자세히 살펴보도록 하자.

```bash
javax.validation.ConstraintViolationException: getQuizList.category: 널이어서는 안됩니다 
    at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:120) ~[spring-context-5.3.14.jar:5.3.14] 
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) ~[spring-aop-5.3.14.jar:5.3.14] 
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:753) ~[spring-aop-5.3.14.jar:5.3.14] 
    at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:698) ~[spring-aop-5.3.14.jar:5.3.14] 
    at com.mangkyu.employment.interview.app.quiz.controller.QuizController$$EnhancerBySpringCGLIB$$b23fe1de.getQuizList(<generated>) ~[main/:na] 
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~
```

### @Validated의 동작 원리
- 특정 ArgumnetResolver에 의해 유효성 검사가 진행되었던 @Valid와 달리, @Validated는 AOP 기반으로 메소드 요청을 인터셉트하여 처리된다.
  - @Validated를 클래스 레벨에 선언하면 해당 클래스에 유효성 검증을 위한 AOP의 어드바이스 또는 인터셉터(MethodValidationInterceptor)가 등록된다.
  - 그리고 해당 클래스의 메소드들이 호출될 때 AOP의 포인트 컷으로써 요청을 가로채서 유효성 검증을 진행한다.
- 이러한 이유로 @Validated를 사용하면 컨트롤러, 서비스, 레포지토리 등 계층에 무관하게 스프링 빈이라면 유효성 검증을 진행할 수 있다.
  - 대신 클래스에는 유효성 검증 AOP가 적용되도록 @Validated를, 검증을 진행할 메소드에는 @Valid를 선언해주어야 한다.
- 이러한 이유로 @Valid에 의한 예외는 MethodArgumentNotValidException이며, @Validated에 의한 예외는 ConstraintViolationException이다.
 

### 다양한 제약조건 어노테이션 
- JSR 표준 스펙은 다양한 제약 조건 어노테이션을 제공하고 있는데, 대표적인 어노테이션으로는 다음과 같은 것들이 있다.

```
@NotNull: 해당 값이 null이 아닌지 검증함
@NotEmpty: 해당 값이 null이 아니고, 빈 스트링("") 아닌지 검증함(" "은 허용됨)
@NotBlank: 해당 값이 null이 아니고, 공백(""과 " " 모두 포함)이 아닌지 검증함
@AssertTrue: 해당 값이 true인지 검증함
@Size: 해당 값이 주어진 값 사이에 해당하는지 검증함(String, Collection, Map, Array에도 적용 가능)
@Min: 해당 값이 주어진 값보다 작지 않은지 검증함
@Max: 해당 값이 주어진 값보다 크지 않은지 검증함
@Pattern: 해당 값이 주어진 패턴과 일치하는지 검증함
```

- 그 외에도 주어진 값이 이메일 형식에 해당하는지 검증하는 @Email 등 다양한 어노테이션을 제공하고 있는데, 필요한 어노테이션이 있는지는 자바 공식 문서를 참고하면 된다.
- 그 외에도 hibernate의 Validator는 해당 값이 URL인지를 검증하는 @URL 등과 같은 어노테이션을 제공하고 있다.
	- 즉, 우리가 필요로 하는 대부분의 제약 사항 어노테이션은 이미 구현되어 있으므로 잘 찾아서 이를 활용하면 된다.

 

 

 

## 2. @Valid를 이용한 검증 예시
### @Valid를 이용한 검증 예시
#### 1) 의존성 추가
- 제약조건 검증을 사용하기 위해서는 다음의 의존성을 build.gradle에 추가해주어야 한다.

```php
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

#### 2) 검증할 객체 및 제약사항 추가
- 예를 들어 새로운 사용자를 추가하는 API를 개발중이라고 하자. 그리고 이에 대한 입력값의 요구사항 및 제약사항은 다음과 같다.
  - 이메일: 이메일 형식으로 입력을 받아야 한다. 
  - 비밀번호: 빈 값을 받으면 안된다.
  - 역할: 사용자의 역할은 null이면 안된다.
  - 나이: 사용자의 나이는 12세 이상이여야 한다.
 
- 우리는 위와 같은 요구사항과 제약사항을 보고 다음과 같은 AddUserRequest를 생성할 수 있다.

```java
@Getter
@RequiredArgsConstructor
public class AddUserRequest {

	@Email
	private final String email;

	@NotBlank
	private final String pw;

	@NotNull
	private final UserRole userRole;

	@Min(12)
	private final int age;

}
```
 
- 그리고 이러한 DTO를 요청으로 받는 UserController를 다음과 같이 작성해줄 수 있다.

```java
@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/users")
    public ResponseEntity<Void> addUser(final @RequestBody @Valid AddUserRequest addUserRequest) {
        userService.registerUser(addUserRequest);

        return ResponseEntity.noContent().build();
    }

}
```

#### 3) 테스트 코드 작성 및 실행
- 해당 제약 조건들이 정상적으로 검증되는지 확인하기 위해서 테스트 코드를 작성해야 한다.
- UserController에서 email 값이 email 형식이 아닌 경우에 대한 테스트 코드는 다음과 같이 작성할 수 있는데, @Valid에 의한 반환값은 400 BadRequest이므로 이를 결과로 예측하면 된다.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

	@Autowired
	private MockMvc mockMvc;
    
	@Autowired
	private Gson gson;

	@MockBean
	private UserService userService;

	@DisplayName("사용자 추가 실패 - 이메일 형식이 아님")
	@Test
	void addUserFail_NotEmailFormat() throws Exception {
		// given
		final AddUserRequest addUserRequest = AddUserRequest.builder()
				.email("mangkyu")
				.pw("password")
				.userRole(UserRole.USER)
				.age(28).build();

		// when
		final ResultActions resultActions = mockMvc.perform(
			MockMvcRequestBuilders.post("/users")
				.content(gson.toJson(addUserRequest))
				.contentType(MediaType.APPLICATION_JSON)
		).andExpect(status().isBadRequest());

		// then
	}

}
```

#### 4) 테스트 코드 실행 및 결과 확인
- 테스트 코드를 실행해보면 email이 올바른 이메일 포맷이 아니고, 그 결과로 400 BAD_REQUEST를 반환하였음을 확인할 수 있다.

```bash
WARN org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver 
   - Resolved [org.springframework.web.bind.MethodArgumentNotValidException: 
   Validation failed for argument [0] in public org.springframework.http.ResponseEntity<java.lang.Void>
   com.example.simpletest.app.test.controller.UserController.addUser
   (com.example.simpletest.app.test.dto.AddUserRequest): 
   [Field error in object 'addUserRequest' on field 'email': rejected value [mangkyu]; 
   codes [Email.addUserRequest.email,Email.email,Email.java.lang.String,Email]; arguments 
   [org.springframework.context.support.DefaultMessageSourceResolvable: codes 
   [addUserRequest.email,email]; arguments []; 
   default message [email],[Ljavax.validation.constraints.Pattern$Flag;@6730b6c1,.*]; 
   default message [must be a well-formed email address]] ]
DEBUG org.springframework.test.web.servlet.TestDispatcherServlet - Completed 400 BAD_REQUEST
```

### @Valid와 @Validated 유효성 검증 차이 
- 위에서 살펴보았듯 @Validated의 기능으로 유효성 검증 그룹의 지정도 있지만 거의 사용되지 않으므로 유효성 검증 진행을 기준으로 차이를 살펴보도록 하자.

#### @Valid
- JSR-303 자바 표준 스펙
- 특정 ArgumentResolver를 통해 진행되어 컨트롤러 메소드의 유효성 검증만 가능하다.
- 유효성 검증에 실패할 경우 MethodArgumentNotValidException이 발생한다.

#### @Validated
- 자바 표준 스펙이 아닌 스프링 프레임워크가 제공하는 기능
- AOP를 기반으로 스프링 빈의 유효성 검증을 위해 사용되며 클래스에는 @Validated를, 메소드에는 @Valid를 붙여주어야 한다.
- 유효성 검증에 실패할 경우 ConstraintViolationException이 발생한다.
