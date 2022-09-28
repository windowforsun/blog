--- 
layout: single
classes: wide
title: "[Kafka] Kafka 명령어 사용하기"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 를 운영하고 제어하는데 필요한 몇가지 명령어를 사용하고 익혀보자'
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
  - Command
toc: true
use_math: true
---  

## Kafka 명령어 사용해보기 
[Kafka on Kubernetes using Strimzi]({{site.baseurl}}{% link _posts/kafka/2022-09-18-kafka-concept-kafka-operator-test.md %})
에서 `Strimzi` 를 사용해서 `Kubernetes` 위에 `Kafka` 를 구성하는 방법에 대해 알아보았다. 
이번에는 구성한 `Kafka` 를 조작할 수 있는 몇가지 명령어에 대해서 알아본다.  


### Kafka 구성
[Kafka on Kubernetes using Strimzi]({{site.baseurl}}{% link _posts/kafka/2022-09-18-kafka-concept-kafka-operator-test.md %})
에서 구성한 것과는 다르게 이번에는 직접 `yaml` 템플릿을 작성해서 다중 노드로 구성된 `Kafka` 를 실행하도록 한다. 
기본적인 설치 방법은 위 링크에서 확인 할 수 있기 때문에 설치에 대한 자세한 내용은 다루지 않는다.  

```yaml
# my-cluster-triple.yaml 

apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: kafka
spec:
  kafka:
    version: 3.2.1
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: false
      # nodeport 추가 
      - name: external
        port: 9094
        type: nodeport
        tls: false
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.2"
      # 토픽 삭제 가능하도록 옵션 추가
      delete.topic.enable: true
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 10Gi
          deleteClaim: false
    template:
      pod:
        securityContext:
          runAsUser: 0
          fsGroup: 0
        # 서로 다른 노드에서 실행 될 수있도록 설정 추가
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - kafka
                topologyKey: "kubernetes.io/hostname"
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: false
    template:
      pod:
        securityContext:
          runAsUser: 0
          fsGroup: 0
        # 서로 다른 노드에서 실행 될 수있도록 설정 추가
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - zookeeper
                topologyKey: "kubernetes.io/hostname"
  entityOperator:
    topicOperator: {}
    userOperator: {}
```  

`minikube` 또한 다중 노드를 위해 아래 명령어를 사용해서 구성해 준다. 

```bash
$ minikube start --nodes=3 --memory=4096

```  

> Apple Silicon 에서는 아래 명령으로 실행이 필요하다. (2022/09/28 기준)(minikube, podman 설치 필요)
> 
> ```bash
> $ minikube start --nodes=3 --memory=4096 --driver=podman --kubernetes-version=v1.23.1
> ```

이제 명령어로 템플릿을 `Kubernetes` 클러스터에 적용해주면 아래와 같이 실행된 구성을 확인 할 수 있다. 

```yaml
$ kubectl apply -f my-cluster-triple.yaml

$ kubectl get all -n kafka
NAME                                              READY   STATUS    RESTARTS      AGE
pod/my-cluster-entity-operator-5746694d45-nhsxw   3/3     Running   0             45m
pod/my-cluster-kafka-0                            1/1     Running   0             46m
pod/my-cluster-kafka-1                            1/1     Running   0             46m
pod/my-cluster-kafka-2                            1/1     Running   0             46m
pod/my-cluster-zookeeper-0                        1/1     Running   0             46m
pod/my-cluster-zookeeper-1                        1/1     Running   0             46m
pod/my-cluster-zookeeper-2                        1/1     Running   0             46m
pod/strimzi-cluster-operator-9b9c856f9-4knsf      1/1     Running   2 (90m ago)   27h

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                               AGE
service/my-cluster-kafka-0                    NodePort    10.102.188.48    <none>        9094:30342/TCP                        46m
service/my-cluster-kafka-1                    NodePort    10.110.161.165   <none>        9094:31916/TCP                        46m
service/my-cluster-kafka-2                    NodePort    10.108.82.47     <none>        9094:31899/TCP                        46m
service/my-cluster-kafka-bootstrap            ClusterIP   10.109.166.5     <none>        9091/TCP,9092/TCP,9093/TCP            46m
service/my-cluster-kafka-brokers              ClusterIP   None             <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   46m
service/my-cluster-kafka-external-bootstrap   NodePort    10.107.211.223   <none>        9094:30470/TCP                        46m
service/my-cluster-zookeeper-client           ClusterIP   10.108.80.73     <none>        2181/TCP                              46m
service/my-cluster-zookeeper-nodes            ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP            46m

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-cluster-entity-operator   1/1     1            1           45m
deployment.apps/strimzi-cluster-operator     1/1     1            1           27h

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/my-cluster-entity-operator-5746694d45   1         1         1       45m
replicaset.apps/strimzi-cluster-operator-9b9c856f9      1         1         1       27h
```  

