--- 
layout: single
classes: wide
title: "[Kubernetes 실습] StatefulSet Volume Mount"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'StatefulSet 을 사용해서 상태를 유지할 수 있는 파드에 볼륨을 마운트해 영구 저장소를 사용해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - StatefulSet
  - Volume
  - Storageclass
toc: true
use_math: true
---  
## 환경
- `Docker desktop for Windows` 
    - `docker v20.10.2` 
    - `kubernetes v1.19.3`

## StatefulSet
[여기]({{site.baseurl}}{% link _posts/kubernetes/2020-07-07-kubernetes-concept-controller-deamonset-statefulset.md %})
에서 설명했던것 같이, `StatefulSet` 은 상태가 있는 파드를 관리하는 컨트롤러이다. 
이러한 특징으로 대부분 영구적인 데이터 저장이 필요한 `MySQL`, `MongoDB`, `Redis`, `ElasticSearch` 
구성에 사용된다. `StatefulSet` 을 사용해서 영구적인 데이터 저장을 할 수 있도록 볼륨을 마운트 하는 방법에 대해 알아본다.  

`StatefulSet` 에서 관리하는 파드는 `Deployment` 와 달리 이름에 규칙성을 갖는다. 
만약 `StatefulSet` 이름이 `test-statefulset` 이고, `replica` 가 3개라면 아래와 같다. 
- `test-statefulset-0`
- `test-statefulset-1`
- `test-statefulset-2`

위 처럼 이름에 규칙성도 존재하기 때문에, 파드를 생성할때 한번에 생성하지 않고 `0 ~2` 까지 순서대로 생성한다. 
즉 이러한 특징으로 데이터 저장소를 구성할 때 `Master-Slave` 구조와 같은 상황에서도 유용하게 사용할 수 있다.  

또한 이번 포스트에서 주로 다룰 파드와 볼륨 관리에 있어서도 `Pod-PVC-PV` 가 1:1 관계를 가지면서 생성되고 관리된다. 
설정에 따라 실제 동작이 달라질 순 있겠지만, `StatefulSet` 의 파드가 삭제되더라도 `PV-PVC` 는 삭제되지 않기 때문에 
데이터를 지속적으로 저장 할 수 있다. 다시 `Pod` 가 살아 난다면 저에 사용하던 데이터 그대로 사용할 수 있다.  

## 예제
예제에서는 `StorageClass`, `Service`, `StatefulSet` 템플릿을 사용해서 예제를 진행한다. 

### StorageClass
`PersistentVolume` 을 동적으로 프로비저닝 할 수 있는 `StorageClass` 를 정의하는 템플릿은 아래와 같다. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: test-storage-class
provisioner: docker.io/hostpath
reclaimPolicy: Retain
```  

> `node` 의 호스트 이름이 `docker-desktop` 인 노드를 선택해서 사용하는데, 
>이때 현재 `WSL` + `docker desktop` 환경이라서 경로에 대한 부가적인 설명은 아래와 같다. 
>
>구분|경로
>---|---
>Template(yaml)|`/run/desktop/mnt/host/c/k8s/test-storage-class`
>WSL|`/c/k8s/test-storage-class`
>Windows|`C:\k8s\test-storage-class`

### Service, StatefulSet
다음으로 `Nginx` 이미지를 사용해서 간단하게 구성한 `StatefulSet` 템플릿은 아래와 같다. 
`.spec.volumeClaimTemplates.spec.storageClassName` 에 앞서 생성한 `StorageClass` 를 지정해 주었다. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
      targetPort: 80
      protocol: TCP
      nodePort: 30080
  selector:
    app: nginx
  type: NodePort
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: nginx-service
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: nginx
          image: k8s.gcr.io/nginx-slim:0.8
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: html-volume
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: test-storage-class
        resources:
          requests:
            storage: 1Gi
```  

### 적용
앞서 나열한 모든 템플릿을 `Kubernetes Cluster` 에 적용하고 생성된 구성을 살펴보면 아래와 같다. 

```bash
$ kubectl apply -f test-storage-class.yaml
storageclass.storage.k8s.io/test-storage-class created
$ kubectl apply -f nginx-service-statefulset.yaml
service/nginx-service created
statefulset.apps/nginx-statefulset created
```  

생성한 모든 오브젝트와 컨트롤러를 조회하면 아래와 같다.  

```bash
$ kubectl get sc,svc,statefulset,pod -l group=test-statefulset
NAME                                             PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/test-storage-class   docker.io/hostpath   Retain          Immediate           false                  6m18s

NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/nginx-service   NodePort   10.110.79.79   <none>        80:30080/TCP   14s

NAME                                 READY   AGE
statefulset.apps/nginx-statefulset   3/3     14s

NAME                      READY   STATUS    RESTARTS   AGE
pod/nginx-statefulset-0   1/1     Running   0          4s
pod/nginx-statefulset-1   1/1     Running   0          3s
pod/nginx-statefulset-2   1/1     Running   0          2s
```  

`StatefulSet` 의 `Pod` 에서 사용하는 `PV` 와 `PVC` 를 조회하면 아래와 같다. 

