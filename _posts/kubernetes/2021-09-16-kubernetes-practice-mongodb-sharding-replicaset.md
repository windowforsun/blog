--- 
layout: single
classes: wide
title: "[Kubernetes 실습] K3S 소개와 클러스터 구성"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Lightweight Kubernetes 도구인 K3S에 대해 알아보고 클러스터를 구성하는 방법에 대해서도 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - K3S
  - Cluster
toc: true
use_math: true
---  

## 환경
- `Docker v20.17.7`
- `Kubernetes v1.21.2`
- `Docker Desktop for Windows`

## MongoDB Sharding + ReplicaSet
[Kubernetes MongoDB Replicaset]({{site.baseurl}}{% link _posts/kubernetes/2021-04-04-kubernetes-practice-statefulset-mongodb.md %})
에서 `Kubernetes` 의 `StatefulSet` 을 사용해서 `MongoDB ReplicaSet` 을 구성하는 방법에 대해서 알아 보았다. 
이번에는 `MongoDB Sharding` 까지 추가해서 아래와 같은 구조로 구성하는 방법에 대해 알아 본다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-mongodb-sharding-replicaset-1drawio.drawio.png)  

전체를 구성하는 각 요소의 역할은 아래와 같다.  

요소|설명
---|---
shard|각 `shard` 는 하나의 컬렉션에 대한 부분집합으로 구성된다. 즉 하나의 컬렉션의 데이터가 `N` 개의 `shard` 에 분리돼서 저장된다. 그리고 `shard` 의 구성은 `Replica Set` 으로 다시 구성될 수 있다. 
mongos|`mongos` 는 클라이언트 애플리케이션과 `shard cluster` 간의 인터페이스를 제공하는 쿼리 라우터이다. 즉 `shard cluster` 사용을 위해서는 `mongos` 를 통해 쿼리를 수행해야 한다. 
config server|`config server` 는 전체 `MongoDB Sharding Cluster` 를 구성하는 서버의 메타데이터와 설정 정보를 저장한다. 

사용할 파일의 구성은 아래와 같다. 
각 파일의 역할과 설정 방법에 대해서는 아래 부분에서 차례대로 살펴보도록 한다.  

```
.
├── key.txt
├── mongodb-configserver-statefulset.yaml
├── mongodb-mongos-statefulset.yaml
├── mongodb-pv-template.yaml
├── mongodb-shard-1-statefulset.yaml
├── mongodb-shard-2-statefulset.yaml
├── mongodb-shard-3-statefulset.yaml
└── mongodb-storageclass.yaml
```  

### Key 파일 생성
`MongoDB` 에서 인증으로 사용할 `key.txt` 파일을 아래 명령을 통해 생성한다.  

```bash
$ openssl rand -base64 741 > ./key.txt
```  

그래서 생성된 `key.txt` 파일을 사용해서 `kubernetes` 의 `secret` 으로 생성해 준다.  

```bash
$ kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=./key.txt
$ kubectl get secret
NAME                    TYPE                                  DATA   AGE
shared-bootstrap-data   Opaque                                1      40s
```  

### StorageClass 생성
`MongoDB` 는 모두 `StatefulSet` 을 사용할 예정이므로 데이터를 영구적으로 저장할 공간을 할당 받을 수 있는 `StorageClass` 를 생성해 준다. 
빠른 테스트를 위해 `reliclimPolicy` 는 `Delete` 로 설정해서 삭제시 저장 공간도 함께 삭제될 수 있도록 설정했다.  

```yaml
# mongodb-storageclass.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mongodb-storage
  labels:
    group: mongodb
provisioner: docker.io/hostpath
allowVolumeExpansion: true
# pvc 삭제시 pv를 유지하는 경우
#reclaimPolicy: Retain
# pvc 삭제시 pv도 함께 삭제할 경우
reclaimPolicy: Delete
```  

`kubectl apply` 명령을 사용해서 클러스터에 적용한다.  

```bash
$ kubectl apply -f mongodb-storageclass.yaml
$ kubectl get storageclass -l group=mongodb
NAME              PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
mongodb-storage   docker.io/hostpath   Delete          Immediate           true                   50s
```  

### PV 
`StorageClass` 을 통해 `Pod` 에게 영구 저장공간을 제공하는 `PersistentVolume` 템플릿을 아래와 같이 작성해 준다.  

```yaml
# mongodb-pv-template.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv-<SIZE>g-<ID>
  labels:
    group: mongodb
spec:
  capacity:
    storage: <SIZE>Gi
  accessModes:
    - ReadWriteOnce
  # pvc 삭제시 pv를 유지하는 경우
  #reclaimPolicy: Retain
  #persistentVolumeReclaimPolicy: Retain
  # pvc 삭제시 pv 의 파일도 함께 삭제할 경우
  persistentVolumeReclaimPolicy: Delete
  storageClassName: mongodb-storage
  hostPath:
    path: /run/desktop/mnt/host/c/k8s-volume/mongodb-<SIZE>g-<ID>
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                # hostname
                - docker-desktop
```  

