--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Kubernetes Jenkins Master Slave(Agent) 구성 및 사용하기"
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
  - Jenkins Agent
toc: true
use_math: true
---  

## 환경
- `K3S v1.21.4+k3s1 (3e250fdb)`
- `Kubernetes v1.21.4`
- `Docker 19.03.15`

## Jenkins Master/Slave(Agent)
본 포스트는 [ Kubernetes Jenkins 설치(StatefulSet)]({{site.baseurl}}{% link _posts/kubernetes/2021-10-27-kubernetes-practice-jenkins.md %})
에 이어서 `Kubernetes` 클러스터를 사용해서 분산 빌드환경인 `Master/Slave` 구조를 구성하는 방법에 대해 알아 본다.  

분산 빌드 환경을 구성하지 않더라도 단일 노드 `Jenkins` 를 사용해서 빌드나 배치 작업은 수행할 수 있다. 
하지만 단일 노드를 사용하게 되면 작업이 많아질수록 부하도 함께 늘어난다. 
이때 `Master/Slave` 구조를 통해 분산 환경을 구성하게 되면 작업을 여러 노드로 분산해서 수행할 수 있기 때문에 
동시성도 늘어날 뿐만아니라, 부하 분산에도 큰 도움이 된다. 
그리고 특정 노드에서만 수행할 수 있는 작업이 있을 때도 해당 노드에 `Agent` 를 구성해서 작업을 수행할 수 있다.  

`Master/Slave` 구조를 만들게 되면 `Master` 는 작업 등록 및 관리, GUI 제공을 담당하고 `Slave` 가 실제 작업을 수행하는 역할을 하게 된다.  


### Kubernetes 추가 설정
`Master` 가 사용할 `Slave` 노드를 동적으로 필요할 때마다 실행하는 구조로 만들 계획이므로, 
`Master` 가 `Kubernetes` 클러스터에 필요한 권한을 아래 템플릿을 사용해서 설정해 준다. 

```yaml
# jenkins-rbac.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
  namespace: jenkins
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
```  

아래 명령어로 위 템플릿을 `Kubernetes` 에 적용해 준다.  

```bash
$ kubectl get serviceaccount,rolebinding -n jenkins
NAME                     SECRETS   AGE
serviceaccount/default   1         2d19h
serviceaccount/jenkins   1         24s

NAME                                            ROLE           AGE
rolebinding.rbac.authorization.k8s.io/jenkins   Role/jenkins   24s
```  

### Jenkins 플러그인 추가
`Kubernetes` 클러스터에 `Master/Slave` 를 적용하기 위해서는 아래와 같은 플러그인이 필요하다. 

- Kubernetes

Jenkins 관리 > 플러그인 관리 > 설치 가능

practice-jenkins-master-slave-1.png


Jenkins 관리 > 시스템 설정 > Cloud(가장 아래쪽)








---
## Reference
[HOW TO INSTALL JENKINS ON A KUBERNETES CLUSTER](https://admintuts.net/server-admin/how-to-install-jenkins-on-a-kubernetes-cluster/)  