```bash
kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                     STORAGECLASS         REASON   AGE
persistentvolume/pvc-146ef4c5-cd7a-4bbc-8c9b-57ffa6a92d6e   1Gi        RWO            Retain           Bound    default/html-volume-nginx-statefulset-0   test-storage-class            7m29s
persistentvolume/pvc-1e893902-7932-47d2-bf3e-cfc8dadb4175   1Gi        RWO            Retain           Bound    default/html-volume-nginx-statefulset-2   test-storage-class            7m21s
persistentvolume/pvc-f23f5712-5996-4804-80c3-47f2de472bd2   1Gi        RWO            Retain           Bound    default/html-volume-nginx-statefulset-1   test-storage-class            7m25s

NAME                                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
persistentvolumeclaim/html-volume-nginx-statefulset-0   Bound    pvc-146ef4c5-cd7a-4bbc-8c9b-57ffa6a92d6e   1Gi        RWO            test-storage-class   7m29s
persistentvolumeclaim/html-volume-nginx-statefulset-1   Bound    pvc-f23f5712-5996-4804-80c3-47f2de472bd2   1Gi        RWO            test-storage-class   7m25s
persistentvolumeclaim/html-volume-nginx-statefulset-2   Bound    pvc-1e893902-7932-47d2-bf3e-cfc8dadb4175   1Gi        RWO            test-storage-class   7m21s
```  

`PV`, `PVC` 모두 `StatefulSet` 의 `Pod` 쌍에 맞게 생성되고 마운트 된 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice_statefulset_volume_mount_1.png)  

### 테스트
테스트를 위해 `pod/nginx-statefulset-0 ~ 2` 까지 파드에 접속해 마운트 경로에 아래와 같이 파일을 생성해 준다. 

```bash
.. 0 ~ 2 까지 반복 ..
$ kubectl exec -it nginx-statefulset-0 -c nginx -- bash
root@nginx-statefulset-0:/# echo "im 0" > /usr/share/nginx/html/whoami.html


$ kubectl exec -it nginx-statefulset-1 -c nginx -- bash
root@nginx-statefulset-1:/# echo "im 1" > /usr/share/nginx/html/whoami.html


$ kubectl exec -it nginx-statefulset-2 -c nginx -- bash
root@nginx-statefulset-2:/# echo "im 2" > /usr/share/nginx/html/whoami.html
```  

호스트에서 `NodePort` 로 열어둔 포트로 `curl` 요청 혹은 브라우저에서 접속하면 아래와 같은 결과를 얻을 수 있다. 

```bash
$ curl localhost:30080/whoami.html
im 0
$ curl localhost:30080/whoami.html
im 1
$ curl localhost:30080/whoami.html
im 2
```  

다음으로 `StatefulSet` 의 `replicas` 를 조정해서 1, 2 파드를 삭제한다.  

```bash
$ kubectl scale --replicas=1 statefulset nginx-statefulset
statefulset.apps/nginx-statefulset scaled
kubectl get statefulset,pod -l group=test-statefulset
NAME                                 READY   AGE
statefulset.apps/nginx-statefulset   1/1     30m

NAME                      READY   STATUS    RESTARTS   AGE
pod/nginx-statefulset-0   1/1     Running   0          30m
```  

스케일 조정으로 인해 0번 파드만 실행중이고, 1 ~ 2번 파드는 종료되었다. 
이상태에서 요청을 하면 아래와 같이 0번 파드에서만 요청을 받게 된다. 

```bash
$ curl localhost:30080/whoami.html
im 0
$ curl localhost:30080/whoami.html
im 0
$ curl localhost:30080/whoami.html
im 0
```  

현재 상태를 도식화 하면 아래와 같은 상황이다. 

![그림 1]({{site.baseurl}}/img/kubernetes/practice_statefulset_volume_mount_2.png)  

다시 `StatefulSet` 의 `replicas` 를 3으로 조정해 1, 2번 파드를 실행하면 아래와 같다. 

```bash
$ kubectl scale --replicas=3 statefulset nginx-statefulset
statefulset.apps/nginx-statefulset scaled
$ kubectl get statefulset,pod -l group=test-statefulset
NAME                                 READY   AGE
statefulset.apps/nginx-statefulset   3/3     48m

NAME                      READY   STATUS    RESTARTS   AGE
pod/nginx-statefulset-0   1/1     Running   0          48m
pod/nginx-statefulset-1   1/1     Running   0          18s
pod/nginx-statefulset-2   1/1     Running   0          16s
$ curl localhost:30080/whoami.html
im 0
$ curl localhost:30080/whoami.html
im 1
$ curl localhost:30080/whoami.html
im 2
```  

실제로 각 파드에 접속해 파일을 확인해도 삭제된 파드에서 이전에 생성한 파일이 존재하는 것을 확인 할 수 있다.  

```bash
$ kubectl exec -it nginx-statefulset-1 -c n
ginx -- bash
root@nginx-statefulset-1:/# cat /usr/share/nginx/html/whoami.html
im 1

$ kubectl exec -it nginx-statefulset-2 -c n
ginx -- bash
root@nginx-statefulset-2:/# cat /usr/share/nginx/html/whoami.html
im 2
```  


---
## Reference
[Docker for mac Kubernetes StorageClass provisioner](https://github.com/docker/for-mac/issues/2460)  








