--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 퍼시스턴트 볼륨 과 클레임(PV, PVC)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '볼륨을 더욱 유연하게 구성할 수 있는 PV 와 PVC 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Volume
  - PV
  - PVC
toc: true
use_math: true
---  

## PV, PVC
쿠버네티스에서는 `PV` 라고 불리는 퍼시스턴트 볼륨(`PersistentVolume`) 과 `PVC` 라고 하는 퍼시스턴스 볼륨 클레임(`PersistentVolumeClaim`) 을 사용해서 구성한다.  

`PV` 는 볼륨 그 자체를 의미하고 클러스터에서 자원으로 관리된다. 파드와 별개의 생명주기를 가지고 관리되는 요소이다. 
그리고 `PVC` 는 사용자가 클러스터에 존재하는 `PV` 에게 수행하는 요청을 의미한다. 
사용하려는 용량, 읽기, 쓰기 등을 설정해서 요청을 할 수 있다.  

이렇게 쿠버네티스에서는 볼륨을 사용할때 파드에서 직접 구성하는 것이 아닌, 
`PVC` 라는 매개체를 사용해서 구성한 저장소인 `PV` 를 사용할 수 있도록 했다. 
이런 구조를 통해 파드는 필요에 따라 다양한 스토리지를 사용할 수 있다.  

클라우드 서비스를 사용한다면 서비스에서 제공하는 스토리지를 사용할 수 있고, 
자체적으로 구성한 스토리지가 있다면 해당 스토리지를 사용할 수도 있다. 
이는 `PV` 와 `PVC` 를 통해 파드와 연결되기 때문에 파드에서는 실제 스토리지의 종류에 의존성없이 다양한 구성이 가능하다.  

## PV, PVC 생명주기
`PV` 와 `PVC` 의 생명주기는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_persistent_volume_pvpvc_lifecycle_plant.png)

### 프로비저닝
볼륨을 사용하기 위해서는 먼저 `PV` 를 만들어야 한다. 
`PV` 를 만드는 단계를 프로비저닝(`Provisioning`) 이라고 한다. 
그리고 프로비저닝의 방법으로는 `PV` 를 미리 만들고 사용하는 정적 방법과 요청이 있을 때 `PV` 를 만드는 동적 방법이 있다.  

정적인 방법은 클러스터 관리자가 미리 적정 용량의 `PV` 를 만들어 두고 요청이 있으면 만들어둔 `PV` 를 할당한다. 
이러한 방법은 사용가능한 용량 제한이 있을 경우 유용하다. 
만약 `PV` 용량이 `100GB` 로 만들어져 있을 때, 
`150GB` 에 대한 요청은 모두 실패하게 된다.  

동적인 방법은 `PVC` 를 통해 `PV` 를 요청했을 때 생성하고 볼륨이 제공된다. 
사용 가능한 용량이 있을때 이를 넘지않는 용량을 사용자가 요청한다면, 
요청한 용량만큼 `PV` 를 생성해서 사용 할 수 있다. 
그리고 `PVC` 는 동적 프로비저닝을 수행할 때 원하는 스토리지를 스토리지 클래스(`Storage Class`) 로 `PV` 를 생성한다.  

### 바인딩
바인딩(`Binding`) 은 프로비저닝 단계에서 만든 `PV` 와 `PVC` 를 연결하는 단계이다. 
`PVC` 를 통해 원하는 스토리지와 용량 및 접근에 대한 설정으로 요청하면 해당하는 `PV` 가 할당된다. 
만약 적절한 `PV` 가 없다면 요청은 실패하게 된다. 
요청이 실패하게 되면 해당하는 `PV` 가 생길때까지 대기하다가 `PVC` 에 바인딩 된다. 
`PVC` 와 `PV` 는 1:1 관계이므로 `PVC` 하나가 여러 `PV` 에 바인딩 될 수 없다.  

