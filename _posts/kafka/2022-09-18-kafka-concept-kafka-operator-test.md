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

#### Kafkacat 사용하기
[Kafkacat](https://www.confluent.io/blog/best-kafka-tools-that-boost-developer-productivity/)
은 카프카를 쉽게 테스트하고 디버깅 할 수 있는 도구이다. 
사용을 위해서는 별도의 설치가 필요한데 아래 명령어로 설치를 해준다.  

```bash
$ apt install kafkacat -y
```  



#### Kafka Pod 바로 사용하기








---
## Reference
[Strimzi Quick Starts](https://strimzi.io/quickstarts/)  