# 서블릿 필터(Servlet Filter)가 스프링 빈으로 등록 가능한 이유
## 1. 필터(Filter)는 스프링 빈으로 등록이 불가능했다!
- 몇몇 포스팅과 예전 책들을 보면 필터(Filter)는 서블릿 기술이라서 Spring의 빈으로 등록할 수 없다는 내용이 나온다.
- 또한 필터는 J2EE 표준 스펙이지만 인터셉터는 스프링 프레임워크가 제공하는 기술이므로 필터와 달리 인터셉터는 스프링 빈으로 등록 가능하다.
- 우선 위의 내용의 진위 여부를 살펴보기 전에 필터와 스프링 간의 관계부터 살펴보도록 하자.
- 서블릿 스펙의 기술인 필터는 스프링 범위 밖인 서블릿 범위에서 관리되는데, 이를 그림으로 표현하면 다음과 같다.

<p align="center"><img src="../images/spring_interceptor.png" width="800"></p>

- 상식적으로 스프링 컨테이너보다 큰 범위인 서블릿 컨테이너의 필터가 스프링 컨테이너에 의해 관리된다는 것은 이상하다.
  - 하지만 테스트를 해보면 필터 역시 스프링 빈으로 등록이 가능하며, 빈을 주입받을 수도 있는 것을 확인할 수 있다.
- 이러한 잘못된 설명이 나오는 이유는 과거에 실제로 필터(Filter)가 스프링 컨테이너에 의해 관리되지 않았기 때문이다.
  - 그래서 필터를 빈으로 등록할 수도 없고, 다른 빈을 주입받을 수도 없었다.
- 하지만 __DelegatingFilterProxy__ 가 등장하면서 이제 이러한 설명은 더이상 유효하지 않은데, 해당 내용에 대해 자세히 살펴보도록 하자.

## 2. DelegatingFilterProxy와 SpringBoot의 등장
### DelegatingFilterProxy의 등장 이전
- 필터(Filter)는 서블릿이 제공하는 기술이므로 서블릿 컨테이너에 의해 생성되며 서블릿 컨테이너에 등록이 된다.
  - 그렇기 때문에 스프링의 빈으로 등록도 불가능했으며, 빈을 주입받을 수 없었다. 
- 하지만 필터에서도 DI와 같은 스프링 기술을 필요로 하는 상황이 발생하면서 필터도 스프링 빈을 주입받을 수 있도록 대안을 마련했는데, 그것이 바로 DelegatingFilterProxy이다.

### DelegatingFilterProxy의 등장 이후 
- Spring 1.2부터 DelegatingFilterProxy가 나오면서 서블릿 필터(Servlet Filter) 역시 스프링에서 관리가 가능해졌다.
- DelegatingFilterProxy는 서블릿 컨테이너에서 관리되는 프록시용 필터로써 우리가 만든 필터를 가지고 있다.
- 우리가 만든 필터는 스프링 컨테이너의 빈으로 등록되는데, 요청이 오면 DelegatingFilterProxy가 요청을 받아서 우리가 만든 필터(스프링 빈)에게 요청을 위임한다.

#### 동작 원리
1. Filter 구현체가 스프링 빈으로 등록됨
2. ServletContext가 Filter 구현체를 갖는 DelegatingFilterProxy를 생성함
3. ServletContext가 DelegatingFilterProxy를 서블릿 컨테이너에 필터로 등록함
4. 요청이 오면 DelegatingFilterProxy가 필터 구현체에게 요청을 위임하여 필터 처리를 진행함
 
#### 필터를 등록하는 코드
- 예를 들어 다음과 같이 우리가 만든 필터가 있다고 하자.

```java
@Component
public class MyFilter implements Filter {

   @Override
   public void doFilter(final ServletRequest request, final ServletResponse response, final FilterChain chain) throws ServletException, IOException {
      chain.doFilter(request, response);
   }

   @Override
   public void init(final FilterConfig filterConfig) throws ServletException {
      Filter.super.init(filterConfig);
   }

   @Override
   public void destroy() {
      Filter.super.destroy();
   }

}
```