`PseristentVolume` 은 먼저 생성한 `StorageClass` 인 `mongodb-storage` 를 통해 `Pod` 과 연결될 수 있도록 설정했다. 
그리고 `PersistentVolume` 내용은 `<SIZE>`, `<ID>` 라는 변수를 사용해서 `MongoDB Sharding` 구성에서 사용되는 `PV` 템플릿을 동적으로 생성해서 적용할 수 있도록 할 계획이다. 
`mongodb-pv-template.yaml` 을 사용해서 `<SIZE>=2` 이고 `<ID>=1` 인 `mongodb-pv-1.yaml` 이라는 템플릿을 생성하는 명령어는 아래와 같다. 

```bash
$ ID=1; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ kubectl apply -f mongodb-pv-1.yaml
```  

이후 각 구성을 클러스터에 적용할때 사용할 계획이다.  
`MongoDB Sharding` 구성 적용을 위한 사전 준비는 여기까지 모두 설명이 되었다. 
이제 아래부터는 구성을 하나씩 클러스터에 적용하는 방법과 절차에 대해서 알아 본다.  

### StatefulSet ConfigServer(ReplicaSet)
`ConfigServer` 는 `ReplicaSet` 을 사용해서 구성한다. 
`ConfigServer` 의 템플릿 파일 내용은 아래와 같다.  

```yaml
# mongodb-configserver-statefulset.yaml

apiVersion: v1
kind: Service
metadata:
  name: mongodb-configserver-service
  labels:
    app: mongodb-configserver
    group: mongodb
spec:
  ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
      nodePort: 31999
  selector:
    app: mongodb-configserver
  type: NodePort
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-configserver-statefulset
  labels:
    app: mongodb-configserver
    environment: test
    group: mongodb
spec:
  serviceName: mongodb-configserver-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb-configserver
      role: mongo-configserver
      environment: test
  template:
    metadata:
      labels:
        app: mongodb-configserver
        role: mongo-configserver
        environment: test
        group: mongodb
        replicaset: ConfigDBRepSet
    spec:
      terminationGracePeriodSeconds: 10
      volumes:
        - name: secrets-volume
          secret:
            secretName: shared-bootstrap-data
            defaultMode: 256
      containers:
        - name: configserver
          image: mongo:4.4.4
          command:
            - "mongod"
            - "--port"
            - "27017"
            - "--wiredTigerCacheSizeGB"
            - "0.25"
            - "--bind_ip"
            - "0.0.0.0"
            - "--configsvr"
            - "--replSet"
            - "ConfigDBRepSet"
            - "--auth"
            - "--clusterAuthMode"
            - "keyFile"
            - "--keyFile"
            - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
            - "--setParameter"
            - "authenticationMechanisms=SCRAM-SHA-1"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: secrets-volume
              readOnly: true
              mountPath: /etc/secrets-volume
            - name: mongodb-volumeclaim
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-volumeclaim
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: mongodb-storage
        resources:
          requests:
            storage: 2Gi
```  

먼저 `mongodb-configserver-service` 는 `NodePort` 를 사용해서 쿠버네티스 클러스터 외부에서도 `ConfigServer` 에 접속 가능하도록 했다. 
만약 쿠버네티스 클러스터 내부에서만 접근가능해야 한다면 `ClusterIP` 로 수정해주면 된다. 
그리고 `ReplicaSet` 의 이름을 `ConfigDBRepSet` 으로 설정한 것을 확인할 필요가 있고, 이는 이후 `ReplicaSet` 구성에서 사용되는 이름이다. 
그리고 `mongodb-configserver-statefulset` 은 총 3개의 `Pod` 을 생성하고 이를 `ReplicaSet` 으로 구성하게 된다.  
볼륨은 앞서 생성한 `secret-volume` 과 이후 생성할 `mongodb-pv-1 ~ 3.yaml` 의 `PV` 와 매핑 될 수 있는 `mongodb-storage` 를 사용하고 있다.


`ConfigServer` 또한 메타데이터, 설정정보 등을 저장하고 관리해야 하기 때문에 `PV` 사용이 필요하다.
먼저 아래 명령어를 통해 `ConfigServer` 에서 사용할 `PV`를 생성한다.  

```bash
$ ID=configserver-0; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=configserver-1; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=configserver-2; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ kubectl apply -f mongodb-pv-configserver-0.yaml
$ kubectl apply -f mongodb-pv-configserver-1.yaml
$ kubectl apply -f mongodb-pv-configserver-2.yaml
$ kubectl get pv
NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS      REASON   AGE
mongodb-pv-2g-configserver-0   2Gi        RWO            Delete           Available           mongodb-storage            11s
mongodb-pv-2g-configserver-1   2Gi        RWO            Delete           Available           mongodb-storage            7s
mongodb-pv-2g-configserver-2   2Gi        RWO            Delete           Available           mongodb-storage            4s
```  


아래 명령을 통해 `ConfigServer` 템플릿을 적용하고 확인이 필요한 오브젝트를 확인해보면 아래와 같다.  

