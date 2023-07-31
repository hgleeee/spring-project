# 생성자 주입을 사용해야 하는 이유

- 최근에는 Spring을 포함한 DI 프레임워크의 대부분이 생성자 주입을 권장하고 있는데, 자세한 이유를 살펴보도록 하자.

```
1) 객체의 불변성 확보
2) 테스트 코드의 작성
3) final 키워드 작성 및 Lombok과의 결합
4) 스프링에 비침투적인 코드 작성
5) 순환 참조 에러 방지
```

## 1. 객체의 불변성 확보
- 실제로 개발을 하다 보면 의존 관계의 변경이 필요한 상황은 거의 없다.
- 하지만 수정자 주입이나 일반 메소드 주입을 이용하면 불필요하게 수정의 가능성을 열어두어 유지보수성을 떨어뜨린다.
- 그러므로 생성자 주입을 통해 변경의 가능성을 배제하고 불변성을 보장하는 것이 좋다.

## 2. 테스트 코드의 작성
- 테스트가 특정 프레임워크에 의존하는 것은 침투적이므로 좋지 못하다.
- 그러므로 가능한 순수 자바로 테스트를 작성하는 것이 가장 좋은데, 생성자 주입이 아닌 다른 주입으로 작성된 코드는 순수한 자바 코드로 단위 테스트를 작성하는 것이 어렵다.

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private MemberService memberService;

    public void register(String name) {
        userRepository.add(name);
    }

}
```

- 예를 들어, 위와 같은 코드에 대해 순수 자바 테스트 코드를 작성하면 다음과 같이 작성할 수 있다.

```java
public class UserServiceTest {

    @Test
    public void addTest() {
        UserService userService = new UserService();
        userService.register("Lee");
    }

}
```

- 위의 테스트 코드는 Spring 위에서 동작하지 않으므로 의존 관계 주입이 되지 않을 것이고, userRepository가 null이 되어 add 호출 시 NPE가 발생할 것이다.
  - 이를 해결하기 위해 Setter를 사용하면 변경 가능성을 열어두게 되는 단점을 갖게 된다.
  - 반대로 테스트 코드에서 @Autowired를 사용하기 위해 스프링을 사용하면 단위 테스트가 아닐 뿐만 아니라, 컴포넌트들을 등록하고 초기화하는 시간 때문에 테스트 비용이 증가하게 된다.
  - 그렇다고 대안으로 리플렉션을 사용하면 깨지기 쉬운 테스트가 된다.
- 반면에 생성자 주입을 사용하면 컴파일 시점에 객체를 주입받아 테스트 코드를 작성할 수 있으며, 주입하는 객체가 누락된 경우 컴파일 시점에 오류를 발견할 수 있다.
  - 심지어 우리가 테스트를 위해 만든 Test객체를 생성자로 넣어 편리함을 얻을 수도 있다.

## 3. final 키워드 작성 및 Lombok과의 결합
- 생성자 주입을 사용하면 필드 객체에 final 키워드를 사용할 수 있으며, 컴파일 시점에 누락된 의존성을 확인할 수 있다.
  - 반면에 다른 주입 방법들은 객체의 생성(생성자 호출) 이후에 호출되므로 final 키워드를 사용할 수 없다.
- 또한 final 키워드를 붙이면 Lombok과 결합되어 코드를 간결하게 작성할 수 있다.
- Lombok에는 final 변수를 위한 생성자를 대신 생성해주는 @RequiredArgsConstructor 가 존재한다.
- Spring과 같은 DI 프레임워크는 Lombok과 환상적인 궁합을 보여주는데, 위에서 작성했던 생성자 주입 코드를 Lombok과 결합시키면 다음과 같이 간편하게 작성할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final MemberService memberService;

    public void register(String name) {
        userRepository.add(name);
    }

}
```
- 이러한 코드가 가능한 이유는 앞서 설명했듯 Spring에서는 생성자가 1개인 경우 @Autowired를 생략할 수 있도록 도와주고 있으며, 해당 생성자를 Lombok으로 구현하였기 때문이다.

 
## 4. 스프링에 비침투적인 코드 작성
- 필드 주입을 사용하려면 @Autowired를 이용해야 하는데, 이것은 스프링이 제공하는 어노테이션이다.
- 그러므로 @Autowired를 사용하면 다음과 같이 UserService에 스프링 의존성이 침투하게 된다. 