테스트를 위해서 환경변수 설정이 필요하다. 
먼저 `kafka-bootstrap` 의 `Kubernetes` 서비스 도메인과 포트를 아래와 같이 지정해 준다.  

```bash
$ SVCDNS=my-cluster-kafka-bootstrap.kafka.svc:9092
$ echo $SVCDNS
my-cluster-kafka-bootstrap.kafka.svc:9092
```  

그리고 현재 `Kafka` 관련 포트는 아래 명령어로 확인 할 수 있다.  

```bash
$ kubectl get kafka -n kafka my-cluster -o jsonpath={.status} | jq -r ".listeners"
[
  {
    "addresses": [
      {
        "host": "my-cluster-kafka-bootstrap.kafka.svc",
        "port": 9092
      }
    ],
    "bootstrapServers": "my-cluster-kafka-bootstrap.kafka.svc:9092",
    "name": "plain",
    "type": "plain"
  },
  {
    "addresses": [
      {
        "host": "my-cluster-kafka-bootstrap.kafka.svc",
        "port": 9093
      }
    ],
    "bootstrapServers": "my-cluster-kafka-bootstrap.kafka.svc:9093",
    "name": "tls",
    "type": "tls"
  },
  {
    "addresses": [
      {
        "host": "192.168.49.3",
        "port": 30470
      },
      {
        "host": "192.168.49.4",
        "port": 30470
      },
      {
        "host": "192.168.49.2",
        "port": 30470
      }
    ],
    "bootstrapServers": "192.168.49.3:30470,192.168.49.4:30470,192.168.49.2:30470",
    "name": "external",
    "type": "external"
  }
]
```  

> Mac 에서는 아래 명령으로 실행이 필요하다.
>
> ```bash
> $ kubectl get kafka -n kafka my-cluster -o jsonpath={.status.listeners}
> 
> $ kubectl get kafka -n kafka my-cluster -o json | jq -r ".status.listeners"
> ```

테스트를 위해 추가로 설정한 `NodePort` 인 `30470` 을 이후 테스트에서 사용하기 위해 환경변수로 지정해 준다.  

```bash
$ NODEPORT=$(kubectl get svc -n kafka my-cluster-kafka-external-bootstrap -o jsonpath={.spec.ports[0].nodePort})
$ echo $NODEPORT
30470
```  

### 명령어 실행 방법
`Kafka` 를 조작할 수 있는 명령어를 사용할 수 있는 방법 중 3가지 방법을 소개한다.  


#### Kafka Cluster Pod 바로 사용하기
`Strimzi` 를 사용해서 구성한 `Kafka` 클러스터의 `Pod` 을 사용해서 명령어를 수행하는 방법이다. 
사용 가능한 실행 파일은 아래와 같이 확인 할 수 있다.  

```bash
$ kubectl exec -n kafka my-cluster-kafka-0 -c kafka -- ls bin
connect-distributed.sh
connect-mirror-maker.sh
connect-standalone.sh
kafka-acls.sh
kafka-broker-api-versions.sh
kafka-cluster.sh
kafka-configs.sh
kafka-console-consumer.sh
kafka-console-producer.sh
kafka-consumer-groups.sh
kafka-consumer-perf-test.sh
kafka-delegation-tokens.sh
kafka-delete-records.sh
kafka-dump-log.sh
kafka-features.sh
kafka-get-offsets.sh
kafka-leader-election.sh
kafka-log-dirs.sh
kafka-metadata-shell.sh
kafka-mirror-maker.sh
kafka-producer-perf-test.sh
kafka-reassign-partitions.sh
kafka-replica-verification.sh
kafka-run-class.sh
kafka-server-start.sh
kafka-server-stop.sh
kafka-storage.sh
kafka-streams-application-reset.sh
kafka-topics.sh
kafka-transactions.sh
kafka-verifiable-consumer.sh
kafka-verifiable-producer.sh
trogdor.sh
windows
zookeeper-security-migration.sh
zookeeper-server-start.sh
zookeeper-server-stop.sh
zookeeper-shell.sh
```  

