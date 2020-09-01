---
layout: post
title: Spring  ControllerAdvice
tags: [spring, error]
---


요새 추세는 예전과 같이 WAS에 FRON-END와 BACK-END 모두 올리지 않고 FRONT-END는 보통 Node서버에 따로 올리는 방식을 사용하곤 하지만 아직도 WAS에 FRONT까지 모두 올리는 곳도 있습니다.

이번 포스팅에서는 이렇게 WAS에 FRONT-END와 BACK-END를 모두 구현할 경우 요청에 따른 Exception를 어떻게 처리할 것 인가에 대한 내용입니다.

흔히 Spring에서 Exception을 처리하기 위해 ControllerAdvice를 활용하곤 합니다. 

이 ControllerAdvice를 통해 Exception을 잡은 후 요청에 따라 Error Json을 반환하든 Error Page를 반환하든 처리하고 싶을 때가 있는데요.

이전 토이 프로젝트 진행 중 위와 같은 상황이 있었고 그 당시 처리했던 내용을 공유하고자 합니다.

아래는 간략하게 작성해본 Controller 인데 일반적으로 아래와 같이 @Controller 어노테이션을 통해 사용자의 요청에 대해 응답을 해줍니다. 

이 경우 기본적으로 반환되는 문자열은 이동 할 HTML Page 명이 됩니다.    

```java
@Controller
@RequestMapping
public class FrontController {
    @GetMapping("/index")
    public String idx(){
        throw new RuntimeException();
    }
}
```

이때 발생한 Exception은 AOP로 동작하는 ControllerAdvice가 Exception을 잡아서 적절한 처리를 해주게 됩니다.

```java
@ControllerAdvice
public class GlobalControllerAdvice {
    @ExceptionHandler(Exception.class)
    public String handleException(Exception e){
        return "error";
    }
}
```   

따라서 위 예시의 handlerException 메서드는 Error Page를 반환하게 됩니다. 하지만 항상 기대하는 응답이 Page가 아닌 Json 데이터일 경우가 있습니다.

모든 Exception을 위의 ControllerAdivce로 처리한다면 Rest요청에 대한 처리 중 Exception이 발생했을 때도 Error Json이 아닌 Error Page를 반환하게 될 것입니다.

Ajax로 Rest 요청을 하였을 때 응답 값이 성공이든 실패든 일관되게 Json으로 오지 않는다면 화면을 개발할 때 여러 이슈가 생기게 됩니다.

먼저 제 경우 보통 Rest 요청과 Page 요청을 구분하기 위해 각각 따로 Controller를 작성하는 편입니다. 

따라서 아래와 같이 Rest 요청에 대해 처리하기 위한 Controller를 추가하였습니다.

```java
@RestController
@RequestMapping
public class BackController {
    @GetMapping("/index-rest")
    public ResponseEntity idx(){
        throw new RuntimeException();
    }
}
```

이 경우도 역시 Error Json이 아닌 Error Page가 반환될 것 입니다. 아래와 같이 말이죠.

<img src="/assets/img/controlleradvice-1.png" width="90%">

그렇다면 어떻게 이 문제를 해결해야 할까요?

제 경우 위에서 말한대로 Rest 요청과 Page 요청을 구분하여 Controller를 작성하기에 이 점을 이용해서 문제를 해결해보았습니다.

Controller도 요청에 따라 구분한거와 같이 ControllerAdvice도 구분하였습니다.

```java
@Order(0)
@RestControllerAdvice(annotations = RestController.class)
public class GlobalRestControllerAdvice {
    @ExceptionHandler(Exception.class)
    public Map<String, Object> handleException(Exception e){
        Map<String, Object> errorMap = new HashMap<>();
        errorMap.put("code", "E001");
        errorMap.put("message", "에러 발생했어요");
        return errorMap;
    }
}

@Order(1)
@ControllerAdvice(annotations = Controller.class)
public class GlobalControllerAdvice {
    @ExceptionHandler(Exception.class)
    public String handleException(Exception e){
        return "error";
    }
}
```

GlobalRestControllerAdvice는 @RestController 어노테이션이 붙은 Controller에서 발생한 Exception을 처리하도록하고 GlobalControllerAdvice는 @Controller 어노테이션이 붙은 Controller에서 발생한 Exception을 처리하도록 하였습니다.  

위와 같이 @RestControllerAdvice와 @ControllerAdvice는 타겟 어노테이션을 설정할 수 있게되어있습니다. 

하지만 타겟 어노테이션만 설정하게 된다면 GlobalControllerAdvice에서 @Controller 어노테이션에서 발생한 Exception만 잡게 설정을 하여도 특정 조건에 의해 RestController에서 발생한 Exception도 잡는 경우가 생깁니다.   

이 현상이 발생하는 이유는 @RestController 어노테이션의 내용을 확인해보면 알 수 있는데요. 

@RestController 어노테이션은 @Controller 어노테이션에 약간의 기능을 추가한 어노테이션이기 때문입니다.

위에서 말한 이 특정 조건은 무엇일까요? 이 조건은 class명 입니다.

제 경우 @ControllerAdvice는 GlobalControllerAdvice라는 이름으로 class를 생성하였고 @RestControllerAdvice는 GlobalRestControllerAdvice라는 이름으로 class를 생성하였습니다.

이 경우 class명의 우선순위에서 GlobalControllerAdvice가 GlobalRestControllerAdvice보다 높은 우선순위를 갖게 됩니다. 

그에 따라 RestController에서 발생한 Exception도 GlobalControllerAdvice에서 모두 잡아 처리하여 Error Page로 응답을 주게 된 것이죠.

이 문제를 해결하기 위해서는 GlobalRestControllerAdvice 의 class명을 GlobalControllerAdvice보다 우선순위가 높게 변경해야합니다.

근데 매번 이렇게 우선순위를 생성해서 class명을 생성하게 되면 원하는 class명을 사용할 수도 없을 뿐더러 누군가 실수로 class명을 바꿨을 경우 또 다시 원하지 않는 Error 응답을 받을 수 있습니다.

그래서 사용한 다른 해결책으로는 위 소스에서 사용한 @Order 어노테이션 있습니다.

class명이 아닌 명시적으로 @Order 어노테이션을 통해서 ControllerAdvice간의 순서를 정할 수 있습니다.

따라서 RestController에서 발생한 Exception을 잡기위해 생성한 GlobalRestControllerAdvice에서는 Controller에서 발생한 Exception은 잡지 않으므로 @Order 어노테이션을 통해서 우선순위를 GlobalControllerAdvice보다 높게 변경하였습니다.   

위 설정을 통해 아래와 같이 처리 중 Exception이 발생하였을 때 의도한대로 Rest 요청에 대해서는 Error Json으로 Page 요청에 대해서는 Error Page로 응답을 받을 수 있게 되었습니다.

<img src="/assets/img/controlleradvice-2.png" width="90%">
