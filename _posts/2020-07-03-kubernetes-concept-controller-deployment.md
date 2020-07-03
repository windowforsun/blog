--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨트롤러(Deployment)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 파드를 관리하는 컨트롤러 중 Deployment 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Controller
  - Deployment
toc: true
use_math: true
---  

## 디플로이먼트
디플로이먼트(`Deployment`) 는 쿠버네티스에서 상태가 없는(`stateless`) 애플리케이션 배포시에 사용하는 기본적인 컨트롤러이다. 
레플리카세트를 관리하면서 애플리케이션 배포에 대해 더욱 세밀하게 관리한다. 
[레플리카세트]({{site.baseurl}}{% link _posts/2020-07-02-kubernetes-concept-controller-replicationcontroller-replicaset.md %})
에서는 관리해야 할 파드의 수만 설정 했다면, 
롤링 업데이트(`Rolling-Update`), 배포 중 중지 및 다시 배포 등의 동작을 제공한다. 

### 디플로이먼트 템플릿
아래는 `Nginx` 이미지를 사용해서 구성한 디플로이먼트의 템플릿 예시이다. 

```yaml
# deployment-nginx.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nginx
  labels:
    app: deployment-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: deployment-nginx
  template:
    metadata:
      labels:
        app: deployment-nginx
    spec:
      containers:
        - name: deployment-nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```  

- `.spec.replicas` : 구성하는 디플로이먼트에서 몇개의 파드를 유지할 지 설정한다. 기본값은 1
- `.spec.selector.matchLabels.app` : `.spec.selector.matchLabels` 의 하위 설정은 `.metadata.labels` 와 동일한 설정이여야 한다. 
- `.spec.template.spec.containers[]` : 파드에서 구성하는 컨테이너에 대한 구체적인 명세를 정의한다. 

구성된 템플릿을 `kubectl apply -f deployment-nginx.yaml` 명령을 통해 실행한다. 
그리고 `kubectl get deploy, rs, pods` 명령을 통해 조회를 하면 아래와 같다. 
`deploy` 는 `deployment` 를, `rs` 는 `replicaset` 를 의미한다.

```bash
$ kubectl apply -f
 deployment-nginx.yaml
deployment.apps/deployment-nginx created
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-nginx   3/3     3            3           27s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-nginx-58cdfb8dc   3         3         3       27s

NAME                                   READY   STATUS    RESTARTS   AGE
pod/deployment-nginx-58cdfb8dc-bvzzk   1/1     Running   0          27s
pod/deployment-nginx-58cdfb8dc-ddfdl   1/1     Running   0          27s
pod/deployment-nginx-58cdfb8dc-ts2xr   1/1     Running   0          27s
```  

`eployment.apps/deployment-nginx` 이름으로 디플로이먼트가 생성되었고, 
디플로이먼트가 관리하는 `replicaset.apps/deployment-nginx-58cdfb8dc` 레플리카세트가 생성된 것을 확인 할 수 있다. 
그리고 레플리카세트에서 관리하는 `deployment-nginx-58cdfb8dc-` 프리픽스가 붙은 파드들도 생성되었다.  

구성한 디플로이먼트의 이미지를 변경했을 때의 상황을 살펴본다. 
이미지를 수정하는 방법으로는 아래와 같은 3가지방법 이 있다. 
- `kubectl set` 명령을 통해 직접 컨테이너 이미지 수정
- `kubectl edit` 명령을 통해 현재 파드 설정 정보의 컨테이너 이미지 수정
- 구성한 템플릿에서 컨테이너 이미지를 수정하고 다시 `kubectl apply` 명령 수행

첫번째로 `kubectl set image deployment/<디플로이먼트 이름> <컨테이너이름>=<이미지이름>` 
으로 직접 컨테이너 이미지를 `nginx:1.9.1` 로 수정한다.

```bash
$ kubectl set image deployment.apps/deployment-nginx deployment-nginx=nginx:1.9.1
deployment.apps/deployment-nginx image updated
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-nginx   3/3     3            3           15m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-nginx-58cdfb8dc   0         0         0       15m
replicaset.apps/deployment-nginx-fbcfb857    3         3         3       33s

NAME                                  READY   STATUS    RESTARTS   AGE
pod/deployment-nginx-fbcfb857-d5gc7   1/1     Running   0          33s
pod/deployment-nginx-fbcfb857-fjmqk   1/1     Running   0          14s
pod/deployment-nginx-fbcfb857-kg6wd   1/1     Running   0          19s
```  

`kubectl set` 명령을 통해 컨테이너 이미지를 수정하자 새로운 레플리카세트인 `replicaset.apps/deployment-nginx-fbcfb857` 가 생기고, 
파드들 또한 새로운 레플리카세트가 관리하는 파드로 갱신 돼었다. 
이렇게 디플로이먼트 설정을 변경하게 되면 새로운 레플리카세트가 생성되고 그에 따라 파드들 또한 변경된다. 