현재 `Kafka` 에 생성된 토픽을 확인 하고 싶다면 아래 명령어로 가능하다.  

```bash
$ kubectl exec -n kafka my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --bootstrap-server :9092 --list
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
```  

자기 자신을 `bootstrap-server` 로 지정하면 되기 때문에 별도로 도메인이나 주소를 알아낼 필요가 없다.  

추후에 사용한다면 아래와 같은 형식이 될 것이다.

```bash
$ kubectl exec -n kafka my-cluster-kafka-0 -c kafka -- bin/<shell-name>.sh --bootstrap-server :9092 <option and command>
```  

별도의 구성없이 가장 간편하게 사용할 수 있지만,
실제로 사용해본 결과 `conumser` 를 실행하고 종료했을 때 간혹 정상종료가 되지 않고 컨슈머 클라이언트가 잔존해 있는 경우가 있다. (터미널 이슈 일 수도 있음)


#### Kafka Client Pod 실행하기 
명령어를 실행해주는 `Pod` 를 동일한 `Kubernetes` 클러스터에 `Pod` 으로 실행해서 사용하는 방법이다. 
사용을 위해서는 아래 템플릿을 사용한다.  

```yaml
# test-kafka-client.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-kafka-client
  labels:
    app: test-kafka-client
spec:
  containers:
    - name: test-kafka-client
      image: bitnami/kafka:3.2
      command: ["tail"]
      args: ["-f", "/dev/null"]
```  

```bash
$ kubectl apply -f test-kafka-client.yaml
```  

실행 가능한 파일은 아래와 같이 있다.  

```bash
$ kubectl exec -it test-kafka-client -- ls /opt/bitnami/kafka/bin
connect-distributed.sh        kafka-mirror-maker.sh
connect-mirror-maker.sh       kafka-producer-perf-test.sh
connect-standalone.sh         kafka-reassign-partitions.sh
kafka-acls.sh                 kafka-replica-verification.sh
kafka-broker-api-versions.sh  kafka-run-class.sh
kafka-cluster.sh              kafka-server-start.sh
kafka-configs.sh              kafka-server-stop.sh
kafka-console-consumer.sh     kafka-storage.sh
kafka-console-producer.sh     kafka-streams-application-reset.sh
kafka-consumer-groups.sh      kafka-topics.sh
kafka-consumer-perf-test.sh   kafka-transactions.sh
kafka-delegation-tokens.sh    kafka-verifiable-consumer.sh
kafka-delete-records.sh       kafka-verifiable-producer.sh
kafka-dump-log.sh             trogdor.sh
kafka-features.sh             windows
kafka-get-offsets.sh          zookeeper-security-migration.sh
kafka-leader-election.sh      zookeeper-server-start.sh
kafka-log-dirs.sh             zookeeper-server-stop.sh
kafka-metadata-shell.sh       zookeeper-shell.sh
```  

