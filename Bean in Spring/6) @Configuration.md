# @Configuration와 싱글톤

## 개요
### AppConfig
```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public OrderService orderService() {
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }

    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
}
```
- @Bean memberService → new MemoryMemberRepository()
- @Bean orderService → new MemoryMemberRepository()
- 결과적으로 각각 다른 2개의 MemoryMemberRepository가 생성되면서 싱글톤이 깨지는 것처럼 보인다.

## 스프링 컨테이너의 문제 해결
### ConfigurationSingletonTest

```java
public class ConfigurationSingletonTest {

    @Test
    void configurationTest() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);

        MemberServiceImpl memberService = ac.getBean("memberService", MemberServiceImpl.class);
        OrderServiceImpl orderService = ac.getBean("orderService", OrderServiceImpl.class);
        MemberRepository memberRepository = ac.getBean("memberRepository", MemberRepository.class);

        MemberRepository memberRepository1 = memberService.getMemberRepository();
        MemberRepository memberRepository2 = orderService.getMemberRepository();

        System.out.println("memberService -> memberRepository1 = " + memberRepository1);
        System.out.println("orderService -> memberRepository2 = " + memberRepository2);
        System.out.println("memberRepository = " + memberRepository);

        assertThat(memberService.getMemberRepository()).isSameAs(memberRepository);
        assertThat(orderService.getMemberRepository()).isSameAs(memberRepository);
    }
}
```

- memberRepository 인스턴스는 모두 같은 인스턴스가 공유되어 사용된다.
- AppConfig의 자바 코드를 보면 분명히 각각 2번 new MemoryMemberRepository호출해서 다른 인스턴스가 생성되어야 하는데..?

### AppConfig에 호출 로그 추가
```java
@Configuration
public class AppConfig {

    @Bean
    public MemberService memberService() {
        System.out.println("Call AppConfig.memberService");
        return new MemberServiceImpl(memberRepository());
    }

    @Bean
    public OrderService orderService() {
        System.out.println("Call AppConfig.orderService");
        return new OrderServiceImpl(memberRepository(), discountPolicy());
    }

    @Bean
    public DiscountPolicy discountPolicy() {
        return new RateDiscountPolicy();
    }

    @Bean
    public MemberRepository memberRepository() {
        System.out.println("Call AppConfig.memberRepository");
        return new MemoryMemberRepository();
    }
}
```

- 스프링 컨테이너가 각각 @Bean을 호출해서 스프링 빈을 생성한다.
- 그래서 memberRepository()는 다음과 같이 총 3번 호출될 것으로 추정된다.
  - 스프링 컨테이너가 스프링 빈에 등록하기 위해 @Bean이 붙어 있는 memberRepository() 호출
  - memberService() 로직에서 memberRepository() 호출
  - orderService() 로직에서 memberRepository() 호출

### 실행결과
```bash
call AppConfig.memberService
call AppConfig.memberRepository    // 한 번만 호출된다!!!
call AppConfig.orderService
```

## @Configuration과 바이트코드 조작
- 스프링 컨테이너는 싱글톤 레지스트리다. 따라서 스프링 빈이 싱글톤이 되도록 보장해주어야 한다.
  - 그런데 스프링이 자바 코드까지 어떻게 하기는 어렵다.
  - 저 자바 코드를 보면 분명 3번 호출되어야 하는 것이 맞다.
- 그래서 스프링은 클래스의 바이트코드를 조작하는 라이브러리를 사용한다.
- 모든 비밀은 @Configuration을 적용한 AppConfig에 있다.

```java
ConfigurationSingletonTest
public class ConfigurationSingletonTest {
	@Test
    void configurationDee() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AppConfig.class);
        AppConfig bean = ac.getBean(AppConfig.class);

        System.out.println("bean = " + bean.getClass());

    }
}
```
- 사실 AnnotationConfigApplicationContext에 파라미터로 넘긴 값은 스프링 빈으로 등록된다.
  - 그래서 AppConfig는 스프링 빈이 된다.
- AppConfig 스프링 빈을 조회하여 클래스 정보를 출력하면 다음과 같다.
  - bean = class hello.core.config.AppConfig$$EnhancerBySpringCGLIB$$e4a3e35f
- 순수한 클래스라면 class hello.core.AppConfig로 출력되어야 한다.
  - 그런데 예상과는 다르게 클래스명에 xxxCGLIB가 붙으면서 상당히 복잡해진 것을 볼 수 있다.
  - 이것은 스프링이 CGLIB라는 바이트코드 조작 라이브러리를 사용해서 AppConfig 클래스를 상속받은 임의의 다른 클래스를 만들고, 그 다른 클래스를 스프링 빈으로 등록한 것이다.
  - 그 임의의 다른 클래스가 바로 싱글톤이 보장되도록 해준다.
- 아마도 다음과 같이 바이트 코드를 조작해서 작성되어 있을 것이다.
  - 실제로는 CGLIB의 내부 기술을 사용하는데, 매우 복잡하다.

### AppConfig@CGLIB 예상 코드
- @Bean이 붙은 메서드마다 이미 스프링 빈이 존재하면 존재하는 빈을 반환하고, 스프링 빈이 없으면 생성해서 스프링 빈으로 등록하고 반환하는 코드가 동적으로 만들어진다.
  - 덕분에 싱글톤이 보장되는 것이다.
- AppConfig@CGLIB는 AppConfig의 자식 타입으로, AppConfig 타입으로 조회할 수 있다.

### @Configuration을 적용하지 않고, @Bean만 적용하면 어떻게 될까?
- @Configuration을 붙이면 바이트코드를 조작하는 CGLIB 기술을 사용해서 싱글톤을 보장하지만, 만약 @Bean만 적용하면 어떻게 될까?

#### AppConfig
```java
//@Configuration
public class AppConfig {
	...
}
```

#### 실행 결과
```bash
call AppConfig.memberService
call AppConfig.memberRepository
call AppConfig.orderService
call AppConfig.memberRepository
call AppConfig.memberRepository
bean = class hello.core.AppConfig
```

- MemberRepository가 총 3번 호출되었으며, AppConfig가 CGLIB 기술 없이 순수한 AppConfig로 스프링 빈에 등록된 것을 확인할 수 있다.
  - 1번은 @Bean에 의해 스프링 컨테이너에 등록하기 위해서,
  - 2번은 각각 memberRepository()를 호출하면서 발생한 코드다.
  - 각각의 인스턴스는 모두 다르다.
 

## 정리
- @Bean만 사용해도 스프링 빈으로 등록되지만, 싱글톤을 보장하지 않는다.
- memberRepository()처럼 의존관계 주입이 필요해서 메서드를 직접 호출할 때 싱글톤을 보장하지 않는다.
