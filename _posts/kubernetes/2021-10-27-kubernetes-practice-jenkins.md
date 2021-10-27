--- 
layout: single
classes: wide
title: "[Kubernetes 실습] "
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Jenkins
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
      nodePort: 35000
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

```java

```  



---
## Reference
[]()  