### 사용
`PVC` 는 파드에 설정되고 파드는 설정된 `PVC` 를 볼륨으로 인식하고 사용한다. 
`PVC` 에는 '사용 중인 스토리지 오브젝트 보호(`Storage Object in Use Protection`)' 기능이 있어, 
파드에서 사용중인 `PVC` 를 삭제할 수 없다. 
파드가 사용중인 `PVC` 를 삭제하려고하면 상태가 `Terminating` 이지만 삭제되지 않고 남아있다. 
이 상태에서 `kubectl describe` 명령으로 `PVC` 의 디테일한 정보를 확인하면 `pvc-protection` 메시지를 확인 할 수 있다.  

### 반환
사용이 끝난 `PVC` 는 삭제되고 `PVC` 에 할당된 `PV` 는 초기화 과정을 거친다. 
이러한 단계를 반환(`Reclaiming`) 이라고 하고, 
초기화 정책으로는 `Retain`, `Delete`, `Recycle` 이 있다.  

#### Retain
`Retain` 은 `PV` 를 삭제하지 않고 보존한다. 
`PVC` 가 삭제되면 할당된 `PV` 는 `Released` 상태가 되고 다른 `PVC` 가 사용할 수 없다. 
해제 상태이므로 기존에 사용 했던 데이터는 모두 남아있고, 
재사용하기 위해서는 직접 아래의 초기화 과정을 수행해야 한다. 
1. `PV` 삭제한다. 만약 `PV` 가 외부 스토리지를 사용한다면 `PV` 는 삭제되더라도 외부 스토리지에 볼륨은 그대로 남아있다. 
1. 스토리지에 남은 데이터를 정리한다. 
1. 남은 스토리지의 볼륨을 삭제하거나 재사용을 한다면 해당 볼륨을 이용하는 `PV` 를 다시 생성한다. 

#### Delete
`PV` 를 삭제하고 연결된 외부 스토리지의 볼륨도 삭제한다. 
프로비저닝을 동적 볼륨으로 수행했다면 `PV` 의 기본 반환 정책은 `Delete` 가 된다. 
필요에다가 반환 정책을 수정해야 한다. 

#### Recycle
`PV` 데이터를 삭제하고 새로운 `PVC` 에서 `PV` 를 할당할 수 있도록 한다. 
현재는 `Deprecated` 된 정책이고, 
동적 프로비저닝을 사용하는 것을 권장하고 있다. 

## 퍼시스턴스 볼륨 템플릿
아래는 퍼시스턴스 볼륨의 예시 템플릿이다. 

```yaml
# pv-hostpath.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /var/lib/pv-hostpath
    type: DirectoryOrCreate
```  

- `.spec.capacity.storage` : 스토리지 용량으로 `1GB` 를 설정한다. 
- `.spec.volumeMode` : 볼륨을 파일 시스템 모드로 설정한다. 
추가로 설정 가능한 값들로는 `raw`, `awsElasticBlockStore`, `azureDisk`, `fc`, `gcePersistentDisk`, `iscsi`, `local volume`, `rbd` 등이 있다. 
- `.spec.accessModes` : 볼륨에서 수행할 수 있는 읽기/쓰기 옵션을 설정한다. 
설정 가능한 값들로는 아래와 같은 것들이 있다.
    - `ReadWriteOnce` : 노드 하나에만 볼륨을 읽기/쓰기 사용
    - `ReadOnlyManay` : 여러 개 노드에서 읽기 전용으로 사용
    - `ReadWriteMany` : 여러개 노드에서 읽기/쓰기 가능
- `.spec.storageClassName` : 스토리지 클래스를 설정하는 필드이다. 
특정 스토리지 클래스를 설정하게 되면 해당하는 `PVC` 와만 연결된다. 
만약 설정하지 않으면 스토리지 클래스 설정이 되지 않은 `PVC` 와 연결된다.
- `.spec.persistentVolumeReclaimPolicy` : `PV` 가 해제되었을 때 초기화 옵션을 설정한다. 
- `.spec.hostPath` : `PV` 볼륨 플러그인을 설정한다. 현재는 `hostPath` 를 사용하고 호스트 경로를 설정했다. 

구성한 템플릿을 `kubectl apply -f pv-hostpath.yaml` 명령으로 클러스터에 적용한다. 
그리고 `PV` 를 조회하면 아래와 같다. 

