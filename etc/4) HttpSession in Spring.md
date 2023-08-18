# HttpSession은 어떻게 만들어지고 어떻게 유지될까
- 코드를 통해 JSESSION의 생성 방법과 기존 세션 확인 방법을 알아보자.

## 서론
- 클라이언트와 서버는 Stateless인 HTTP 통신을 하게 되지만 로그인과 같이 접속을 했던 정보가 저장이 되어야 할 때가 있다.
  - 이 때 인증과 인가가 필요하게 된다.
- 인증은 클라이언트에서 보낸 정보에 담긴 아이디와 패스워드가 서버에서 저장하고 있는 정보와 일치할 때 인증이 발생한다.
- 그 이후 로그인을 한 사용자는 자신의 계정으로 글을 쓰거나 물건을 살 때 무상태로 유지되게 되면 본인이 그 사용자임을 계속해서 인증을 해야한다.
  - 이를 방지하기 위해 서버는 인증이 된 클라이언트에게 쿠키 또는 세션을 발급해주고
  - 클라이언트는 이를 요청 정보에 담아 보냄으로써 지속적인 인증 없이 사용자 인증이 필요한 요청들을 할 수 있게 된다.

## 쿠키
- 쿠키는 서버에서 필요한 정보를 지정하면, 클라이언트 측에서 저장하고 HTTP 요청마다 메세지에 담아 보내게 된다.
- 이로써, 서버에 메모리 부담을 줄일 수 있지만 요청 시 쿠키 내부에 있는 보안 정보들이 그대로 노출될 수 있어서 보안 상의 문제가 있고, 쿠키의 크기가 클 경우 네트워크 부하가 커질 수 있다.
- 그리고 사용자가 보안 상 쿠키 수집을 거절할 경우 사용이 불가능하며 웹 브라우저마다 지원 형태가 달라 호환이 불가능하다.

## 세션
- 세션은 인증된 사용자의 정보를 SESSION ID와 매핑하여 서버에 저장하고 클라이언트에게 식별자와 문자열로 이루어진 SESSION ID를 응답 헤더에 넣어 전송한다.
- 매 응답마다 SESSION ID만 보내기 때문에 네트워크 부하가 커지지 않고, 서버에 저장되므로 클라이언트의 웹 브라우저의 호환성 문제가 해결된다.
  - 그리고 많은 정보를 유지할 수 있게 된다.
- 하지만 서버에 데이터 저장량이 많아지므로 서버 메모리에 부담이 갈 수 있다.

## 쿠키와 세션 정리
- 세션도 결국 쿠키를 사용하기 때문에 역할과 동작 원리는 비슷하다.

### 사용자의 정보 저장 위치
- 위에서 설명한대로 가장 큰 차이점은 사용자의 정보가 저장되는 위치이다.
  - 쿠키는 서버의 자원을 전혀 사용하지 않지만 네트워크에 부담이 생길 수 있고,
  - 세션은 서버의 자원을 사용하지만 네트워크에 부담이 가지 않는다.

### 보안
- 보안 면에서는 세션이 더 우수하지만, 세션은 서버에서 처리하기 때문에 요청 속도는 쿠키가 더 빠르다.

### 만료 시간
- 쿠키도 만료 시간이 있지만 파일로 저장되기 때문에 브라우저를 종료해도 계속해서 정보가 남아 있을 수 있다.
  - 또한 만료 시간을 넉넉하게 잡아두면 쿠키 삭제를 할 때까지 유지될 수도 있다.
- 반면에 세션도 만료 시간을 정할 수 있지만 브라우저가 종료되면 만료 시간에 상관 없이 삭제된다.
  - 예를 들어, 크롬에서 다른 탭을 사용해도 세션은 공유된다.
  - 다른 브라우저를 사용하게 되면 다른 세션을 사용하게 된다.

 
## HttpSession
여기서 알아볼 것은 Spring MVC에서의 HttpSession을 알아보도록 하겠다.

