--- 
layout: single
classes: wide
title: "[Kubernetes 실습] StatefulSet 을 이용한 MongoDB ReplicaSet"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'StatefulSet 을 이용해서 MongoDB ReplicaSet 을 구성하고 관련 테스트를 진행해보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - StatefulSet
  - MongoDB
  - ReplicaSet
toc: true
use_math: true
---  

## 환경
- `Docker desktop for Windows` 
    - `docker v20.10.2` 
    - `kubernetes v1.19.3`

## StatefulSet MongoDB
`Kubernetes` 의 `StatefulSet` 컨트롤러를 사용해서 `MongoDB` `ReplicaSet` 구성하는 방법에 대해 알아본다.  

## 예제
### Key 파일 생성
가장 먼저 `MongoDB` 클러스터에서 사용할 키파일을 아래 명령어로 생성해 준다. 

```bash
openssl rand -base64 741 > ./key.txt
```  

그리고 생성된 키파일을 `kubernetes` 의 `Secret` 으로 생성한다. 

```bash
kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=./key.txt
```  

### StorageClass
`MongoDB-StatefulSet` 에서 사용할 `PV-PVC` 를 동적으로 프로비저닝 할 수 있는 `StorageClass` 를 아래 템플릿으로 정의 한다. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mongodb-storage
  labels:
    group: mongodb
provisioner: docker.io/hostpath
allowVolumeExpansion: true
reclaimPolicy: Retain
```  

### Service, StatefulSet
다음으로 `MongoDB` 를 구성에 필요한 `Service`, `StatefulSet` 템플릿을 아래와 같이 정의 한다. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    app: mongodb
    group: mongodb
spec:
  ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
      nodePort: 30270
  selector:
    app: mongodb
  type: NodePort

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-statefulset
  labels:
    app: mongodb
    environment: test
    replicaset: MainRepSet
    group: mongodb
spec:
  serviceName: mongodb-service
  replicas: 2
  selector:
    matchLabels:
      app: mongodb
      role: mongo
      environment: test
      replicaset: MainRepSet
  template:
    metadata:
      labels:
        app: mongodb
        role: mongo
        environment: test
        replicaset: MainRepSet
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
          image: mongo:4.2
          command:
            - "mongod"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - "MainRepSet"
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

### 적용
앞서 정의한 모든 템플릿을 `Kubernetes Cluaster` 에 적용한다. 

```bash
$ kubectl apply -f storageclass-mongodb.yaml
storageclass.storage.k8s.io/mongodb-storage created
$ kubectl apply -f service-mongodb.yaml
service/mongodb-service created
statefulset.apps/mongodb-statefulset created
```  

생성한 컨트롤러와 오브젝트를 조회하면 아래와 같다. 

```bash
$ kubectl get sc,svc,statefulset,pod -l group=mongodb
NAME                                          PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/mongodb-storage   docker.io/hostpath   Retain          Immediate           true                   4m39s

NAME                      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
service/mongodb-service   NodePort   10.96.172.98   <none>        27017:30270/TCP   67s

NAME                                   READY   AGE
statefulset.apps/mongodb-statefulset   2/2     67s

NAME                        READY   STATUS    RESTARTS   AGE
pod/mongodb-statefulset-0   1/1     Running   0          61s
pod/mongodb-statefulset-1   1/1     Running   0          40s
```  

### MongoDB ReplicaSet 구성

>`MongoDB` `ReplicaSet` 을 정상적으로 구성하기 위해서는 최소 3개의 노드가 필요하다. 
>현재 예제 스텝에서는 2개로 구성을 시작하고 이후 진행되는 예제에서 3개의 노드로 최종적으로 구성한다.  

`ReplicaSet` 구성은 `Kubernetes Cluster` 내부에서 사용할 수 있는 `Service Discovery DNS` 를 사용할 것이다.  
`Pod` 이름, `Service` 이름, `Namespace` 이름 등으로 추론할 수 있지만, 
각 `Pod` 에 접속해서 확인하면 아래와 같다.  

```bash
$ kubectl exec -it mongodb-statefulset-0 -c mongodb -- bash
root@mongodb-statefulset-0:/# hostname -f
mongodb-statefulset-0.mongodb-service.default.svc.cluster.local


