# 서블릿과 서블릿 컨테이너

## 서블릿
- 서블릿은 Java를 이용하여 웹 페이지를 동적으로 생성하는 서버 측 프로그램이다.
- 예를 들어, '안녕하세요, xxx님' 과 같은 메시지를 브라우저에 띄울 때 들어오는 유저마다 다른 이름을 출력하는 것을 동적 페이지라고 할 수 있다.

## 서블릿 컨테이너
- 서블릿 컨테이너는 구현되어 있는 Servlet 클래스의 규칙에 맞게 서블릿 객체를 생성, 초기화, 호출 종료하는 생명 주기를 관리한다.
- 톰캣처럼 서블릿을 지원하는 WAS를 서블릿 컨테이너라고 할 수 있다.
- 이 때, 서블릿 객체는 빈과 마찬가지로 싱글톤으로 관리된다. 그 외에도 서블릿 컨테이너는 제공하는 기능이 많다.

### 1) 통신 지원
- 서블릿과 웹 서버가 통신할 수 있는 손쉬운 방법을 제공한다.
- 만약 해당 유저의 이름 값을 FORM을 통해 입력받는다고 가정해보자. 그러면 아래와 같은 수많은 작업이 필요하다.

<p align="center"><img src="../images/servlet_web_server_connection.png" width="700"></p>
 
- FORM 인증을 하면 HTTP 메시지가 전송되는데 그것을 읽어들이기 위해 여러 가지 과정을 거쳐야 하고 응답하기 위해서도 또 번거로운 과정들을 거쳐야 한다.
  - 사실 중요하게 해야할 일은 비즈니스 로직인데도 말이다.
- 서블릿은 개발자가 비즈니스 로직에 집중할 수 있도록 해당 과정을 모두 자동으로 해준다.
  - 우리는 단순히 HTTP 요청 메시지로 생성된 request를 읽어서 비즈니스 로직을 수행하고 response를 반환하면 된다.

<p align="center"><img src="../images/request_response_process.png" width="800"></p>

- 좀 더 구체적으로 해당 흐름을 설명하자면,
- 먼저, 특정 URL이 호출되면 서블릿 코드가 실행되면서 HTTP 요청 메시지를 기반으로 HttpServletRequest(request)를 생성한다.
- 그리고 개발자는 여러가지 비즈니스 로직을 거친 뒤 서블릿이 제공하는 HttpServletResponse를 활용하여 HTTP 응답 (response)를 생성할 수 있다.

### 2) 멀티 스레딩 관리
- 서블릿 컨테이너는 요청이 올 때마다 새로운 자바 쓰레드를 하나 생성하는데, HTTP 서비스 메소드를 실행하고 나면 쓰레드는 자동으로 종료된다.
- 원래는 쓰레드를 관리해야 하지만 서버가 다중 쓰레드를 생성 및 운영해주니 쓰레드의 안정성에 대해서 걱정하지 않아도 된다.

### 3) 선언적인 보안 관리
- 서블릿 컨테이너는 보안 관련된 기능을 지원하므로 서블릿 코드 안에 보안 관련된 메소드를 구현하지 않아도 된다.

### 4) JSP 지원

## 서블릿을 구현한 예제 코드
```java
@WebServlet(name = "requestBodyStringServlet", urlPatterns = "/request-body-string")
public class RequestBodyStringServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        // 비즈니스 로직 작성
    }
}
```

- 위와 같이 서블릿의 이름을 설정하고 URI를 지정해 줄 수 있다.
- 그 이후 HttpServlet에 상속을 받아서 service() 메소드를 재정의함으로써 개발자가 비로소 비즈니스 로직을 작성할 수 있다.

 
<p align="center"><img src="../images/servlet_hierarchy.png" width="600"></p>

- 참고로 JDK에서 정의된 서블릿의 계층 구조는 위와 같다.

 
## 정리
- 서블릿은 동적 웹페이지를 위해 사용되는 프로그램이고, 서블릿 컨테이너는 서블릿을 관리하면서 여러 부가적인 기능을 제공한다.
