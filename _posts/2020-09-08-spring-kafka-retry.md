
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

이 시기만해도 Kafka라는걸 처음 접해봤기에 당연히 실패한 OFFSET 머물러 있을 것이다라고 생각했습니다. 

하지만 생각과는 다르게 동작하고 있는걸 데이터베이스가 복구되고나서야 알 수 있었습니다. 

만약 100번 OFFSET 데이터를 소비하는 중에 Exception이 발생해서 더이상 진행을 못하면 당연히 100번 OFFSET 가르키고 있으니 데이터베이스만 복구하면 정상적으로 다시 100번 OFFSET 

데이터를 소비하여 적재를 하겠지라고 생각하며 데이터베이스를 복구하였는데 데이터를 확인해보니 100번 OFFSET 데이터가 아닌 한참 지난 최신의 데이터가 데이터베이스에 적재되고 있었습니다. 


왜 이런 현상이 발생하고 있을까요? Kafka는 적재된 데이터의 OFFSET 가지고 있고 설정상 auto commit 도 false인데 왜 OFFSET 100번이 아닌걸까요? 

이 상황은 조금 더 Kafka에 대해 학습을 하고 이해할 수 있었습니다. 


Spring Kafka 는 *설정을 하지 않는한 Listner 에서 문제가 발생 시 재시도를 하지 않습니다.* 그에 따라 계속 다음 데이터를 소비하게 되면서 OFFSET 은 증가하게 된 것이죠. 

이 소비하는 시점만 하더라도 실제 Kafka에 commit 된 OFFSET은 여전히 100번이었습니다. 

그러나 이후 데이터베이스가 복구되면서 문제가 발생했던 로직이 정상 동작을 하게되고 그에 따라 그 시점의 OFFSET 이 commit 된 것이죠. 

(참고로 Kafka 의 comsumer group 당 OFFSET 은 Topic 내에 저장됩니다.)