`kubectl get deploy <디플로이먼트 이름> -o=jsonpath="{.spec.template.spec.container[].image}{'\n'}"` 
명령을 수행해서 컨테이너의 이미지 정보를 확인하면 아래와 같이 정상적으로 변경 된 것을 확인 할 수 있다. 

```bash
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.template.spec.containers[].image}{'\n'}"
nginx:1.9.1
```  

다음으로 `kubectl edit deploy <디플로이먼트 이름>` 명령으로 현재 구성된 디플로이먼트 설정에서 컨테이너 부분을 수정해 본다. 
편집기가 열리면 `.spec.template.spec.containers[].image` 필드를 `nginx:1.10.1` 로 변경한다. 

```bash
$ kubectl edit deploy deployment-nginx

.. 생략 ..
spec:
  .. 생략 ..
  template:
    spec:
      containers:
      - image: nginx:1.10.1

..생략.. 


deployment.apps/deployment-nginx edited
```  

다시 `kubectl get deploy <디플로이먼트 이름> -o=jsonpath` 명령어로 컨테이너 이미지를 확인하면 `nginx:1.10.1` 로 수정된 것을 확인 할 수 있다. 

```bash
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.template.spec.containers[].image}{'\n'}"
nginx:1.10.1
```  

마지막으로 구성한 템플릿 `deployment-nginx.yaml` 의 `.spec.template.spec.containers[].image` 필드를 
`nginx:1.11.1` 로 수정하고 `kubectl apply -f` 명령을 수행한다. 
그리고 이미지를 조회하면 아래와 같이 변경된 것을 확인 할 수 있다. 

```bash
spec:
  template:
    spec:
      containers:
          image: nginx:1.11.1

$ kubectl apply -f deployment-nginx.yaml
deployment.apps/deployment-nginx configured
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.template.spec.containers[].image}{'\n'}"
nginx:1.11.1
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-nginx   3/3     3            3           29m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-nginx-58cdfb8dc    0         0         0       29m
replicaset.apps/deployment-nginx-5b5d6bddc9   0         0         0       5m
replicaset.apps/deployment-nginx-7b5b45dd84   3         3         3       79s
replicaset.apps/deployment-nginx-fbcfb857     0         0         0       14m

NAME                                    READY   STATUS    RESTARTS   AGE
pod/deployment-nginx-7b5b45dd84-7tzg5   1/1     Running   0          78s
pod/deployment-nginx-7b5b45dd84-jtk8f   1/1     Running   0          60s
pod/deployment-nginx-7b5b45dd84-t7knk   1/1     Running   0          64s
```  

### 롤백
앞에서 여러 방식을 사용해 이미지를 변경한 이력은 `kubectl rollout history deploy <디플로이먼트 이름>` 명령으로 확인 할 수 있다. 

```bash
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         <none>
```  

총 4개의 리비전(`Revision`) 이 있는 것을 확인 할 수 있고, 
현재 사용중인 리비전은 4이다. 
특정 리비전의 정보 확인은 `--revision=<리비전>` 을 통해 가능하다. 

```bash
$ kubectl rollout history deploy deployment-nginx --revision=4
deployment.apps/deployment-nginx with revision #4
Pod Template:
  Labels:       app=deployment-nginx
        pod-template-hash=7b5b45dd84
  Containers:
   deployment-nginx:
    Image:      nginx:1.11.1
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

```  

이전 리비전으로 롤백은 `kubectl rollout undo deploy <디플로이먼트 이름>` 명령을 통해 가능하다. 
현재 롤백을 하게 되면 리비전 3으로 돌아가게 되고, 
리비전 3은 `kubectl set image` 을 통해 `nginx:1.10.1` 이미지로 변경한 상태이다. 

```bash
$ kubectl rollout undo deploy deployment-nginx
deployment.apps/deployment-nginx rolled back

$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-nginx   3/3     3            3           36m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-nginx-58cdfb8dc    0         0         0       36m
replicaset.apps/deployment-nginx-5b5d6bddc9   3         3         3       12m
replicaset.apps/deployment-nginx-7b5b45dd84   0         0         0       8m29s
replicaset.apps/deployment-nginx-fbcfb857     0         0         0       21m

NAME                                    READY   STATUS    RESTARTS   AGE
pod/deployment-nginx-5b5d6bddc9-2fqll   1/1     Running   0          35s
pod/deployment-nginx-5b5d6bddc9-6fq5s   1/1     Running   0          39s
pod/deployment-nginx-5b5d6bddc9-tfgnl   1/1     Running   0          43s
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.template.spec.containers[].image}{'\n'}"
nginx:1.10.1
```  

