# @Async의 사용
## @Async in Spring boot
- 스프링 부트에서 개발자에게 비동기 처리를 손쉽게 할 수 있도록 다양한 방법을 제공하고 있다.
- 대세는 Reactive stack, CompletableFuture를 쓰지만 역시 가장 쉬운 방법으로는 @Async annotation을 적용하는 것이다.

## @Async 사용법
```
1) @EnableAsync로 @Async를 쓰겠다고 스프링에게 알린다.
2) 비동기로 수행되었으면 하는 메서드 위에 @Async를 적용한다.
```
- 스프링 가이드에도 마찬가지로 설명한다.
- 만약에 별도로 @Async에 대한 설정이 없으면 새로운 비동기 작업을 스레드 풀에서 처리하는게 아니라 새로운 스레드를 매번 생성해서 작업을 수행시키는 것이 디폴트 설정이다.
  - 그래서 스레드 풀을 빈으로 등록시켜 자동으로 해당 스레드 풀로 작업을 넘기도록 설정한다.

```java
@Configuration
@EnableAsync
public class AsyncThreadConfiguration {
	@Bean
	public Executor asyncThreadTaskExecutor() {
		ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
		threadPoolTaskExecutor.setCorePoolSize(8);
		threadPoolTaskExecutor.setMaxPoolSize(8);
		threadPoolTaskExecutor.setThreadNamePrefix("jeong-pro-pool");
		return threadPoolTaskExecutor;
	}
}
```

- 위와 같이 configuration 클래스를 하나 만들고 Bean을 등록하면 자동으로 내가 만든 스레드 풀에 작업이 할당될 것이다.
- springboot 2.0 이상이라면 auto configuration으로 Executor를 등록해주기 때문에 아래와 같이 설정 파일에서 설정해도 똑같이 스레드 풀이 생성 및 적용된다. (application.yml)

```
spring:
  task:
    execution:
      pool:
        core-size: 8
        max-size: 8
```
- 위와 같이 설정 단계를 거쳤다면 아래 코드처럼 @Async를 통해 호출할 수 있다.

```java
@RestController
public class TestController {
	@Autowired
	private TestService testService;
	
	@GetMapping("/test1")
	public void test1() {
		for(int i=0;i<10000;i++) {
			testService.asyncHello(i);
		}
	}
}
```
```java
@Service
public class TestService {
	private static final Logger logger = LoggerFactory.getLogger(TestService.class);
	
	@Async
	public void asyncHello(int i) {
		logger.info("async i = " + i);
	}
}
```
- 아래와 같이 설정하고 테스트를 해볼 수 있다.
<p align="center"><img src="../images/async_result_1.png" width="700"></p>

- 보이는 대로 브라우저에서 "localhost:8080/test1"로 연결해보면 로그를 10,000번 찍는 과정이 비동기로 호출되는 것을 확인할 수 있다.
  - threadName을 보았을 때 jeong-pro-pool로 적은 prefix가 잘 적용되어 해당 풀을 사용하고 있음을 알 수 있고,
  - 0부터 9999까지 순서대로 찍히는 게 아니라 비동기로 수행되기 때문에 순서가 뒤죽박죽인 것을 확인할 수 있다.

## @Async 사용 시 주의점
```
1) private 메서드에는 적용이 안 된다. public만 된다.
2) self-invocation(자가 호출)해서는 안된다. → 같은 클래스 내부의 메서드를 호출하는 것은 안 된다.
```

```java
@RestController
public class TestController {
	@Autowired
	private TestService testService;
	
	@GetMapping("/test2")
	public void test2() {
		for(int i=0;i<10000;i++) {
			testService.innerMethodCall(i);
		}
	}
}
```
```java
@Service
public class TestService {
	private static final Logger logger = LoggerFactory.getLogger(TestService.class);
	
	@Async
	public void innerMethod(int i) {
		logger.info("async i = " + i);
	}
	
	public void innerMethodCall(int i) {
		innerMethod(i);
	}
}
```
- 위 코드를 테스트해보면 controller에서 testService.innerMethodCall()를 동기로 호출하지만,
- 내부에서 하는 작업이 비동기로 @Async가 걸린 innerMethod를 호출하므로 결국에는 비동기로 로그가 찍힐 것을 예상할 수 있다.

<p align="center"><img src="../images/async_result_2.png" width="700"></p>

- 하지만 틀렸다. 위의 결과처럼 하나의 스레드로 동기 처리됨을 볼 수 있다.

### 이유
- 결론부터 말하면 AOP가 적용되어 Spring context에 등록되어 있는 빈 객체의 메서드가 호출되었을 때 스프링이 끼어들 수 있고,
- @Async가 적용되어 있다면 스프링이 메서드를 가로채서 다른 스레드(풀)에서 실행시켜주는 메커니즘이라는 것이다.

<p align="center"><img src="../images/async_aop.jpg" width="700"></p>

- 이로써 위의 제약조건들이 설명된다.
  - public 이어야 가로챈 스프링의 다른 클래스에서 호출이 가능할 것이고,
  - self-invocation 이 불가능한 이유도 spring context에 등록된 빈의 메서드 호출이어야 프록시를 적용받을 수 있 내부 메서드 호출은 프록시 영향을 받지 않기 때문이다.

### 해결(?)
```java
@Service
public class AsyncService {
	@Async
	public void run(Runnable runnable) {
		runnable.run();
	}
}
```
- 위와 같이 AsyncService를 하나 두고 해당 서비스는 유틸 클래스처럼 전역에서 사용하도록 두는 것이다.
  - @Async 메서드 run을 통해 들어오는 Runnable을 그냥 실행만 해주는 메서드다.

```java
@Service
public class TestService {
	private static final Logger logger = LoggerFactory.getLogger(TestService.class);
	@Autowired
	private AsyncService asyncService;
	
	public void innerMethod(int i) {
		logger.info("async i = " + i);
	}
	
	public void innerMethodCall(int i) {
		asyncService.run(()->innerMethod(i));
		
	}
}
```
- 그 다음에 비동기 메서드 호출이 필요할 때 해당 서비스로 메서드를 호출해버리는 것이다.
