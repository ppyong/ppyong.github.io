---
layout: post
title: Kubernetes in action 2장~3장 정리
---

- Docker 이미지 실행 

기존 컨테이너 이미지를 실행할 경우 아래와 같이 입력합니다. 실행할 명령어는 대게 이미지 자체에 구워지지만 원하는 경우 무시할 수 있습니다. 

ex) 
ENTRYPOINT - 기본 값으로 동작하면 argument로는 대체되지 않고 append 된다. 
아래와 같이 기존 Dockerfile에 ENTRYPOINT를 지정 후 실행 시 "docker run <image> node app2.js"로 실행 시 "node app.js node app2.js" 와 같이 실행된다.

'''bash
FROM node:7
ADD app.js /app.js
CMD ["node", "app.js"]
'''

CMD - 실행 시 argument를 주면 대체된다. 
아래와 같이 기존 Dockerfile에 CMD를 지정하여도 실행 시 "docker run <image> node app2.js"로 실행 시 app2.js가 실행된다. 

'''bash
FROM node:7
ADD app.js /app.js
CMD ["node", "app.js"]
'''

'''bash
$docker run <image>
'''

- Docker 이미지 버전 

도커는 동일한 이름의 여러 버전 또는 동일한 이미지의 다양한 버전을 지원합니다
태그를 지정하지 않을 시 최신 태그를 참조하는 것으로 가정합니다. 

ex) docker run busybox 는 docker run busybox:latest 와 동일 

'''bash
$docker run <image>:<tag>
'''