```bash
$ kubectl apply -f mongodb-
configserver-statefulset.yaml
service/mongodb-configserver-service created
statefulset.apps/mongodb-configserver-statefulset created
$ kubectl get pod,pv,pvc
NAME                                     READY   STATUS    RESTARTS   AGE
pod/mongodb-configserver-statefulset-0   1/1     Running   0          37s
pod/mongodb-configserver-statefulset-1   1/1     Running   0          33s
pod/mongodb-configserver-statefulset-2   1/1     Running   0          29s

NAME                                            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                            STORAGECLASS      REASON   AGE
persistentvolume/mongodb-pv-2g-configserver-0   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-configserver-statefulset-0   mongodb-storage            102s
persistentvolume/mongodb-pv-2g-configserver-1   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-configserver-statefulset-2   mongodb-storage            98s
persistentvolume/mongodb-pv-2g-configserver-2   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-configserver-statefulset-1   mongodb-storage            95s

NAME                                                                           STATUS   VOLUME                         CAPACITY  ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/mongodb-volumeclaim-mongodb-configserver-statefulset-0   Bound    mongodb-pv-2g-configserver-0   2Gi  RWO            mongodb-storage   37s
persistentvolumeclaim/mongodb-volumeclaim-mongodb-configserver-statefulset-1   Bound    mongodb-pv-2g-configserver-2   2Gi  RWO            mongodb-storage   33s
persistentvolumeclaim/mongodb-volumeclaim-mongodb-configserver-statefulset-2   Bound    mongodb-pv-2g-configserver-1   2Gi  RWO            mongodb-storage   29s
```  

`StatefulSet` 에 관리되는 `Pod` 도 모두 정상적으로 실행 중이고, `PV`, `PVC` 도 모두 정상적으로 생성된 것을 확인 할 수 있다.  

다음으로 `ConfigServer` 로 생성한 3개의 `Pod` 을 하나의 `ReplicaSet` 으로 구성하는 방법은 아래와 같다.  

```bash
$ kubectl exec -it mongodb-configserver-statefulset-0 -- bash
root@mongodb-configserver-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb

.. service discorvery dns 를 사용해서 3개의 Pod 을 ReplicaSet 으로 등록한다. 
> rs.initiate({
    _id: "ConfigDBRepSet",
    members: [
        {
          _id: 0, host: "mongodb-configserver-statefulset-0.mongodb-configserver-service.default.svc.cluster.local:27017"
        },
        {
          _id: 1, host: "mongodb-configserver-statefulset-1.mongodb-configserver-service.default.svc.cluster.local:27017"
        },
        {
          _id: 2, host: "mongodb-configserver-statefulset-2.mongodb-configserver-service.default.svc.cluster.local:27017"
        }
    ]
});
{
        "ok" : 1,
        "$gleStats" : {
                "lastOpTime" : Timestamp(1631979592, 1),
                "electionId" : ObjectId("000000000000000000000000")
        },
        "lastCommittedOpTime" : Timestamp(0, 0)
}

ConfigDBRepSet:SECONDARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
true

.. flase 가 나올 떄까지 반복해 준다. flase 가 나오면 모든 구성이 완료 된 것이다 ..

ConfigDBRepSet:PRIMARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
false
```  


### StatefulSet MongoDB Shard(ReplicaSet)
샤드는 우선 2개만 생성해서 샤드 클러스터를 추가하고 나머지 하나는 이후에 추가하는 방향으로 진행한다. 
하나의 샤드는 3개의 `Pod` 로 구성되고 이를 `ReplicaSet` 으로 설정한다. 
아래는 하나의 샤드의 템플릿을 자동으로 생성할 수 있는 템플릿 파일 내용이다.  

```yaml
# mongodb-shard-template.yaml

apiVersion: v1
kind: Service
metadata:
  name: mongodb-shard-<ID>-service
  labels:
    app: mongodb-shard-<ID>
    group: mongodb
spec:
  ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
      nodePort: 327<ID>
  selector:
    app: mongodb-shard-<ID>
  type: NodePort

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-shard-<ID>-statefulset
  labels:
    app: mongodb-shard-<ID>
    environment: test
    replicaset: shard-<ID>
    group: mongodb
spec:
  serviceName: mongodb-shard-<ID>-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb-shard-<ID>
      role: mongo
      environment: test
      replicaset: shard-<ID>
  template:
    metadata:
      labels:
        app: mongodb-shard-<ID>
        role: mongo
        environment: test
        replicaset: shard-<ID>
        group: mongodb
    spec:
      terminationGracePeriodSeconds: 10
      volumes:
        - name: secrets-volume
          secret:
            secretName: shared-bootstrap-data
            defaultMode: 256
      containers:
        - name: mongodb
          image: mongo:4.4.4
          command:
            - "mongod"
            - "--port"
            - "27017"
            - "--wiredTigerCacheSizeGB"
            - "0.25"
            - "--bind_ip"
            - "0.0.0.0"
            - "--shardsvr"
            - "--replSet"
            - "shard-<ID>"
            - "--auth"
            - "--clusterAuthMode"
            - "keyFile"
            - "--keyFile"
            - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
            - "--setParameter"
            - "authenticationMechanisms=SCRAM-SHA-1"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: secrets-volume
              readOnly: true
              mountPath: /etc/secrets-volume
            - name: mongodb-volumeclaim
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-volumeclaim
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: mongodb-storage
        resources:
          requests:
            storage: 2Gi
```  

