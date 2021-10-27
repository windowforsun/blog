--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Kubernetes Jenkins 설치(StatefulSet)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Kubernetes 클러스터에서 StatefulSet 을 사용해서 Jenkins 를 구성해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Jenkins
  - StatefulSet
toc: true
use_math: true
---  

## 환경
- `K3S v1.21.4+k3s1 (3e250fdb)`
- `Kubernetes v1.21.4`
- `Docker 19.03.15`

## Kubernetes Jenkins 구성하기
`CI/CD` 혹은 `Batch` 툴로 범용적으로 사용하는 `Jenkins` 를 `Kubernetes Statefulset` 을 사용해서 
구성하는 방법에 대해 알아본다.  

`Jenkins` 를 구성하는 `Statefulset` 과 `Service` 에 대한 템플릿 내용은 아래와 같다.  

```yaml
# jenkins-master.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
  namespace: jenkins
spec:
  updateStrategy:
    type: RollingUpdate
  serviceName: jenkins
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
      namespace: jenkins
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: jenkins-master
                    operator: In
                    values:
                      - "true"
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:2.317-jdk11
          ports:
            - name: http-port
              containerPort: 8080
            - name: jnlp-port
              containerPort: 50000
          volumeMounts:
            - name: jenkins-volumeclaim
              mountPath: /var/jenkins_home
  volumeClaimTemplates:
    - metadata:
        name: jenkins-volumeclaim
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: jenkins-storage
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
#      nodePort: 35000
  selector:
    app: jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
  namespace: jenkins
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
```  

`Jenkins` 는 `nodeAffinity` 에 의해 `jenkins-master` 라는 레이블의 값이 `true` 인 노드에 실행된다. 
그리고 해당 노드의 로컬경로에 마운트를 수행해서 빌드 및 배치 이력이 유지될 수있도록 한다.  

`Jenkins` 에 접속하는 포트인 `8080` 은 `NodePort` 를 사용해서 외부에서 접속할 수 있다. 
그리고 `Kubernetes` 클러스터에서 실행될 `Remove Agent(Slave)` 는 `ClusterIP` 를 통해 클러스터 내부에서만 `50000` 을 사용해서 접근할 수 있다.

다음으로는 `Jenkins` 에 볼륨을 마운트 할때 사용하는 `StorageClass` 와 `PV` 템플릿 내용이다.  

```yaml
# jenkins-storageclass.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: jenkins-storage
    namespace: jenkins
provisioner: docker.io/hostpath
allowVolumeExpansion: true
#reclaimPolicy: Retain
reclaimPolicy: Delete
```  

```yaml
# jenkins-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-master-pv
  namespace: jenkins
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: jenkins-storage
  local:
    path: /data/k8s-volume/jenkins
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                # hostname
                - node-03
```  

`Jenkins` 는 `node-03` 노드에 실행 시킬 예정이므로 명령어를 사용해서 `jenkins-master=true` 레이블을 `node-03` 에 설정해 준다.  

```bash
$ kubectl label nodes node-03 jenkins-master=true
```  

그리고 `jenkins` 네임스페이스도 생성해 준다.  

```bash
$ kubectl create ns jenkins
```  

아래 명령으로 작성한 템플릿을 클러스터에 적용해 준다.  

```bash
$ kubectl apply -f jenkins-master.yaml
$ kubectl apply -f jenkins-storageclass.yaml
$ kubectl apply -f jenkins-pv.yaml

$ kubectl get all -n jenkins
NAME            READY   STATUS    RESTARTS   AGE
pod/jenkins-0   0/1     Pending   0          29s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/jenkins        NodePort    10.43.224.102   <none>        8080:30983/TCP   29s
service/jenkins-jnlp   ClusterIP   10.43.104.80    <none>        50000/TCP        29s

NAME                       READY   AGE
statefulset.apps/jenkins   0/1     29s
```  

지금은 `Pod` 에 마운트할 경로가 존재하지 않아 `Pending` 상태이다. 
아래 명령어로 `Pod` 가 실행되는 노드에 아래 명령어로 마운트할 경로를 생성해 준다.  

```bash
$ mkdir -p /data/k8s-volume/jenkins
```  

생성 후 다시 클러스터의 리소스를 조회하면 아래와 같다.  

```bash
$ kubectl get all -n jenkins
NAME            READY   STATUS    RESTARTS   AGE
pod/jenkins-0   1/1     Running   0          2m44s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/jenkins        NodePort    10.43.224.102   <none>        8080:30983/TCP   2m44s
service/jenkins-jnlp   ClusterIP   10.43.104.80    <none>        50000/TCP        2m44s

NAME                       READY   AGE
statefulset.apps/jenkins   1/1     2m44s
$ kubectl get pv,pvc -n jenkins | grep jenkins
persistentvolume/jenkins-master-pv                     20Gi       RWO            Retain           Bound    jenkins/jenkins-volumeclaim-jenkins-0                            jenkins-storage            3m45s
persistentvolumeclaim/jenkins-volumeclaim-jenkins-0   Bound    jenkins-master-pv   20Gi       RWO            jenkins-storage   8m26s
```  

외부에서 `Jenkins` 웹에 접근할 수 있는 포트는 `30983` 으로 설정 된것을 확인 할 수 있다. 
아래 `URL` 로 접근하면 아래와 같은 페이지를 확인할 수 있다.  

```
http://<node-ip>:30983
```

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-1.png)  

초기 관리자 비밀번호는 아래 명령어로 확인할 수 있다.  

```bash
$ kubectl exec -it -n jenkins jenkins-0 -- cat /var/jenkins_home/secrets/initialAdminPassword
55f04d717eaa4f7b89a6ee2f5cfcac48
```  

위 패스워드로 접근하면 아래 화면을 볼수 있고 `Install suggesed plugins` 를 눌러 초기 설정을 마무리 해준다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-2.png)  

완료되면 사용할 계정을 만들어 준다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-3.png)  

모든 설정이 완료되면 아래와 같이 `Jenkins` 페이지을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-4.png)


---
## Reference
[HOW TO INSTALL JENKINS ON A KUBERNETES CLUSTER](https://admintuts.net/server-admin/how-to-install-jenkins-on-a-kubernetes-cluster/)  