디플로이먼트 롤백을 수행하면서 리비전 3에서 사용했던 레플리카세트로 돌아가고 파드 또한 변경 된것을 확인 할 수 있다. 
그리고 이미지는 리비전 3에서 변경한 `nginx:1.10.1` 의 이미지인 것도 확인 가능하다.  

다시 `kubectl rollout history deploy` 명령으로 리비전을 조회하면 아래와 같다. 

```bash
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
4         <none>
5         <none>
```  

리비전 3이 리비전 5로 변경된 것을 확인 할 수 있다.  

특정 리비전으로 롤백하려면 `--to-revision=<리비전>` 옵션을 통해 가능하다. 
리비전 4로 롤백을 하고 조회하면 아래와 같다. 

```bash
$ kubectl rollout undo deploy deployment-nginx --to-revision=4
deployment.apps/deployment-nginx rolled back
$ kubectl get deploy,rs,pods
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-nginx   3/3     3            3           42m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-nginx-58cdfb8dc    0         0         0       42m
replicaset.apps/deployment-nginx-5b5d6bddc9   0         0         0       17m
replicaset.apps/deployment-nginx-7b5b45dd84   3         3         3       13m
replicaset.apps/deployment-nginx-fbcfb857     0         0         0       27m

NAME                                    READY   STATUS        RESTARTS   AGE
pod/deployment-nginx-5b5d6bddc9-2fqll   0/1     Terminating   0          6m1s
pod/deployment-nginx-5b5d6bddc9-6fq5s   0/1     Terminating   0          6m5s
pod/deployment-nginx-5b5d6bddc9-tfgnl   1/1     Terminating   0          6m9s
pod/deployment-nginx-7b5b45dd84-2n5j7   1/1     Running       0          9s
pod/deployment-nginx-7b5b45dd84-f28sm   1/1     Running       0          13s
pod/deployment-nginx-7b5b45dd84-rflcl   1/1     Running       0          5s
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.template.spec.containers[].image}{'\n'}"
nginx:1.11.1
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
5         <none>
6         <none>
```  

리비전 4에서 변경한 `nginx:1.11.1` 이미지로 변경된 것을 확인 가능하다. 
히스토리에 조회되는 리비전의 수는 디플로이먼트 템플릿에서 `.spec.reivisionHistoryLimit` 필드를 통해 설정가능하다. 기본값은 10이다.  

히스토리 조회에서 `CHANGE-CAUSE` 필드는 리비전의 주요 내용을 표시하는 부분이다. 
리비전 숫자만으로는 해당 리비전의 내용이나 상태에 대한 확인이 어렵기 때문에 해당 필드를 사용 할 수 있다. 
필드를 활용하는 방법은 디플로이먼트 템플릿에서 `.metadata.annotations.kubernetes.io/change-cause` 에 해당하는 메시지를 적어 주면 된다. 

```yaml
metadata:
  name: deployment-nginx
  labels:
    app: deployment-nginx
  annotations:
    kubernetes.io/change-cause: version 1.11.1
```  

다시 `kubectl apply -f` 를 통해 템플릿을 적용하고 리비전 히스토리를 조회하면 아래와 같이 추가한 내용이 표시된다. 

```bash
$ kubectl apply -f deployment-nginx.yaml
deployment.apps/deployment-nginx configured
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
5         <none>
6         version 1.11.1
```  


### 파드 수 조정
실행 중인 디플로이먼트의 파드 수를 조정하는 방법은 `kubectl sacle deploy <디플로이먼트 이름> --replicas=<수>` 명령을 통해 가능하다. 

```bash
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
deployment-nginx-7b5b45dd84-2n5j7   1/1     Running   0          23m
deployment-nginx-7b5b45dd84-f28sm   1/1     Running   0          23m
deployment-nginx-7b5b45dd84-rflcl   1/1     Running   0          23m
$ kubectl scale de
ploy deployment-nginx --replicas=5
deployment.apps/deployment-nginx scaled
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
deployment-nginx-7b5b45dd84-2n5j7   1/1     Running   0          24m
deployment-nginx-7b5b45dd84-d75wn   1/1     Running   0          7s
deployment-nginx-7b5b45dd84-f28sm   1/1     Running   0          24m
deployment-nginx-7b5b45dd84-rflcl   1/1     Running   0          24m
deployment-nginx-7b5b45dd84-wbhn4   1/1     Running   0          7s
```  

새로운 파드 2개가 추가로 실행된 것을 확인 할 수 있다. 

### 배포 정지, 재개, 재시작
`kubectl rollout` 명령을 사용하면 진행 중인 배포를 잠시 멈추거나 다시 재시작하는 등의 동작을 수행할 수 있다. 
먼저 `kubectl rollout pause deployment/<디플로이먼트 이름>` 으로 업데이트를 중지시킨다. 