```java
import org.springframework.beans.factory.annotation.Autowired;
// 스프링 의존성이 UserService에 import되어 코드로 박혀버림

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private MemberService memberService;

}
```

- 우리가 사용하는 프레임워크는 언제 바뀔지도 모를 뿐만 아니라, 사용자와 관련된 책임을 지는 UserService에 스프링 코드가 박혀버리는 것은 바람직하지 않다.
  - 프레임워크는 비즈니스 로직을 작성하는 서비스 계층에서 알아야 할 대상이 아니다.
- 물론 이는 필요한 자바 파일을 임포트해야 하는 정적 언어인 자바의 한계이기도 하다.
  - 그래도 가능하다면 스프링이 없이 코드가 작성되면 더욱 유연한 코드를 확보하게 된다.
- 프레임워크가 자주 바뀌는 것도 아니므로 비록 스프링 코드가 침투하는게 치명적인 문제는 아니긴하다.
  - 하지만 그래도 더 좋은 방법(생성자 주입)이 있는데, 굳이 사용할 필요는 없다.

## 5. 순환 참조 에러 방지
- 생성자 주입을 사용하면 애플리케이션 구동 시점(객체의 생성 시점)에 순환 참조 에러를 예방할 수 있다.
- 예를 들어 다음과 같이 필드을 사용해 서로 호출하는 코드가 있다고 하자.

```java
@Service
public class UserService {

    @Autowired
    private MemberService memberService;
    
    @Override
    public void register(String name) {
        memberService.add(name);
    }

}
```

- UserSerivce가 이미 MemberService에 의존하고 있는데, MemberService 역시 UserService에 의존하는 것이다.

```java
@Service
public class MemberService {

    @Autowired
    private UserService userService;

    public void add(String name){
        userService.register(name);
    }

}
``` 

- 위의 두 메소드는 서로를 계속 호출할 것이고, 메모리에 함수의 CallStack이 계속 쌓여 StackOverflow 에러가 발생하게 된다.

```bash
Caused by: java.lang.StackOverflowError: null
	at com.mang.example.user.MemberService.add(MemberService.java:20) ~[main/:na]
	at com.mang.example.user.UserService.register(UserService.java:14) ~[main/:na]
	at com.mang.example.user.MemberService.add(MemberService.java:20) ~[main/:na]
	at com.mang.example.user.UserService.register(UserService.java:14) ~[main/:na]
```

 
- 만약 이러한 문제를 발견하지 못하고 서버가 운영된다면 어떻게 되겠는가? 해당 메소드의 호출 시에 StackOverflow 에러에 의해 서버가 죽게 될 것이다.
- 하지만 생성자 주입을 이용하면 이러한 순환 참조 문제를 방지할 수 있다.


```bash
Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  memberService defined in file [C:\Users\Mang\IdeaProjects\build\classes\java\main\com\mang\example\user\MemberService.class]
↑     ↓
|  userService defined in file [C:\Users\Mang\IdeaProjects\build\classes\java\main\com\mang\example\user\UserService.class]
└─────┘
```

- 애플리케이션 구동 시점(객체의 생성 시점)에 위와 같은 에러가 발생하기 때문이다.
- 그러한 이유는 Bean에 등록하기 위해 객체를 생성하는 과정에서 다음과 같이 순환 참조가 발생하기 때문이다.

```java
new UserService(new MemberService(new UserService(new MemberService()...)))
```

- @Autowired를 이용한 필드 주입에서 이러한 문제가 애플리케이션 구동 시점에 에러가 발생하지 않는 이유는 빈의 생성과 조립(@Autowired) 시점이 분리되어 있기 때문이다.
  - 생성자 주입은 객체의 생성과 조립(의존관계 주입)이 동시에 실행되다 보니 위와 같은 에러를 사전에 잡을 수 있다.
  - 하지만 필드 주입은 모든 객체의 생성이 완료된 후에 조립(의존관계 주입)이 처리된다.
- 그러다 보니 위와 같이 호출이 되고 나서야 순환 이슈를 확인할 수 있는 것이다.


## 생성자 주입 사용 이유 정리
- 객체의 불변성을 확보할 수 있다.
- 테스트 코드의 작성이 용이해진다.
- final 키워드를 사용할 수 있고, Lombok과의 결합을 통해 코드를 간결하게 작성할 수 있다.
- 스프링에 침투적이지 않은 코드를 작성할 수 있다.
- 순환 참조 에러를 애플리케이션 구동(객체의 생성) 시점에 파악하여 방지할 수 있다.
