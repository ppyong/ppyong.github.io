---
layout: post
title: Checked vs UnChecked Exceptions 
image: /img/hello_world.jpeg
---

자바 개발자로서 이직을 준비하다보면 자주 받게되는 질문중 하나인 Checked Exception 과 UnChecked Exception 에 대해서 알아보고자 합니다. 

자바에서는 위 두 타입의 Exception 이 존재합니다. 

먼저 **Checked Exception** 입니다. Checked Exception 은 컴파일 시에 체크 됩니다. 

아래와 같이 코드가 있고 해당 코드 내에서 Checked Exception 을 발생 시킨다면 Exception 을 처리하던 호출한 상위로 Throws 를 통해 던지던 해야 정상적으로 컴파일을 할 수 있습니다. 


예를 들어볼까요? 

```java
public static void main(String[] args) {
    File f = new File("filePath");
    FileInputStream fis = new FileInputStream(f);
    fis.close();
}
```

위의 코드는 컴파일에 실패하게 됩니다. 

아래 에러와 같이 총 두곳에서 에러가 발생하네요.

```
Error:(7, 31) java: unreported exception java.io.FileNotFoundException; must be caught or declared to be thrown
Error:(8, 18) java: unreported exception java.io.IOException; must be caught or declared to be thrown
```

위의 컴파일 에러를 처리하기 위해서는 위에서 설명한 대로 먼저 Exception 을 처리하는 방법입니다. 

```java
public static void main(String[] args) {
    File f = new File("filePath");
    FileInputStream fis = null;
    
    try {
        fis = new FileInputStream(f);
        fis.close();
    } catch (IOException e){
        e.printStackTrace();
    }
}
```

try ~ catch 를 사용하여 Exception 을 처리할 수 있습니다. 

이번에는 throws 를 통해 해당 메서드를 호출한 곳으로 Exception 을 던져서 해결해보겠습니다. 

```java
public static void main(String[] args) throws IOException {
    File f = new File("filePath");
    FileInputStream fis = new FileInputStream(f);
    fis.close();
}
```

다음으로 **UnChecked Exception** 입니다. 

UnChecked Exception 은 컴파일 시에 체크되지 않습니다. 예외를 지정하고 처리하는 건 모두 프로그래머에게 달렸습니다. Exception 클래스의 자식 클래스인 RuntimeExcetion 하위 클래스, Error 클래스의 하위는 모두 UnChecked Exception 입니다. 그 외에는 Exception 의 다른 하위 클래스는 모두 Checked Exception 입니다. 

그럼 이 두가지 예외를 어떨 때 사용하면 될까요? 오라클 공식 문서에서는 아래와 같이 설명하고 있습니다. 

```
If a client can reasonably be expected to recover from an exception, make it a checked exception. If a client cannot do anything to recover from the exception, make it an unchecked exception.
```

클라이언트가 예외를 복구할 수 있을 것으로 예상되는 상황에 Checked Exception 을 사용하고 클라이언트가 예외를 복구하기 위해 아무 것도 할 수 없는 경우 UnChecked Exception 을 사용한다고 합니다. 


이는 UnChecked Exception 인 RuntimeException 의 경우 프로그래머의 실수에 의해서 발생하기 때문입니다. 이 경우 프로그램적으로 해결하기 보다는 적절하게 코드를 수정하는게 올바른 방법이기 때문입니다. 

Checked Exception 의 경우 외부적인 요인 예로 사용자의 잟못된 프로그램 사용에 의해 발생하기 때문에 이런 상황에 대해 프로그램적으로 어떤식으로든 대처하도록 처리를 강제하는 것입니다. 