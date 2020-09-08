---
layout: post
title: Spring security oauth2
tags: [spring, security, OAuth2]
---    


이번에 사내 프로젝트 중 Admin의 인증 부분을 개발하게 되었고 개발을 진행하던 중 학습한 내용과 평소에 궁금했던 내용들을 정리하고자 글을 작성하게 되었습니다.

이 글에서 주로 다룰 내용은 설정보다는 어떤 식으로 Spring security의 Filter들이 동작하는가와 동작 과정 중 예외처리는 어떤 식으로 이뤄지는지에 대한 내용입니다.

Spring security는 표준 서블릿 기반 Filter를 기반으로 합니다. Spring boot를 사용 시 EnableWebSecurity 어노테이션을 사용할 경우 자동으로 Filter가 등록되어 Request에 대해 Spring security 필터가 동작되게 됩니다.

현재 Spring security 세션 기반이 아닌 OAuth2 Token 방식으로 개발을 진행하고 있으므로 설명은 OAuth2 설정 기준으로 진행될 것입니다.

흔히 Spring security에 대해 설명할 때 아래와 같은 Filter 리스트에 대해서 보여주곤 하는데, 모든 Request와 Response는 조건에 따라 이런 일련의 Filter들을 거치게 됩니다.

<img src="/assets/img/spring-security-1-14.png" width="90%">

직접 Intellij 디버그 모드를 통해 확인해보도록 하겠습니다.

Spring security OAuth2를 사용하면서 한 프로젝트로 Authorization 서버와 Resource 서버를 모두 구현할 경우 @EnableAuthorizationServer, @EnableResourceServer 각 어노테이션을 붙여서 설정 파일을 작성하게 될 것입니다.

<img src="/assets/img/spring-security-1-13.png" width="90%">

위와 같이 설정을 함으로써 기본 설정을 Override 하여 변경할 수 있습니다. 이런 설정들은 WebSecurityConfigurer.setFilterChainProxySecurityConfigurer(…) 에서 설정들을 정렬하고 중복을 제거하여 추후 사용을 위해 준비하게 됩니다.

이렇게 준비된 설정들은 springSecurityFilterChain라는 이름의 Bean으로 스프링 컨테이너에 올라가게 됩니다. 기존 web.xml 방식으로 XML에 Filter 설정을 하였다면 Spring boot에서는 기본적으로 security를 사용하면 설정이 됩니다.

<img src="/assets/img/spring-security-1-12.png" width="90%">

이런 과정은 애플리케이션 기동 중에 일하는 일로 각 요청에 대한 Filter 처리는 Client로부터의 Request 가 있을 경우에 동작하게 됩니다.

현재 Spring boot에 포함된 내장 톰캣을 사용하고 있으며 그에 따라 모든 Request는 StandardHostValve.invoke(…) 로부터 시작됩니다.

<img src="/assets/img/spring-security-1-11.png" width="90%">

로직을 타고 들어가면 결국 마주하는 곳은 StandardWapperValve.invoke(…) 이고 이 안에서 기동과 동시에 설정되었던 Filter 정보를 가지고 오게 되어있습니다.

ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

위 로직을 통해 Request에 따른 Filter들을 가져와 적용하는 방식으로 동작하는데, 조금 더 로직을 상세히 들어가 보면 FilterChain을 생성하는 과정을 확인해볼 수 있습니다.

디버깅을 통해 확인해보면 Servlet Request의 속성들 중 “org.apache.catalina.core.DISPATCHER_TYPE” 라는 속성을 확인할 수 있는데요. 이 DispatcherType과 URL을 통해 어떤 속성을 사용할 것인가 정하게 됩니다.

<img src="/assets/img/spring-security-1-10.png" width="90%">

위 이미지는 실제 Intellij 디버깅 중 context의 filterMaps 값을 들여다본 건데요. Spring security 설정에 의해서 추가된 위에서 언급한 springSecurityFilterChain이라는 필터도 보입니다. 
(정확히는 FilterChain인데 이 FilterChain도 Filter 인터페이스의 구현체입니다.)

보이는 거와 같이 각 Filter는 urlPattern 속성을 갖고 있습니다. 이 urlPattern과 위에서 언급한 DispatcherType으로 어떤 Filter를 적용할 것인가를 정하게 됩니다.

<img src="/assets/img/spring-security-1-9.png" width="90%">

위에 이미지의 드래그 되어있는 matchDispatcher를 호출하는 부분을 보면 Loop를 돌며 각 FilterMap과 현재 Request의 DispatcherType을 넘기고 있습니다.

