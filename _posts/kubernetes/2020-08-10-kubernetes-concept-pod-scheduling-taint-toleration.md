--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 파드 스케쥴링 (테인트, 톨러레이션, 커든, 드레인)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에 구성된 노드에서 파드 스케쥴링을 설정할 수 있는 테인트, 톨레이션, 커든, 드레인에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Pod Scheduling
  - Taint
  - Toleration
  - Cordon
  - Drain
toc: true
use_math: true
---  

## 테인트, 톨러레이션
쿠버네티스 클러스터에서 테인트(`Taint`) 를 설정한 노드는 파드 스케쥴링을 수행하지 않는다. 
테인트를 설정한 노드에 파드를 스케쥴링하기 위해서는 톨러레이션(`Toleration`) 을 설정해야 하는데, 
톨러레이션에서 설정된 특정 파드면 해당 노드에 스케쥴링하고 다른 파드는 스케쥴링하지 않는다.  

테인트와 톨러레이션을 사용하면 별도로 특정 역할을 수행하는 노드를 구성할 수 있다. 
데이터베이스 파드를 실행 할때 노드에는 테인트를 설정하고 톨러레이션에 데이터베이스 파드를 설정하면, 
데이터베이스 파드는 노드의 모든 자원을 독점해서 사용할 수 있게 된다.  

먼저 테인트의 설정은 아래와 같은 3가지 구성을 갖는다.
- 키
- 값
- 효과

`kubectl taint nodes <노드이름> <키>=<값>:<효과>` 와 같은 명령 형식으로 테인트를 적용할 수 있다. 
아래 명령어로 현재 노드에 새로운 테인트를 설정한다. 
그리고 `kubectl describe nodes <노드이름>` 으로 설정한 테인트를 확인하면 아래와 같다. 

```bash
kubectl taint nodes docker-desktop key01=value01:NoSchedule
node/docker-desktop tainted
root@CS_NOTE_2:/mnt/c/Users/ckdtj/Documents/kubernetes-example# kubectl describe nodes docker-desktop
Name:               docker-desktop
Roles:              master

.. 생략 ..

Taints:             key01=value01:NoSchedule

.. 생략 ..
```  

노드에 테인트를 설정을 하게되면 실제로 파드 스케쥴링이 수행되지 않는지 아래 예제 디플로이먼트 템플릿을 사용해서 테스트를 해본다. 

```yaml
# deploy-taint.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-pod
  labels:
    app: deploy-pod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-pod-taint
  template:
    metadata:
      labels:
        app: deploy-pod-taint
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
```  

```bash
$ kubectl apply -f deploy-taint.yaml
deployment.apps/deploy-pod-taint created
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
deploy-pod-taint-787df4965d-xz74w   0/1     Pending   0          21s
```  

클러스터에 등록된 파드를 조회하면 디플로이먼트로 배포한 파드의 상태가 `Pending` 상태에서 변화가 없는 것을 확인 할 수 있다.  

테인트 설정으로 파드 스케쥴링이 되지 않을 때는, 톨러레이션을 사용해 특정 파드만 스케쥴링 대상으로 만들 수 있다고 했었다. 
아래 디플로이먼트 템플릿은 톨러레이션을 적용한 예시이다. 

```yaml
# deploy-toleration.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-pod-taint
  labels:
    app: deploy-pod-taint
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-pod-taint
  template:
    metadata:
      labels:
        app: deploy-pod-taint
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
      tolerations:
        - key: "key01"
          operator: "Equal"
          value: "value01"
          effect: "NoSchedule"
```  

- `.spec.template.spec.tolerations[]` : 앞서 노드에 설정했던 톨러레이션을 설정한다. 

