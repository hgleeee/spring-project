# JUnit과 Mockito 기반의 Spring 단위 테스트 코드 작성

## 1. Mockito 소개 및 사용법
### Mockito란?
- Mockito는 개발자가 동작을 직접 제어할 수 있는 가짜 객체를 지원하는 테스트 프레임워크이다.
- 일반적으로 Spring으로 웹 애플리케이션을 개발하면, 여러 객체들 간의 의존성이 생긴다.
  - 이러한 의존성은 단위 테스트를 작성을 어렵게 하는데, 이를 해결하기 위해 가짜 객체를 주입시켜주는 Mockito 라이브러리를 활용할 수 있다.
- Mockito를 활용하면 가짜 객체에 원하는 결과를 Stub하여 단위 테스트를 진행할 수 있다.

### Mockito 사용법
#### 1) Mock 객체 의존성 주입
- Mockito에서 가짜 객체의 의존성 주입을 위해서는 크게 3가지 어노테이션이 사용된다.
  - @Mock: 가짜 객체를 만들어 반환해주는 어노테이션
  - @Spy: Stub하지 않은 메소드들은 원본 메소드 그대로 사용하는 어노테이션
  - @InjectMocks: @Mock 또는 @Spy로 생성된 가짜 객체를 자동으로 주입시켜주는 어노테이션
- 예를 들어 UserController에 대한 단위 테스트를 작성하고자 할 때,
  - UserService를 사용하고 있다면 @Mock 어노테이션을 통해 가짜 UserService를 만들고,
  - @InjectMocks를 통해 UserController에 이를 주입시킬 수 있다.

#### 2) Stub으로 결과 처리
- 앞서 설명하였듯, 의존성이 있는 객체는 가짜 객체를 주입하여 어떤 결과를 반환하라고 정해진 답변을 준비시켜야 한다.
- Mockito에서는 다음과 같은 stub 메소드를 제공한다.
  - doReturn(): 가짜 객체가 특정한 값을 반환해야 하는 경우
  - doNothing(): 가짜 객체가 아무 것도 반환하지 않는 경우(void)
  - doThrow(): 가짜 객체가 예외를 발생시키는 경우
 
#### 3) Mockito와 Junit의 결합
- Mockito도 테스팅 프레임워크이기 때문에 JUnit과 결합되기 위해서는 별도의 작업이 필요하다.
- 기존의 JUnit4에서 Mockito를 활용하기 위해서는 클래스 어노테이션으로 @RunWith(MockitoJUnitRunner.class)를 붙여주어야 연동이 가능했다.
- 하지만 SpringBoot 2.2.0부터 공식적으로 JUnit5를 지원함에 따라, 이제부터는 @ExtendWith(MockitoExtension.class)를 사용해야 결합이 가능하다.

## 2. Spring 컨트롤러 단위 테스트 작성 예시
### 사용자 회원 가입 / 목록 조회 API
- 예를 들어 다음과 같은 회원 가입 API와 사용자 목록 조회 API가 있고, 이에 대한 단위 테스트를 작성해주어야 한다고 하자.

```java
@RestController
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/users/signUp")
    public ResponseEntity<UserResponse> signUp(@RequestBody SignUpRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.signUp(request));
    }

    @GetMapping("/users")
    public ResponseEntity<List<UserResponse>> findAll() {
        return ResponseEntity.ok(userService.findAll());
    }
}
``` 

### 단위 테스트(Unit Test) 작성 준비
- 앞서 설명하였듯 JUniit5와 Mockito를 연동하기 위해서는 @ExtendWith(MockitoExtension.class)를 사용해야 한다.
- 이를 클래스 어노테이션으로 붙여 다음과 같이 테스트 클래스를 작성할 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {

}
```
- 이제 의존성 주입을 해주어야 한다.
- 먼저 테스트 대상인 UserController에는 가짜 객체 주입을 위한 @InjectMocks를 붙여주어야 한다.
  - 그리고 UserService에는 가짜 객체 생성을 위해 @Mock 어노테이션을 붙여주면 된다.

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {

    @InjectMocks
    private UserController userController;

    @Mock
    private UserService userService;

}
```