기본적인 명령어는 아래와 같이 사용할 수 있다. 
현재 `Kafka` 에 생성된 토픽 리스트를 확인하는 명령어이다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --list
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
```  

`Kubernetes` 클러스터에서 실행 중인 `Kafka` `Pod` 을 `bootstrap-server` 로 설정해줘야 하기 때문에 도메인을 추가로 지정해 줘야 한다.  

추후에 사용한다면 아래와 같은 형식이 될 것이다.  

```bash
$ kubectl exec -it test-kafka-client -- <shell-name>.sh --bootstrap-server $SVCDNS <option and command>
```  


#### Kafkacat 사용하기
[Kafkacat](https://www.confluent.io/blog/best-kafka-tools-that-boost-developer-productivity/)
은 카프카를 쉽게 테스트하고 디버깅 할 수 있는 도구이다. 
사용을 위해서는 별도의 설치가 필요한데 아래 명령어로 설치를 해준다.  

```bash
$ apt install kafkacat -y
```  

`Kafkacat` 은 `Kubernetes` 클러스터 내부가 아니라 호스트에 설치가 된 상태이다. 
그러므로 `Kubernetes` 클러스터 내부의 `Pod` 을 외부 프로세스가 접근해야 하기 때문에, 
앞서 `Kafka` 템플릿에 추가했었던 `NodePort` 를 사용해서 연결을 해줘야 한다. 
`Kafka` 구성이 모두 정상적으로 됐다면, 아래 명령어를 통해 `NodePort` 값을 추출해올 수 있다. 

```bash
$ NODEPORT=$(kubectl get svc -n kafka my-cluster-kafka-external-bootstrap -o jsonpath={.spec.ports[0].nodePort})
$ echo $NODEPORT
30470
```  

`Kafkacat` 을 사용해서 생성된 토픽과 `Kafka` 클러스터의 상세 정보를 리스팅하면 아래와 같다.  

```bash
.. 192.168.49.2 은 minikube node ip ..
$ kafkacat -L -b 192.168.49.2:$NODEPORT
Metadata for all topics (from broker -1: 192.168.49.2:30470/bootstrap):
 3 brokers:
  broker 0 at 192.168.49.4:30342
  broker 2 at 192.168.49.2:31899
  broker 1 at 192.168.49.3:31916 (controller)
 3 topics:
  topic "__consumer_offsets" with 50 partitions:
    partition 0, leader 2, replicas: 2,1,0, isrs: 1,0,2
    partition 1, leader 1, replicas: 1,0,2, isrs: 1,0,2

    .. 생략 ..

    partition 48, leader 2, replicas: 2,1,0, isrs: 1,0,2
    partition 49, leader 1, replicas: 1,0,2, isrs: 1,0,2
  topic "__strimzi-topic-operator-kstreams-topic-store-changelog" with 1 partitions:
    partition 0, leader 1, replicas: 1,0,2, isrs: 1,0,2
  topic "__strimzi_store_topic" with 1 partitions:
    partition 0, leader 0, replicas: 0,2,1, isrs: 1,0,2
```  

추후에 사용한다면 아래와 같은 형식이 될 것이다.

```bash
$ kafkacat -b <MINIKUBE_NODE_IP>:$NODEPORT <option and command>
```  

### Topic 조회
현재 `Kafka` 에 생성된 모든 `Topic` 의 리스트는 아래 명령어로 가능하다.  

```bash
.. kafka-topics.sh --bootstrap-server $SVCDNS --list ..
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --list
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
```  

모든 `Topic` 에 대한 자세한 정보는 아래 명령어로 한번에 조회 할 수 있다.  

```bash
.. kafka-topics.sh --bootstrap-server $SVCDNS --describe ..
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --describe
Topic: __strimzi_store_topic    TopicId: To8DEqdrQ7ynnDfx2iDOCg PartitionCount: 1       ReplicationFactor: 3    Configs: min.insync.replicas=2,message.format.version=3.0-IV1
        Topic: __strimzi_store_topic    Partition: 0    Leader: 0       Replicas: 0,2,1 Isr: 1,0,2
Topic: __strimzi-topic-operator-kstreams-topic-store-changelog  TopicId: _MZQk56OQDWBDLcIhIbqIA PartitionCount: 1       ReplicationFactor: 3    Configs: min.insync.replicas=2,cleanup.policy=compact,segment.bytes=67108864,message.format.version=3.0-IV1,min.compaction.lag.ms=0,message.timestamp.type=CreateTime
        Topic: __strimzi-topic-operator-kstreams-topic-store-changelog  Partition: 0    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
Topic: __consumer_offsets       TopicId: CrQY-RtPQyahyDTnbsvYWQ PartitionCount: 50      ReplicationFactor: 3    Configs: compression.type=producer,min.insync.replicas=2,cleanup.policy=compact,segment.bytes=104857600,message.format.version=3.0-IV1
        Topic: __consumer_offsets       Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 1,0,2
        Topic: __consumer_offsets       Partition: 1    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2

        ... 생략 ...

        Topic: __consumer_offsets       Partition: 48   Leader: 2       Replicas: 2,1,0 Isr: 1,0,2
        Topic: __consumer_offsets       Partition: 49   Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