`kubectl apply -f deploy-toleration.yaml` 명령으로 클러스터에 적용하고, 
실행 중인 파드를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f deploy-toleration.yaml
deployment.apps/deploy-pod-taint configured
$ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
deploy-pod-taint-6bbfcb7f9f-zf8nf   1/1     Running   0          5s
```  

이전 템플릿과 톨러레이션을 추가한 템플릿에서 사용하는 레이블이 같기 때문에, 
기존에 `Pending` 상태이던 파드는 사라지고, 새로운 파드가 실행 중인 것을 확인 할 수 있다.  

노드에 추가한 테인트 설정은 `kubectl taint nodes <노드이름> <키>:<효과>-` 명령으로 삭제할 수 있다. 

```bash
$ kubectl taint nodes docker-desktop key01:NoSchedule-
node/docker-desktop untainted
```  

### 설정 값
앞서 살펴본 것과 같이 테인트를 설정할 때는 키, 값, 효과 이렇게 3가지 구성요소가 필요하다. 
각 구성요소에서 필요한 조건은 아래와 같다. 
- 키(`key`) : 최대 253자까지 가능하고 영문, 숫자로 시작해야한다. 그리고 영문, 숫자, `-`, `.`, `_` 등 툭수문자를 사용할 수 있다.
- 값(`value`) : 키와 동일한 문자 조합으로 사용가능하고 최대 63자 까지 작성할 수 있다. 
- 효과(`effect`) : 아래와 같은 3가지 설정가능한 값을 사용할 수 있다.
    - `NoSchedule` : 톨러레이션 설정이 없으면 파드를 노드에서 스케쥴링하지 않는다. 
    기존에 노드에서 실행 중이던 파드는 적용되지 않는다. 
    - `PreferNoSchedule` : 톨러레이션 설정이 없으면 파드를 노드에서 스케쥴링하지 않는다. 
    클러스터의 자원이 부족할 경우, 테인트가 설정된 노드에서도 파드가 스케쥴링 될 수 있다. 
    - `NoExecute` : 톨러레이션 설정이 없으면 파드를 노드에서 스케쥴링하지 않는다. 
    기존 노드에서 실행 중이던 파드에도 적용된다. 

템플릿에서 톨러레이션 설정을 할때 `.operator` 필드에 조건을 검사할 연산을 설정한다. 
- `Equal` : 키, 값, 효과가 설정된 값과 노드에 설정된 값이 동일한지 확인한다. 
- `Exists` : 3가지 설정요소를 선별적으로 확인할 때 사용한다. `Exists` 를 사용할 때는 값 필드는 사용할 수 없다. 

만약 `.spec.template.spec.tolerations[].operator` 필드 값에 `Exists` 만 있는 경우, 
노드에 어떤 테인트 설정이 있던 해당 파드는 스케쥴링 될 수 있다. 

```yaml
spec:
  template:
    spec:
      tolerations:
        operator: "Exists"
```  

그리고 `.spec.template.spec.tolerations[].operator` 필드에 `Exists` 를 설정하고 `.key` 필드 값만 설정하면, 
설정된 `.key` 에 해당하는 테인트 설정만 있다면 해당 파드는 스케쥴링 될 수 있다. 

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: "someKey"
          operator: "Exists"
```  


## 커든, 드레인
커든(`Cordon`) 과 드레인(`Drain`) 은 특정 노드에 있는 파드를 모두 다른 노드로 옮겨야 하건, 
특정 노드의 파드 스케쥴링을 잠시 중지해야 하는 상황에서 사용할 수 있는 명령어이다. 

### 커든
커든은 `kubectl cordon` 명령으로 사용 할 수 있다. 
해당 명령은 지정된 노드에 추가로 파드를 스케쥴링 하지 않도록 하는 동작을 수행한다. 
먼저 현재 클러스터에 구성된 노드를 확인하고 `kubectl cordon <노드이름>` 명령으로 커든을 적용하면 아래와 같다. 

```bash
$ kubectl get nodes
NAME             STATUS   ROLES    AGE   VERSION
docker-desktop   Ready    master   42d   v1.16.6-beta.0
$ kubectl cordon docker-desktop
node/docker-desktop cordoned
$ kubectl get nodes
NAME             STATUS                     ROLES    AGE   VERSION
docker-desktop   Ready,SchedulingDisabled   master   42d   v1.16.6-beta.0
```  

실행 중인 파드를 확인해보면 `테인트, 톨러레이션` 에서 실행 했던 파드가 정상적으로 실행 중인 것을 확인 할 수 있다. 

```bash
NAME                                READY   STATUS    RESTARTS   AGE
deploy-pod-taint-6bbfcb7f9f-zf8nf   1/1     Running   0          27m
```  

파드가 스케쥴링 가능한지 `kubectl scale deploy <디플로이먼트 이름> --replicas=2` 로 파드의 수를 늘리면 아래 결과를 확인 할 수 있다. 

```bash
$ kubectl scale deploy deploy-pod-taint --replicas=2
deployment.apps/deploy-pod-taint scaled
$ kubectl get deploy,pod
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deploy-pod-taint   1/2     2            1           34m

NAME                                    READY   STATUS    RESTARTS   AGE
pod/deploy-pod-taint-6bbfcb7f9f-g2lms   0/1     Pending   0          23s
pod/deploy-pod-taint-6bbfcb7f9f-zf8nf   1/1     Running   0          28m
```  

