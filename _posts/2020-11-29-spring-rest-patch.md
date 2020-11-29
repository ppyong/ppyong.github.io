---
layout: post
title: Spring REST PATCH
---

REST API를 만들 때 기존에 존재하는 리소스를 수정하는 API의 경우 PUT을 사용하곤 하였습니다. 
프론트 개발자랑 협업하다보면 간혹 "수정되는 데이터(프로퍼티)만 보내도 되나요?" 라는 문의를 많이 듣곤합니다. 

기존에 백앤드 개발자 경험이 적어서 그런가 "전체 데이터를 다 주면 되는데 왜 번거롭게 변경된거만 준단건지.. 번거롭게.." 라고 생각을 하곤 했는데, 듣기론 변경되지 않은 데이터까지 모두 받게 되면 그중 큰 사이즈의 데이터가 있을 경우 더 많은 네트워크 부하가 걸리기 때문에 종종 프론트 개발자 분들이 요청을 하곤 한다 합니다. 


그에 따라 기존에 사용중인 DTO -> Entity로 변환시키기 위해 사용하던  ModelMapper 라이브러리의 옵션을 활용해서 해결하곤 하였습니다.

```java
modelMapper.getConfiguration().setSkipNullEnabled(null)
```

위와 같이 Null일 경우에는 값을 복사하지 않는 옵션을 켜서 사용하도록 수정하였는데 여기서 또 문제가 발생하였습니다. 
특정 데이터를 수정하는게 아닌 지우고 싶을 때 딱히 방법이 떠오르지 않았습니다. null로 받고 싶어도 이미 백앤드에서는 프론트의 Request를 DTO로 받고고 있어서 값이 변경되지 않아서 넘겨주지 않을 때와 값 삭제를 위해 null로 넘겨줄 때를 구분할 방법이 없었던거죠. 대략 아래와 같이 DTO를 사용하고 있었습니다. 

 ```java
public class UpdateReq {
    private String title;

    private String content;
}
 ```

이때부터 뭔가 잘못 설계하고 있다 생각을 하기 시작했습니다. 
"애초에 PUT은 전체 데이터를 삽입 혹은 수정인데, 부분적인 데이터 수정에 PUT이 맞을까?" 라는 의문이 들었고 조금 더 다른 서비스의 REST API들을 찾아보기 시작하였습니다. 

또한 안되는 영어실력으로 Stackoverflow에 문의를 올리기도 하였죠. 
(https://stackoverflow.com/questions/64940600/how-should-i-design-a-rest-api/64940695#64940695)

정보를 찾다보니 이렇게 부분적인 수정일 경우에는 PATCH가 맞다고 판단이 되었습니다. 

그럼 "이제 다시 PATCH로 처리를 해볼까? PATCH에서는 어떤식으로 처리하지?" 라고 생각하며 다시 추가적으로 정보를 알아보았습니다. 

```java
PATCH /my/data HTTP/1.1
Host: example.org
Content-Length: 326
Content-Type: application/json-patch+json
If-Match: "abc123"

[
	{ "op": "test", "path": "/a/b/c", "value": "foo" },
	{ "op": "remove", "path": "/a/b/c" },
	{ "op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ] },
	{ "op": "replace", "path": "/a/b/c", "value": 42 },
	{ "op": "move", "from": "/a/b/c", "path": "/a/b/d" },
	{ "op": "copy", "from": "/a/b/d", "path": "/a/b/e" }
]
```

위와 같은 구조로 REST API를 만들고 op 값을 통해서 데이터를 처리하는 것으로 보였습니다. 
아.. 생각보다 간단하지 않구나 싶었죠. 

일단 위에 구조가 좋단건 알겠지만 막상 어떤식으로 백앤드를 구현해야 처리가 가능한지 떠오르지 않아 이후에 제대로 다시 구현하고 일단은 처음 생각했던대로 삭제하고 싶은 값이 있으면 null로 보내서 삭제하도록 처리하자. 라고 마음을 굳히고 조금 더 이 방법을 찾아보게 되었습니다. 

그러던중 저와 비슷한 고민을 하는 문의글을 찾게되었고 그 답변을 통해 해결법을 찾을 수 있었습니다. 

그건 바로 Java8에서 등장한 Optional을 활용하는 것이었죠. 

```java
public class PatchReq {
    private Optional<String> title;

    private Optional<String> content;

    public void setTitle(String title){
        this.title = Optional.ofNullable(title);
    }

    public void setContent(String content){
        this.content = Optional.ofNullable(content);
    }
}
```

위와 같이 PATCH를 따로 구분하고 각 프로퍼티를 Optional로 선언하였습니다. 위와 같이 할 경우 아예 값이 넘어오지 않는 경우 setter가 호출되지 않아 null이 들어가고 null로 값을 넘길 경우 Optional.empty로 값이 들어가게 됩니다. 

그에 따라 빼서 쓸땐 아래와 같이 매번 null check를 해야하는 번거로움이 있었지만 일단 임시 방편이 될 수 있지만 문제를 해결할 수 있었습니다. 

```java
if(req.getTitle() != null) {
	board.changeTitle(req.getTitle().orElse(null));
}
if(req.getContent() != null) {
	board.changeContent(req.getContent().orElse(null));
}
```

이 부분에 대해서는 조금 더 공부하고 PATCH 를 제대로 활용할 수 있게 개선해나갈 생각 입니다. 