먼저 사용했던 `mongodb-pv-template.yaml` 과 동일한 방식을 사용한다. 
`<ID>` 라는 변수를 사용해서 샤드를 구분 짓게 되고, `<ID>` 는 기본 2자리 숫자로 설정해야 한다. (01, 02, 11, ..)
`<ID>=01` 인 샤드의 템플릿 파일을 생성은 아래 명령어를 통해 가능하다.  

```bash
$ ID=01; sed -e "s/<ID>/${ID}/g" mongodb-shard-template-statefulset.yaml > mongodb-shard-${ID}-statefulset.yaml
```  

전체적인 템플릿 구성은 `mongodb-configserver-statefulset.yaml` 과 동일하다.  

하나의 샤드 템플릿을 생성하고 쿠버네티스 클러스터에 적용하는 명령은 아래와 같다.  

```bash
.. shard-01 PV ..
$ ID=shard-01-0; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=shard-01-1; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=shard-01-2; sed -e "s/<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ kubectl apply -f mongodb-pv-shard-01-0.yaml
$ kubectl apply -f mongodb-pv-shard-01-1.yaml
$ kubectl apply -f mongodb-pv-shard-01-2.yaml

.. shard-01 ..
$ ID=01; sed -e "s/<ID>/${ID}/g" mongodb-shard-template-statefulset.yaml > mongodb-shard-${ID}-statefulset.yaml
$ kubectl apply -f mongodb-shard-01-statefulset.yaml
service/mongodb-shard-01-service created
statefulset.apps/mongodb-shard-01-statefulset created
```  

위 절차를 `shard-02` 에 대해서도 동일하게 수행하고 관련 오브젝트를 조회하면 아래와 같다.  

```bash
$ kubectl get `kubectl get pod,pv -o=name | grep shard`
pod/mongodb-shard-01-statefulset-0   1/1     Running   0          19m
pod/mongodb-shard-01-statefulset-1   1/1     Running   0          19m
pod/mongodb-shard-01-statefulset-2   1/1     Running   0          19m
pod/mongodb-shard-02-statefulset-0   1/1     Running   0          40s
pod/mongodb-shard-02-statefulset-1   1/1     Running   0          37s
pod/mongodb-shard-02-statefulset-2   1/1     Running   0          33s

NAME                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                        STORAGECLASS      REASON   AGE
persistentvolume/mongodb-pv-2g-shard-01-0   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-01-statefulset-0   mongodb-storage            20m
persistentvolume/mongodb-pv-2g-shard-01-1   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-01-statefulset-1   mongodb-storage            20m
persistentvolume/mongodb-pv-2g-shard-01-2   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-01-statefulset-2   mongodb-storage            20m
persistentvolume/mongodb-pv-2g-shard-02-0   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-02-statefulset-0   mongodb-storage            73s
persistentvolume/mongodb-pv-2g-shard-02-1   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-02-statefulset-2   mongodb-storage            69s
persistentvolume/mongodb-pv-2g-shard-02-2   2Gi        RWO            Delete           Bound    default/mongodb-volumeclaim-mongodb-shard-02-statefulset-1   mongodb-storage            64s
```  

이제 생성된 2개의 샤드(`shard-01`, `shard-02`)에서 0번 `Pod` 에 접속해서 아래 명령을 실행해서 각 샤드 마다 `ReplicaSet` 을 구성한다.  

```bash
.. shard-01 ..
$ kubectl exec -it mongodb-shard-01-statefulset-0 -- bash
root@mongodb-shard-01-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
> rs.initiate({
  _id: "shard-01",
  version: 1,
  members: [
    {
      _id: 0,
      host: "mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:27017"
    },
    {
      _id: 1,
      host: "mongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017"
    },
    {
      _id: 2,
      host: "mongodb-shard-01-statefulset-2.mongodb-shard-01-service.default.svc.cluster.local:27017"
    }
  ]
});
{ "ok" : 1 }
shard-01:SECONDARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
true

.. flase 가 나올 떄까지 반복해 준다. flase 가 나오면 모든 구성이 완료 된 것이다 ..

shard-01:PRIMARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
false


.. shard-02 ..
$ kubectl exec -it mongodb-shard-02-statefulset-0 -- bash
root@mongodb-shard-02-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
> rs.initiate({
  _id: "shard-02",
  version: 1,
  members: [
    {
      _id: 0,
      host: "mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:27017"
    },
    {
      _id: 1,
      host: "mongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017"
    },
    {
      _id: 2,
      host: "mongodb-shard-02-statefulset-2.mongodb-shard-02-service.default.svc.cluster.local:27017"
    }
  ]
});
{ "ok" : 1 }
shard-02:SECONDARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
true

.. flase 가 나올 떄까지 반복해 준다. flase 가 나오면 모든 구성이 완료 된 것이다 ..

shard-02:PRIMARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
false
```  