```yaml
$ kubectl apply -f pv-hostpath.yaml
persistentvolume/pv-hostpath created
$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-hostpath   1Gi        RWO            Delete           Available           manual                  18s
```  

`STATUS` 상태가 `Available` 인 것을 확인 할 수 있다. 
`PVC` 와 연결되면 `Bound`, `PVC` 는 삭제되었지만 `PV` 는 아직 초기화 않았다면 `Released` 상태이고, 자동 초기화가 실패하면 `Failed` 가 된다.  

## 퍼시스턴트 볼륨 클레임 템플릿
아래는 퍼시스턴트 볼륨 클레임의 예시 테플릿이다. 

```yaml
# pvc-hostpath.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-hostpath
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 200Mi
  storageClassName: manual
```  

- `.spec.resources.requests.storage` : `PVC` 에서 사용할 자원을 설정한다. 
자원의 크기는 요청하는 `PV` 에 할당된 크기보다 크면 할당되지 않고 `Pending` 상태로 남게 된다. 

구성한 템플릿을 `kubectl apply -f pvc-hostpath.yaml` 명령으로 클러스터에 적용하고, 
`PVC` 를 조회하면 아래와 같다. 

```yaml
$ kubectl apply -f pvc-hostpath.yaml
persistentvolumeclaim/pvc-hostpath created
$ kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   Bound    pv-hostpath   1Gi        RWO            manual         4s
```  

`VOLUME` 필드에 `pv-hostpath` 가 설정된 것을 확인할 수 있다. 
다시 `PV` 를 조회하면 `STATUS` 필드의 값이 `Bound` 인 것을 확인 할 수 있다. 

```bash
$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pv-hostpath   1Gi        RWO            Delete           Bound    default/pvc-hostpath   manual                  3m13s
```  

## 레이블을 사용해서 PV, PVC 연결하기
이전 예제에서는 스토리지 클래스를 사용해서 `PV` 와 `PVC` 를 연결했다. 
이번에는 기존에 사용하던 레이블을 통해 `PV` 와 `PVC` 를 연결하는 템플릿 예제에 대해 알아본다. 

```yaml
# pv-hostpath-label.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath-label
  labels:
    location: local
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: /var/lib/pv-hostpath-label
    type: DirectoryOrCreate
```  

- `.metadata.lables` : `location` 을 키로 `local` 값을 갖는 레이블을 추가한다. 

```yaml
# pvc-hostpath-label.yaml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-hostpath-label
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 200Mi
  storageClassName: manual
  selector:
    matchLabels:
      location: local
```  

- `.spec.selector.matchLables` : `PV` 템플릿에서 추가한 `location: local` 레이블을 설정한다. 

`kubectl apply -f ` 명령을 사용해서 두개 템플릿을 클러스터에 적용한다. 
그리고 `PV` 와 `PVC` 를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f pv-hostpath-label.yaml
persistentvolume/pv-hostpath-label created
$ kubectl apply -f pvc-hostpath-label.yaml
persistentvolumeclaim/pvc-hostpath-label created
root@CS_NOTE_2:/mnt/c/Users/ckdtj/Documents/kubernetes-example/volume/pv-pvc# kubectl get pv,pvc
NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
persistentvolume/pv-hostpath         1Gi        RWO            Delete           Bound    default/pvc-hostpath         manual                  8m46s
persistentvolume/pv-hostpath-label   1Gi        RWO            Delete           Bound    default/pvc-hostpath-label   manual                  86s

NAME                                       STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvc-hostpath         Bound    pv-hostpath         1Gi        RWO            manual         6m46s
persistentvolumeclaim/pvc-hostpath-label   Bound    pv-hostpath-label   1Gi        RWO            manual         12s
```  

`pvc-hostpath-label` 의 `VOLUME` 필드에 `pv-hostpath-label` 이 설정된 것을 확인 할 수 있다.  

추가로 `.spec.selector.matchLables` 에 `matchExpressions[]` 필드를 사용해서 아래와 같이 조건을 설정할 수도 있따. 

```yaml
spec:
  selector:
    matchExpressions:
      - {key: stage, operator: In, values [DEV]}