이 메서드 내부로 들어가면 FilterMap의 getDispatcherMapping()을 통해 가져온 값과 FilterMap의 상수 필드를 AND 연산자로 비교하여 적용 여부를 판단하여 반환하게 되어있습니다.

위 로직을 FilterMaps의 요소만큼 반복문을 통해 확인하며 Request에 대한 ApplicationFilterChain 값을 구하여 FilterChain을 실행하게 됩니다.

<img src="/assets/img/spring-security-1-8.png" width="90%">

이 FilterChain 내부에서 각 Filter를 처리하기 위해서 위 이미지 상 드래그된 부분과 같이 this를 통해 자기 참조를 넘겨 FilterChain 내부의 Filter 배열을 순회하게 됩니다.

글을 적다 보니 너무 Filter 관련 얘기만 하고 있는 거 같은데요. 이 과정을 이해해야 Spring security의 Filter 동작 과정을 이해할 수 있기에 주절주절 적었는데 이제 다시 Spring security의 Filter에 대해서 볼 차례입니다.

springSecurityFilterChain라는 이름으로 등록된 Filter는 DelegatingFilterProxy라는 클래스로 생성된 객체입니다. 이 객체 역시 Filter 인터페이스의 구현체여서 doFilter(…) 메서드를 갖고 있는데요. 

이 springSecurityFilterChain의 경우 앞에서 지나친 다른 Filter와는 조금 다른 점이 있습니다.

springSecurityFilterChain는 Filter이자 FilterChain입니다. 즉 내부에 추가로 Filter를 들고 있단 거죠.

<img src="/assets/img/spring-security-1-7.png" width="90%">

정확히 따지면 springSecurityFilterChain은 3개의 FilterChain을 갖고 있고 또 그 각 FilterChain은 다수의 Filter를 갖고 있습니다.

순서대로 보면 Authorization 서버 설정, Resource 서버 설정, Security 기본 설정에 따라 각각 등록된 FilterChain 들입니다.

이 FilterChain 들 역시 서블릿 Filter 들 중 Request에 따라 어떤 Filter를 적용할 건지 결정한 거와 비슷하게 Request에 따라 다시 한번 추려지게 됩니다. 

여기서는 위 서블릿 Filter를 추리는 부분보다는 단순하게 각 FilterChain (SecurityFilterChain) 이 가지고 있는 RequestMatcher에 Matcher 객체가 갖고 있는 AntPattern을 통해 Request의 URL 만을 체크하여 추려집니다.

이 과정은 역시나 springSecurityFilterChain 하위 3개의 FilterChain 만이 아닌 또 그 하위의 다수의 Filter들에게도 적용됩니다. 

이런 과정을 통해서 Request에 대해 적절한 Filter가 적용되는 것이지요.

추가로 이런 Filter 동작 중 Error가 발생했을 경우 어떤 식으로 동작하는지 살펴보고 추가로 Filter가 아닌 인증 로직 중 Controller에서 Error가 발생 시 어떻게 동작하는지 살펴보겠습니다.

이런 과정을 디버깅하기 위해서 일부로 Basic 인증에 사용될 Client ID 정보를 서버에 등록되지 않은 걸로 바꿔서 진행할 것이며 기본 Authorization 서버의 경우 기본 인증 URL은 /oauth/token을 통해 하게 되어있지만 현재 설정을 통해 /api/v1/login으로 변경하였습니다.

변경 파일은 oAuth2AuthorizationConfig입니다.

<img src="/assets/img/spring-security-1-6.png" width="90%">

위 설정을 적용함으로 인해 기본 인증 URL은 /api/v1/login으로 동작합니다.

HTTP Basic 인증을 처리하는 Filter는 BasicAuthenticationFilter입니다. 이 Filter는 Header의 Authorization 속성의 값을 Base64로 디코딩하여 Client Id, Cliend Secret을 정보를 가져와 Basic 인증을 처리합니다.

참고로 현재 OAuth2 인증 방식은 password grant 방식을 사용하고 있기에 header 정보에 Client Id, Secret을 모두 포함한 것입니다.

Spring security에서 인증은 ProviderManager를 통해 처리합니다. ProvierManager는 여러 Provicer를 가지고 있고 이 중 적절한(적용 가능한) Provider를 통해 인증을 처리하는 방식이지요.

아래는 ProviderManager.authenticate(…)의 일부입니다.

