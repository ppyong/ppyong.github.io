---
layout: post
title: AWS-IOT-SOLUTION 이란?
image: /img/aws-iot-core.png
tags: [aws, iot]
---

# DEVICE SHADOW

## 언제든 디바이스 상태 확인 및 설정 

AWS Iot 공식 사이트에서는 아래와 같이 설명하고 있다.

>"AWS Iot Core에서는 언제든 학인하거나 설정할 수 있도록 커넥티드 디바이스의 최신 상태를 저장하므로, 애플리케이션에는 디바이스가 언제나 온라인인 것처럼 표시됩니다. 즉, 디바이스가 연결되어 있지 않아도 애플리케이션에서 디바이스의 상태를 확인할 수 있고, 사용자가 상태를 설정하여 디바이스가다시 연결될때 설정한 상태가 구현되도록 할 수도 있습니다." 

**이런 기능이 가능한건 디바이스 섀도라는 구성을 통해서인데 먼저 이 구성에 대해 알아보자.**

<img src="/assets/img/aws-iot-device-shadow-mechanism.png" width="90%">

대략적인 동작 방식은 위 이미지와 같다.    

MQTT Client인 각 장비들은 장비에 변화가 있을 시 (상태가 변경 시) MQTT message를 broker의 특정 topic으로 publish 하고 publish 된 message는 
DEVICE SHADOW SERVICE 가 subscribe 한다.   

받은 message 에 따라 새로운 장비에 대한 요청이면 장비 정보(DEVICE SHADOW)를 새로 생성하고 기존 장비일 경우는 기존 정보를 수정한다. 그리고 이 결과를 다시 
broker로 publish 한다.   

위 내용 중 topic에 대해 상세하게 알아보자. 

*용어를 각각 아래와 같이 줄여서 설명*
> DEVICE SHADOW SERVICE -> DSS    
> DEVICE SHADOW -> DS

1. $aws/things/myLightBulb/shadow/update/accepted   
-> DSS가 성공적으로 DS를 수정(반영)했을 때 메시지를 보냄 

2. $aws/things/myLightBulb/shadow/update/rejected   
-> DSS가 DS 수정(반영)에 실패 했을 때 메시지를 보냄 

3. $aws/things/myLightBulb/shadow/update/delta   
-> DSS가 전달 받은 DS의 상태와 유지되어야 하는 DS의 상태에 차이가 있을 때 메시지를 보냄     

4. $aws/things/myLightBulb/shadow/get/accepted
-> DSS가 DS에 대한 요청을 성공적으로 처리했을 때 메시지를 보냄 

5. $aws/things/myLightBulb/shadow/get/rejected   
-> DSS가 DS에 대한 요청에 실패 했을 때 메시지를 보냄 

6. $aws/things/myLightBulb/shadow/delete/accepted    
-> DSS가 DS를 삭제 했을 때 메시지를 보냄

7. $aws/things/myLightBulb/shadow/delete/rejected    
-> DSS가 DS 삭제에 실패 했을 때 메시지를 보냄

8. $aws/things/myLightBulb/shadow/update/documents   
-> DS 수정이 성공적으로 동작 했을 때마다 DSS는 state document(??)를 발행   

<hr/>    
     
**위에서 내용을 바탕으로 데이터 흐름을 나열 해보자.**    


<img src="/assets/img/aws-iot-device-shadow-flow.png?1" width="90%">

아래 과정은 device 상태 변경에 따른 FLOW 이다. 

1. MQTT Client(device)에 상태 변화가 발생하여 device에서 메시지를 보냄    
(topic: $aws/things/myLightBulb/shadow/update)   

2. DSS에서 topic에서 메시지를 subscribe 함    

3. DSS가 변경 사항을 DS에 반영   

4. DSS에서 변경사항 DS에 반영 후 메시지를 보냄     
(topic: $aws/things/myLightBulb/shadow/update/accepted)   

5. DSS에서 변경 후 메시지를 보냄   
(topic: $aws/things/myLightBulb/shadow/update/documents)    




