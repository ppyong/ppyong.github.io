---
layout: post
title: Kubernetes in action 3장 정리
---

- ### 3.1 포드 소개 
> 포드에 여러 컨테이너가 포함돼 있을 때 항상 포드 전부가 단일 워커 노드에서 실행된다. 

- 3.1.1 포드의 필요성

> - 관련 없는 여러 프로세스를 하나의 컨테이너에서 실행하는 경우 모든 프로세스를 실행 상태로 유지하고 로그를 관리하는 것은 사용자의 책임 
> - 충돌이 발생한 경우 개별 프로세스를 자동으로 다시 시작하는 매커니즘을 포함 시켜야함 -> kubernetes 자체 기능이 아닌 직접 구현이 필요하다는 것을 의미한다.
> - 모든 프로세스는 동일한 표준 출력으로 로그를 남기므로 어떤 프로세스가 어떤 내용을 기록했는지 파악하기 어려울 수 있다. 

> -> 포드로 관리할 경우 특정 컨테이너의 로그만을 볼 수 있다. 
```shell
kubectl logs kubia-manual -c ${container명} 
```

- 3.1.2 포드 

> 여러 컨테이너를 단일 단위로 관리할 수 있는 상위 레벨 구조가 필요하다. 컨테이너 포드는 밀접하게 연관된 프로세스를 함께 실행하고 마치 하나의 컨테이너에서 실행되는 것처럼 동일한 환경을 제공하면서 다소 격리된  상태로 유지한다. 포드의 모든 컨테이너는 동일한 네트워크 및 UTS 네임스페이스에서 실행되기 때문에 모두 같은 호스트 이름 및 네트워크 인터페이스를 공유한다. 파일 시스템의 경우 각 컨테이너의 파일 시스템은 다른 컨테이너와 완전히 분리돼 있다. 