$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# hostname -f
mongodb-statefulset-1.mongodb-service.default.svc.cluster.local
```  


0번 파드에 접속해서 현재 `MongoDB` 상태를 조회하면 아래와 같다.  


```bash
$ kubectl exec -it mongodb-statefulset-0 -c mongodb -- bash
root@mongodb-statefulset-0:/# mongo
MongoDB shell version v4.2.13
> rs.status();
{
        "ok" : 0,
        "errmsg" : "no replset config has been received",
        "code" : 94,
        "codeName" : "NotYetInitialized"
}

```  

위에서 알아본 `DNS` 주소를 사용해서 아래 명령어로 `ReplicaSet` 을 구성한다.  

```bash
> rs.initiate({_id: "MainRepSet", version: 1, members: [
...     {_id: 0, host: "mongodb-statefulset-0.mongodb-service.default.svc.cluster.local:27017"},
...     {_id: 1, host: "mongodb-statefulset-1.mongodb-service.default.svc.cluster.local:27017"}
... ]});
{ "ok" : 1 }
```  

다시 `MongoDB` 상태를 확인하면 `ReplicaSet` 이 정상적으로 구성된 것을 확인 할 수 있다. 

```bash
MainRepSet:SECONDARY> rs.status();
{
        "set" : "MainRepSet",

        .. 생략 ..

        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-statefulset-0.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 570,
                        "optime" : {
                                "ts" : Timestamp(1617530812, 1),
                                "t" : NumberLong(-1)
                        },
                        "optimeDate" : ISODate("2021-04-04T10:06:52Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "could not find member to sync from",
                        "configVersion" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-statefulset-1.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 7,
                        "optime" : {
                                "ts" : Timestamp(1617530812, 1),
                                "t" : NumberLong(-1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1617530812, 1),
                                "t" : NumberLong(-1)
                        },
                        "optimeDate" : ISODate("2021-04-04T10:06:52Z"),
                        "optimeDurableDate" : ISODate("2021-04-04T10:06:52Z"),
                        "lastHeartbeat" : ISODate("2021-04-04T10:06:59.293Z"),
                        "lastHeartbeatRecv" : ISODate("2021-04-04T10:06:59.461Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "configVersion" : 1
                }
        ],
        "ok" : 1
}
```  

### ReplicaSet 기본 테스트
테스트를 위해 아래 명령어로 0번 파드에서 `admin` 계정을 생성한다. 

```bash
MainRepSet:SECONDARY> db.getSiblingDB("admin").createUser({
...       user : "main_admin",
...       pwd  : "abc123",
...       roles: [ { role: "root", db: "admin" } ]
...  });
Successfully added user: {
        "user" : "main_admin",
        "roles" : [
                {
                        "role" : "root",
                        "db" : "admin"
                }
        ]
}
```  

그리고 계속해서 0번 파드에서 아래와 같이 데이터를 추가하고 조회하면 아래와 같다.  

```bash
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> use test;
switched to db test
MainRepSet:PRIMARY> db.testcoll.insert({a:1});
WriteResult({ "nInserted" : 1 })
MainRepSet:PRIMARY> db.testcoll.insert({b:2});
WriteResult({ "nInserted" : 1 })
MainRepSet:PRIMARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

0번 파드에서 나와 1번 파드로 접속해 데이터를 조회하면, 
아래와 같이 0번 파드에서 추가한 데이터가 조회되는 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# mongo
MongoDB shell version v4.2.13
MainRepSet:SECONDARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:SECONDARY> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:SECONDARY> use test;
switched to db test
MainRepSet:SECONDARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

### ReplicaSet 추가
`ReplicaSet` 추가를 위해 먼저 `StatefulSet` 의 `replicas` 를 3으로 수정한다. 

```bash
$ kubectl scale --replicas=3 statefulset mongodb-statefulset
statefulset.apps/mongodb-statefulset scaled
$ kubectl get statefulset,pod -l group=mongodb
NAME                                   READY   AGE
statefulset.apps/mongodb-statefulset   3/3     26m

NAME                        READY   STATUS    RESTARTS   AGE
pod/mongodb-statefulset-0   1/1     Running   0          26m
pod/mongodb-statefulset-1   1/1     Running   0          26m
pod/mongodb-statefulset-2   1/1     Running   0          19s
```  