추가로 이후 테스트를 위해 `shard-01`, `shard-02` 에 아래 명령어로 `root` 권한을 가진 계정을 추가해 준다.  

```bash
.. shard-01, shard-02 수행 ..
shard-01:PRIMARY> use admin;
switched to db admin
shard-01:PRIMARY> db.createUser({
  user: 'root',
  pwd: 'root',
  roles: [ { role: 'root', db: 'admin' } ]
});
Successfully added user: {
        "user" : "root",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
```  

### StatefulSet Mongos
메타데이터, 설정 정보를 저장하는 `ConfigServer` 와 실제 데이터가 저장되는 2개의 샤드 구성이 완료 됐다. 
이제 마지막으로 남은건 실제 `Sharding` 동작을 수행해주는 `Mongos` 의 구성이다. 
그리고 미리 생성한 2개의 샤드를 `Mongos` 에 등록하는 작업으로 샤딩을 구성하는 방법에 대해 알아본다.  

`Mongos` 또한 `StatefulSet` 으로 구성하지만 별도의 `PV` 는 사용하지 않는다. 
아래는 `Mongos` 에 대한 템플릿 파일 내용이다.  

```yaml
# mongodb-mongos-statefulset.yaml

apiVersion: v1
kind: Service
metadata:
  name: mongodb-mongos-service
  labels:
    app: mongodb-mongos
    group: mongodb
spec:
  ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
      nodePort: 32700
  selector:
    app: mongodb-mongos
  type: NodePort

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-mongos-statefulset
  labels:
    app: mongodb-mongos
    environment: test
    group: mongodb
spec:
  serviceName: mongodb-mongos-service
  replicas: 2
  selector:
    matchLabels:
      app: mongodb-mongos
      role: mongo
      environment: test
  template:
    metadata:
      labels:
        app: mongodb-mongos
        environment: test
        group: mongodb
        role: mongo
        replicaset: routers
    spec:
      terminationGracePeriodSeconds: 10
      volumes:
        - name: secrets-volume
          secret:
            secretName: shared-bootstrap-data
            defaultMode: 256
      containers:
        - name: mongos
          image: mongo:4.4.4
          command:
            - "mongos"
            - "--port"
            - "27017"
            - "--bind_ip"
            - "0.0.0.0"
            - "--configdb"
            - "ConfigDBRepSet/mongodb-configserver-statefulset-0.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-statefulset-1.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-statefulset-2.mongodb-configserver-service.default.svc.cluster.local:27017"
            - "--clusterAuthMode"
            - "keyFile"
            - "--keyFile"
            - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
            - "--setParameter"
            - "authenticationMechanisms=SCRAM-SHA-1"
          resources:
            requests:
              cpu: 0.25
              memory: 512Mi
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: secrets-volume
              readOnly: true
              mountPath: /etc/secrets-volume
```  

전체적인 템플릿의 구성은 앞서 적용한 `ConfigServer`, 샤드와 대부분 비슷하고 `PV` 를 사용하지 않는 부분에서 차이가 있다. 
그리고 한가지 눈여겨 볼 부분은 `Deployment` 의 `.spec.template.spec.containers[0].command` 에서 `--configdb` 의 옵션 인자 부분으로, `ConfigServer` 의 `Service Discorvery DNS` 주소가 아래와 같이 등록 된 것을 확인 할 수 있다.  

```bash
"ConfigDBRepSet/mongodb-configserver-statefulset-0.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-statefulset-1.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-statefulset-2.mongodb-configserver-service.default.svc.cluster.local:27017"
```  

`Mongos` 의 구동 시점부터 `ConfigServer` 와 연결되어 `Sharding Cluster` 관련 메타, 설정 데이터를 저장하는 공간으로 사용하게 된다. 
아래 명령을 통해 쿠버네티스 클러스터에 적용한다.  

```bash
$ kubectl apply -f mongodb-mongos-statefulset.yaml
service/mongodb-mongos-service created
statefulset.apps/mongodb-mongos-statefulset created
$ kubectl get `kubectl get pod -o=name | grep mongos`
NAME                           READY   STATUS    RESTARTS   AGE
mongodb-mongos-statefulset-0   1/1     Running   0          2m3s
mongodb-mongos-statefulset-1   1/1     Running   0          2m1s
```  

샤드 2개를 `Mongos` 에 추가하는 명령은 아래와 같다.  
샤드 등록은 `Primiary` 만 수행해 주면 된다.  