```  

특정 `Topic` 에 대한 상세 정보도 조회 할 수 있다.  

```bash
.. kafka-topics.sh --bootstrap-server $SVCDNS --describe --topic __strimzi_store_topic ..
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --describe --topic __strimzi_store_topic
Topic: __strimzi_store_topic    TopicId: To8DEqdrQ7ynnDfx2iDOCg PartitionCount: 1       ReplicationFactor: 3    Configs: min.insync.replicas=2,message.format.version=3.0-IV1
        Topic: __strimzi_store_topic    Partition: 0    Leader: 0       Replicas: 0,2,1 Isr: 1,0,2
```  

마지막으로 `kubectl` 을 사용해서 `kafka` 리소스를 바탕으로 생성된 `Topic` 을 조회할 수도 있다.  

```bash
$ kubectl get kafkatopics -n kafka
NAME                                                                                               CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                                        my-cluster   50           3                    True
strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                                     my-cluster   1            3                    True
strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a9263134de3507fc0cc4017b   my-cluster   1            3                    True
```  

### Topic 생성
새로운 `Topic` 을 생성하는 방법은 2가지가 있는데 하나는 `Strimzi` 템플릿을 만들어서 생성하는 것과 다른 하나는 명령어를 사용하는 방법이다. 
먼저 `Strimzi` 템플릿을 통해 `CRD` 를 사용하는 방법은 아래와 같은 템플릿을 생성해주면 된다.  

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: test-topic-first
  labels:
    strimzi.io/cluster: "my-cluster"
  namespace: kafka
spec:
  partitions: 2
  replicas: 2
  config:
    retention.ms: 7200000
    segment.bytes: 1073741824
    min.insync.replicas: 2%
```  

그리고 명령어 사용해서 `test-topic-first` 토픽을 생성하는 명령어는 아래와 같다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --create --replication-factor 2 --partitions 1 --topic test-topic-first
Created topic test-topic-first.
```  

`Topic` 을 조회하면 아래와 같이 `replication-factor = 2`, `partitions = 1` 로 생성된 것을 확인 할 수 있다.  


```bash
$ kubectl get kafkatopics -n kafka
NAME                                                                                               CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
test-topic-first                                                                                   my-cluster   1            2                    True
```  

### Producer, Consumer 사용
`Producer`, `Consumer` 를 사용하는 테스트 진행을 위해서는 2개 이상의 터미널이 필요하다. 
먼저 한 터미널에 아래 명령으로 `test-topic-first` 에 메시지를 생산하는 `Producer` 를 실행해 준다.   

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-producer.sh --bootstrap-server $SVCDNS --topic test-topic-first
> {메시지 입력}
```  

그리고 다른 터미널에 `test-topic-first` 에서 방출되는 메시지를 소비하는 `Consumer` 를 실행해 준다. 

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first
```  

이제 `Producer` 터미널에 입력한 메시지가 `Consumer` 터미널에서 출력되는 것을 확인 할 수 있다.  

```bash
.. Producer ..
>myMessage-1
>myMessage-2
>myMessage-3

.. Consumer ..
myMessage-1
myMessage-2
myMessage-3
```  

만약 `Consumer` 가 해당 `Topic` 의 처음부터 구독하고 싶다면 `--from-beginning` 옵션을 추가해주면 아래와 같이 처음 메시지 부터 현시점까지 방출된 메시지를 모두 받아 볼 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --from-beginning
myMessage-1
myMessage-2
myMessage-3
```

### Consumer Group 조회
앞서 `kafka-console-consumer.sh` 를 사용해서 토픽을 구독해 메시지를 수신했다. 
`Kafka` 는 이런 토픽을 구동하는 것을 `Consumer Group` 이라는 단위로 관리한다. 
우리가 `kafka-console-consumer.sh` 로 토픽의 메시지를 구독할때 해당 터미널에서 사용할 `Consumer Group` 이 새롭게 생성된 것이다. 
지금까지 생성된 `Consumer Group` 을 조회하는 명령어는 아래와 같다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --list