```bash
$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                               STORAGECLASS         REASON   AGE
persistentvolume/pvc-41e7d01f-d1a1-459e-bd02-acff4ab6abf5   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-2   mongodb-storage               16m
persistentvolume/pvc-7d2c2c63-d843-449d-86c8-1aa40ef10dfc   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-1   mongodb-storage               42m
persistentvolume/pvc-b2eab001-bd08-4b48-b86c-6fd264e0c320   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-0   mongodb-storage               45m

NAME                                                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-0   Bound    pvc-b2eab001-bd08-4b48-b86c-6fd264e0c320   2Gi        RWO            mongodb-storage      45m
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-1   Bound    pvc-7d2c2c63-d843-449d-86c8-1aa40ef10dfc   2Gi        RWO            mongodb-storage      42m
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-2   Bound    pvc-41e7d01f-d1a1-459e-bd02-acff4ab6abf5   2Gi        RWO            mongodb-storage      16m
```

0번 파드에 접속해서 아래 명령어로 새로 추가된 2번 파드를 `ReplicaSet` 에 추가한다. 

```bash
kubectl exec -it mongodb-statefulset-0 -c mongodb -- bash
root@mongodb-statefulset-0:/# mongo
MongoDB shell version v4.2.13
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> rs.add("mongodb-statefulset-2.mongodb-service.default.svc.cluster.local:27017")
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1617532128, 1),
                "signature" : {
                        "hash" : BinData(0,"nM/ud95p8pLIYM6d5EK0rHmwI5U="),
                        "keyId" : NumberLong("6947241985056964611")
                }
        },
        "operationTime" : Timestamp(1617532128, 1)
}
```  

`rs.status()` 로 조회하면 2번 파드까지 함께 구성된 것을 확인 할 수 있다. 

```bash
MainRepSet:PRIMARY> rs.status();
{
        "set" : "MainRepSet",
        
        .. 생략 ..        

        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb-statefulset-0.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 1888,
                        "optime" : {
                                "ts" : Timestamp(1617532128, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2021-04-04T10:28:48Z"),
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1617530823, 1),
                        "electionDate" : ISODate("2021-04-04T10:07:03Z"),
                        "configVersion" : 2,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb-statefulset-1.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 1324,
                        "optime" : {
                                "ts" : Timestamp(1617532128, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1617532128, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2021-04-04T10:28:48Z"),
                        "optimeDurableDate" : ISODate("2021-04-04T10:28:48Z"),
                        "lastHeartbeat" : ISODate("2021-04-04T10:28:56.430Z"),
                        "lastHeartbeatRecv" : ISODate("2021-04-04T10:28:56.980Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "configVersion" : 2
                },
                {
                        "_id" : 2,
                        "name" : "mongodb-statefulset-2.mongodb-service.default.svc.cluster.local:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 8,
                        "optime" : {
                                "ts" : Timestamp(1617532128, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1617532128, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2021-04-04T10:28:48Z"),
                        "optimeDurableDate" : ISODate("2021-04-04T10:28:48Z"),
                        "lastHeartbeat" : ISODate("2021-04-04T10:28:56.448Z"),
                        "lastHeartbeatRecv" : ISODate("2021-04-04T10:28:56.901Z"),
                        "pingMs" : NumberLong(2),
                        "lastHeartbeatMessage" : "",
                        "syncingTo" : "",
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "configVersion" : 2
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1617532128, 1),
                "signature" : {
                        "hash" : BinData(0,"nM/ud95p8pLIYM6d5EK0rHmwI5U="),
                        "keyId" : NumberLong("6947241985056964611")
                }
        },
        "operationTime" : Timestamp(1617532128, 1)
}
```  

2번 파드에 접속해서 전에 추가한 데이터를 조회하면 아래와 같이 정상적으로 조회되는 것을 확인 할 수 있다. 