<img src="/assets/img/spring-security-1-5.png" width="90%">

for문 첫 번째 라인을 보면 supports라는 메서드를 통해 해당 provider의 대상인지를 판단하고 인증 과정을 진행하는데 특이한 점은 성공하면 break를 통해 for문은 벗어나게 되지만 실패할 경우 혹은 성공하지 못했을 경우에 계속 for문 돌게 되어있습니다. 실패했을 경우라면 이해되지만 의아한 부분은 Exception이 발생해도 계속 동작하면서 마지막 Exception만 담는 로직입니다.

인증 과정 중 Exception이 발생했을 경우 이 마지막 Exception의 경우 Throw를 통해 메서드 밖으로 던져집니다.

중간에 Exception이 발생하지 않았다면 FilterChain의 Filter들의 로직을 모두 수행 후 Servlet 로직단 (Controller)까지 Request가 도착했어야 하지만 Exception 발생으로 인해서 Chain은 끊기고 StandardWrapperValve을 거쳐 Request 처리의 시작점이었던 StandardHostValve.invoke(…) 로 다시 돌아오게 됩니다.

여기까지만 봤을 때 그럼 “Exception 발생에 따른 처리는 어떻게 해줘?”라고 생각할 수 있습니다만 이 처리는 이제부터 시작됩니다. 이 위치까지 지나오면서 StandardWrapperValve을 거쳐왔는데요. StandardWrapperValve.invoke(…) 에서 Response에는 Error 값이 세팅되어 넘어옵니다.

세팅된 Error 값은 아래와 같이 StandardHostValve.invoke(…) 에서 isErrorReportRequired()의 값을 true로 세팅하여 추가 적인 Error에 대한 처리를 하게 만듭니다.

<img src="/assets/img/spring-security-1-4.png" width="90%">

status(…)를 지나면서 Error에 따른 ErrorPage를 구하고 (따로 Tomcat 설정을 하지 않았다면 /error) custom(…)로 이동하여 request를 forward 처리합니다.

<img src="/assets/img/spring-security-1-3.png" width="90%">

이 Forward로 인해 ApplicationDispatcher에서 다시 한번 ApplicationFilterFactory를 통해 FilterChain을 구합니다. 단, 처음 Request 와는 다르게 URL은 /error로 변경되었으며 request의 DispatcherType은 ERROR가 됩니다.

초반에 등장했던 FilterMaps의 5개의 FilterMap 중 이 조건에 부합하는 건 springSecurityFilterChain 필터뿐이므로 해당 Error Request는 springSecurityFilterChain의 하위 Filter 들을 거쳐 Servlet에 도달하게 되고 Spring boot에서 기본으로 설정된 BasicErrorController에서 Error Request에 대한 처리를 하게 됩니다.

이런 긴 과정을 통해서 Filter에서 발생한 또는 Controller에서 예외처리가 되지 않은 Exception은 처리되는 겁니다.

그럼 이번에는 인증 요청을 받은 /v1/api/login 에서 Error가 발생했을 때는 어떤 식으로 처리될까요?

이 인증 로직은 Filter에서 문제가 없을 경우 TokenEndpoint.postAccessToken(…) 로 진입하게 됩니다.

<img src="/assets/img/spring-security-1-2.png" width="90%">

보통은 이렇게 Controller 단에서 Exception이 발생 시 ControllerAdvice를 Global로 생성하였을 경우 AOP에 의해 ControllerAdvice로 Exception이 넘어가서 Exception을 처리할 수 있는데 TokenEndpoint 에는 위와 같이 @ExceptionHandler로 Exception을 잡아서 처리하고 있습니다.

인증 과정에서 발생한 Exception을 아무리 ControllerAdvice로 Exception을 잡으려 해도 잡을 수 없는 이유입니다.
그럼 이런 ExceptionHandler는 커스터마이징 할 수 없을까요?

물론 Spring은 이 모든 상황에 대해서 처리할 수 있게 설정을 제공합니다.

<img src="/assets/img/spring-security-1-1.png" width="90%">

역시나 OAuth2AuthorizationConfig 의 설정인데 이 설정을 통해 Exception 처리에 대한 커스터마이징을 할 수 있습니다.

Spring security에서 인증에 관한 Exception 처리는 하고자 하는데, @ExceptionHandler로 Exception을 잡을 경우 다시 Throw 하여도 ControllerAdvice는 Exception을 잡을 수 없는 구조기에 위와 같이 커스터마이징을 지원하는게 아닌가 싶습니다.
