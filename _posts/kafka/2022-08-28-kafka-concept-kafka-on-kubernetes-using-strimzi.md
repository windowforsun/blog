--- 
layout: single
classes: wide
title: "[Kafka 개념] Kafka on Kubernetes using Strimzi"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Strimzi 를 사용해서 Kubernetes 에 Kafka 를 구성하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
  - Kafka
  - Concept
  - Kafka
  - Kubernetes
  - Strimzi
  - Minikube
toc: true
use_math: true
---  

## Kubernetes 와 Kafka
`Kuberntes` 와 `Kafka` 이 두개의 키워드로 검색을 하면 아래 질문에 대한 글이 보인다.

> `Kafka`, `Kuberntes` 위에 올리는게 좋을까 ?

`Kuberntes` 를 사용해서 `Kafka` 를 어떻게 구성하나요 ?
라는 질문을 하기전에 꼭 `Kubernetes` 를 사용해야 하는지에 대해 먼저 고민이 필요하다.

먼저 `Kafka` 를 `Kuberntes` 를 사용해서 구성해야 하는 이유와 장점은 아래와 같은 것들이 있다.

- 표준화된 배포 환경
  현재 서비스를 구성하는 모든 혹은 대부분의 애플리케이션들이 `Kubernetes` 환경이라면,
  `Kafka` 만 별도의 환경에서 구성하고 관리하는게 유지보수 측면에서 더 많은 비용이 들어갈 수 있다.
  또한 `Kubernetes` 가 지니고 있는 장점을 그대로 `Kafka` 인프라에도 사용할 수 있다는 점이 있다. (간편한 `scale out`, `scale up` 등..)

- 빠른 서비스 구성
  `Kubernetes` 를 사용해서 새로운 인프라를 구성하는 것은 이미 표준화된 작업이 완료된 것을 사용한다는 의미이기도 하다.
  복잡한 설치와 설정 과정이 이미 간편하게 제공되고 있기 때문에 사용하는 입장에서는 쉽고 간편하다.

위와 같은 장점들이 있지만 다시한번 고민해보라는 것은 아래와 같은 이유와 단점들이 있기 때문이다.

- Resource
  `Kubernetes` 환경은 말그대로 추상화된 레이어를 사용해서 그 위에 애플리케이션을 올리는 것을 의미한다.
  `Kuberntes` 를 사용하지 않는 환경과 비교해서 추상화가 늘어 났기 때문에 추가적인 리소스 사용과 함께 성능에 영향을 줄 수 있다.
  `Kafka` 는 운영체제 위에 바로 설치한 후에 튜닝을 하며 고성능을 끌어올리는 경우가 많은데 `Kuberntes` 위에 올리는 경우 튜닝에 어려움이 있다고 한다.

- Monitoring
  위에서 언급한 추상화 레이어가 늘어는 것은 모니터링에도 영향을 끼쳐 모니터링을 할 요소들이 더 늘어날 수 있다.
  `Kafka` 는 큰 부하가 발생하는 요소들이 실시간으로 변경되는 특성이 있다.
  그리고 일반적인 클러스터 구조를 갖는 애플리케이션들과 달리 클러스터 수를 조절한다고 해서 부하가 해소되는 특성을 가지지 않기 때문에 `Kubernetes` 환경에서 모니터링이 더 어려울 수 있다.

- Stateful
  `Kafka` 는 일반적인 서비스 애플리케이션이 가지는 `Stateless` 가 아닌 `Stateful` 성격을 갖는 애플리케이션이다.
  이러한 이유로 설정이 다른 애플리케이션을 구성하는 것에 비해 까다롭고, 동작을 위해 유지가 필요한 구성에 따른 어려움도 있다.

- 표준화된 `Kafka` 도커 이미지의 부재
  `Kafka` 는 `Kafka Server` 와 `Zookeeper` 의 조합으로 구성되기 때문에 2개 모두 구성을 반드시 해줘야 한다.
  `Zookeeper` 의 경우 표준화된 공식 이미지가 존재해서 이를 사용하면 되지만,
  `Kafka` 는 공식 이미지가 아직 존재하지 않는 상태이다.
  어떤 이미지를 사용할지, 해당 이미지에 대한 정보 및 자료는 적절하게 있는지에 대한 조사부터 필요할 수 있다.