```  

## 파드에서 PVC 를 볼륨으로 사용하기
아래는 파드에서 `PVC` 를 볼륨으로 사용하는 디플로이먼트 템플릿 예시이다. 

```yaml
# deploy-pvc.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-app
  labels:
    app: pvc-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pvc-app
  template:
    metadata:
      labels:
        app: pvc-app
    spec:
      containers:
        - name: pvc-app
          image: nginx:latest
          ports:
            - containerPort: 80
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: "/usr/share/nginx/html"
              name: pvc-volume
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: pvc-hostpath
```  

- `.spec.template.spec.volumes[].name` : 사용할 볼륨을 설정한다. 
`persistentVolumeClaim` 필드에 앞서 생성해둔 `pvc-hostpath` 를 사용하도록 설정한다. 
- `.spec.template.spec.containers[].volumeMounts[]` : `volumes` 에서 설정한 이름을 사용해서 컨테이너와 연결한다. 
`mountPath` 에는 `nginx` 공식 이미지의 `HTML` 파일 경로를 마운트 한다. 

구성한 템플릿을 `kubectl apply -f deploy-pvc.yaml` 명령으로 클러스터에 적용한다. 
그리고 파드와 마운트된 호스트 경로로 가서 `index.html` 파일을 생성해 준다. 

```yaml
$ kubectl apply -f deploy-pvc.yaml
deployment.apps/pvc-app created
$ cd /mnt/wsl/docker-desktop-data/data/pv-hostpath
$ echo "welcome pv, pvc" > index.html
$ ls
index.html
```  

`kubectl port-forward` 명령을 사용해서 실행 중인 파드의 포트 80과 호스트 포트 8123을 포워딩한다. 

```bash
$ kubectl get pod
NAME                          READY   STATUS    RESTARTS   AGE
pvc-app-6dcdb6cf8c-g4skr      1/1     Running   0          9m14s
$ kubectl port-forward pods/pvc-app-6dcdb6cf8c-g4skr 8123:80
Forwarding from 127.0.0.1:8123 -> 80
Forwarding from [::1]:8123 -> 80
```  

`localhost:8123` 으로 `curl` 명령을 사용하거나 웹 브라우저를 사용해서 요청하면 `welcome pv, pvc` 가 출력되는 것을 확인 할 수 있다. 

```bash
$ curl localhost:8123
welcome pv, pvc
```  

## PVC 용량 확장하기
`gcePersistentDisk`, `awsElasticBlockStore`, `cinder`, `glusterfs`, `rbd`, `azureFile`, `azureDisk`, `portworkVolume` 등의 
볼륨 플러그인을 사용하면 할당된 `PVC` 의 용량 확장이 가능하다.  
용량을 확장하기 위해서는 템플릿에서 `.spec.storageClassName.allowVolumeExpansion` 필드의 값이 `true` 로 설정되어야 한다.  

`PVC` 용량을 늘릴때는 기존에 설정된 `.spec.resources.requests.storage` 필드 값에 더 큰 용량을 설정한 후 다시 클러스터에 적용한다. 
만약 파일 시스템이 `XFS`, `Ext3`, `Ext4` 인 경우에도 볼륨 크기를 확장 할 수 있다. 
볼륨 크기를 늘리는 동작은 `PVC` 를 사용하는 새로운 파드가 실행할 때만 진행된다. 
기존에 실행중인 파드들은 재시작을 해야 한다. 
파드 실행 중 볼륨 크기를 조절하는 기능은 [여기](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/)
에서 참고할 수 있다.  

## 노드별 볼륨 개수 제한
쿠버네티스의 볼륨은 노드하나에 설정가능한 볼륨수를 제한한다. 
`kube-scheduler` 컴포넌트의 `KUBE_MAX_PD_VOLS` 환경 변수를 통해 설정할 수 있다. 
추가로 클라우드 서비스의 경우 제약사항은 아래와 같다.  

클라우드 서비스|최대 볼륨 수
---|---
Amazon Elastic Block Store| 39
Google Persistent Disk| 16
Microsoft Azure Disk Storage| 16

---
## Reference
