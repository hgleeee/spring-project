# 싱글톤 레지스트리
## 스프링에서의 빈 객체
- 컨테이너(Object Factory)에서 같은 객체를 생성하게 되면 두 객체가 가진 값은 같지만, 실제 등록된 객체는 다른 객체이다.
  - 즉, '동등성'은 만족하나 '동일성'은 만족하지 않는다. 객체를 생성할 때마다 중복해서 만들게 된다.

```java
DaoFactory factory = new DaoFactory(); // AppConfig 객체를 의미한다.
 
// 이 두 객체는 다르다.
UserDao dao1 = factory.userDao();
UserDao dao2 = factory.userDao();
 
// my.package.UserDao@118f375
// my.package.UserDao@117A8bd
System.out.println(dao1 + "\n" + dao2);
```

- 스프링에서는 이러한 중복(낭비)을 막기 위해 객체를 싱글톤으로 등록한다.

```java
// Object Factory 객체를 스프링 설정파일로 넘겨주어, 스프링 컨테이너를 사용하는 방법
ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
 
// 그렇다면 이 두 객체도 동일성은 만족하지 못하는 걸까?
UserDao dao1 = context.getBean("userDao", UserDao.class);
UserDao dao2 = context.getBean("userDao", UserDao.class);
 
// 놀랍게도 동일성을 만족한다. 몇 개를 생성하던 동일한 하나의 객체만 생성하고, 참조한다.(싱글톤)
// my.package.UserDao@ee22f7
// my.package.UserDao@ee22f7
System.out.println(dao1 + "\n" + dao2);
```

## 싱글톤 레지스트리로써 ApplicationContext
- 스프링에서 사용하는 컨테이너, Application Context는 Object Factory와 비슷한 방식으로 동작한다.
  - 하지만 스프링 컨테이너는 이와 동시에 싱글톤을 저장하고 관리하는 싱글톤 레지스트리(singleton registry)이기도 하다.
- 스프링은 필요에 따라 따로 설정하는 게 아니라면, 기본적으로 생성하는 빈 객체는 전부 싱글톤으로 만든다.
  - 디자인 패턴에서 나오는 싱글톤 패턴과 비슷한 개념이지만, 구현 방법은 확연히 다르다.

### 스프링에서 싱글톤을 만드는 방법
- 스프링은 CGLIB라는 바이트코드 조작 라이브러리를 이용하여 싱글톤을 구현한다.
- 실제 컴파일한 바이트코드를 살펴보면 AppConfig.class가 메모리에 로드되는게 아닌, AppConfig@CGLIB 라는 상속받은 클래스가 사용된다.

```java
// CGLIB는 자바 코드가 아닌 바이트코드를 직접 조작하기 때문에 사용 방법이 어렵고 복잡하다.
// 이해를 돕기 위해 자바 코드로 비유하면 아래와 같이 동작한다.

class AppConfig@CGLIB123 extends AppConfig {
  @Bean
  public MemberRepository memberRepository() {
    if (memoryMemberRepository가 스프링 컨테이너에 등록되어 있다면) {
      return 스프링 컨테이너에서 찾아서 반환;
    } else { // 스프링 컨테이너에 없다면
      기존 로직을 호출해 MemoryMemberRepository를 생성하고 스프링 컨테이너에 등록
      return 반환;
    }
  }
  // ... 생략 ...
}
```

### 코드로 구현한 싱글톤의 한계
- 근데 이렇게 바이트코드를 조작하면서까지 싱글톤을 구현해야 하는 이유는 무엇일까?
  - 그 이유는 스프링은 태생적으로 서버(웹 서버)에 주로 사용되는 프레임워크이기 때문이다.
- 스프링이 처음 설계되었던 대규모 서비스를 처리하는 서버 환경은 초당 수십, 수백 번에 이르는 클라이언트의 요청을 처리한다.
  - 그 하나의 요청을 처리하기 위해 [데이터 엑세스(DB) 객체, 서비스 객체, 핵심 비즈니스 로직 객체] 등 수많은 객체들이 참여하는 계층적인 구조로 이루어지는 것이 대부분이다. 