파드 수를 늘리며 추가된 파드의 상태가 `Pending` 상태로 멈춰 있는 것을 확인 할 수 있다. 
클러스터에 다른 노드가 있었단 해당 노드에서 실행 가능하지만 현재는 한 개 노드의 구성이므로 실행이 가능한 노드가 없어서 발생한 결과이다.  

이제 `kubectl uncordon <노드이름>` 명령을 실행해서 파드 스케쥴링이 가능하게 하면, 
아래와 같이 기존에 `Pending` 이던 파드가 `Running` 으로 정상 실행되는 것을 확인 할 수 있다. 

```bash
$ kubectl uncordon docker-desktop
node/docker-desktop uncordoned
$ kubectl get deploy,pod
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deploy-pod-taint   2/2     2            2           36m

NAME                                    READY   STATUS    RESTARTS   AGE
pod/deploy-pod-taint-6bbfcb7f9f-g2lms   1/1     Running   0          3m12s
pod/deploy-pod-taint-6bbfcb7f9f-zf8nf   1/1     Running   0          31m
```  

### 드레인
드레인은 `kubeclt drain` 명령으로 사용할 수 있다. 
해당 명령은 다양한 상황으로 인해 노드에 실행 중인 파드를 다른 노드로 모두 옮겨야 할때 사용하는 명령이다. 
파드를 다른 노드로 다 옮기더라도 파드 스케쥴링이 가능하면, 
다시 다른 파드가 실행 될 수 있기 때문에 드레인 명령어는 먼저 커든과 같이 파드 스케쥴링을 멈추고 동작을 수행한다.  

하지만 데몬세트를 사용해서 실행한 파드는 드레인에 적용되지 않는다. 
데몬세트는 파드를 삭제하더라도 바로 재실행하는 성절로 인해, 
노드에 데몬세트 파드가 있는 경우 해당 파드는 무시하는 `--ignore-daemonset=true` 옵션과 함께 실행해야 한다.  

컨트롤러를 사용하지 않고 실행한 파드 또한 드레인에 적용되지 않는다. 
컨트롤러로 관리되는 파드는 현재 노드에서 삭제되더라도 컨트롤러가 다른 노드에 다시 실행하지만, 
그렇지 않은 파드가 삭제되면 다시 복구할 수 없기 때문이다. 
이런 상황에 강제로 모든 파드를 삭제해서 드레인을 적용하기 위해서는 `--force` 옵션을 사용해야 한다.  

추가로 노드에서 `kubelet` 가 실행한 스태틱 파드들이 있다. 
해당 파드가 있는 상태에서 드레인을 적용하면 그레이스풀 하게 파드를 종료하도록 설정돼 있다. 
드레인의 수행은 종료 명령을 받은 즉지 파드가 종료되는 것이 아닌, 
기존에 수행 중이던 작업을 모두 수행한 후 종료할 수 있도록 대기하게 된다.  

현재 구성된 노드를 확인하고 `kubectl drain <노드 이름>` 으로 드레인을 적용하면, 
아래와 같이 데몬세트, 레플리카세트 등으로 인해 드레인 적용이 불가능하다는 메시지가 출력된다. 

```bash
$ kubectl drain docker-desktop
node/docker-desktop cordoned
error: unable to drain node "docker-desktop", aborting command...

There are pending nodes to be drained:
 docker-desktop
cannot delete Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet (use --force to override): kube-system/storage-provisioner, kube-system/vpnkit-controller
cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/daemonset-fluentd-ccztl, kube-system/kube-proxy-wzl67
```  

`--ignore-daemonsets=true --force` 옵션을 주고 다시 명령을 수행하면 아래와 같이 적용 되는 것을 확인 할 수 있다. 

```bash
$ kubectl drain --ignore-daemonsets=true --force docker-desktop
node/docker-desktop already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/daemonset-fluentd-ccztl, kube-system/kube-proxy-wzl67; deleting Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-system/storage-provisioner, kube-system/vpnkit-controller
evicting pod "vpnkit-controller"

.. 생략 ..

pod/compose-api-6ffb89dc58-8qvmz evicted
node/docker-desktop evicted
```  

현재 노드에 드레인이 적용된 상태에서 노드, 디플로이먼트, 파드 등등을 모든 네임스페이스로 조회해서 자원 상태를 확인하면 아래와 같다. 

