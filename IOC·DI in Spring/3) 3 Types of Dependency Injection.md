# 의존성 주입의 3가지 방법

## 1. 생성자 주입(Constructor Injection) 
- 생성자 주입(Constructor Injection)은 생성자를 통해 의존 관계를 주입하는 방법이다.

```java
@Service
public class UserService {

    private UserRepository userRepository;
    private MemberService memberService;

    @Autowired
    public UserService(UserRepository userRepository, MemberService memberService) {
        this.userRepository = userRepository;
        this.memberService = memberService;
    }
    
}
```

- 생성자 주입은 생성자의 호출 시점에 1회 호출되는 것이 보장된다.
  - 그렇기 때문에 주입받은 객체가 변하지 않거나, 반드시 객체의 주입이 필요한 경우에 강제하기 위해 사용할 수 있다.
- 또한 Spring 프레임워크에서는 생성자 주입을 적극 지원하고 있기 때문에, 생성자가 1개만 있을 경우에 @Autowired를 생략해도 주입이 가능하도록 편의성을 제공하고 있다.
  - 그렇기 때문에 위의 코드는 아래와 동일한 코드가 된다.

```java
@Service
public class UserService {

    private UserRepository userRepository;
    private MemberService memberService;

    public UserService(UserRepository userRepository, MemberService memberService) {
        this.userRepository = userRepository;
        this.memberService = memberService;
    }
}
```

## 2. 수정자 주입(Setter 주입, Setter Injection)
- 수정자 주입(Setter 주입, Setter Injection)은 필드 값을 변경하는 Setter를 통해서 의존 관계를 주입하는 방법이다.
- Setter 주입은 생성자 주입과 다르게 주입받는 객체가 변경될 가능성이 있는 경우에 사용한다. (실제로 변경이 필요한 경우는 극히 드물다.)

```java
@Service
public class UserService {

    private UserRepository userRepository;
    private MemberService memberService;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Autowired
    public void setMemberService(MemberService memberService) {
        this.memberService = memberService;
    }
}
```

- @Autowired로 주입할 대상이 없는 경우에는 오류가 발생한다.
  - 위의 예제에서는 XXX 빈이 존재하지 않을 경우에 오류가 발생하는 것이다.
- 주입할 대상이 없어도 동작하도록 하려면 @Autowired(required = false)를 통해 설정할 수 있다.
- 스프링 초기에는 수정자 주입이 자주 사용되었는데, 그 이유는 바로 getX, setX 등 프로퍼티를 기반으로 하는 자바 기본 스펙 때문이였다.
  - 하지만 시간이 지나면서 점차 수정자 주입이 아닌 다른 방식이 주목받게 되었다.

## 3. 필드 주입(Field Injection)
- 필드 주입(Field Injection)은 필드에 바로 의존 관계를 주입하는 방법이다.
- IntelliJ에서 필드 인젝션을 사용하면 Field injection is not recommended이라는 경고 문구가 발생한다.

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private MemberService memberService;

}
```

- 필드 주입을 이용하면 코드가 간결해져서 과거에 상당히 많이 이용되었던 주입 방법이다.
- 하지만 필드 주입은 외부에서 접근이 불가능하다는 단점이 존재하는데, 테스트 코드의 중요성이 부각됨에 따라 필드의 객체를 수정할 수 없는 필드 주입은 거의 사용되지 않게 되었다.
- 또한 필드 주입은 반드시 __DI 프레임워크가 존재해야 하므로__ 사용을 지양해야 한다.
- 그렇기에 애플리케이션의 실제 코드와 무관한 테스트 코드나 설정을 위해 불가피한 경우에만 이용하도록 하자.