- 매번 클라이언트의 요청이 올 때마다 각 로직을 담당하는 객체를 새로 만든다고 생각해보자.
  - 초당 500개의 요청이 들어온다면 2500개의 새로운 객체가 생성된다.
  - 1분이면 십만 개, 한 시간이면 서버에 900만 개의 오브젝트가 생성되고 사용될 것이다.
  - 아무리 JVM과 GC의 기능이 좋아졌다고 한들, 객체를 생성하고 소멸시키는 과정은 연산 비용이 비싼 작업이다.
  - 그래서 여러 스레드에서 하나의 객체를 공유해 돌려쓰도록 싱글톤을 구현하는 것이다.
- 다만 코드로도 싱글톤을 구현할 수는 있다. 하지만 실제 서비스에서 사용하기에는 아래와 같은 한계점이 있다.

```java
public class DoubleLockSingleton {
    // 단순한 싱글톤 구현 방법
    private static final Singleton instance = new Singleton();
    
    // private 생성자를 이용하여 같은 객체가 중복 생성되는 것을 막음.
    private Singleton(){}
 
    public static Singleton getInstance(){
        // 멀티스레드에서 싱글톤 객체 생성의 Thread-safety 보장
        if(instance == null){
            synchronized (Singleton.class) {
                if(instance == null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

#### 한계점
1. private 생성자 때문에 객체를 상속할 수 없다. 서비스 객체에 다형성을 적용하기 어렵다.
2. 실제 객체가 아닌 테스트용 객체(mock)를 사용하기 어렵다. 즉 테스트 환경 구성이 어려워진다.
3. 서버에서 클래스 로더를 어떻게 구성하고 있는가에 따라 싱글톤 클래스임에도 하나 이상의 오브젝트가 만들어질 수 있다.
    1. 여러 개의 JVM에 분산되어 설치되는 경우에도 싱글톤 보장 X
4. 싱글톤 객체가 '전역 변수'처럼 사용될 수 있는 가능성을 가지고 있다. 이러한 방식은 side-effect를 유발한다.

### 싱글톤 레지스트리의 등장
- 그래서 스프링은 바이트코드 조작을 통해 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공하는데, 이를 싱글톤 레지스트리라고 부른다.
- 싱글톤 레지스트리는 코드 상에서 static 메서드와 private 생성자를 이용해야하는 비정상적인 싱글톤 클래스가 아니라, 평범한 자바 클래스를 싱글톤 객체로 사용할 수 있게 만들어준다.
- 싱글톤 레지스트리는 싱글톤 패턴과 달리 자바의 객체지향적인 설계 방식이나 코드를 활용하는데 아무런 제약이 없다는 큰 장점이 있다.
  - 다만, '싱글톤' 패턴이 가지는 위험성은 변함이 없기에 주의해서 사용하는 것을 잊으면 안 된다.

## 주의! 싱글톤과 오브젝트 상태
- 싱글톤은 멀티스레드 환경이라면 여러 스레드가 동시에 접근해서 사용할 수 있다. 따라서 '객체'가 상태 값을 가져서는 안 된다.
  - 상태 데이터를 가지고 있지 않는 Stateless 방식으로 싱글톤을 사용해야 한다.
  - 더 쉽게 말하면 싱글톤 오브젝트의 인스턴스 변수를 수정하는 작업은 매우 위험하다는 말이다.
- 여러 스레드에서 사용될 가능성이 있기에 Thread-Safe 하지 않다.
- 더 큰 문제는 스프링에서 싱글톤 레지스트리를 통해 관리해주다 보니, 개발자가 '싱글톤 객체'라는 사실을 잊고 상태를 저장할 수 있다는 위험성이 있다.
- 멀티스레드 환경에서 이러한 오류가 발생하게 되면, 그 오류를 잡는 것은 상상 이상으로 어렵다.
  - 사용자가 로그인을 했는데 다른 사람의 데이터가 보이는 상황이라면, 어디서부터 어떤 코드를 디버깅해야 할까 상상해보자.
  - 그러니 스프링 빈 객체를 사용할 때 기본값이 싱글톤 객체임을 잊지 말고 주의해서 'Stateless'하게 사용하도록 하자.


