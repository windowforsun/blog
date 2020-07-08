--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 오브젝트와 컨트롤러, 네임스페이스와 템플릿"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터를 구성하는 오브젝트와 컨트롤러를 구분하고 정의하는 네임스페이스와 템플릿에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Object
  - Controller
  - Namespace
  - Template
toc: true
use_math: true
---  

## 오브젝트와 컨트롤러
쿠버네티스의 시스템은 크게 오브젝트(`Object`) 와 오브젝트를 관리하는 컨트롤러(`Controller`) 로 구성된다. 
사용자는 템플릿이나 커맨드라인을 통해 바라는 상태(`desired state`) 를 정의하면,
컨트롤러는 현재 상태가 정의된 바라는 상태가 될 수 있도록 오브젝트를 생성/삭제 한다.  

오브젝트는 아래와 같은 종류가 있다.
- [파드(Pod)](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)
- [서비스(Service)](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
- [볼륨(Volume)](https://kubernetes.io/ko/docs/concepts/storage/volumes/)
- [네임스페이스(Namespace)](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/)

그리고 컨트롤러에는 아래와 같은 종류가 있다. 
- [디플로이먼트(Deployment)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)
- [데몬셋(DaemonSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/daemonset/)
- [스테이트풀셋(StatefulSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/statefulset/)
- [레플리카셋(ReplicaSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicaset/)
- [잡(Job)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/jobs-run-to-completion/)

오브젝트와 클러스터의 다른 종류에 대해서는 추후에 다루고, 
클러스터를 논리적인 단위로 구분할 수 있도록 해주는 네임스페이스와 오브젝트와 컨트롤러의 상태를 정의하는 템플릿에 대해 알아본다. 

## 네임스페이스
쿠버네티스에서 네임스페이스는 하나의 클러스터를 여러 개의 논리적인 단위로 나눠 사용할 수 있도록 해준다. 
이런 네임스페이스를 통해 클러스터 하나에 여러 앱의 구성을 동시에 사용 할 수 있다. 
그리고 클러스터에서 용도에 맞춰 실행해야 하는 앱을 구분 할 수도 있다. 
또한 네임스페이스별로 사용량을 제한 할 수도 있다.  

쿠버네티스를 처음 설치하게 되면 구성되는 기본 네임스페이스는 `kubectl get namespaces` 를 통해 확인 할 수 있다. 

```bash
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   18h
docker            Active   18h
kube-node-lease   Active   18h
kube-public       Active   18h
kube-system       Active   18h
```  

- `default` : 기본 네임스페이스로 네임스페이스를 지정하지 않게 되면 설정되는 네임스페이스이다. 
- `kube-system` : 쿠버네티스 시스템에서 사용하는 네임스페이스로 관리용 파드나 설정이 있다. 
- `kube-public` : 클러스터를 사용하는 모든 사용자가 접근할 수 있는 네임스페이스로 클러스터 사용량 정보 등을 관리한다. 
- `kube-node-lease` : 클러스터에 구성된 각 노드의 임대 오브젝트(`Lease object`) 를 관리하는 네임스페이스이다. 

`kubectl` 명령을 통해 네임스페이스를 지정할 때는 `--namespace=<네임스페이스>` 와 같이 사용한다.  

앞서 설명한 것처럼 네임스페이스를 지정하지 않으면 기본 네임스페이스(`default`) 가 설정되는데, 
기본 네임스페이스를 변경하는 방법은 우선 `kubectl config current-context` 를 통해 현재 컨텍스트 정보를 확인해야 한다. 

```bash
$ kubectl config current-context
docker-desktop
```  

현재 컨텍스트는 `docker-desktop` 인걸 확인 할 수 있다. 
컨텍스트 관련 정보는 `kubectl config get-contexts <컨텍스트이름>` 을 통해 확인 가능하다. 

```bash
$ kubectl config get-contexts docker-desktop
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```  

`NAMESPACE` 부분이 공백으로 되어 있는데, 기본 네임스페이스인 `default` 라는 의미이다. 
`kubectl config set-context <컨텍스트이름> --namespace=<기본네임스페이스>` 을 통해 기본 네임스페이스를 `kube-system` 으로 변경해 본다. 

```bash
$ kubectl config set-context docker-desktop --namespace=kube-system
Context "docker-desktop" modified.

$ kubectl config get-contexts docker-desktop
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop   kube-system
```  

아래 명령어를 통해 네임스페이스가 잘 변경 됐는지 확인해 본다. 

```bash
$ kubectl config view | grep namespace
    namespace: kube-system
```  

쿠버네티스 클러스터에서 실행 중인 전체 파드들의 기본 네임스페이스를 확인하는 방법은 
`kubectl get pods --all-namespaces` 을 통해 가능하다. 

```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
docker        compose-78f95d4f8c-vf2qp                 1/1     Running   0          19h
docker        compose-api-6ffb89dc58-8qvmz             1/1     Running   0          19h
kube-system   coredns-5644d7b6d9-qtrp2                 1/1     Running   0          19h
kube-system   coredns-5644d7b6d9-trb6q                 1/1     Running   0          19h
kube-system   etcd-docker-desktop                      1/1     Running   0          19h
kube-system   kube-apiserver-docker-desktop            1/1     Running   0          19h
kube-system   kube-controller-manager-docker-desktop   1/1     Running   0          19h
kube-system   kube-proxy-wzl67                         1/1     Running   0          19h
kube-system   kube-scheduler-docker-desktop            1/1     Running   0          19h
kube-system   storage-provisioner                      1/1     Running   0          19h
kube-system   vpnkit-controller                        1/1     Running   0          19h
```  

기본 네임스페이스 초기화를 위해 아래 명령어를 통해 다시 `default` 로 변경한다. 

```bash
$ kubectl config set-context `kubectl config current-context` --names
pace=default
Context "docker-desktop" modified.
$ kubectl config get-contexts `kubectl config current-context`
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop   default

# 혹은
$ kubectl config set-context `kubectl config current-context` --namespace=""
Context "docker-desktop" modified.
$ kubectl config get-contexts `kubectl config current-context`
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```  

### kubens
`kubens` 는 쿠버네티스 클러스터에서 네임스페이스의 변경을 더욱 간편하게 도와주는 도구이다. 
`Bash` 스크립트 기반으로 다운로드해서 권한 부여하고 바로 사용 가능하다. 

```bash
$ wget https://raw.githubusercontent.com/ahmetb/kubectx/master/kubens
--2020-06-29 22:22:07--  https://raw.githubusercontent.com/ahmetb/kubectx/master/kubens
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.88.133, 23.235.32.32, 104.156.80.32, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.88.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5555 (5.4K) [text/plain]
Saving to: ‘kubens’

kubens                    100%[===================================>]   5.42K  --.-KB/s    in 0.005s

2020-06-29 22:22:07 (1.17 MB/s) - ‘kubens’ saved [5555/5555]

$ alias kubens='<경로>/kubens'
$ kubens
default
docker
kube-node-lease
kube-public
kube-system
$ kubens --help
USAGE:
  kubens                    : list the namespaces in the current context
  kubens <NAME>             : change the active namespace of current context
  kubens -                  : switch to the previous namespace in this context
  kubens -c, --current      : show the current namespace
  kubens -h,--help          : show this message
```  

기본 네임스페이스를 변경할 때는 `kubens <네임스페이스>` 를 사용한다. 

```bash
$ kubens kube-system
Context "docker-desktop" modified.
Active namespace is "kube-system".
```  

> 추가로 `kubens` 가 있는 `kubectx` 저장소는 `kubeconfig` 에 저장된 클러스터가 여러개 일 떄,
>클러스터를 빠르게 변경 할 수 있도록 도와주는 도구이다. 


## 템플릿
쿠버네티스 클러스터에서 오브젝트, 컨트롤러의 상태를 정의할 때는 `YAML` 포맷의 템플릿(`Template`) 을 사용한다.  

`YAML` 파일 형식은 아래와 같은 형식으로 표현한다. 
- `Scalars(strings/numbers)`
    
    ```yaml
    Str: hello
    num: 2020
    ```  
  
- `Sequences(array/lists)`
    
    ```yaml
    list:
      - keyboard
      - mouse
      - monitor
    ```  
  
- `Mapping(hashes/dictionaries)`

    ```yaml
    data:
      one: 1
      two: 2
      three: 3
    ```  

주석은 `#` 문자로 시작하고, `---` 은 성격이 다른 문서가 한 파일에 여러개 존재할 때 구분자로 사용한다. 

```yaml
# 주석
profile: dev

---
profile: prod
```  

쿠버네티스 클러스터 상태를 정의하는 `YAML` 의 기본 템플릿은 아래와 같다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
spec:
```  

- `apiVersion` : 사용하는 쿠버네티스 API 버전을 명시한다. 
여러 버전이 있기 때문에 사용하려는 버전을 정확하게 작성해야 한다. 
`kube api-versions` 를 통해 현재 클러스터에서 사용가능한 버전을 확인 할 수 있다. 
- `kind` : 오브젝트, 컨트롤러의 종류를 명시한다. 
`Pod` 라는 작성돼 있으면 파드를 정의하는 템플릿을 의미한다. 
`Pod`, `Deployment`, `Ingress` 등 다양한 종류가 있다. 
- `metadata` : 정의하는 오브젝트나 컨트롤러의 이름, 레이블과 같은 메타데이터를 설정한다. 
- `spec` : 파드로 예를 들면 어떤 컨테이너를 실행하고, 구체적인 동작을 설정한다. 

대략적인 템플릿 구조에 대해서 설명 했지만, 
쿠버네티스 클러스터를 설정하기 위한 템플릿의 구조는 다양한 것들이 있다.
이는 `kubectl explain <필드>` 명령을 통해 더욱 자세히 확인 가능하다. 

```bash
$ kubectl explain pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status       <Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```  

위 출력결과에서 `metadata`, `spec` 등은 타입이 `Object` 인 것을 확인 할 수 있다. 
`Object` 일 경우 다시 `kubectl explain <상위필드명>.<하위필드명>` 을 통해 하위 필드에 대한 정보를 확인 할 수 있다. 

```bash
$ kubectl explain pods.metadata
KIND:     Pod
VERSION:  v1

RESOURCE: metadata <Object>

DESCRIPTION:
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

     ObjectMeta is metadata that all persisted resources must have, which
     includes all objects users must create.

FIELDS:
   annotations  <map[string]string>
     Annotations is an unstructured key value map stored with a resource that
     may be set by external tools to store and retrieve arbitrary metadata. They
     are not queryable and should be preserved when modifying objects. More
     info: http://kubernetes.io/docs/user-guide/annotations

   clusterName  <string>
     The name of the cluster which the object belongs to. This is used to
     distinguish resources with same name and namespace in different clusters.
     This field is not set anywhere right now and apiserver is going to ignore
     it if set in create or update request.

.. 생략 ..

   selfLink     <string>
     SelfLink is a URL representing this object. Populated by the system.
     Read-only. DEPRECATED Kubernetes will stop propagating this field in 1.20
     release and the field is planned to be removed in 1.21 release.

   uid  <string>
     UID is the unique in time and space value for this object. It is typically
     generated by the server on successful creation of a resource and is not
     allowed to change on PUT operations. Populated by the system. Read-only.
     More info: http://kubernetes.io/docs/user-guide/identifiers#uids
```  

상위 필드가 있는 하위 필드를 표현할 때는 `.` 으로 연결해 `<상위 필드>.<하위필드>` 와 같이 표현한다.  

한 필드에 포함되는 모든 하위 필드까지 다 확인하고 싶다면 `kubectl explain <필드명> --recursive` 를 통해 확인 할 수 있다.

```bash
$ kubectl explain pods --recursive
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
      annotations       <map[string]string>
      clusterName       <string>
      creationTimestamp <string>
      deletionGracePeriodSeconds        <integer>

.. 생략 ..

   spec <Object>
      activeDeadlineSeconds     <integer>
      affinity  <Object>
         nodeAffinity   <Object>
            preferredDuringSchedulingIgnoredDuringExecution     <[]Object>
               preference       <Object>

.. 생략 ..

      automountServiceAccountToken      <boolean>
      containers        <[]Object>
         args   <[]string>
         command        <[]string>
         env    <[]Object>

.. 생략 ..

   status       <Object>
      conditions        <[]Object>
         lastProbeTime  <string>
         lastTransitionTime     <string>
         message        <string>
         reason <string>
         status <string>
         type   <string>

.. 생략 ..

      qosClass  <string>
      reason    <string>
      startTime <string>
```  

---
## Reference
