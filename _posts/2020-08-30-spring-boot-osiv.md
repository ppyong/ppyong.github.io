---
layout: post
title:  spring boot OSIV
tags: [spring, jpa, osiv]
---    

JPA를 학습하다가 무심코 넘긴 내용인데 면접 질문으로 인해서 모르고 있다는 사실을 깨닫고 정리하기 위해 글을 작성합니다. 

먼저, OSIV는 Open Session In View 의 약자로 뷰까지 JPA의 영속성 컨텍스트를 유지하는 걸 뜻 합니다.

굳이 왜 뷰까지 영속성 컨텍스트를 유지하지? 라고 제일 먼저 궁금할 수 밖에 없는데요. 

요새는 보통 Spring boot로 API만을 개발하고 화면은 Node 같은 별도의 프론트를 위한 서버로 구성하는게 일반적이지만 Spring boot 서버에서 뷰를 생성할 경우 

Controller에서 반환된 Entity를 뷰 생성 시 사용할 때 Lazy로 값을 가져오는 필드에 대해서 정상적으로 가져오지 못하는 되어 이 부분을 개선하고자 추가된 기능(개념)으로 보입니다. 


왜 뷰에서는 영속성 컨텍스트에 접근해서 값을 가져오지 못할까요? 

스프링은 기본적으로 "트랜잭션 범위"와 "영속성 컨텍스트 범위"가 같습니다. @Transactional 어노테이션이 붙은 Method는 Spring 의 AOP에 의해 진입과 동시에 트랜잭션이 시작되고 

해당 Method를 벗어나면 트랜잭션은 종료됩니다. 그와 동시에 영속성 컨텍스트도 종료되고 영속성 컨텍스트에서 영속성 상태(Persistence)였던 객체들은 준영속 상태(Detached)로 변하게 됩니다. 


트랜잭션의 범위 내에서 영속성 상태였던 객체는 트랜젝션 종료와 함께 변경사항을 감지하고 데이터베이스와 동기화를 하게 되는데, 위와 같이 트랜젝션 종료와 함께 준영속 상태가 된 객체에 대해서는 

이런 데이터베이스와의 연결이 끊긴다고 생각하시면 될 것 같습니다. 


이에 대한 결과로 뷰에서 Lazy로 설정된 값을 가져오려 할 때 정상적으로 가져오지 못하게 되는 문제가 생기게 된 것이죠. 그에 대한 결과로 아래와 같은 Exception을 만나게 됩니다.


```java
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role: com.kingbbode.model.Team.members, could not initialize proxy - no Session
```

이런 문제를 해결하기 위해 OSIV가 등장하게 되었습니다. 

처음 나온 개념은 영속성 컨텍스트를 포함하는 Hibernate의 Session 객체를 서블릿 필터부터 열면서 그와 동시에 트랜잭션을 시작하고 뷰 랜더링 작업이 모두 완료 된 후에 트랜잭션을 종료하고 Session을 

종료하도록 하기도 하였는데, 이에 따른 문제로 뷰 랜더링 시에 의도치 않게 객체 변경과 동시에 데이터베이스에 값이 동기화 되기도 하고 너무 오랜 시간동안 데이터베이스 커넥션을 유지하는 등의 문제를 갖게 되었습니다. 

(Flush Mode는 FlushMode.AUTO 로 동작)


다음으로 위 문제를 조금 더 우아한 방식으로 해결한 Spring boot 의 OSIV가 나오게 되었습니다. 

그럼 Spring boot의 OSIV는 어떤식으로 이런 문제들을 우아하게 처리할 수 있는걸까요? 


이전 OSIV의 경우에는 서블릿 필터부터 Session을 열고 트랜잭션을 시작한다고 요청을 완료 후에 트랜잭션을 닫고 Session을 닫는다 하였는데, Spring boot에서는 이 두개를 분리하여 동작하게 하였습니다. 

Spring boot에서는 Request 당 Session을 열고 @Transactional 어노테이션 범위에서만 트랜잭션을 동작하게 하였으며 뷰 랜더링까지 Session을 유지하도록 하였습니다. 


그에 따른 결과로 뷰 랜더링에서도 Lazy로 설정된 값을 읽어올 수 있으며 변경 사항은 반영하지 않게 되었습니다. 

이는 기본 Flush Mode가 Manual로 되어있어서기도 하고 트랜잭션은 뷰 랜더링 시점에는 이미 종료되었기에 때문이기도 합니다. 

(Flush Mode는 FlushMode.MANUAL 로 동작)


이런 기능은 기본적으로 Spring boot 설정으로 활성화가 되어 있으며 아래 설정을 통해 동작하고 있습니다. 기본으로 동작하도록 설정되어있으니 필요 없다면 설정을 통해 종료해야 합니다. 

```profile
spring.jpa.open-in-view=true 
```

**개인적인 견해로는 JPA에서의 Entity는 도메인 계층에 속하기에 뷰랜더링 시점까지 가져오기 보다는 UI 계층인 뷰에서 사용하기 위한 DTO 객체로 변환하여 반환하는게 좋다고 생각합니다.**

위 계층과는 별개로 영속성 컨텍스트를 뷰까지 가져온다는 의미가 데이터베이스와의 연결을 오랜 시간 유지한다는게 됨으로 해당 기능은 꼭 필요한 경우가 아니고서는 사용하지 않는게 좋지 않을까요?



*참고자료*
- https://kingbbode.tistory.com/27
- https://www.baeldung.com/spring-open-session-in-view  
- https://joont92.github.io/jpa/웹-어플리케이션과-영속성-관리/  
- https://www.inflearn.com/course/스프링부트-JPA-API개발-성능최적화  
     