console-consumer-91491
console-consumer-1854
__strimzi-topic-operator-kstreams
```  

`kafka-console-consumer.sh` 을 사용해서 생성된 `Consumer Group` 들은 `console-consumer-<숫자>` 형식으로 생성되는 것을 확인 할 수 있다.  

`Consumer Group` 의 자세한 정보를 보면 토픽의 상태도 가늠해 볼 수 있을 정도로 중요하다. 
파티션, `offset`, `lag` 에 대한 자세한 정보를 조회하는 명령어는 아래와 같다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --group console-consumer-60038 --describe

GROUP                  TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
console-consumer-60038 test-topic-first 0          -               3               -               console-consumer-feb14eeb-cc0a-46a1-b7fd-ddd2520346fb /10.244.2.5     console-consumer
```  

파티션은 1개, `offset` 은 4, `lag` 는 존재하지 않는 상태이다.  


### Consumer Group 생성 및 사용
특정 `Consumer Group` 은 `consumer` 사용시 `Consumer Group` 이름을 명시적으로 지정해주면 생성할 수 있다. 
아래는 `test-topic-first-group-1` 을 생성하는 명령어이자, 특정 `Consumer Group` 을 지징해서 토픽을 구독하는 명령이다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1 --from-beginning
myMessage-1
myMessage-2
myMessage-3
```  

다시 `Consumer Group` 목록을 조회하면 `test-topic-first-group-1` 이 정상적으로 생성된 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --list
test-topic-first-group-1
console-consumer-91491
console-consumer-1854
```  

### Consumer Group Offset 초기화
`offset` 초기화는 메시지를 다시 전송받아야 하는 경우에 `offset` 을 초기화해서 토픽의 메시지를 다시 수신할 수 있다. 
`--topic` 으로 특정 토픽을 지정할 수도 있고, `--all-topic` 으로 전체 토픽에 설정할 수 있다. 
`--execute` 옵션을 제거하고 실행하면 실제 반영되지 않고 어떤 결과가 나오는지 출력만 한다.  


옵션|설명
---|---
`--shift-by <Long: number-of-offsets>`|현재 offset에서 +/- 만큼 offset을 증가/감소 한다.
`--to-offset <Long: offset>`|지정한 offset 으로 이동한다.
`--by-duration <String: duration>`|현재 시간에서 특정 시간만큼 이전으로 offset을 이동한다. 형식 `PnDnHmMnSn` (e.g. P7D : 일주일전)
`--to-datetime <String: datetime>`|특정 날씨로 offset을 이동시킨다. 형식 `YYYY-MM-DDTHH:mm:SS.sss`
`--to-latest`|가장 마지막 offset 값(offset 최대값)으로 이동한다. 
`--to-earlist`|가장 처음 offset 값(offset 최소값)으로 이동한다.


현재 `test-topic-first-group-1` 의 상세 정보를 확인하면 아래와 같다. 

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --group test-topic-first-group-1 --describe

Consumer group 'test-topic-first-group-1' has no active members.

GROUP                    TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test-topic-first-group-1 test-topic-first 0          4               4               0               -               -               -
```  

`offset` 이 최대값이므로 마지막(최신) 메시지가까지 모두 수신한 상태이다. 
위 상태에서 `--to-earlist` 로 `--execute` 없이 `offset` 초기화명령어롤 수행해주면 아래와 같다. 

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1 --reset-offsets --to-earliest
WARN: No action will be performed as the --execute option is missing.In a future major release, the default behavior of this command will be to prompt the user before executing the reset rather than doing a dry run. You should add the --dry-run option explicitly if you are scripting this command and want to keep the current default behavior without prompting.

GROUP                          TOPIC                          PARTITION  NEW-OFFSET     
test-topic-first-group-1       test-topic-first               0          0   
```  

`NEW-OFFSET` 이라는 필드를 통해 해당 명령을 수행하면 가장 처음값 `0` 으로 `offset` 이설정 된다는 것을 출력해준다. 
다시 `test-topic-first-group-1` 을 사용해서 `test-topic-first` 토픽을 구독하더라도 `offset` 초기화가 실제되 되지 않았고, 이후 생상된 메시지도 없기 때문에 아무런 메시지도 출력되지 않는다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-first-topic-group-1

.. 메시지 출력 없음 ..
```  

이제 `--execute` 옵션을 추가해서 실제로 수행해본다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1 --reset-offsets --to-earliest --execute

GROUP                          TOPIC                          PARTITION  NEW-OFFSET     
test-topic-first-group-1       test-topic-first               0          0  
```  