```bash
kubectl get nodes,deploy,daemonset,pod --all-namespaces
NAME                  STATUS                     ROLES    AGE   VERSION
node/docker-desktop   Ready,SchedulingDisabled   master   42d   v1.16.6-beta.0

NAMESPACE       NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
default         deployment.apps/deploy-pod-taint           0/2     2            0           52m
docker          deployment.apps/compose                    0/1     1            0           42d
docker          deployment.apps/compose-api                0/1     1            0           42d
ingress-nginx   deployment.apps/nginx-ingress-controller   0/1     1            0           20d
kube-system     deployment.apps/coredns                    0/2     2            0           42d

NAMESPACE     NAME                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/daemonset-fluentd   1         1         0       1            0           <none>                        37d
kube-system   daemonset.apps/kube-proxy          1         1         1       1            1           beta.kubernetes.io/os=linux   42d

NAMESPACE       NAME                                           READY   STATUS             RESTARTS   AGE
default         pod/deploy-pod-taint-6bbfcb7f9f-k5krn          0/1     Pending            0          2m12s
default         pod/deploy-pod-taint-6bbfcb7f9f-rjb2k          0/1     Pending            0          2m12s
docker          pod/compose-78f95d4f8c-8l4v8                   0/1     Pending            0          2m12s
docker          pod/compose-api-6ffb89dc58-w5jw2               0/1     Pending            0          2m12s
ingress-nginx   pod/nginx-ingress-controller-f9d45bc68-ffgpg   0/1     Pending            0          2m12s
kube-system     pod/coredns-5644d7b6d9-8cvkg                   0/1     Pending            0          2m12s
kube-system     pod/coredns-5644d7b6d9-qk88m                   0/1     Pending            0          2m12s
kube-system     pod/daemonset-fluentd-ccztl                    0/1     CrashLoopBackOff   318        37d
kube-system     pod/etcd-docker-desktop                        1/1     Running            0          42d
kube-system     pod/kube-apiserver-docker-desktop              1/1     Running            0          42d
kube-system     pod/kube-controller-manager-docker-desktop     1/1     Running            0          42d
kube-system     pod/kube-proxy-wzl67                           1/1     Running            0          42d
kube-system     pod/kube-scheduler-docker-desktop              1/1     Running            0          42d
```  

노드 조회에서 `STATUS` 필드를 보면 현재 노드는 스케쥴링이 불가능한 상태인 것을 확인 할 수 있다. 
그리고 각 파드나 다른 자원들은 `Pending` 혹은 실행 불가 상태인 것을 확인 가능하다. 
다른 노드가 있는 상황이라면 다른 노드로 자원들이 넘어 가게 되지만, 노드가 1개 이기 때문에 
현재 파드를 스케쥴링하려 하지만 노드는 파드 스케쥴링이 불가능하기 때문에 `Pending` 상태로 남아 있게 된다.  

`kubectl uncordon <노드 이름>` 명령으로 드레인 설정을 해제할 수 있다. 
드레인 설정을 해제하고 다시 노드의 자원을 조회하면 아래와 같다. 

```bash
kubectl uncordon docker-desktop
node/docker-desktop uncordoned
root@CS_NOTE_2:/mnt/c/Users/ckdtj/Documents/kubernetes-example/pod-scheduling/taint-toleration# kubectl get nodes,deploy,daemonset,pod --all-namespaces
NAME                  STATUS   ROLES    AGE   VERSION
node/docker-desktop   Ready    master   42d   v1.16.6-beta.0

NAMESPACE       NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
default         deployment.apps/deploy-pod-taint           2/2     2            2           86m
docker          deployment.apps/compose                    1/1     1            1           42d
docker          deployment.apps/compose-api                1/1     1            1           42d
ingress-nginx   deployment.apps/nginx-ingress-controller   1/1     1            1           20d
kube-system     deployment.apps/coredns                    2/2     2            2           42d

NAMESPACE     NAME                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/daemonset-fluentd   1         1         0       1            0           <none>                        37d
kube-system   daemonset.apps/kube-proxy          1         1         1       1            1           beta.kubernetes.io/os=linux   42d

NAMESPACE       NAME                                           READY   STATUS             RESTARTS   AGE
default         pod/deploy-pod-taint-6bbfcb7f9f-k5krn          1/1     Running            0          36m
default         pod/deploy-pod-taint-6bbfcb7f9f-rjb2k          1/1     Running            0          36m

.. 생략 ..

kube-system     pod/kube-proxy-wzl67                           1/1     Running            0          42d
kube-system     pod/kube-scheduler-docker-desktop              1/1     Running            0          42d
```  

노드 상태는 `Ready` 로 변경되었고, 파드들도 모두 정상 실행 되는 것을 확인 할 수 있다. 

---
## Reference
