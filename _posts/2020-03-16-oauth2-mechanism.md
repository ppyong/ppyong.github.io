---
layout: post
title: OAuth2
tags: [OAuth2]
---

# OAuth란?

> OAuth 2.0은 다양한 플랫폼 환경에서 권한 부여를 위한 산업 표준 프로토콜입니다.

먼저 OAuth를 구성하는 주요 4가지 객체를 알아보자. 

1. client   
자원 소유자의 보호된 자원에 접근하는 애플리케이션. 웹or앱 등..

2. resource owner   
자원 소유자. 보호된 자원에 접근하는 권한을 제공 

3. resource server   
요청을 수신하고 권한을 검증하여 결과(자원)을 응답 

4. authorization server   
자원 소유자를 권한을 부여    

위 4가지 구성요소를 위주로 클라이언트가 access token을 얻는 과정 4가지 grant type을 설명하고자 한다. 


* Authorization Code

<img src="/assets/img/authorization_code.png" width="90%">

Client를 위한 backend server가 있을 경우 동작하는 방식이다. 

동작 과정은 아래와 같다

1. resource owner는 client를 통해 authorization server에 인증을 받고 권한을 허가한다.   

```html
Method: GET 
https://{인증서버}/oauth/authorize?response_type=code&client_id={client_id}&redirect_uri={redirect_uri}&scope={scope}
```
      
2. authorization server는 권한 허가와 동시에 Authorization Code를 발급한다. code를 담은 {리다이렉트 URL}을 응답한다.   

```html
{리다이렉트 URL}?code={code}
```
   
3. 2번에서 전달 받은 {리다이렉트 URL}을 통해 브라우저는 client backend 서버로 리다이렉트 되며 Authorization Code를 전달한다.    

4. client backend는 {인증서버}로 client backend에서 가지고 있는 client_id, client_secret고 리다이렉트를 통해 전달 받은 grant type, Authrization Code, {리다이렉트 URL}을 보낸다.    

5. {인증서버}로부터 access token을 발급 받는다.   