다시 `test-topic-first-group-1` 의 상세 정보를 확인하면 `offset` 이 `0` 으로 초기화 돼면서 4개의 `lag` 이 생긴것을 확인 할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --group test-topic-first-group-1 --describe

Consumer group 'test-topic-first-group-2' has no active members.

GROUP                    TOPIC            PARTITION  CURRENT-OFFSET  LOG-END-OFFSET LAG            CONSUMER-ID     HOST            CLIENT-ID
test-topic-first-group-1 test-topic-first 0          0               4              4              -               -               -
```

실제로 `offset` 이 초기화 됐기 때문에 `test-topic-first` 를 구독할때 `--from-beginning` 옵션이 없더라도 처음 메시지부터 다시 수신하게 된다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1
myMessage-1
myMessage-2
myMessage-3
```  

### Consumer Group 삭제
`Consumer Group` 을 삭제할 때는 `Consumer Group` 을 사용하고 있는 `Consumer`(`Application`) 이 존재해서는 안된다. 
만약 아직 종료되지 않은 `Consumer` 가 존재한 상태에서 삭제하려고 하면 아래와 같은 메시지가 출력 된다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --delete --group test-topic-first-group-1 

Error: Deletion of some consumer groups failed:
* Group 'test-topic-first-group-1' could not be deleted due to: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.GroupNotEmptyException: The group is not empty.
```  

실행 중인 `Consumer` 를 모두 종료하고 식재할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --delete --group test-topic-first-group-1
Deletion of requested consumer groups ('test-topic-first-group-1') was successful.
```  

### Topic 수정
`test-topic-first` 토픽은 `Partitions` 이 1개인 상태로 생성했다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --topic test-topic-first --describe
Topic: test-topic-first TopicId: L3xWP_IASIqXeAIrp7bBKg PartitionCount: 1       ReplicationFactor: 2    Configs: min.insync.replicas=2,message.format.version=3.0-IV1
        Topic: test-topic-first Partition: 0    Leader: 2       Replicas: 2,1   Isr: 2,1
```  

토픽의 처리량을 늘리기위해 `Partition` 을 1개에서 2개로 늘려야 한다면 아래 명령어로 가능하다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --topic test-topic-first -alter --partitions 2
```  

다시 `test-topic-first` 의 상세 정보를 조회하면 아래와 같이 `Partition` 이 2개로 설정된 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --topic test-topic-first --describe
Topic: test-topic-first TopicId: L3xWP_IASIqXeAIrp7bBKg PartitionCount: 2       ReplicationFactor: 2    Configs: min.insync.replicas=2,message.format.version=3.0-IV1
        Topic: test-topic-first Partition: 0    Leader: 2       Replicas: 2,1   Isr: 2,1
        Topic: test-topic-first Partition: 1    Leader: 0       Replicas: 0,1   Isr: 0,1
```  

이전에 `Partition` 이 1개 일때 동일한 `Consumer Group` 을 1개를 초과하는 `Consumer` 가 사용중이라면 실제 메시지는 특정 1개 `Consumer` 에게만 전달된다. 

> 매핑된 `Consumer` 외 다른 `Consumer` 들은 현재 매핑된 `Consumer` 가 가용 불가 상황이 됐을 떄 바로 메세지를 받아 처리 할 수 있도록 백업 용도로 사용 가능하다.  

지금은 `Partition` 을 2개로 늘린 상태이므로 `Consumer Group` 에 2개의 `Consumer` 까지는 병렬로 메시지 전달이 가능하다. 
여기서 병령이라는 것은 푸시 된 메시지가 2개의 `Partition` 중 한개에 저장되고, 만약 2개의 `Consumer` 를 사용한다면 각각 서로다른 하나의 `Partition` 과 연결을 맺여 메시지를 수신받게 된다. 
이를 통해 `Consumer` 의 성능이 부족해 지속적인 `Consumer Lag` 이 발생할 때 `Partition` 과 `Consumer` 를 늘려 동시성을 증가 시킬 수 있다. 


```bash
.. Producer ..
$ kubectl exec -it test-kafka-client -- kafka-console-producer.sh --bootstrap-server $SVCDNS --topic test-topic-first
>myMessage2-1
>myMessage2-2
>myMessage2-3
>myMessage2-4