### Kafka on Kubernetes
위 글을 통해 먼저 `Kafka` 를 꼭 `Kubernetes` 에 구성해야 하는지에 대해 알아보았다.
우선 현재 대부분의 애플리케이션들이 `Kubernetes` 환경에서 구성되고, 모니터링, 배포 등의
전반적은 `Devops` 가 `Kubernetes` 를 주축으로 사용중이기 때문에 `Kubernetes` 환경에 `Kafka` 를 구성할 필요가 있다고 느껴졌다.

그리고 꼭 서비스 뿐만 아니더라도, 테스트 용도로 간단하게 구성해서 사용하고 깔끔하게 제거 하기위해서도 `Kubernetes` 환경에 `Kafka` 를 구성할 필요가 있다고 생각한다.  
`Kubernetes` 에 `Kafka` 를 구성하는 방법은 자료조사를 해보면 아주 다양한 방법들이 보인다.
그 중에 소개할 방법은
[Strimzi](https://strimzi.io/) 를 사용하는 방법이다.

## Strimzi
[Strimzi](https://strimzi.io/)
는 `Kubernetes` 클러스터에서 `Apache Kafka` 를 구성하는 프로세스를 단순화한 오픈소스 프로젝트이다. 
`Strimzi` 를 사용하면 `Kubernetes` 기능을 확장해서 `Kafka` 배포와 관련된 일반적이고 복잡한 작업을 자동화 할 수 있다. 

`Strimzi` 에서는 `Kubernetes` 에서 `Kafka` 구성, 실행과 관리를 단순화 하기 위해서 `Operator` 라는 것을 제공한다. 
즉 `Operator` 는 `Strimzi` 에서 기본이 되는 구성 요소라고 할 수 있다. 
`Strimzi` 에서는 `Operator` 를 사용해서 아래와 같은 요소들을 단순화 한다. 

- `Kafka` 클러스터 배포 및 실행
- `Kafka` 구성 요소 배포 및 실행
- `Kafka` 에 대한 액세스 구성
- `Kafka` 에 대한 액세스 보안
- `Kafka` 업그레이드
- `Broker` 관리
- `Topic` 생성 및 관리
- 사용자 생성 및 관리

### Operators
`Strimzi` 에서 실제로 제공되는 `Operators` 는 아래와 같다. 

- Cluster Operator
`Apache Kafka Cluster`, `Kafka Connect`, `Kafka MirrorMaker`, `Kafka Bridge`, `Kafka Exporter`, `Cruise Control`, `Entity Operator` 배포 및 관리(클러스터 생성)

- Entity Operator
`Topic Operator` 와 `User Operator` 연산자로 구성
  
- Topic Operator
`Kafka` 의 `Topic` 관리, `Broker` 의 `Topic` 을 생성하거나 삭제하는 역할
  
- User Operator
`Kafka` 사용자 관리, `Kafka` 접근시 접근 승인 및 권한을 부여하는 역할
  

![그림 1]({{site.baseurl}}/img/kafka/concept-kafka-on-kubernetes-using-strimzi-1.png)


### Strimzi 설치와 Kafka 구성하기
`Kubernetes` 클러스터에서 `Strimzi` 설치와 `Kafka` 구성은 `Strimzi` 의 `Quick Starts` 를 바탕으로 
간단하게 살펴본다. 

`Strimzi` 설치를 위해서 [minikube](https://kubernetes.io/docs/tasks/tools/#installation) 
를 사용해서 `Kubernetes` 환경을 구성한다.  

`Docker` 와 `minikube` 버전은 아래와 같다. 

- `Docker` : 20.10.14
- `minikube` : v1.23.2

메모리 설정이 `4GB` 인 `Kubernetes Cluster` 하나를 `minikube` 명령어로 실행한다.  

```bash
$ minikube start --memory=4096
😄  minikube v1.23.2 on Ubuntu 20.04
🎉  minikube 1.26.1 is available! Download it: https://github.com/kubernetes/minikube/releases/tag/v1.26.1
💡  To disable this notice, run: 'minikube config set WantUpdateNotification false'

✨  Automatically selected the docker driver. Other choices: none, ssh
❗  Your cgroup does not allow setting memory.
    ▪ More information: https://docs.docker.com/engine/install/linux-postinstall/#your-kernel-does-not-support-cgroup-swap-limit-capabilities
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.22.2 on Docker 20.10.8 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
💡  kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```  

`minikube` 로 생성한 `Kubernetes Cluster` 상태를 확인하면 정상적으로 실행 중임을 확인 할 수 있다. 

```bash
$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```  

이제 `kubectl` 명령을 사용해서 `Strimzi` 를 사용해서 `Kafka` 를 설치할 `kafka` 네임스페이스를 생성한다.  

```bash
$ kubectl create namespace kafka
namespace/kafka created
$ kubectl get ns | grep kafka
kafka             Active   11s
```  

다음은 `ClusterRoles`, `ClusterRoleBidings` 와 `Custom Resource Definitions(CRD)` 를 포함한 `Strimzi` 설치 파일을 `kafka` 네임스페이스에 적용한다.  

```bash
$ kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
customresourcedefinition.apiextensions.k8s.io/kafkas.kafka.strimzi.io created
clusterrole.rbac.authorization.k8s.io/strimzi-cluster-operator-namespaced created
clusterrole.rbac.authorization.k8s.io/strimzi-kafka-broker created
customresourcedefinition.apiextensions.k8s.io/kafkatopics.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkaconnectors.kafka.strimzi.io created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-kafka-client-delegation created
customresourcedefinition.apiextensions.k8s.io/kafkamirrormaker2s.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkabridges.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/strimzipodsets.core.strimzi.io created
clusterrole.rbac.authorization.k8s.io/strimzi-kafka-client created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-entity-operator created
clusterrole.rbac.authorization.k8s.io/strimzi-cluster-operator-global created
customresourcedefinition.apiextensions.k8s.io/kafkausers.kafka.strimzi.io created
deployment.apps/strimzi-cluster-operator created
rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-entity-operator-delegation created
customresourcedefinition.apiextensions.k8s.io/kafkaconnects.kafka.strimzi.io created
customresourcedefinition.apiextensions.k8s.io/kafkamirrormakers.kafka.strimzi.io created
clusterrolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-kafka-broker-delegation created
configmap/strimzi-cluster-operator created
customresourcedefinition.apiextensions.k8s.io/kafkarebalances.kafka.strimzi.io created
rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator created
serviceaccount/strimzi-cluster-operator created
```  

그리고 아래 명령어로 `Pod` 이 정상적으로 `Running` 상태가 되는지 확인한다.  

```bash
$ kubectl get pod -n kafka -w
NAME                                        READY   STATUS              RESTARTS   AGE
strimzi-cluster-operator-597d67c7d6-l59qv   0/1     ContainerCreating   0          3s
strimzi-cluster-operator-597d67c7d6-l59qv   0/1     Running             0          31s
strimzi-cluster-operator-597d67c7d6-l59qv   1/1     Running             0          70s
```  

`kafka` 네임스페이스에 `Strimzi` 의 `Operator` 중 하나인 `Cluster Operator` 가 정상적으로 설치 된 것을 확인 할 수 있다. 
지금까지 구성된 `Kubernetes` 의 오브젝트는 아래와 같다.  

```bash
$ kubectl get all -n kafka
NAME                                            READY   STATUS    RESTARTS   AGE
pod/strimzi-cluster-operator-597d67c7d6-l59qv   1/1     Running   0          98s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/strimzi-cluster-operator   1/1     1            1           99s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/strimzi-cluster-operator-597d67c7d6   1         1         1       98s
```  

이제 `kafka` 네임스페이스에 `Custom Resource` 생성을 통해 `Apache Zookeeper`, `Apache Kafka` 그리고 `Entity Operator` 를 설치해준다. 
`Custom Resource` 는 `Strimzi` 에서 기본적으로 제공하는 단일 노드환경의 `Custom Reosurce` 를 사용한다.  

```bash
$ kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-persistent-single.yaml -n kafka
kafka.kafka.strimzi.io/my-cluster created
```  

그리고 아래 명령어를 사용해서 구성에 필요한 `Kubernetes` 오브젝트인 `Pod`, `Service` 가 모두 시작할 때까지 기다린다.  

```bash
$ kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka

.. 실행이 완료되면 아래 메시지 출력 ..
kafka.kafka.strimzi.io/my-cluster condition met
```  

이제 `Strimzi` 를 사용해서 필요한 `Kafka` 구성은 모두 설치가 왼료 됐다. 
`kafka` 네임스페이스에 실행된 모든 `Kubernetes` 오브젝트를 확인하면 아래와 같다.  

```bash
$ kubectl get all -n kafka
NAME                                             READY   STATUS    RESTARTS   AGE
pod/my-cluster-entity-operator-5df896f79-bz7tg   3/3     Running   0          26s
pod/my-cluster-kafka-0                           1/1     Running   0          49s
pod/my-cluster-zookeeper-0                       1/1     Running   0          102s
pod/strimzi-cluster-operator-597d67c7d6-l59qv    1/1     Running   0          4m51s

NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
service/my-cluster-kafka-bootstrap    ClusterIP   10.106.114.14   <none>        9091/TCP,9092/TCP,9093/TCP            49s
service/my-cluster-kafka-brokers      ClusterIP   None            <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   49s
service/my-cluster-zookeeper-client   ClusterIP   10.102.36.223   <none>        2181/TCP                              103s
service/my-cluster-zookeeper-nodes    ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP            103s

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-cluster-entity-operator   1/1     1            1           26s
deployment.apps/strimzi-cluster-operator     1/1     1            1           4m52s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/my-cluster-entity-operator-5df896f79   1         1         1       26s
replicaset.apps/strimzi-cluster-operator-597d67c7d6    1         1         1       4m51s
```  

이제 간단한 `Producer` 와 `Consumer` 를 실행해서 `Kafka` 동작을 테스트해본다. 
테스트를 위해서는 `Producer` 와 `Consumer` 실행이 각각 필요하기 때문에 2개의 터미널이 필요하다.  

먼저 `my-topic` 토픽에 메시지를 `Push` 하는 `Producer` 를 아래 명령을 사용해서 실행한다.  

```bash
$ kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.30.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic
If you don't see a command prompt, try pressing enter.
>first message
>second message
>third message
```  

그리고 다른 터미널에서 `my-topic` 토픽을 구독해서 메시지를 `Pull` 하는 `Consumer` 를 아래 명령을 사용해서 실행한다.  

```bash
$ kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.30.0-kafka-3.2.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
If you don't see a command prompt, try pressing enter.
first message
second message
third message
```  

`Producer` 에서 입력한 메시지가 `Consumer` 로 정상적으로 전달되는 것을 확인 할 수 있다.  

테스트가 모두 완료된 이후에는 아래 명령으로 `Kafka` 을 구성한 `Kubernetes Cluster` 를 삭제해주면 된다.  

```bash
$ minikube delete
🔥  Deleting "minikube" in docker ...
🔥  Deleting container "minikube" ...
🔥  Removing /home/windowforsun/.minikube/machines/minikube ...
💀  Removed all traces of the "minikube" cluster.
```  


---
## Reference
[Strimzi Quick Starts](https://strimzi.io/quickstarts/)  
[Apache Kafka on Kubernetes – Could You? Should You?](https://www.confluent.io/blog/apache-kafka-kubernetes-could-you-should-you/)  
[Strimzi Quick Start guide (In Development)](https://strimzi.io/docs/operators/in-development/quickstart.html)  
