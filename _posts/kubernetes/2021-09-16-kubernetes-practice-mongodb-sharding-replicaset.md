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

## MongoDB Sharding + ReplicaSet
[Kubernetes MongoDB Replicaset]({{site.baseurl}}{% link _posts/kubernetes/2021-04-04-kubernetes-practice-statefulset-mongodb.md %})
에서 `Kubernetes` 의 `StatefulSet` 을 사용해서 `MongoDB ReplicaSet` 을 구성하는 방법에 대해서 알아 보았다. 
이번에는 `MongoDB Sharding` 까지 추가해서 아래와 같은 구조로 구성하는 방법에 대해 알아 본다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-mongodb-sharding-replicaset-1drawio.drawio.png)  

사용할 파일의 구성은 아래와 같다. 
각 파일의 역할과 설정 방법에 대해서는 아래 부분에서 차례대로 살펴보도록 한다.  

```
.
├── key.txt
├── mongodb-configserver-statefulset.yaml
├── mongodb-mongos-statefulset.yaml
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
```  

### StorageClass 생성
`MongoDB` 는 모두 `StatefulSet` 을 사용할 예정이므로 데이터를 영구적으로 저장할 공간을 할당 받을 수 있는 `StorageClass` 를 생성해 준다. 
빠른 테스트를 위해 `reliclimPolicy` 는 `Delete` 로 설정해서 삭제시 저장 공간도 함께 삭제될 수 있도록 설정했다.  

```yaml
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

### PV ???

### StatefulSet ConfigServer(ReplicaSet)

### StatefulSet MongoDB Shard(ReplicaSet)

### StatefulSet Mongos

### Sharding Test

### Shard 추가 

### Shard 삭제





---
## Reference
[Sharding in MongoDB](https://www.mongodb.com/basics/sharding)  
[Sharded Mongodb in Kubernetes StatefulSets on GKE](https://medium.com/google-cloud/sharded-mongodb-in-kubernetes-statefulsets-on-gke-ba08c7c0c0b0)  
[pkdone/gke-mongodb-shards-demo](https://github.com/pkdone/gke-mongodb-shards-demo)  