- 컨트롤러를 테스트하기 위해서는 HTTP 호출이 필요하다.
  - 일반적인 방법으로는 HTTP 호출이 불가능하므로 스프링에서는 이를 위한 MockMVC를 제공하고 있다.
  - MockMvc는 다음과 같이 생성할 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {

    @InjectMocks
    private UserController userController;

    @Mock
    private UserService userService;

    private MockMvc mockMvc;

    @BeforeEach
    public void init() {
        mockMvc = MockMvcBuilders.standaloneSetup(userController).build();
    }

}
```

- 그러면 이제 UserController 테스트를 위한 준비가 끝났으므로, 다음의 케이스들에 대해 테스트 코드를 작성해주도록 하자.
  - 회원가입 성공
  - 사용자 목록 조회
 
#### 1) 회원가입 성공 테스트
- 우선 회원가입 요청을 보내기 위해서는 SignUpRequest 객체 1개와 userService의 signUp에 대한 stub이 필요하다.
- 이러한 준비 작업을 해주면 given 단계에 다음과 같은 테스트 코드가 작성된다.

```java
@DisplayName("회원 가입 성공")
@Test
void signUpSuccess() throws Exception {
    // given
    SignUpRequest request = signUpRequest();
    UserResponse response = userResponse();
    
    doReturn(response).when(userService)
        .signUp(any(SignUpRequest.class));
}

private SignUpRequest signUpRequest() {
    return SignUpRequest.builder()
        .email("test@test.test")
        .pw("test")
        .build();
}

private UserResponse userResponse() {
    return UserResponse.builder()
        .email("test@test.test")
        .pw("test")
        .role(UserRole.ROLE_USER)
        .build();
}
```

- HTTP 요청을 보내면 Spring은 내부에서 MessageConverter를 사용해 Json String을 객체로 변환한다.
  - 그런데 이것은 Spring 내부에서 진행되므로, 우리가 API로 전달되는 파라미터인 SignUpRequest를 조작할 수 없다.
- 그래서 SignUpRequest 클래스 타입이라면 어떠한 객체도 처리할 수 있도록 any()가 사용되었다.
  - any()를 사용할 때에는 특정 클래스의 타입을 지정해주는 것이 좋다.
- 그 다음 when 단계에서는 mockMVC에 데이터와 함께 POST 요청을 보내야 한다.
  - 요청 정보는 mockMvc의 perform에서 작성 가능한데, 요청 정보에는 MockMvcRequestBuilders가 사용되며 요청 메소드 종류, 내용, 파라미터 등을 설정할 수 있다.
  - 보내는 데이터는 객체가 아닌 문자열이여야 하므로 별도의 변환이 필요하므로 Gson을 사용해 변환하였다.

```java
@DisplayName("회원 가입 성공")
@Test
void signUpSuccess() throws Exception {
    // given
    SignUpRequest request = signUpRequest();
    UserResponse response = userResponse();
    
    doReturn(response).when(userService)
        .signUp(any(SignUpRequest.class));

    // when
    ResultActions resultActions = mockMvc.perform(
        MockMvcRequestBuilders.post("/users/signUp")
                .contentType(MediaType.APPLICATION_JSON)
                .content(new Gson().toJson(request))
    );

}
```

- 마지막으로 호출된 결과를 검증하는 then 단계에서는 회원가입 API 호출 결과로 200 Response와 응답 결과를 검증해야 한다.
- 응답 검증 시에는 jsonPath를 이용해 해당 json 값이 존재하는지 확인하면 된다.

```java
@DisplayName("회원 가입 성공")
@Test
void signUpSuccess() throws Exception {
    // given
    SignUpRequest request = signUpRequest();
    UserResponse response = userResponse();
    
    doReturn(response).when(userService)
        .signUp(any(SignUpRequest.class));

    // when
    ResultActions resultActions = mockMvc.perform(
        MockMvcRequestBuilders.post("/users/signUp")
                .contentType(MediaType.APPLICATION_JSON)
                .content(new Gson().toJson(request))
    );

    // then
    MvcResult mvcResult = resultActions.andExpect(status().isOk())
        .andExpect(jsonPath("email", response.getEmail()).exists())
        .andExpect(jsonPath("pw", response.getPw()).exists())
        .andExpect(jsonPath("role", response.getRole()).exists())
}
```

#### 2) 사용자 목록 조회 테스트
- 사용자 목록 조회의 given 단계에서는 UserService의 findAll에 대한 Stub이 필요하다.
- 그리고 when 단계에서는 호출하는 HTTP 메소드를 GET으로, URL을 "/users/list"로 작성해주어야 한다.
- 마지막으로 then 단계에서는 HTTP Status가 OK이며, 주어진 데이터가 올바른지를 검증해야 하는데 이번에는 Json 응답을 객체로 변환하여 확인해보록 하자.

```java
@DisplayName("사용자 목록 조회")
@Test
void getUserList() throws Exception {
    // given
    doReturn(userList()).when(userService)
        .findAll();

    // when
    ResultActions resultActions = mockMvc.perform(
        MockMvcRequestBuilders.get("/user/list")
    );

    // then
    MvcResult mvcResult = resultActions.andExpect(status().isOk()).andReturn();
    
    UserListResponseDTO response = new Gson().fromJson(mvcResult.getResponse().getContentAsString(), UserListResponseDTO.class);
    assertThat(response.getUserList().size()).isEqualTo(5);
}