```bash
$ kubectl exec -it mongodb-statefulset-2 -c mongodb -- bash
root@mongodb-statefulset-2:/# mongo
MongoDB shell version v4.2.13
MainRepSet:SECONDARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:SECONDARY> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:SECONDARY> use test;
switched to db test
MainRepSet:SECONDARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

### Pod, StatefulSet 삭제 테스트
현재 `Primary` 인 0번 파드를 삭제 `kubectl` 명령으로 삭제한다. 
`StatefulSet` 컨트롤러에 의해 관리되기 때문에 삭제된 0번 파드는 다시 실행된다. 

```bash
$ kubectl delete pod mongodb-statefulset-0
pod "mongodb-statefulset-0" deleted
$ kubectl get pod -l group=mongodb
NAME                    READY   STATUS    RESTARTS   AGE
mongodb-statefulset-0   1/1     Running   0          13s
mongodb-statefulset-1   1/1     Running   0          37m
mongodb-statefulset-2   1/1     Running   0          11m
```  

1번 파드에 접속하면 1번 파드의 `MongoDB` 가 `Primary` 로 승격된 것을 확인 할 수 있고, 
데이터도 정상적으로 조회 된다. 

```bash
$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# mongo
MongoDB shell version v4.2.13
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> use test;
switched to db test
MainRepSet:PRIMARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

삭제 되었다고 다시 실행된 0번은 `Secondary` 로 강등 되었고, 모드 데이터도 정상 조회 가능하다. 

```bash
kubectl exec -it mongodb-statefulset-0 -c mongodb -- bash
root@mongodb-statefulset-0:/# mongo
MongoDB shell version v4.2.13
MainRepSet:SECONDARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:SECONDARY> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:SECONDARY> use test;
switched to db test
MainRepSet:SECONDARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

이번에는 `StatefulSet` 컨트롤러를 삭제해서 테스트를 진행해 본다. 

```bash
$ kubectl delete statefulset mongodb-statefulset
statefulset.apps "mongodb-statefulset" deleted
$ kubectl get statefulset,pod -l group=mongodb
No resources found in default namespace.
```  

`Pod` 에서 사용하던 `PV-PVC` 는 삭제 되지 않는다. 

```bash
kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                               STORAGECLASS         REASON   AGE
persistentvolume/pvc-41e7d01f-d1a1-459e-bd02-acff4ab6abf5   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-2   mongodb-storage               19m
persistentvolume/pvc-7d2c2c63-d843-449d-86c8-1aa40ef10dfc   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-1   mongodb-storage               45m
persistentvolume/pvc-b2eab001-bd08-4b48-b86c-6fd264e0c320   2Gi        RWO            Retain           Bound    default/mongodb-volumeclaim-mongodb-statefulset-0   mongodb-storage               49m

NAME                                                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-0   Bound    pvc-b2eab001-bd08-4b48-b86c-6fd264e0c320   2Gi        RWO            mongodb-storage      49m
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-1   Bound    pvc-7d2c2c63-d843-449d-86c8-1aa40ef10dfc   2Gi        RWO            mongodb-storage      45m
persistentvolumeclaim/mongodb-volumeclaim-mongodb-statefulset-2   Bound    pvc-41e7d01f-d1a1-459e-bd02-acff4ab6abf5   2Gi        RWO            mongodb-storage      19m
```  

전에 사용하던 템플릿에서 `replicas` 를 3으로 수정하고 다시 `StatefulSet` 을 `Kubernetes Cluaster` 에 적용하면, 각 파드들은 자신이 전에 사용하던 `PV-PVC` 와 바인딩 되는 것을 확인 할 수 있다.  

```bash
$ kubectl apply -f service-mongodb.yaml
service/mongodb-service unchanged
statefulset.apps/mongodb-statefulset created
$ kubectl get statefulset,pod -l group=mongodb
NAME                                   READY   AGE
statefulset.apps/mongodb-statefulset   3/3     89s

NAME                        READY   STATUS    RESTARTS   AGE
pod/mongodb-statefulset-0   1/1     Running   0          89s
pod/mongodb-statefulset-1   1/1     Running   0          87s
pod/mongodb-statefulset-2   1/1     Running   0          3s