.. First Consumer ..
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1
myMessage2-1
myMessage2-3

.. Second Consumer ..
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1
myMessage2-2
myMEssage2-4
```  

### Topic 메시지 삭제
토픽에 저장된 메시지를 삭제가 필요한 경우 `json` 파일을 작성해서 가능하다. 

먼저 테스트용으로 사용할 `test-topic-first` 의 전체 메시지 목록은 아래와 같다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --from-beginning
myMessage2-1
myMessage2-2
myMessage2-3
myMessage2-4
```  

아래 `json` 파일은 2개 파티션 중 `Partition 0` 의 `offset` 을 `-1` 로 설정해서 파티션에 해당하는 모든 메시지를 지우는 내용이다.  


```json
{
  "partitions": [
    {
      "topic": "test-topic-first",
      "partition": 0,
      "offset": -1
    }
  ],
  "version": 1
}
```  

해당 명령어 실행을 위해서는 `test-kafka-client` 에서 직접 실행이 필요하다. 
먼저 아래 명령으로 위 내용이 작성된 `delete-partition-all-recrods.sh` 파일을 `test-kafka-client` 로 복사해준다.  

```bash
$ kubectl cp ./delete-partition-all-records.json default/test-kafka-client:/tmp/delete-partition-all-records.json
```  

그리고 `test-kafka-client` 에 접속한 다음 아래 명령으로 `Partition 0` 에 해당하는 모든 메시지만 삭제한다.  

```bash
$ kubectl exec -it test-kafka-client -- bash
$ kafka-delete-records.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --offset-json-file ./tmp/delete-partition-all-records.json
Executing records delete operation
Records delete operation completed:
partition: test-topic-first-0   low_watermark: 55
```  

그리고 `--from-beginning` 을 사용해서 `test-topic-first` 에 있는 모든 메시지를 수신하면 `Parition 0` 은 메시지는 모두 삭제 됐기 때문에 절반의 메시지만 수신되는 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --from-beginning
myMessage2-1
myMessage2-3
```  

아래 내용은 토픽에 포함된 모든 파티션의 `offset` 을 초기화하는 내용이기 때문에 전체 메시지가 모두 삭제 된다.  

```bash
{
  "partitions": [
    {
      "topic": "test-topic-first",
      "partition": 0,
      "offset": -1
    },
    {
      "topic": "test-topic-first",
      "partition": 1,
      "offset": -1
    }
  ],
  "version": 1
}
```  

다시 `Container` 로 복사하고 명령을 수행하면 `--from-beginning` 으로 수신하더라도 어떤 메시지도 수신되지 않는 것을 확인 할 수 있다.  

```bash
$ kubectl cp ./delete-topic-all-records.json default/test-kafka-client:/tmp/delete-topic-all-records.json

$ kafka-delete-records.sh --bootstrap-server my-cluster-kafka-bootstrap.kafka.svc:9092 --offset-json-file ./tmp/delete-topic-all-records.json
Executing records delete operation
Records delete operation completed:
partition: test-topic-first-1   low_watermark: 24
partition: test-topic-first-0   low_watermark: 55

$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --from-beginning
.. 모든 메시지가 삭제 됐기 때문에 메시지 출력이 되지 않는다 ..
```  


### Topic 삭제
사용을 마친 토픽은 삭제할 수 있는데 실행 중인 `Consumer` 가 존재하는 하더라도 토픽은 삭제되기 때문에, 
실행중인 `Consumer` 애플리케이션에서 에러가 발생할 수 있기 때문에 주의해야 하고 모든 `Consumer` 를 종료하고 삭제를 수행해야 한다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-topics.sh --bootstrap-server $SVCDNS --topic test-topic-first --delete
```  




---
## Reference
[Kafka CLI Tutorials](https://www.conduktor.io/kafka/kafka-cli-tutorial)  
[APACHE KAFKA QUICKSTART](https://kafka.apache.org/quickstart)  
[Important Kafka CLI Commands to Know in 2022](https://hevodata.com/learn/kafka-cli-commands/)  
[KAFKA TUTORIAL: USING KAFKA FROM THE COMMAND LINE](http://cloudurable.com/blog/kafka-tutorial-kafka-from-command-line/index.html)  