> * [UTS 네임스페이스 참고자료] (https://www.44bits.io/ko/post/container-network-1-uts-namespace)

- 컨테이너가 동일한 IP 및 포트 공간을 공유하는 방법 

> - 포드의 컨테이너가 동일한 네트워크에서 실행되므로 네임스페이스의 경우 같은 IP 주소와 포트 공간을 공유한다는 것이다. 따라서 동일한 포트 번호에 바인딩 될 경우 충돌이 발생한다. 
> - 각 포드에는 별도의 공간이 있으므로 다른 포드의 컨테이너는 포트 충동을 일으킬 수 없다. 
> - 포드 내부의 모든 컨테이너에 역시 동일한 루프백 네트워크 인터페이스 가지므로 컨테이너는 localhost로 동일한 포드에서 다른 컨테이너와 통신할 수 있다. 

- 플랫 인터 포드 네트워크 소개 

<img src="/assets/img/kubernetes-in-action-network-image1.png" width="90%">

> 쿠버네티스 클러스터의 모든 포드는 공유된 단일 플랫, 네트워크 주소 공간에 위치하며 이는 모든 포드가 다른 포드의 IP 주소에 있는 다른 모든 포드에 액세스 할 수 있음을 의미한다. 
> 랜상의 컴퓨터와 마찬가지로 각 포드는 자체 IP 주소를 가지며 포드 전용으로 설정된 네트워크를 통해 다른 모든 포드에서 액세스 할 수 있다. 

<img src="/assets/img/k8s-network-new.png" width="90%">

> 위에 책 자료를 기반으로 조금 자료를 찾아봤다. docker의 경우 각 컨테이너별로 네트워크 네임스페이스를 갖고 있기에 각 컨테이너가 IP를 갖고 있는 형태다. 포드의 경우 같은 포드안에 속한 컨테어는 동일한 IP 즉 네트워크를 공유한다. 이는 pause 컨테이너와 같은 특수한 목적을 가진 컨테이너를 통해서 가능해진다. pause 컨테이너는 veth라는 가상의 네트워크 인터페이스를 생성하고 이 veth는 pod 내부와 host에 pair로 생성된다. 포드의 컨테이너들은 pause 컨테이너가 생성한 이 veth 를 공유하므로서 모두 같은 네트워크를 공유하게 된다. 또한 이 pair로 생성된 가상 네트워크 인터페이스를 통해 pod 외부와 통신할 수 있게 된다. 

> 책에서 단순 플랫한 네트워크라고 표현되어 있었는데, 이해가 되지 않았던 부분은 docker를 설치 할 경우 기본적으로 docker0이라는 브릿지가 생성되고 이 브릿지를 통해서 컨테이너간 통신이 가능하게 되는데, 이 브릿지는 각 네트워크 대역을 갖고 있게 되는데 쿠버네티스 같이 여러 Node를 클러스터로 묶을 경우 "각 Node별 동일한 IP를 가진 포드가 생성될 수 있는거 아닌가?" 라는 의문을 갖게 되었고 그와 동시에 "Node1의 포드A(10.1.1.2)에서 Node2의 포드B(10.1.1.2)로 통신을 시도 할 경우 어떻게 포드를 구분하고 통신이 가능할 수 있지?" 라는 의문 또한 갖게 되었다. 

> 그에 대해 조사하고 위와 같은 그림을 그려보았다. 어떤식으로 이렇게 통신이 가능할까? 

>> "간략하게 설명하면 CNI별 동작은 다르지만 calico 기준 모든 노드의 포드를 위한 IP 대역을 생성하고 해당 대역의 subnet을 각 노드에 할당하게 된다. 그러면 각 노드는 해당 subnet 범위에서 IP를 할당받기 때문에 IP는 중복되지 않을 수 있다. 또한 각 노드에 Routing table 규칙을 추가하여 subnet 범위에 따른 노드 IP를 찾을 수 있게 해준다."

> 이를 위해서 CNI에 대한 이해가 필요하다.
> * [CNI 참고자료 - 1] (https://velog.io/@seunghyeon/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EA%B5%AC%EC%84%B1%EB%8F%84)
> * [CNI 참고자료 - 2] (https://ssup2.github.io/theory_analysis/Kubernetes_Calico_Plugin/)

- 3.1.3 컨테이너를 포드 전체에 적절하게 구성하기 

> - 다수의 포드로 멀티티어 애플리케이션 분할하기 
>> 여러 레이어의 컨테이너가 동일한 포드에 있으면 둘 다 항상 동일한 시스템에서 실행된다. 즉 단일 워커에서만 실행되므로 나머지 노드에서 처리할 수 있는 연산 리소스를 활용하지 않게 된다. 포드를 나누면 각각 다른 노드에서 스케쥴링 할 수 있으므로 인프라의 활용도가 향상된다.

> - 각각 스케일링이 가능한 포드로 분할하기 
>> 포드는 스케일링의 기본 단위다. 한 개의 포드에 프론트와 백엔드 컨테이너로 구성된 경우 각 컨테이너별로 확장이 불가능하다. 컨테이너를 개별적으로 확장해야 하는 경우에는 별도의 포드에 배포되어야 한다. 

> - 하나의 포드에서 다수의 컨테이너 사용 시점 
>> 단일 포드에 여러 컨테이너를 배치하는 주된 이유는 주요 프로세스 한개와 한개 이상의 보조 프로세서로 구성되기 때문이다. 추가 컨테이너(사이드카)의 종류에는 로그 로테이션 및 수집 장치, 데이터 프로세서, 통신 어댑터 등이 있다. 

> - 포드에서 다수의 컨테이너를 사용할 때 결정할 사항 
>> - 컨테이너가 같이 실행돼야 할 필요가 있거나 다른 호스트에서 실행할 수 있는가? 
>> - 각 컨테이너들이 단일화된 전체 혹은 독립적인 컴포넌트로 표현해야 하는가? 
>> - 함께 혹은 개별적으로 컨테이너의 크기가 조정돼야 하는가? 


- ### 3.2 YAML이나 JSON 파일 디스크립터에서 포드 만들기 

- 3.2.1 포드의 YAML 디스크립터 검사

> kubectl get 명령과 -o yaml 옵션을 사용해 포드 전체의 YAML 정의를 가져온다. 
```shell
kubectl get po kubia-zxzij -o yaml
```

>> 포드 정의의 주요 부분  소개 
>> - 메타 데이터에는 포드와 관련된 이름, 네임스페이스, 라벨, 그 밖의 정보가 있다. 
>> - 스펙에는 포드의 컨테이너, 볼륨, 그 밖의 데이터와 같은 포드 내용의 실제 설명이 있다. 
>> - 상태에는 포드의 상태, 각 컨테이너의 설명 및 상태, 포드 내부의 IP 및 그 밖에의 기본 정보 등 실행 중인 포드의 현재 정보가 들어 있다. 

- 3.2.2 포드의 간단한 YAML 디스크립터 만들기 

```shell
apiVersion: v1
kind: Pod
metadata:
    name: kubia-manunal
spec:
    containers:
    - image: luksa/kubia
      name: kubia
      ports: 
      - containerPort: 8080
        protocol: TCP
```

> 컨테이너 포드 지정 
>> 포드 정의에서 포트를 지정하는 것은 정보를 제공하려는 것이다. 포트를 생략해도 클라이언트가 포트를 통해 포드에 연결 가능 여부에는 영향을 미치지 않는다. 

- 3.2.3 kubectl을 사용해 포드 만들기 

> 포드를 만드려면 kubectl create 명령을 사용해야 한다. 
```shell
kubectl create -f kubia-manual.yaml
```

> 실행 중인 포드 전체의 정의 검색 
```shell
kubectl get po kubia-manual -o yaml 
```

> 포드 목록에 새로 생성된 포드 보기 
```shell
kubectl get pods
```

- 3.2.4 애플리케이션 로그 보기 

> kubectl log로 포드의 로그 가져오기 
```shell
kubectl logs kubia-manual
```

> 컨테이너 이름을 지정해 다중 컨테이너 포드의 로그 가져오기 
```shell
kubectl logs kubia-manual -c kubia
```

- 3.2.5 포드에 요청 보내기 

> 포드의 포트에 로컬 네트워크 포트 포워딩 
```shell
kubectl port-forward kubia-manual 8888:8080
```

> 포트 전달자를 통한 포드 연결 
```shell
curl localhost:8888
```

<img src="/assets/img/kubernetes-in-action-port-forward.png" width="90%">

- ### 3.3 라벨을 이용한 포드 구성 
> 포드 수가 증가함에 따라 포드를 하위 집합으로 분류해야 한다는 점이 더욱 명확해진다. 조직화하는 매커니즘이 없다면 이해할 수 없는 복잡한 모양이 된다. 또한 각 포드에서 개별적으로 작업하지 않고 특정 그룹에 속한 모든 포드를 단일 작업으로 실행할 것이다. 

<img src="/assets/img/kubernetes-in-action-label.png" width="90%">

- 3.3.1 라벨 소개

> 라벨은 리소스에 첨부하는 임의의 키/값 쌍이다. 라벨 셀렉터를 사용해 리소스를 선택할 때 활용된다. 리소스는 해당 라벨의 키가 해당 자원 내에서 고유한 경우 한 개 이상의 라벨을 가질 수 있다. 

- 3.3.2 포드를 만들 때 라벨 지정하기 

```shell
apiVersion: v1
kind: Pod
metadata:
    name: kubia-manunal-v2
    labels:
        creation_method: manual
        env: prod
spec:
    containers:
    - image: luksa/kubia
      name: kubia
      ports: 
      - containerPort: 8080
        protocol: TCP
```

- 3.3.3 포드의 라벨 수정 

```shell
kubectl label po kubia-manual creation_method=manual # kubia-manual 포드에 creation_method=manual 라벨을 붙인다 
```

```shell
kubectl label po kubia-manual-v2 env=debug --overwrite # env=prod 라벨을 env=debug로 변경한다. 변경할 때는 --overwrite 옵션을 사용해야 한다 
```

- ### 3.4 라벨 셀렉터를 통한 하위 집합 나열하기 
> 라벨 셀렉터를 사용하면 특정 라벨로 태그가 지정된 포드의 하위 집합을 선택하고 해당 포드에서 작업을 수행할 수 있다. 

> - 특정 키가 있는 라벨 포함
> - 특정 키와 값이 있는 라벨 포함
> - 특정 키가 있지만 지정한 값과 다른 값이 있는 라벨을 포함 

- 3.4.1 라벨 셀렉터를 사용한 포드 나열 

> creation_method=manual 라벨을 지정한 포드 
```shell
kubectl get po -l creation_method=manual
```

> env 라벨을 포함하는 모든 포드
```shell
kubectl get po -l env
```

> env 라벨이 없는 포드
```shell
kubectl get po -l '!env'
```
> - create_method!=manual 은 manual 이외의 다른 값으로 creation_method 라벨이 있는 포드를 선택한다. 
> - env in (prod, devel) 은 env 라벨이 prod 또는 devel로 설정된 포드를 선택한다.
> - env not in (prod, devel) 은 env 라벨이 prod 또는 devel이 아닌 다른 값으로 설정된 포드를 선택한다.

- 3.4.2 라벨 셀렉터에서 다중 조건 사용하기 
> 셀렉터는 여러 개의 쉼표로 구분된 기준을 포함할 수 있다 ex) app=pc, rel=beta

- ### 3.5 포드 스케줄링 제약을 위한 라벨과 셀렉터의 사용 
> 포드를 어디로 스케줄링 해야하는지 최소한의 의견을 제시해야 하는 경우가 있다. 정확한 노드를 지정하는 대신 포드를 스케줄링할 위치를 결정하려면 노드 요구 사항을 설명한 후 쿠버네티스가 해당 요구 사항에 맞는 노드를 선택하도록 해야한다.

- 3.5.1 라벨을 사용한 워커 노드 분류 
> 노드를 포함한 몯느 쿠버네티스 객체에 라벨을 붙일 수 있다. 일반적으로 노드에서 제공하는 하드웨어 유형이나 포드 스케쥴링 시 유용한 항목을 지정한 후 라벨을 붙여 노드를 분류한다

- 3.5.2 노드 지정을 위한 포드 스케줄링 

```shell
apiVersion: v1
kind: Pod
metadata:
    name: kubia-gpu
spec:
    nodeSelector:
        gpu: "true"  # 포드를 만들면 스케쥴러는 gpu=true 라벨이 있는 노드 중에서 선택한다
    containers:
    - image: luksa/kubia
      name: kubia
```

- 3.5.3 하나의 지정된 노드에 스케줄링 
> 각 노드에는 kubernetes.io/hostname 키가 있는 고유한 라벨이 있고 노드의 실제 호스트 이름으로 설정된 값이 있으므로 포드를 정확한 노드로 스케쥴링 할 수도 있다. 그러나 특정 노드로 설정하면 노드가 오프라인인 경우 포드가 예상치 못한 상태가 될 수 있다. 

- ### 3.6 포드에 주석 달기 
> 라벨 외에도 포드와 그 밖의 객체에 주석을 넣을 수 있다. 주석은 키/값 쌍이기도 하므로 본질적으로 라벨과 유사하지만 식별 정보를 보유하지는 않는다. 주석을 잘 사용하면 각 포드나 API 객체 설명이 추가되므로 클러스터를 사용하는 모든 사람이 각 객체의 정보를 빠르게 찾을 수 있다. 

- ### 3.7 그룹 리소스의 네임스페이스 사용하기 
> 2장에서 리눅스 네임스페이스가 아니며 서로 프로세스를 격리시키는데 사용된다. 쿠버네티스 네임스페이스는 객체 이름의 범위를 제공한다. 단일 네임스페이스 하나에 모든 리소스를 갖는 대신 여러개의 네임스페이스로 분할할 수 있으므로 동일한 리소스 이름을 여러 네임스페이스에서 여러 번 사용할 수 있다. 

> 네임스페이스의 목적 
> - 네임스페이스별 리소스 할당
> - 사용자별 네임스페이스 접근 권한

> * [네임스페이스 참고자료] (https://artist-developer.tistory.com/33)

- 3.7.1 네임스페이스의 필요성
> 여러 네임스페이스를 사용하면 많은 구성 요소를 포함하는 복잡한 시스템을 더 작은 그룹으로 분할할 수 있다. 또한 멀티 테넌트 환경에서 리소스를 분리하고 리소스를 운영, 개발 및 QA 환경으로 또는 기타 필요한 방식으로 분할하는데 사용할 수 있다. 

- 3.7.2 다른 네임스페이스와 네임스페이스의 코드
> 클러스터에 있는 모든 네임스페이스 나열
```shell
kubectl get ns
```

> 특정 네이스페이스에 속한 객체 확인
```shell
kubectl get po --namespace kube-system
```

- 3.7.3 네임스페이스 만들기 

```shell
apiVersion: v1
kind: Namespace
metadata:
    name: custom-namespace
```

- 3.7.4 다른 네임스페이스의 객체 관리 
> 생성한 네임스페이스에 리소스를 만들려면 namespace:custom-namespace 항목을 메타 데이터 섹션에 추가하거나 kubctl create 명령을 사용해 리소스를 만들 때 네임스페이스를 지정해야한다. 
```shell
kubectl create -f kubia-manual.yaml -n custom-namespace 
```

> 네임스페이스를 지정하지 않으면 현재 kubectl 컨텍스트에 구성된 기본 네임스페이에서 작업을 수행한다. 
> 현재 컨텍스트의 네임스페이스와 컨텍스트 자체는 kubectl config 명령을 통해 변경할 수 있다. 

- 3.7.5 네임스페이스가 제공하는 격리 
> 네임스페이스를 시용하면 오브젝트를 별도 그룹으로 분리해 특정한 네임스페이스 안에 속한 리소스를 대상으로 작업할 수 있게 해주지만， 실행 중인 오브젝트에 대한 격리는 제공하지 않는다.

- ### 3.8 포드의 중지와 삭제 

- 3.8.1 이름으로 포드 삭제

```shell
kubectl delete po kubia-gpu
```

- 3.8.2 라벨 셀렉터로 포드 삭제

```shell
kubectl delete po -l createion_method=manual
```

```shell
kubectl delete po -l rel=canary
```

- 3.8.3 전체 네임스페이스를 삭제해 포드 삭제

```shell
kubectl delete ns custom-namespace
```

- 3.8.4 네임스페이스가 유지되는 동안 모든 포드 삭제

```shell
kubectl delete po --all
```

- 3.8.5 네임스페이스의 (거의)모든 리소스 삭제

```shell
kubectl delete all --all
```