- 위와 같은 필터를 등록해주려면 SpringBoot가 아닌 Spring에서는 다음과 같이 서블릿 컨텍스트에 addFilter로 추가해주어야 했다.

```java
public class MyWebApplicationInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

   public void onStartup(ServletContext servletContext) throws ServletException {
      super.onStartup(servletContext);
      servletContext.addFilter("myFilter", DelegatingFilterProxy.class);
   }
}
```

- 위의 코드는 "myFilter"라는 빈을 찾아서 DelegatingFilterProxy에 넣어주고, 서블릿 컨테이너에 myFilter 대신 DelegatingFilterProxy를 등록하여 DelegatingFilterProxy를 통해 myFilter에 요청을 위임하라는 것이다.
- DelegatingFilterProxy가 add 되는 곳은 ServletContext이고, 우리가 만든 필터는 스프링 컨테이너에 빈으로 먼저 등록된 후에 DelegatingFilterProxy에 감싸져 서블릿 컨테이너로 등록이 되는 것이다.
- 이러한 이유로 우리가 개발한 Filter도 스프링 빈으로 등록되며, 스프링 컨테이너에서 관리되기 때문에 빈 등록뿐만 아니라 빈의 주입까지 가능한 것이다.

 
### SpringBoot의 등장 이후 
- 위의 DelegatingFilterProxy를 등록하는 과정은 Spring이기 때문에 필요한 것이고, SpringBoot라면 DelegatingFilterProxy 조차 필요가 없다.
  - 왜냐하면 SpringBoot가 내장 웹 서버를 지원하면서 톰캣과 같은 서블릿 컨테이너까지 SpringBoot가 제어 가능하기 때문이다.
- 그래서 SpringBoot에서는 ServletContext에 필터(Filter) 빈을 DelegatingFilterProxy로 감싸서 등록하지 않아도 된다.
- SpringBoot가 서블릿 필터의 구현체 빈을 찾으면 DelegatingFilterProxy 없이 바로 필터 체인(Filter Chain)에 필터를 등록해주기 때문이다.

- 실제로 필터에서 doFilter 호출 전에 런타임 예외를 발생시켜 로그를 출력해보면 이러한 내용을 확인해볼 수 있다.
- 먼저 Spring 프레임워크에 DelegatingFilterProxy를 이용해 필터를 등록한 경우에는 다음과 같이 로그가 남는다.

```java
java.lang.RuntimeException
    at com.mangkyu.MyFilter.doFilter(MyFilter.java:17)
    at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:358)
    at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:271)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
```

- 위의 로그를 보면 DelegatingFilterProxy가 doFilter로 요청을 받고, 이를 실제 구현체인 MyFilter로 위임(invokeDelegate)하는 것을 볼 수 있다.
- 반면에 SpringBoot에서 동일한 상황을 만들어 로그를 확인하면 DelegatingFilterProxy 관련 내용이 없음을 확인할 수 있다.

```java
at com.mangkyu.MyFilter.doFilter(MyFilter.java:17) [main/:na]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) [tomcat-embed-core-9.0.56.jar:9.0.56]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) [tomcat-embed-core-9.0.56.jar:9.0.56]
    at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) [spring-web-5.3.15.jar:5.3.15]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:117) [spring-web-5.3.15.jar:5.3.15]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) [tomcat-embed-core-9.0.56.jar:9.0.56]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) [tomcat-embed-core-9.0.56.jar:9.0.56]
```

## 정리
- 등장 배경 : 필터에서 다른 스프링 빈의 주입이 필요해짐
- DelegatingFilterProxy의 등장 이전 : 스프링 빈으로 등록 및 다른 빈 주입이 불가능했음
- DelegatingFilterProxy의 등장 이후 : DelegatingFilterProxy를 통해 스프링 빈으로 등록 및 다른 빈 주입이 가능해짐
- SpringBoot의 등장 이후 : 웹 서버를 직접 관리하면서 DelegatingFilterProxy조차 필요없게 됨
 