```bash
$ kubectl rollout
pause deployment/deployment-nginx
deployment.apps/deployment-nginx paused
```  

`kubectl set image deployment/<디플로이먼트 이름> <컨테이너이름>:<이미지>` 로 이미지를 변경한다. 
그리고 `CAHNGE-CAUSE` 메시지 변경을 위해 `kubectl patch deployment/<디플로이먼트 이름> -p "{\"metadata\":{\"annotations\":{\"kubernetes.io/change-cause\":\"version 1.11\"}}}"` 
명령을 수행한다. 

```bash
$ kubectl set image deployment/deployment-nginx deployment-nginx=nginx:1.12
deployment.apps/deployment-nginx image updated
$ kubectl patch deployment/deployment-nginx -p "{\"metadata\":{\"annotations\":{\"kubernetes.io/change-cause\":\"version 1.12\"}}}"
deployment.apps/deployment-nginx patched
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
5         <none>
6         version 1.11.1
```  

지금은 업데이트가 정지 돼있기 때문에 갱신 작업이 진행되지 않는다. 
갱신 작업을 위해 `kubectl rollout resume deploy/<디플로이먼트 이름>` 을 수행하고 확인하면 배포가 진행된 것을 확인 할 수 있다.  

```bash
$ kubectl rollout resume deploy/deployment-nginx
deployment.apps/deployment-nginx resumed
$ kubectl rollout history deploy deployment-nginx
deployment.apps/deployment-nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
5         <none>
6         version 1.11.1
7         version 1.12
```  

위와 같이 배포를 잠시 중지 해뒀다가 필요 시점에 다시 진행 시킬 수 있다.  

실서비스 환경에서는 전체 파드를 다시 시작해야 하는 경우도 발생할 수 있다. 
하지만 이전 쿠버네티스에서는 재시작은 설계 사상과 맞지 않다고 지원하지 않았다. 
`1.15` 버전 부터 디플로이먼트, 스테이트풀세트, 데몬세트에 힌해 `kubectl rollout restart` 명령을 통해 재시작이 가능하다. 
디플로이먼트에서 수행한다면 예시는 `kubectl rollout restart deployment/<디플로이먼트 이름>` 과 같다. 


### 디플로이먼트 상태
디플로이먼트는 배포를 관리하면서 상태가 변하게 된다. 
디플로이먼트에 존재하는 배포 상태는 아래와 같다.
- 우선 진행(`Preprocessing`)
- 완료(`Complete`)
- 실패(`Failed`)

배포 상태는 `kubectl rollout status` 명령을 통해 확인 가능하다.  

배포 상태가 우선 진행인 경우는 아래와 같다. 
- 디플로이먼트가 새로운 레플리카세트를 만들 때
- 디플로이먼트가 새로운 레플리카세트의 파드 개수를 늘릴 때
- 디플로이먼트가 예전 레플리카세트의 파드 개수를 줄일 때
- 새로운 파드가 준비 상태가 되거나 이용 가능한 상태가 되었을 때

배포가 이상없이 완료되면 상태는 완료가 되어 종료 코드를 0으로 표시한다. 
이는 아래 조건을 만족한다는 의미이다. 
- 디플로이먼트가 관리하는 모든 레플리카세트가 업데이트 완료되었을 때
- 모든 레플리카세트가 사용 가능해졌을 때
- 예전 레플리카세트가 모두 종료되었을 때

배포 상태가 실패인 경우는 아래와 같다. 
- 쿼터 부족
- readinessProbe 진단 실패
- 컨테이너 이미지 가져오기 실패
- 권한 부족
- 제한 범위 초과
- 애플리케이션 실행 조건을 잘못 지정

디플로이먼트 템플릿에서 `.spec.progressDeadlineSeconds` 필드에 값을 설정하면, 
설정된 값의 시간이 지났을 때 상태를 `False` 로 설정한다. 
테스트를 위해 `.spec.progressDeadlineSedons` 값을 2초로 아주 작게 준다. 

```bash
$ kubectl patch deployment/deployment-nginx -p "{\"spec\":{\"progressDeadlineSeconds\":2}}"
deployment.apps/deployment-nginx patched
$ kubectl get deploy deployment-nginx -o=jsonpath="{.spec.progressDeadlineSeconds}{'\n'}"
2
$ kubectl set image deploy/deployment-nginx deployment-nginx=nginx:1.8
deployment.apps/deployment-nginx image updated
$ kubectl describe deploy deployment-nginx
.. 생략 ..

Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    False   ProgressDeadlineExceeded
```  

`Progressing` 항목이 `False` 인 것과 메시지를 확인해 보면 데드라인이 지나 실패했다는 것을 확인 할 수 있다. 

---
## Reference
