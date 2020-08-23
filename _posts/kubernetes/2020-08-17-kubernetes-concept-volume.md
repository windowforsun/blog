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
`hostPath` 는 파드가 실행된 호스트에 파일이나 디렉토리를 마운트해서 사용하는 볼륨을 의미한다. 
`emptyDir` 은 임시 디렉토리인 것에 비해, 
`hostPath` 은 호스트의 실제 파일이나 디렉토리를 마운트한다. 
이런 특징으로 `emptyDir` 은 파드 재시작시점에 모든 데이터가 삭제 됐다면, 
`hostPath` 는 파드 재시작을 하더라도 마운트된 볼륨에 데이터는 유지된다.  

`/var/lib/docker` 와 같은 호스트 도커 시스템 디렉토리를 컨테이너에 마운트해서 사용하거나, 
시스템용 디렉토리르 마운트해서 시스템을 모티터링 하는 용도로도 사용할 수 있다.  

아래는 `hostPath` 을 적용한 파드 템플릿의 예시이다. 

```yaml
# pod-hostpath.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
spec:
  containers:
    - name: test
      image: nginx
      volumeMounts:
        - mountPath: /test-hostpath
          name: hostpath-volume
      ports:
        - containerPort: 80
  volumes:
    - name: hostpath-volume
      hostPath:
        path: /var/lib/testdir
        type: Directory
```  

- `.spec.containers[].volumeMounts[]` : `.mountPath` 필드에는 볼륨을 마운트할 컨테이너 경로를 설정한다. 
그리고 `.name` 은 `.spec.volumes[]` 에 설정한 볼륨 이름을 설정해 준다. 
- `.spec.volumes[]` : `.name` 필드에는 볼륨의 이름을 설정하고, 
`.hostPath` 필드를 사용해서 호스트 경로에 대한 볼륨 설정을 한다. 
`.hostPath.path` 는 볼륨을 마운트할 호스트 경로를 설정하고, 
`.hostPath.type` 에는 디렉토리임을 명시한다. 

>현재 docker desktop 이 `wsl2` 환경에서 구동중인 환경이다. 
>그러므로 쿠버네티스 클러스터에서 실행되는 컨테이너와 호스트 경로를 마운트하기 위해서는 아래와 같은 경로를 사용한다.  
>  
>설정 Path|docker-desktop Distro 경로
>---|---
>/var/lib/<dir-or-file>|/mnt/wsl/docker-desktop-data/data/<dir-or-file>
>/var/lib/docker/volumes|/mnt/wsl/docker-desktop-data/data/docker/volumes

`.spec.volums[].hostPath.type` 은 `hostPath` 경로를 설정할 때 경로의 타입을 설정하는 필드이다. 
해당 필드에 사용가능한 값은 아래와 같다. 

Type|설명
---|---
none|볼륨을 마운트할때 확인 절차를 수행하지 않는다.
DirectoryOrCreate|설정한 디렉토리 경로가 없으면 퍼미션 `755` 인 디렉토리를 생성한다.
Directory|설정한 디렉토리 경로가 존재해야 한다. 존재하지 않으면 파드는 `ContainerCreating` 상태로 디렉토리 경로가 생성될때까지 대기한다. 
FileOrCreate|설정한 경로에 파일이 없으면 퍼미션 `644` 인 파일을 생성한다. 
File|설정한 경로에 파일이 존재해야 한다. 존재하지 않으면 파드는 생성되지 않는다.
Socket|설정한 경로에 유닉스 소켓 파일이 존재해야 한다.
CharDevice|설정한 경로에 문자 디바이스가 존재해야 한다.
BlockDevice|설정한 경로에 블록 디바이스가 존재해야 한다.

`kubectl apply -f pod-hostpath.yaml` 명령으로 구성한 템플릿을 클러스터에 적용한다. 
그리고 `kubectl exec <파드이름> -it sh` 명령으로 컨테이너에 접속하고, 
컨테이너 경로인 `/test-hostpath` 에 새로운 파일 하나를 생성한다. 

```bash
$ kubectl apply -f pod-hostpath.yaml
pod/pod-hostpath created
$ kubectl exec pod-hostpath -it sh
# cd test-hostpath
# touch thisisfile
# ls
thisisfile
# exit
```  

그리고 호스트의 경로인 `/var/lib/testdir`(`/mnt/wsl/docker-desktop-data/data/testdir`) 로 이동해서 
파일을 조회하면 아래와 같이 컨테이너에서 생성한 파일이 동일하게 존재하는 것을 확인 할 수 있다. 

```bash
$ cd /mnt/wsl/docker-desktop-data/data/testdir/
$ ls
thisisfile
```  

### nfs
`nfs`(`Network File System`) 볼륨은 네트워크를 사용하는 파일 시스템을 볼륨으로 마운트하는 것을 의미힌다. 
설정하게 되면 볼륨은 `nfs` 클라이언트 역할을 수행한다.  

여러 파드에서 볼륨 하나를 공유해서 읽기/쓰기를 동시에 수행해야 할때 사용할 수 있다. 
안전성 높은 외부 스토리지를 볼륨으로 설정하고, 
해당 파드에 `nfs` 서버를 설정한다. 
그리고 다른 파드에서는 해당 파드의 `nfs` 서버를 `nfs` 볼륨으로 마운트해서 사용 가능하다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_volume_nfs_plant.png)

고성능이 요구되는 상황에서는 사용하기 힘든 구조이다. 
단순히 데이터의 안정성을 높이면서 파드간 파일 공유가 필요할 경우 적합한 구조이다.  

앞서 언급했던 것과 같이 파드 하나에 `nfs` 서버를 설정하고, 다른 파드와 공유해서 사용하는 설정을 구성해본다. 
`hostPath` 를 사용해서 `nfs` 서버를 만들고, 
다른 파드들은 `nfs` 서버에 `nfs` 볼륨을 마운트해서 구성한다. 
아래는 `nfs` 서버를 설정하는 파트 템플릿의 예시이다. 

```yaml

```  
































 



---
## Reference