https://www.baeldung.com/spring-mvc-session-attributes

 
Session Attributes in Spring MVC | Baeldung

Explore the different ways to store attributes in a session with Spring MVC.

www.baeldung.com
Session 생성
 

1. @Autowired 사용
@Controller
public class LoginController {
			@Autowired
			private HttpSession session;
 
			@PostMapping("/login")
	    public String login(MemberLoginDto memberLoginDto, Model model){
	       
	        Member loginMember = loginService.login(memberLoginDto.getUserId(), memberLoginDto.getPassword());
	
	        if (loginMember == null) {
	            result.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
	            return "member/loginForm";
	        }
	
	        //로그인 성공 처리
	        //세션이 있으면 있는 세션 반환, 없으면 신규 세션을 생성
	        HttpSession session = request.getSession();
	        //세션에 로그인 회원 정보 보관
	        session.setAttribute("loginMember", loginMember);
	
	        model.addAttribute("member",loginMember);
	        return "/member/memberInfo";
	    }
}
HttpSession을 빈주입을 받는 것만으로는 Session이 생성되지 않는다. 해당 메서드의 setAttribute()나 getAttribute()이 호출되는 시점에 서블릿 컨테이너에서 session을 받을 수 있다.

 

2. 메서드 주입
@Controller
public class LoginController {
			
			@PostMapping("/login")
	    public String login(MemberLoginDto memberLoginDto, HttpSession session , Model model){
	       
	        Member loginMember = loginService.login(memberLoginDto.getUserId(), memberLoginDto.getPassword());
	
	        if (loginMember == null) {
	            result.reject("loginFail", "아이디 또는 비밀번호가 맞지 않습니다.");
	            return "member/loginForm";
	        }
	
	        //로그인 성공 처리
	        //세션이 있으면 있는 세션 반환, 없으면 신규 세션을 생성
	        HttpSession session = request.getSession();
	        //세션에 로그인 회원 정보 보관
	        session.setAttribute("loginMember", loginMember);
	
	        model.addAttribute("member",loginMember);
	        return "/member/memberInfo";
	    }
}
1번과 방식이 비슷하지만 파라미터로 받게된다면, login메서드가 실행되는 시점에 Session을 서블릿 컨테이너로 부터 전달 받게 됩니다.

 

3. @SessionAttribute, @MudelAttribute로 주입
@Controller
@SessionAttributes("loginMember")
public class LoginController {
			
		@GetMapping("/")
    public String loginForm(@ModelAttribute("loginMember")Member member, Model model){
 
        if(member != null){
            model.addAttribute("member",member);
            return "/member/memberInfo";
        }
        model.addAttribute("memberLoginDto",new MemberLoginDto());
        return "/member/loginForm";
    }
}
해당방식은 위의 두방식과 조금은 다른 방식으로 사용 한다. 이미 생성되어있는 세션이 있을떄 Get요청을 하면서 동일한 키(loginMember)를 조회하여 Member 파라미터로 전달 받게 된다.

 

 

 

 

세션유지
Request시 Heder에 SessionId가 포함 되어 전달된다면 서블릿 컨테이너는 세션을 발급하지 않고 해당 SessionId에 해당하는 세션을 전달하게 되고, Sessionid가 포함되지 않는다면 HttpSession을 요구하는 모든 요청에 대해 새로운 Session을 발급한다. Spring에서는 기본 서블릿 컨테이너를 Tomcat으로 사용하고 있기때문에 톰캣기준의 내용이다.

Tomcat의 경우 SessionID를 JSESSIONID라는 키의 쿠키를 생성하여 클라이언트에게 전달 하고 클라이언트는 JSESSION이 담긴 쿠키를 헤더에 포함하여 인증을 유지한다. 실제로 어떤 형식으로 전달하는지 알아보자. 아래는 프로젝트 중 확인한 내용이다.


로그인을 했을때 Response Header로 JSESSIONID를 가진 쿠키를 전달 받는 것을 볼 수 있다


 

 

 