```bash
$ kubectl exec -it mongodb-mongos-statefulset-0 -- bash
root@mongodb-mongos-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb

.. add shard-01 ..
mongos> sh.addShard("shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:27017");
{
        "shardAdded" : "shard-01",
        "ok" : 1,
        "operationTime" : Timestamp(1631987705, 5),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631987705, 5),
                "signature" : {
                        "hash" : BinData(0,"OUwhKBWuIYL/wfRmDZPS0YTLNqg="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}

.. add shard-02 ..
mongos> sh.addShard("shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:27017");
{
        "shardAdded" : "shard-02",
        "ok" : 1,
        "operationTime" : Timestamp(1631987713, 5),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631987713, 5),
                "signature" : {
                        "hash" : BinData(0,"85je8zUrP/wwyzeto1Pu1PoqIns="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}
```  

`Mongos` 에 샤드가 잘 구성됐는지 확인하는 방법은 여러가지 있는데 리스팅 하면 아래와 같다. 
먼저 `admin` 데이터베이스에 `root` 권한을 가진 계정을 생성해서 확인을 진행한다.  

```bash
.. db 전환 ..
mongos> use admin;
switched to db admin
.. 계정 생성 .. 
mongos> db.createUser({
  user: 'root',
  pwd: 'root',
  roles: [ { role: 'root', db: 'admin' } ]
});
Successfully added user: {
        "user" : "root",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
.. 계정 로그인 ..
mongos> db.auth('root', 'root');
1
.. 샤딩 상태 확인 명령어 ..
mongos> sh.status();
--- Sharding Status ---
  sharding version: {
        "_id" : 1,
        "minCompatibleVersion" : 5,
        "currentVersion" : 6,
        "clusterId" : ObjectId("614608546ae568626c9a3414")
  }
  shards:
        {  "_id" : "shard-01",  "host" : "shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.l
ocal:27017,mongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017,mongodb-shard-01-statefulset-
2.mongodb-shard-01-service.default.svc.cluster.local:27017",  "state" : 1 }
        {  "_id" : "shard-02",  "host" : "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.l
ocal:27017,mongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-
2.mongodb-shard-02-service.default.svc.cluster.local:27017",  "state" : 1 }

.. 생략 ..


.. 샤딩 클러스터 구성 정보 조회 .. 
mongos> db.admin.runCommand('getShardMap');
{
        "map" : {
                "config" : "ConfigDBRepSet/mongodb-configserver-statefulset-0.mongodb-configserver-service.default.svc.cluster.l
ocal:27017,mongodb-configserver-statefulset-1.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-
statefulset-2.mongodb-configserver-service.default.svc.cluster.local:27017",
                "shard-01" : "shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:27017,m
ongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017,mongodb-shard-01-statefulset-2.mongodb-sh
ard-01-service.default.svc.cluster.local:27017",
                "shard-02" : "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:27017,m
ongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-2.mongodb-sh
ard-02-service.default.svc.cluster.local:27017"
        },
        "ok" : 1,
        "operationTime" : Timestamp(1631987948, 2),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631987948, 2),
                "signature" : {
                        "hash" : BinData(0,"JrEPz88rKzGXgOSrESgA58c00so="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}

.. 샤딩 클러스터 구성 정보 조회 .. 
mongos> db.adminCommand('getShardMap');
{
        "map" : {
                "config" : "ConfigDBRepSet/mongodb-configserver-statefulset-0.mongodb-configserver-service.default.svc.cluster.l
ocal:27017,mongodb-configserver-statefulset-1.mongodb-configserver-service.default.svc.cluster.local:27017,mongodb-configserver-
statefulset-2.mongodb-configserver-service.default.svc.cluster.local:27017",
                "shard-01" : "shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:27017,m
ongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017,mongodb-shard-01-statefulset-2.mongodb-sh
ard-01-service.default.svc.cluster.local:27017",
                "shard-02" : "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:27017,m
ongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-2.mongodb-sh
ard-02-service.default.svc.cluster.local:27017"
        },
        "ok" : 1,
        "operationTime" : Timestamp(1631987950, 19),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631987950, 19),
                "signature" : {
                        "hash" : BinData(0,"Y4g9RcPffYp4A9O0PeKwAo5Wrrk="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}

.. 샤드 구성 정보 조회 .. 
mongos> db.adminCommand({listShards: 1});
{
        "shards" : [
                {
                        "_id" : "shard-01",
                        "host" : "shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:270
17,mongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017,mongodb-shard-01-statefulset-2.mongod
b-shard-01-service.default.svc.cluster.local:27017",
                        "state" : 1
                },
                {
                        "_id" : "shard-02",
                        "host" : "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:270
17,mongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-2.mongod
b-shard-02-service.default.svc.cluster.local:27017",
                        "state" : 1
                }
        ],
        "ok" : 1,
        "operationTime" : Timestamp(1631987963, 5),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631987963, 5),
                "signature" : {
                        "hash" : BinData(0,"4BEicVXwyTfMAau3rSpJT6vJJ1I="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}
```  



### Sharding Test
샤딩이 잘 구성됐는지 까지 확인해본 상태이다. 
테스트로 샤딩 동작이 실제로 잘 동작하는지 간단한 테스트 환경을 바탕으로 확인해본다.  