$ kubectl get pod mongodb-statefulset-0 --output=json | jq -j '.spec.volumes[0].persistentVolumeClaim.claimName'
mongodb-volumeclaim-mongodb-statefulset-0
$ kubectl get pod mongodb-statefulset-1 --output=json | jq -j '.spec.volumes[0].persistentVolumeClaim.claimName'
mongodb-volumeclaim-mongodb-statefulset-1
$ kubectl get pod mongodb-statefulset-2 --output=json | jq -j '.spec.volumes[0].persistentVolumeClaim.claimName'
mongodb-volumeclaim-mongodb-statefulset-2
```  

테스트롤 0, 1번 파드에 접속해서 데이터를 조회하면 정상 조회되고, 
`ReplicaSet` 구성도 이전과 동일하게 0번은 `Secondary`, 1번은 `Primary` 인 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it mongodb-statefulset-0 -c mongodb -- bash
root@mongodb-statefulset-0:/# mongo
MongoDB shell version v4.2.13
MainRepSet:SECONDARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:SECONDARY> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:SECONDARY> use test;
switched to db test
MainRepSet:SECONDARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }

$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# mongo
MongoDB shell version v4.2.13
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> use test;
switched to db test
MainRepSet:PRIMARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

### ReplicaSet 삭제
삭제 테스트를 위해 `StatefulSet` 의 `replicas` 를 4로 설정하고, 
`RelicaSet` 에도 추가해준다. 

```bash
$ kubectl scale --replicas=4 statefulset mongodb-statefulset
statefulset.apps/mongodb-statefulset scaled
$ kubectl get pod -l group=mongodb
NAME                    READY   STATUS    RESTARTS   AGE
mongodb-statefulset-0   1/1     Running   0          13m
mongodb-statefulset-1   1/1     Running   0          13m
mongodb-statefulset-2   1/1     Running   0          12m
mongodb-statefulset-3   1/1     Running   0          10s

$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# mongo
MongoDB shell version v4.2.13

.. 생략 ..

MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> rs.add("mongodb-statefulset-3.mongodb-service.default.svc.cluster.local:27017")
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1617533953, 1),
                "signature" : {
                        "hash" : BinData(0,"HicMq+LEhxM5rL9538/oTHWb38M="),
                        "keyId" : NumberLong("6947241985056964611")
                }
        },
        "operationTime" : Timestamp(1617533953, 1)
}
```  

새로 추가된 3번 파드 `MongoDB` 에서도 전에 추가한 데이터는 모두 정상적으로 조회 된다. 

```bash
$ kubectl exec -it mongodb-statefulset-3 -c mongodb -- bash
root@mongodb-statefulset-3:/# mongo
MongoDB shell version v4.2.13
MainRepSet:SECONDARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:SECONDARY> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:SECONDARY> use test;
switched to db test
MainRepSet:SECONDARY> db.testcoll.find();
{ "_id" : ObjectId("606990a8d556e5449e432eab"), "a" : 1 }
{ "_id" : ObjectId("606990add556e5449e432eac"), "b" : 2 }
```  

1번 파드(`Primary`)에서 3번 파드를 `ReplicaSet` 에서 삭제 한다. 

```bash
$ kubectl exec -it mongodb-statefulset-1 -c mongodb -- bash
root@mongodb-statefulset-1:/# mongo
MongoDB shell version v4.2.13
MainRepSet:PRIMARY> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:PRIMARY> rs.remove("mongodb-statefulset-3.mongodb-service.default.svc.cluster.local:27017")
{
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1617534179, 2),
                "signature" : {
                        "hash" : BinData(0,"+Fhr6JcVy6rlnXr1unwYITW7dsk="),
                        "keyId" : NumberLong("6947241985056964611")
                }
        },
        "operationTime" : Timestamp(1617534179, 2)
}
```  

다시 3번 파드에서 `MongoDB` 상태를 조회하면 `ReplicaSet` 에 구성되지 않은 것과 
데이터도 조회되지 않는 것을 확인 할 수 있다.  

```bash
kubectl exec -it mongodb-statefulset-3 -c mongodb -- bash
root@mongodb-statefulset-3:/# mongo
MongoDB shell version v4.2.13
MainRepSet:OTHER> rs.status();
{
        "ok" : 0,
        "errmsg" : "command replSetGetStatus requires authentication",
        "code" : 13,
        "codeName" : "Unauthorized"
}
MainRepSet:OTHER> db.getSiblingDB('admin').auth("main_admin", "abc123");
1
MainRepSet:OTHER> db.getMongo().setSlaveOk()
WARNING: setSlaveOk() is deprecated and may be removed in the next major release. Please use setSecondaryOk() instead.
MainRepSet:OTHER> use test;
switched to db test
MainRepSet:OTHER> db.testcoll.find();
Error: error: {
        "ok" : 0,
        "errmsg" : "node is not in primary or recovering state",
        "code" : 13436,
        "codeName" : "NotPrimaryOrSecondary"
}
```  


---
## Reference