private List<UserResponse> userList() {
    List<UserResponse> userList = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        userList.add(new UserResponse("test@test.test", "test", UserRole.ROLE_USER));
    }
    return userList;
}
```

#### 3) @WebMvcTest
- 위와 같이 MockMvc를 생성하는 등의 작업은 번거롭다.
- 다행히도 SpringBoot는 컨트롤러 테스트를 위한 @WebMvcTest 어노테이션을 제공하고 있다.
  - 이를 이용하면 MockMvc 객체가 자동으로 생성될 뿐만 아니라 ControllerAdvice나 Filter, Interceptor 등 웹 계층 테스트에 필요한 요소들을 모두 빈으로 등록해 스프링 컨텍스트 환경을 구성한다.
  - @WebMvcTest는 스프링 부트가 제공하는 테스트 환경이므로 @Mock과 @Spy 대신 각각 @MockBean과 @SpyBean을 사용해주어야 한다.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @MockBean
    private UserService userService;

    @Autowired
    private MockMvc mockMvc;

    // 테스트 작성

}
```
- 하지만 여기서 주의할 점이 있다.
  - 스프링은 내부적으로 스프링 컨텍스트를 캐싱해두고 동일한 테스트 환경이라면 재사용한다.
  - 그런데 특정 컨트롤러만을 빈으로 만들고 @MockBean과 @SpyBean으로 빈을 모킹하는 @WebMvcTest는 캐싱의 효과를 거의 얻지 못하고 새로운 컨텍스트의 생성을 필요로 한다.
- 그러므로 빠른 테스트를 원한다면 직접 MockMvc를 생성했던 처음의 방법을 사용하는 것이 좋을 수 있다.

 
## 3. Spring 서비스 계층 단위 테스트 작성 예시
### 사용자 회원 가입 / 목록 조회 비즈니스 로직 
- 사용자 회원가입과 목록 조회를 위해서는 다음과 같은 비즈니스 로직 레이어(Service Layer)가 필요하다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl {

    private final UserRepository userRepository;
    private final BCryptPasswordEncoder passwordEncoder;

    @Transactional
    public UserResponse signUp(final SignUpRequest request) {
        final User user = User.builder()
                .email(request.getEmail())
                .pw(passwordEncoder.encode(request.getPw()))
                .role(UserRole.ROLE_USER)
                .build();

        return UserResponse.of(userRepository.save(user));
    }

    public List<User> findAll() {
        return userRepository.findAll().stream()
            .map(UserResponse::of)
            .collect(Collectors.toList()));
    }
}
```

### 단위 테스트(Unit Test) 작성 준비
- 앞서 설명하였듯 @ExtendWith(MockitoExtension.class)와 가짜 객체 주입을 사용해 다음과 같은 테스트 클래스를 작성할 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @InjectMocks
    private UserService userService;

    @Mock
    private UserRepository userRepository;

    @Spy
    private BCryptPasswordEncoder passwordEncoder;

}
```

