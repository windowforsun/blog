--- 
layout: single
classes: wide
title: "[Kubernetes 개념] "
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
toc: true
use_math: true
---  

## 볼륨
파드의 컨테이너 안에 저장하는 데이터는 컨테이너가 종료되는 시점에 함께 삭제된다. 
컨테이너에서 저장한 데이터를 유지하기 위해서는 외부에 데이터를 저장해야 한다. 
이렇게 컨테이너 데이터를 외부에 저장해서 데이터를 보전할 수 있도록 하는 기능을 볼륨(`Volume`) 이라고 한다.  

컨테이너는 상태가 없는(`Stateless`) 성격을 띄는 애플리케이션을 실행하는 용도로 주로 사용된다. 
이렇게 상태가 없는 애플리케이션은 문제가 발생하거나, 장애 등 상황에 컨테이너를 재시작하거나 노드를 옮기는 동작에 큰 제약이 없다. 
하지만 상태가 없다는 성격을 가진 만큼 컨테이너가 동작하며 생성한 데이터는 모두 사라지게 된다.  

컨테이너는 데이터를 보존해야 되는 애플리케이션을 실행하는 용도로도 사용 될 수 있다. 
대표적인 예로 데이터를 파일로 저장하는 `Jenkins` 나 `MySQL` 와 같은 데이터베이스가 있다. 
이러한 애플리케이션은 특정 상황으로 컨테이너가 재시작되더라도 컨테이너에서 저장한 데이터를 유지해야 한다.  

위와 같은 상황에서 볼륨을 사용하면 컨테이너가 재시작되는 상황에서도 데이터를 유지할 수 있다. 
그리고 이후에 다뤄볼 퍼시스턴트 볼륨을 사용하면 데이터가 저장된 노드가 아닌 다른 노드에서 컨테이너가 실행되더라도, 
저장된 볼륨 데이터를 그대로 사용할 수 있다. 
이렇게 볼륨, 퍼시스턴트 볼륨을 사용하면 보다 안정적으로 데이터 유지가 필요한 서비스를 운영할 수 있다.  

쿠버네티스에서 사용할 수 있는 볼륨 플러그인에는 아래와 같은 종류가 있다. 
- `awsElasticBlockStore`
- `azureDisk`
- `cephfs`
- `configMap`
- `csi`
- `downwardAPI`
- `emptyDir`
- `fc`(fibre channel)
- `flcoker`
- `hostPath`
- `gcePersistentDisk`
- `gitRepo`(deprecated)
- `glusterfs`
- `iscsi`
- `local`
- `nfs`
- `persistentVolumeCalim`
- `projected`
- `portworxVolume`
- `quobyte`
- `rbd`
- `scaleIO`
- `secret`
- `storageos`
- `vsphereVolume`

`aws`, `azure`, `gce` 로 시작하는 볼륨 플러그인은 해당하는 클라우드 서비스에서 제공하는 볼륨 서비스이다. 
그외 `glusterfx`, `cephfs` 는 오픈소스 스토리지 서비스이고, 
`configMap`, `secret` 은 쿠버네티스 내부 오브젝트이다. 
그리고 `emptyDir`, `hostPath`, `local` 은 컨테이너가 실행된 노드의 디스크를 볼륨으로 사용하는 옵션이다. 
`nfs` 은 컨테이너 하나를 `NFS` 서버로 구성하고 다른 컨테이너에서 `NFS` 서버 컨테이너를 사용하도록 설정 할 수 있다. 
이렇게 `nfs` 를 사용하면 볼륨에서 자체적으로 멀티 읽기/쓰기를 지원하지 않더라도 비슷한 효과를 낼 수 있다.  

볼륨관련 `.spec.container.volumeMounts.mountPropagation` 필드는 동일한 파드에서 실행되는 컨테이너들간, 
혹은 동일 노드에 실행되는 파드간 볼륨을 공유해서 사용할지 아래와 같은 설정 값으로 설정할 수 있다. 
- `None` : 호스트에서 마운트한 볼륨 디렉토리 하위에 마운트된 다른 마운트를 볼 수 없다. 
컨테이너가 만든 마운트를 호스트에서도 볼 수 없다. (default)
- `HostToContainer` : 호스트에서 마운트한 볼륨 디렉토리 하위에 마운트된 다른 마운트를 해당 볼륨에서 볼 수 있다. 
- `Bidirectional` : `HostToConatiner` 와 같이 하위에 마운트된 디렉토리를 볼 수 있고, 
호스트에서 실행되는 파드, 컨테이너간 같은 볼륨을 사용할 수 있다. 

앞서 나열한 다양한 볼륨 관련 플러그인들 중, 
호스트 내부에서 사용하는 `emptyDir`, `hostPath` 와 `nfs` 를 사용해서 볼륨 하나를 여러 컨테이너에서 공유하는 방법에 대해 살펴 본다. 

### emptyDir
`emptyDir` 은 파드가 실행되는 호스트의 디스크에 임시로 컨테이너에서 사용하는 볼륨을 할당해서 사용한다. 
`emptyDir` 은 파드가 종료되면 마운트 된 볼륨의 데이터도 함께 사라지지만, 컨테이너가 종료되는 경우에는 사라지지 않는다. 
이러한 특성으로 메모리와 디스크를 함께 사용하는 대용량 데이터 계산이나, 
시간이 오래걸리는 상황에서 중간 데이터 저장용 등에 사용하기에 적합하다. 

`emptyDir` 을 적용한 파드 템플릿 예시는 아래와 같다. 

```yaml
# pod-emptydir.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-emptydir
spec:
  volumes:
    - name: emptydir-volume
      emptyDir: {}
  containers:
    - name: test
      image: nginx:latest
      volumeMounts:
        - mountPath: /emptydir
          name: emptydir-volume
```  

- `.spec.volumes[]` : 필드를 사용해서 사용할 볼륨을 선언한다. 
`.name` 필드 값으로 `emptydir-volume` 이름을 설정하고, 
`.emptyDir` 키로 `emptyDir` 을 사용하기 위한 필드 값으로 `{}` 빈값을 설정했다. 
- `.spec.containers[].name.volumeMounts[]` : `.mountPath` 필드에 `/emptydir` 컨테이너 디렉토리를 설정하고, 
앞서 선언한 `emptyDir` 볼륨의 이름을 `.name` 필드에 설정해서 마운트를 수행한다. 

파트 템플릿에서 알 수 있듯이 쿠버네티스에서 볼륨은 선언 부분과 컨테이너에 실제로 설정하는 부분을 분리해서 구성된다.  


### hostPath

 




---
## Reference