```bash
.. sh.enableSharding('<database>'); 명령으로 샤딩 데이터베이스 추가 .. 
mongos> sh.enableSharding('sharding-test');
{
        "ok" : 1,
        "operationTime" : Timestamp(1631988379, 25),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631988379, 26),
                "signature" : {
                        "hash" : BinData(0,"Z6J3VPkqZ8Bq38VSIfpzEbZxlfk="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}
mongos> use sharding-test;
switched to db sharding-test

.. items 라는 컬렉션의 index 컬럼에 hashed 타입의 인덱스 생성 .. 
mongos> db.items.createIndex({"index":"hashed"});
{
        "raw" : {
                "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-
02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-2.mongodb-shard-02-servic
e.default.svc.cluster.local:27017" : {
                        "createdCollectionAutomatically" : true,
                        "numIndexesBefore" : 1,
                        "numIndexesAfter" : 2,
                        "commitQuorum" : "votingMembers",
                        "ok" : 1
                }
        },
        "ok" : 1,
        "operationTime" : Timestamp(1631988456, 22),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631988456, 22),
                "signature" : {
                        "hash" : BinData(0,"Z9ymi7EVSbKgo51ItOIVW26cYpQ="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}


.. sh.shardCollection('<database>.<collection>', {<shard key field> : "hashed", ...}); 명령으로 sharding-test.items의 index 필드를 hash 타입의 샤드 키로 등록 ..
mongos> sh.shardCollection("sharding-test.items", {"index":"hashed"});
{
        "collectionsharded" : "sharding-test.items",
        "collectionUUID" : UUID("134ef624-5e89-4142-bfd1-5e3d4de0070a"),
        "ok" : 1,
        "operationTime" : Timestamp(1631988578, 22),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631988578, 22),
                "signature" : {
                        "hash" : BinData(0,"y9EjH49KpwFoHQlf2nm0CPLFKp0="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}
```  

테스트 환경에 대한 부분은 모두 완료 됐다. 
이제 실제로 10000개의 데이터를 `sharding-test.items` 컬렉션에 넣어 실제 샤딩이 잘 이뤄지는지 확인해 본다.  

```bash
.. 데이터 추가전 row 수 카운트 ..
mongos> db.items.count();
0

.. 데이터 10000개 추가 ..
mongos> for(var n=1; n<=10000; n++) db.items.insert({index:n, name:"test"})
WriteResult({ "nInserted" : 1 })

.. 데이터 추가 후 10000개 정상 추가 확인 ..
mongos> db.items.count();
10000

.. shard-01 Pod 에 접속해서 items 의 row 수 확인 ..
$ kubectl exec -it mongodb-shard-01-statefulset-0 -- bash
root@mongodb-shard-01-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
shard-01:PRIMARY> use admin;
switched to db admin
shard-01:PRIMARY> db.auth('root', 'root');
1
shard-01:PRIMARY> use sharding-test;
switched to db sharding-test
shard-01:PRIMARY> db.items.count();
4993

.. shard-02 Pod 에 접속해서 items 의 row 수 확인 ..
$ kubectl exec -it mongodb-shard-02-statefulset-0 -- bash
root@mongodb-shard-02-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
shard-02:PRIMARY> use admin;
switched to db admin
shard-02:PRIMARY> db.auth('root', 'root');
1
shard-02:PRIMARY> use sharding-test;
switched to db sharding-test
shard-02:PRIMARY> db.items.count();
5007
```  

`Mongos` 에서 데이터 10000개를 추가하고 `shard-01`, `shard-02` 에 각각 데이터가 저장된 것을 확인 할 수 있고, 
그 합이 10000개 인 것도 확인 할 수 있다.  

### Shard 추가 
새로은 샤드를 `Mongos` 에 추가하는 방법에 대해서 알아본다. 
테스트는 `shard-03` 템플릿을 만들어 쿠버네티스 클러스터에 추가하고, 이를 `Mongos` 를 통해 샤드로 추가하고 샤딩 동작을 확인하는 순으로 진행 된다.  

먼저 `shard-03` 템플릿을 생성하고 쿠버네티스 클러스터에 적용한다.  

```bash
$ ID=shard-03-0; sed -e "s/
<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=shard-03-1; sed -e "s/
<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ ID=shard-03-2; sed -e "s/
<SIZE>/2/g; s/<ID>/${ID}/g" mongodb-pv-template.yaml > mongodb-pv-${ID}.yaml
$ kubectl apply -f mongodb-pv-shard-03-0.yaml
persistentvolume/mongodb-pv-2g-shard-03-0 created
$ kubectl apply -f mongodb-pv-shard-03-1.yaml
persistentvolume/mongodb-pv-2g-shard-03-1 created
$ kubectl apply -f mongodb-pv-shard-03-2.yaml
persistentvolume/mongodb-pv-2g-shard-03-2 created
$ ID=03; sed -e "s/<ID>/${ID}/g" mongodb-shard-template-statefulset.yaml > mongodb-shard-${ID}-statefulset.yaml
$ kubectl apply -f mongodb-shard-03-statefulset.yaml
service/mongodb-shard-03-service created
statefulset.apps/mongodb-shard-03-statefulset created
```  