JSESSION
JSESSION은 어떻게 생성될까?

JSSION은 HttpServletRequest의 getSession의 옵션에 따라 자동 생성이 된다. 그렇다면 먼저HttpServletRequest 인터페이스에 있는 getSession을 추적해보자

 

 

 


HttpServletRequest 인터페이스의 getSession 구현체는 아래와 같이 확인 된다. 우리는 이것들 중 ApplicaionHttpRequest를 추적하여본다.

 

 


위는 ApplicaionHttpRequest에 있는 getSession의 구현코드 중 일부이다.

ApplicaionHttpRequest의 부모클래스는HttpServletRequestWrapper 이기 때문에

HttpServletRequestWrapper 의 getSession을 찾아간다.

 

 

 


HttpServletRequestWrapper 에서는 HttpServletRequest 의 getsession을 호출하고 일부 생략하자면

HttpServletRequest에는 Request의 getSession을 호출하게 된다.

 

 

 


Request의 getSession 구현이고 아래는 doGetSession의 일부 코드이다.

 

 

 


세션을 만드는 코드가 있다. 이 세션을 만드는 코드는 Manger 인터페이스를 상속 받은 ManagerBase라는 구현체에 구현 되어 있다.

 

 

 


위와 같이 구현 되어 있는데 여기서 JSESSIONID를 만드는 방법도 볼 수 있었다.

JSESSIONID는 StandardSessionGenerator에서 생성하게되는데 방법은 아래와 같다.

16Byte의 랜덤 값(SHA1PRNGE또는 각 플랫폼의 기본 난수 생성기) 을 16진수의 String으로 변환하여 route로 받느 값이 있는경우 뒤에 “.route”를 추가하게 되고 그렇지 않을 경우 + “.jvmRoute”을 추가하게 된다. 여기서Route나 jvmRoute는 서블릿 컨테이너에 접속한 사용자를 구분하는 값이 되는데 이를 구현한 코드를 아래와 같이 확인할 수 있다.

 

/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      <http://www.apache.org/licenses/LICENSE-2.0>
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.catalina.util;
 
public class StandardSessionIdGenerator extends SessionIdGeneratorBase {
 
    @Override
    public String generateSessionId(String route) {
 
        byte random[] = new byte[16];
        int sessionIdLength = getSessionIdLength();
 
        // Render the result as a String of hexadecimal digits
        // Start with enough space for sessionIdLength and medium route size
        StringBuilder buffer = new StringBuilder(2 * sessionIdLength + 20);
 
        int resultLenBytes = 0;
 
        while (resultLenBytes < sessionIdLength) {
            getRandomBytes(random);
            for (int j = 0;
            j < random.length && resultLenBytes < sessionIdLength;
            j++) {
                byte b1 = (byte) ((random[j] & 0xf0) >> 4);
                byte b2 = (byte) (random[j] & 0x0f);
                if (b1 < 10) {
                    buffer.append((char) ('0' + b1));
                } else {
                    buffer.append((char) ('A' + (b1 - 10)));
                }
                if (b2 < 10) {
                    buffer.append((char) ('0' + b2));
                } else {
                    buffer.append((char) ('A' + (b2 - 10)));
                }
                resultLenBytes++;
            }
        }
 
        if (route != null && route.length() > 0) {
            buffer.append('.').append(route);
        } else {
            String jvmRoute = getJvmRoute();
            if (jvmRoute != null && jvmRoute.length() > 0) {
                buffer.append('.').append(jvmRoute);
            }
        }
 
        return buffer.toString();
    }
}
기존 JESSION은 어떻게 확인할까
위의 과정에서 Request.doGetSession에서 m.findSession부분을 따라가면 아래와 같이 ManagerBase에서는 세션을 Map을 이용해서 관리를 하고 있고, JSESSIONID가 들어오면 그에 맞는 Session을 반환하도록 되어 있다.
