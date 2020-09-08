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

그러던 중 Kafka에서 전달받은 데이터를 저장하는 데이터베이스에 문제가 생겨 Exception이 발생하게 되었습니다. 이 시기만해도 Kafka라는걸 처음 접해봤기에 당연히 실패한 OFFSET에 머물러 있을 것이다라고 생각했습니다.  하지만 생각과는 다르게 동작하고 있는걸 데이터베이스가 복구되고나서야 알 수 있었습니다. 

만약 100번 OFFSET 데이터를 소비하는 중에 Exception이 발생해서 더이상 진행을 못하면 당연히 100번 OFFSET 가르키고 있으니 데이터베이스만 복구하면 정상적으로 다시 100번 OFFSET 데이터를 소비하여 적재를 하겠지라고 생각하며 데이터베이스를 복구하였는데 데이터를 확인해보니 100번 OFFSET 데이터가 아닌 한참 지난 최신의 데이터가 데이터베이스에 적재되고 있었습니다. 


왜 이런 현상이 발생하고 있을까요? Kafka는 적재된 데이터의 OFFSET 가지고 있고 설정상 auto commit 도 false인데 왜 OFFSET 100번이 아닌걸까요? 
이 상황은 조금 더 Kafka에 대해 학습을 하고 이해할 수 있었습니다. 

Spring Kafka 는 *설정을 하지 않는한 Listner 에서 문제가 발생 시 재시도를 하지 않습니다.* 그에 따라 계속 다음 데이터를 소비하게 되면서 OFFSET 은 증가하게 된 것이죠. 

이 소비하는 시점만 하더라도 실제 Kafka에 commit 된 OFFSET은 여전히 100번이었습니다. 
그러나 이후 데이터베이스가 복구되면서 문제가 발생했던 로직이 정상 동작을 하게되고 그에 따라 그 시점의 OFFSET 이 commit 된 것이죠. 
(참고로 Kafka 의 comsumer group 당 OFFSET 은 Topic 내에 저장됩니다.)

그럼 어떻게 Listner에서 문제가 발생 시 재시도를 하게 할 수 있을까요? 

Spring Kafka에서는 다양한 방법을 통해 재시도를 할 수 있는 기능을 제공하고 있습니다. 

먼저 RetryTemplate와 RecoveryCallback을 이용한 방법 입니다.

```java
@Bean
public RetryTemplate retryTemplate(){
    RetryTemplate retryTemplate = new RetryTemplate();

    FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
    fixedBackOffPolicy.setBackOffPeriod(1000l);

    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
    retryPolicy.setMaxAttempts(3);

    retryTemplate.setRetryPolicy(retryPolicy);
    retryTemplate.setBackOffPolicy(fixedBackOffPolicy);

    return retryTemplate;
}

@Bean
public ConsumerFactory<String, String> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "ppyong-group1");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
    return new DefaultKafkaConsumerFactory<>(props);
}

@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(){
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setRetryTemplate(retryTemplate());
    factory.setRecoveryCallback(i->{
        log.info("consumer: {}", i.toString());
        return null;
    });
    //factory.setErrorHandler(new SeekToCurrentErrorHandler());
    return factory;
}
```

위에 코드를 보면 알 수 있듯이 RetryTemplate를 통해 재시도 횟수 및 간격을 설정할 수 있습니다. 또한 이런 재시도가 모두 실패했을 경우 이후 동작은 RecoveryCallback을 통해 지정할 수 있습니다. 
만약 위에 코드에서 RecoveryCallback이 설정되어 있지 않다면 Exception은 컨테이너에서 발생하고 되고 이런 Exception은 ErrorHandler를 통해 처리되게 됩니다. 

하지만 위 코드상의 RetryTemplate를 사용한 재시도는 Consumer Thread를 일시적으로 중지 시킵니다. 그에 따라 max.poll.interval.ms(기본값 5 분)이 지나게 되면 해당 Broker는 할당된 파티션을 취소하고 재조정을 하게 됩니다. 

이런 문제는 ErrorHandler를 통해 해결할 수 있습니다. 정확히는 SeekToConcurrentErrorHandler와 factory.setStatefulRetry()를 통해 말이죠. 
Listener에서 Exception이 발생할 경우 recoveryCallback 이 없거나 recoveryCallback에도 Exception이 발생할 경우 설정된 ErrorHandler에서 처리하게 되는데요. 
이때 동작 할 ErrorHandler를 SeekToConcurrentErrorHandler로 설정 해두면 Exception이 발생한 시점의 OFFSET으로 재시도를 하게 됩니다. 

더불어 factory.setStatefulRetry 설정을 true로 하게되면 단순 재시도가 아닌 현재 Producer에서 가져온 데이터를 모두 버리고 OFFSET을 실패 지점으로 돌려 다시 소비하게 합니다. 

그에 따라 max.poll.interval.ms 시간을 초과하여 재조정 되는 현상을 *거의* 없앨 수 있습니다. (정확히는 이렇게 해도 기본 각 retry 로직에서 시간을 초과하면 재조정은 이뤄지게 됩니다.) 