그리고 `shard-03` 의 0번 `Pod` 에 접속해서 `ReplicaSet` 을 구성해 준다.  

```bash
> rs.initiate({  _id:
   _id: "shard-03",
   version: 1,
   members: [
     {
       _id: 0,
       host: "mongodb-shard-03-statefulset-0.mongodb-shard-03-service.default.svc.cluster.local:27017"
     },
     {
       _id: 1,
       host: "mongodb-shard-03-statefulset-1.mongodb-shard-03-service.default.svc.cluster.local:27017"
     },
     {
       _id: 2,
       host: "mongodb-shard-03-statefulset-2.mongodb-shard-03-service.default.svc.cluster.local:27017"
     }
   ]
 });
{ "ok" : 1 }

.. false 확인 ..
shard-03:SECONDARY> rs.status().hasOwnProperty("myState") && rs.status().myState != 1
false

.. 테스트 계정 생성 ..
shard-03:PRIMARY> use admin;
switched to db admin
shard-03:PRIMARY> db.createUser({
  user: 'root',
  pwd: 'root',
  roles: [ { role: 'root', db: 'admin' } ]
});
Successfully added user: {
        "user" : "root",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
```  

이제 `Mongos` 에 접속해서 새로 구성한 `shard-03` 을 샤드로 추가해 준다.  

```bash
$ kubectl exec -it mongodb-mongos-statefulset-0 -- bash
root@mongodb-mongos-statefulset-0:/# mongo
MongoDB shell version v4.4.4
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
mongos> use admin;
switched to db admin
mongos> db.auth('root', 'root');
1

.. shard-03 샤드 추가 ..
mongos> sh.addShard("shard-03/mongodb-shard-03-statefulset-0.mongodb-shard-03-service.default.svc.cluster.local:27017");
{
        "shardAdded" : "shard-03",
        "ok" : 1,
        "operationTime" : Timestamp(1631989847, 3),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631989847, 4),
                "signature" : {
                        "hash" : BinData(0,"Wtz0x3WQ3Y96jjBuKP1TSd6mazU="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}

.. 샤드 구성 확인 ..
mongos> db.adminCommand({listShards: 1});
{
        "shards" : [
                {
                        "_id" : "shard-01",
                        "host" : "shard-01/mongodb-shard-01-statefulset-0.mongodb-shard-01-service.default.svc.cluster.local:270
17,mongodb-shard-01-statefulset-1.mongodb-shard-01-service.default.svc.cluster.local:27017,mongodb-shard-01-statefulset-2.mongod
b-shard-01-service.default.svc.cluster.local:27017",
                        "state" : 1
                },
                {
                        "_id" : "shard-02",
                        "host" : "shard-02/mongodb-shard-02-statefulset-0.mongodb-shard-02-service.default.svc.cluster.local:270
17,mongodb-shard-02-statefulset-1.mongodb-shard-02-service.default.svc.cluster.local:27017,mongodb-shard-02-statefulset-2.mongod
b-shard-02-service.default.svc.cluster.local:27017",
                        "state" : 1
                },
                {
                        "_id" : "shard-03",
                        "host" : "shard-03/mongodb-shard-03-statefulset-0.mongodb-shard-03-service.default.svc.cluster.local:270
17,mongodb-shard-03-statefulset-1.mongodb-shard-03-service.default.svc.cluster.local:27017,mongodb-shard-03-statefulset-2.mongod
b-shard-03-service.default.svc.cluster.local:27017",
                        "state" : 1
                }
        ],
        "ok" : 1,
        "operationTime" : Timestamp(1631989857, 2),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1631989857, 2),
                "signature" : {
                        "hash" : BinData(0,"5He2V6eBixIrZeu4LGVuCZXxF6Y="),
                        "keyId" : NumberLong("7009299026919030798")
                }
        }
}
```  

`shard-03` 샤드가 추가된 상태에서 다시 10000개의 데이터를 추가해서 샤드 동작을 확인해 본다.  

```bash
mongos> use sharding-test
switched to db sharding-test
mongos> db.items.count();
10000
mongos> for(var n=10001; n<=20000; n++) db.items.insert({index:n, name:"test"})
WriteResult({ "nInserted" : 1 })

.. shard-03 추가 시점 부터 12506 개로 전체 레코드 수가 늘어나 있음 원인 파악 필요 ..
mongos> db.items.count();
22506
```  

### Shard 삭제





---
## Reference
[Sharding in MongoDB](https://www.mongodb.com/basics/sharding)  
[Sharded Mongodb in Kubernetes StatefulSets on GKE](https://medium.com/google-cloud/sharded-mongodb-in-kubernetes-statefulsets-on-gke-ba08c7c0c0b0)  
[pkdone/gke-mongodb-shards-demo](https://github.com/pkdone/gke-mongodb-shards-demo)  








