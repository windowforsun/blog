--- 
layout: single
classes: wide
title: "[Kafka] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
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
관련 정보는 실제로 `Consumer Group` 을 사용하고 있는 애플리케이션이 동작중이여야 조회 할 수 있기 때문에 동작중인 걸로 조회를 수행한다.  

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
$ kubectl exec -it test-kafka-client -- kafka-console-consumer.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1
```  

다시 `Consumer Group` 목록을 조회하면 `test-topic-first-group-1` 이 정상적으로 생성된 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --list
test-topic-first-group-1
```  

### Consumer Group Offset 초기화
`offset` 초기화는 메시지를 다시 전송받아야 하는 경우에 `offset` 을 초기화해서 토픽의 메시지를 다시 수신할 수 있다. 
`--topic` 으로 특정 토픽을 지정할 수도 있고, `--all-topic` 으로 전체 토픽에 설정할 수 있다. 
`--execute` 옵션을 제거하고 실행하면 실제 반영되지 않고 어떤 결과가 나오는지 출력만 한다.  



```bash
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --group test-topic-first-group-1 --describe
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1 --reset-offsets --to-earliest

$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --group test-topic-first-group-1 --describe
$ kubectl exec -it test-kafka-client -- kafka-consumer-groups.sh --bootstrap-server $SVCDNS --topic test-topic-first --group test-topic-first-group-1 --reset-offsets --to-earliest --execute
```  


### Consumer Group 삭제


### Topic 수정
앞서 생성한 `test-topic-first` `Topic` 의 설정 정보을 수정해본다. 
`partitions` 를 1에서 2로 변경하는 명령어는 아래와 같다.  

```bash

```  








### Topic 삭제







---
## Reference
[Strimzi Quick Starts](https://strimzi.io/quickstarts/)  