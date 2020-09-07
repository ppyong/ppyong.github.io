
---
layout: post
title: Spring kafka 재시도 및 에러 처리 
image: /img/hello_world.jpeg
---

프로젝트 진행 중 발생한 예기치 못한 상황에 대해 알아보던 중 학습한 내용을 정리하고자 작성하였습니다. 

현재 사내 IOT 플랫폼에서는 Kafka를 사용하고 있으며 Spring Kafka를 통해 데이터를 소비하고 있습니다. 

대략적인 로직은 외부에서 입수되는 IOT 센서 데이터는 먼저 Kafka에 저장되고 이 데이터를 기반으로 사용자에게 시각화 데이터도 제공하고 백앤드 API도 만드는 방식입니다. 


아래와 같이 데이터를 필요한 곳에서 구독해서 사용하고 있습니다. 

```java
@KafkaListener(topics = "topic")
public void listen(String message, Acknowledgment ack){
    DataEntity data = new DataEntity(message);
    dataRepository.save(data);
    ack.acknowledge();
}
```

```yaml
kafka:
  enable:
    auto:
      commit: false 
  ackmode: MANUAL
```

그러던 중 Kafka에서 전달받은 데이터를 저장하는 데이터베이스에 문제가 생겨 Exception이 발생하게 되었습니다. 

이 시기만해도 Kafka라는걸 처음 접해봤기에 당연히 실패한 offset에 머물러 있을 것이다라고 생각했습니다. 



그에 따라 데이터는 저장할 수 없는 상태가 되었죠. 