- 이번에는 BCryptPasswordEncoder에 @Spy를 사용하였다.
- 앞서 설명하였듯 Spy는 Stub하지 않은 메소드는 실제 메소드로 동작하게 하는데, 위의 예제에서 실제로 사용자 비밀번호를 암호화해야 하므로, @Spy를 사용하였다.

#### 1) 회원가입 성공 테스트
```java
@DisplayName("회원 가입")
@Test
void signUp() {
    // given
    BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    SignUpRequest request = signUpRequest();
    String encryptedPw = encoder.encode(request.getPw());

    doReturn(new User(request.getEmail(), encryptedPw, UserRole.ROLE_USER)).when(userRepository)
        .save(any(User.class));
        
    // when
    UserResponse user = userService.signUp(request);

    // then
    assertThat(user.getEmail()).isEqualTo(request.getEmail());
    assertThat(encoder.matches(signUpDTO.getPw(), user.getPw())).isTrue();

    // verify
    verify(userRepository, times(1)).save(any(User.class));
    verify(passwordEncoder, times(1)).encode(any(String.class));
}
```

- 이번에는 추가적으로 mockito의 verify()를 사용해보았다.
- verify는 Mockito 라이브러리를 통해 만들어진 가짜 객체의 특정 메소드가 호출된 횟수를 검증할 수 있다.
- 위에서는 passwordEncoder의 encode 메소드와 userRepository의 save 메소드가 각각 1번만 호출되었는지를 검증하기 위해 사용하였다.

 
#### 2) 사용자 목록 조회 테스트
```java
@DisplayName("사용자 목록 조회")
@Test
void findAll() {
    // given
    doReturn(userList()).when(userRepository)
        .findAll();

    // when
    final List<UserResponse> userList = userService.findAll();

    // then
    assertThat(userList.size()).isEqualTo(5);
}

private List<User> userList() {
    List<User> userList = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        userList.add(new User("test@test.test", "test", UserRole.ROLE_USER));
    }
    return userList;
}
```

## 4. Spring 레포지토리 계층 단위 테스트 작성 예시
### 사용자 추가/목록 조회 코드 
- 사용자 회원가입과 목록 조회를 위한 JPA 레포지토리 인터페이스는 다음과 같이 구현되어 있다.

```java
public interface UserRepository extends JpaRepository <User, Long> {

}
```

#### @DataJpaTest 어노테이션
- 스프링 부트는 JPA 레포지토리를 손쉽게 테스트할 수 있는 @DataJpaTest 어노테이션을 제공하고 있다.
- @DataJpaTest를 사용하면 기본적으로 인메모리 데이터베이스인 H2를 기반으로 테스트용 데이터베이스를 구축하며, 테스트가 끝나면 트랜잭션 롤백을 해준다.
- 레포지토리 계층은 실제 DB와 통신 없이 단순 모킹하는 것은 의미가 없으므로 직접 데이터베이스와 통신하는 @DataJpaTest를 사용하도록 하자.

 
#### 1) 사용자 추가 테스트
```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @DisplayName("사용자 추가")
    @Test
    void addUser() {
        // given
        User user = user();
        
        // when
        User savedUser = userRepository.save(user);

        // then
        assertThat(savedUser.getEmail()).isEqualTo(user.getEmail());
        assertThat(savedUser.getPw()).isEqualTo(user.getPw());
        assertThat(savedUser.getRole()).isEqualTo(user.getRole());
    }

    private User user() {
        return User.builder()
                .email("email")
                .pw("pw")
                .role(UserRole.ROLE_USER).build();
    }
}
``` 

#### 2) 사용자 목록 조회 테스트
```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @DisplayName("사용자 목록 조회")
    @Test
    void addUser() {
        // given
        userRepository.save(user());
        
        // when
        List<User> userList = userRepository.findAll();

        // then        
        assertThat(userList.size()).isEqualTo(1);
    }
}
```
