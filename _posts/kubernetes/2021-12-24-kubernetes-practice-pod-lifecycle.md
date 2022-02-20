--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Pod Lifecycle 과 Readiness Gates"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Kubernetes Pod 오브젝트가 가지는 Lifecycle 과 상태에 대해 알아보고, readinessGates 에 대해서도 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Pod
  - Pod Lifecycle
  - Pod Phase
  - Container States
  - Pod Conditions
  - Containers Probes
  - Readiness Gates
  - Restart Policy
toc: true
use_math: true
---  

## 환경
- `Kubernetes v1.22.2`
- `Docker 20.10.8`

## Pod Lifecycle
`Pod` 는 `Kubernetes` 를 사용하면서 가장 많이 사용하게 되는 오브젝트이다. 
이런 `Pod` 은 생명주기를 가지고 있는데 이번 포스트에서는 이 생명주기에 대해서 다뤄보려고 한다. 
또한 `Pod` 사용할 때 발생할 수 있는 다양한 이슈에 대해서 파악이 좀더 쉽도록 예제도 함께 진행해 볼 계획이다.  

`Pod` 을 다뤄본 사람이라면 알겠지만 기본적으로 `Pending` 에서 시작해서 컨테이너가 정상 동작하며 `Running` 이 되고, 
성공, 실패 종료 여부에 따라 `Succeeded` 또는 `Failed` 가 된다.  

이후에 `Pod` 템플릿을 사용할 예정인데, 정상적으로 구동이 가능한 템플릿인 `test-pod.yaml` 아래와 같다. 
예제는 아래 정상 템플릿에서 일부분만 수정하거나, 추가 및 삭제해서 특정 상황을 만드는 방법으로 진행 된다.  

```yaml
# test-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  initContainers:
    - name: init-con
      imagePullPolicy: IfNotPresent
      image: nginx:latest
      command: ["/bin/bash", "-c", "sleep 5;"]
  containers:
    - name: test-pod-con
      imagePullPolicy: IfNotPresent
      image: nginx:latest
      ports:
        - containerPort: 80
      lifecycle:
        postStart:
          exec:
            command: ["/bin/bash", "-c", "sleep 5;"]
        preStop:
          exec:
            command: ["/bin/bash", "-c", "sleep 5;"]
      startupProbe:
        exec:
          command: ["/bin/bash", "-c", "sleep 5;"]
        timeoutSeconds: 10
      livenessProbe:
        exec:
          command: ["/bin/bash", "-c", "sleep 5;"]
        timeoutSeconds: 10
      readinessProbe:
        httpGet:
          path: /index.html
          port: 80
  restartPolicy: Never
```  

### Pod Phase
`Pod` 는 아래와 같은 4개의 `Phase`(단계) 로 구성된다. 

값|의미
---|---
Pending|`Pod` 이 `Kubernetes` 클러스터에 의해 승인됐지만, 하나 이상의 컨테이너가 설정되지 않았고 실행준비가 되지 않은 상태이다. 파드 스케쥴링, 컨테이너 이미지 다운로드 시간이 포함된다. 
Running|`Pod` 가 특정 노드에 할당 되었고, 모든 컨테이너가 생성되었으며 적어도 하나의 컨테이너가 실행, 재시작 중에 있다. 
Succeeded|`Pod` 에 있는 모든 컨테이너들이 성공적으로 종료(`exit 0`) 됐고, 재시작 되지 않는다. 
Failed|`Pod` 에 있는 모든 컨테이너가 종료되었고, 적어도 하나 이상의 컨테이너가 실패로 종료 됐다. `none-zero` 상태로 종료 됐거나(`exit`), 시스템에 의해 종료 된걸 의미한다. 
Unknown|`Pod` 의 상태를 가져 올 수 없을 때 발생한다. 주로 `Pod` 이 할당된 노드와 `API` 통신에 문제가 있을 때 발생한다. 

`Pod` 단계 생명주기를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-pod-lifecycle-1.drawio.png)  

만약 현재 `Pod` 의 `Pod Phase` 를 조회하고 싶다면 아래 명령어를 사용할 수 있다.  

```bash
kubectl get pod <pod-name> -o json | jq '.status.phase'
```  

#### Running
가장 먼저 정상적으로 수행되는 `Running` 상태는 아래 명령으로 `test-pod.yaml` 템플릿으로 `Pod` 을 실행해 준다.  

```bash
$ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
pod/all-ok created
"Pending"
"Pending"
"Pending"
"Running"
"Running"
"Running"
```  

`Pending` 단계를 거쳐 `Running` 단계로 계속 유지되는 것을 확인 할 수 있다.   

#### Pending
`Pending` 상태에서 벗어나지 못하는 상황은 여러가지가 있겠지만 대표적으로 아래 몇가지가 있다.  

- `.resources` 에 설정한 리소스로 인해 스케쥴링에 실패한 경우
  - `.resources.requests` 설정으로 아주 큰 숫자를 아래와 같이 추가 후 실행하면 아래와 같다. 
    
    ```yaml
    spec:
      containers:
        - name: test-pod-con
          resources:
            requests:
              cpu: "200"
              memory: "20000Gi"
    ```  
  
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
    pod/test-pod created
    "Pending"
    
    .. 계속 유지 ..
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS    RESTARTS   AGE
    test-pod   0/1     Pending   0          2m25s
    ```  

- 컨테이너에 설정한 이미지를 찾지 못하는 경우(`Pod` 를 구성하는 모든 컨테이너 이미지)
  - 아무 `.image` 필드를 아래와 같이 변경한 후 실행하면 아래와 같다. 
  
    ```yaml
    image: noimage:notag
    ```  
    
    ```bash
    kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
    pod/test-pod created
    "Pending"
    "Pending"
    "Pending"

    .. 계속 반복 ..
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS                  RESTARTS   AGE
    test-pod   0/1     Init:ImagePullBackOff   0          3m30s
    ```  

- `.nodeSelector`, `.affinity` 에 의해 스케쥴링에 실패한 경우
  - `.nodeSelector` 에 존재하지 않는 노드 정보를 아래와 같이 설정한 후 실행하면 아래와 같다.

    ```yaml
    spec:
      nodeSelector:
      kubernetes.io/hostname: nonode
    ```  
    
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
    pod/test-pod created
    "Pending"
    
    .. 계속 유지 ..
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS    RESTARTS   AGE
    test-pod   0/1     Pending   0          2m45s
    ```  
    

#### Succeeded
`Succeeded` 는 말그대로 `Pod` 의 모든 컨테이너가 주어진 동작을 모두 마치고 정상적으로 종료된 단계이다.  
`test-pod.yaml` 템플릿에 아래와 같이 추가한 후 실행하면 결과를 확인 할 수 있다.  

```yaml
spec:
  containers:
    - name: test-pod-con
      command: [ "/bin/bash", "-c", "sleep 10;" ]
  restartPolicy: Never # 수정 필요
```  

```bash
$ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
pod/test-pod created
"Pending"
"Pending"
"Pending"
"Running"
"Succeeded"

$ kubectl get pod test-pod
NAME       READY   STATUS      RESTARTS   AGE
test-pod   0/1     Completed   0          53s
```  

`Running` 을 지나 `Succeeded` 단계로 진입하는 것을 확인 할 수 있다. 

#### Failed
`Failed` 는 `initContainers` 를 포함해서 `Pod` 를 구성하는 컨테이너 중 하나가 실패로 종료된 경우이다. 
아래와 같이 `.command` 를 수정해 준다.  

```yaml
spec:
  - name: test-pod-con
    command: [ "/bin/bash", "-c", "sleep 10; nocommand" ]
  restartPolicy: Never # 수정 필요
```  

```bash
kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.phase'
pod/test-pod created
"Pending"
"Pending"
"Pending"
"Running"
"Failed"

$ kubectl get pod test-pod
NAME       READY   STATUS   RESTARTS   AGE
test-pod   0/1     Error    0          39s
```  

`Running` 을 지나 `Failed` 단계로 진입하는 것을 확인 할 수 있다.


### Container States
`Kubernetes Pod` 에서는 `Pod Phase` 뿐만 아니라, `Pod` 에 구성된 컨테이너들의 상태를 관리한다. 
컨테이너 상태는 `.lifecycle.postStart`, `.lifecycle.preStop` 에서 이벤트 트리거 동작을 수행할 수 있다.  

`Pod` 이 특정 노드에 할당되면, 컨테이너 생성을 시작하면서 확인 할 수 있는 각 컨테이너의 상태는 아래와 같다. 

> `Container States` 는 노드에 스케쥴링이 된 이후 부터 상태값 조회가 가능하다.  

값|의미
---|---
Waiting|`Running`, `Terminated` 가 아니면 `Waiting` 상태이다. `Waiting` 은 컨테이너 시작을 하는데 필요한 이미지 다운로드, 시크릿 데이터 적용 등이 실행 중인 상태이다. 
Running|컨테이너가 문제 없이 실행 중인 상태이고, `.lifecycle.postStart` 는 이미 실행이 완료된 상태아다. 
Terminated|실행 이후 실행이 완료되거나 어떤 이유에 의해서 실패한 상태이다. `.lifecycle.preStop` 는 `Terminated` 이전에 실행이 완료 된다. 

`Pod` 를 구성하고 있는 컨테이너 들의 상태를 조회하고 싶은 경우 아래 명령어를 사용할 수 있다.  

```bash
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[].state'
```  

#### Running
가장 먼저 정상적으로 수행되는 `Running` 상태는 아래 명령으로 `test-pod.yaml` 템플릿으로 `Pod` 을 실행해 준다. 

```bash
$ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
pod/test-pod created
{
  "waiting": {
    "reason": "PodInitializing"
  }
}
{
  "waiting": {
    "reason": "PodInitializing"
  }
}
{
  "waiting": {
    "reason": "PodInitializing"
  }
}
{
  "running": {
    "startedAt": "2022-02-20T17:28:47Z"
  }
}
{
  "running": {
    "startedAt": "2022-02-20T17:28:47Z"
  }
}
```  

`waiting` 상태를 지나 `running` 상태가 되는 것을 확인 할 수 있다.  

#### Waiting
`waiting` 에서 멈추는 상황은 여러가지가 있겠지만 아래 몇개로 정리 할 수 있다.  

- 컨테이너에 설정한 이미지를 찾지 못하는 경우(`Pod` 를 구성하는 모든 컨테이너 이미지)
  - 아무 `.image` 필드를 아래와 같이 변경한 후 실행하면 아래와 같다.

    ```yaml
    image: noimage:notag
    ```  
     
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
       "waiting": {
         "reason": "PodInitializing"
       }
    }
    {
      "waiting": {
        "message": "rpc error: code = Unknown desc = Error response from daemon: Get \"https://registry-1.docker.io/v2/\": dial tcp: lookup registry-1.docker.io on 172.30.107.78:53: lame referral",
        "reason": "ErrImagePull"
      }
    }
    {
      "waiting": {
        "message": "Back-off pulling image \"noimage:latest\"",
        "reason": "ImagePullBackOff"
      }
    }


    .. 계속 유지 ..
    
    $ kubectl get pod test-pod -w
    NAME       READY   STATUS             RESTARTS   AGE
    test-pod   0/1     ImagePullBackOff   0          2m54s
    ```  
    
- `initContainers` 가 정상 종료 되지 못하거나, 종료되지 않는 경우
  - `initContainers.command` 를 아래 2가지 중 하나로 변경해 본다.  

    ```yaml
    spec:
      initContainers:
        - name: init-con
          command: ["/bin/bash", "-c", "no cmmand"]
    
          or
    
          command: ["/bin/bash", "-c", "sleep 999999999"]
    ```  
    
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
       "waiting": {
         "reason": "PodInitializing"
       }
    }

    .. 계속 유지 ..
    
    .. command: ["/bin/bash", "-c", "no cmmand"] ..
    $ kubectl get pod test-pod
    NAME       READY   STATUS       RESTARTS   AGE
    test-pod   0/1     Init:Error   0          2m42s
    
    .. command: ["/bin/bash", "-c", "sleep 999999999"]
    $ kubectl get pod test-pod
    NAME       READY   STATUS     RESTARTS   AGE
    test-pod   0/1     Init:0/1   0          2m34s
    ```  
    
- `postStart` 가 종료되지 않은 경우
  - `.lifecycle.postStart.exec.command` 를 아래와 같이 변경한다.
  
    ```yaml
    spec:
      containers:
        - name: test-pod-con
          lifecycle:
            postStart:
              exec:
                command: ["/bin/bash", "-c", "sleep 999999999"]
    ```  
    
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
       "waiting": {
         "reason": "PodInitializing"
       }
    }

    .. 계속 유지 ..
    
    kubectl get pod test-pod
    NAME       READY   STATUS            RESTARTS   AGE
    test-pod   0/1     PodInitializing   0          2m10s
    ```  

#### Terminated
`terminated` 가 되는 상황은 아래 몇가지로 정리해볼 수 있다.  

- `Pod` 의 모든 컨테이너가 정상 종료
  - `.containers.command` 를 아래와 같이 일정 시간 이후 정상 종료하도록 수정 한다.
    
    ```yaml
    spec:
      containers:
        - name: test-pod-con
          command: [ "/bin/bash", "-c", "sleep 10;" ]
    ```  
    
    ```bash
    kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
          "reason": "PodInitializing"
        }
      }
    {
      "waiting": {
          "reason": "PodInitializing"
        }
      }
    {
      "waiting": {
          "reason": "PodInitializing"
        }
      }
    {
      "running": {
          "startedAt": "2022-02-20T18:01:20Z"
        }
      }
    {
      "terminated": {
        "containerID": "docker://8db397ab54ad21ff72232c11b4d489eef0bb471f98ec90e7e255b6d85297b305",
        "exitCode": 0,
        "finishedAt": "2022-02-20T18:01:30Z",
        "reason": "Completed",
        "startedAt": "2022-02-20T18:01:20Z"
      }
    }
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS      RESTARTS   AGE
    test-pod   0/1     Completed   0          88s
    ```  

- `Pod` 의 컨테이너 중 하나에서 실행 실패
  - `.containers.command` 를 아래와 같이 일정 시간 이후 실패하도록 수정 한다. 
  
    ```yaml
    spec:
      containers:
      - name: test-pod-con
        command: [ "/bin/bash", "-c", "sleep 10; nocommand" ]
    ```  
   
    ```bash
    kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "running": {
        "startedAt": "2022-02-20T18:05:51Z"
      }
    }
    {
      "terminated": {
        "containerID": "docker://fb264389d039e6fc5cdc3014953cb7a5284809b4e7570e931eeeadfef522f3ba",
        "exitCode": 127,
        "finishedAt": "2022-02-20T18:06:01Z",
        "reason": "Error",
        "startedAt": "2022-02-20T18:05:51Z"
      }
    }
   
    $ kubectl get pod test-pod
    NAME       READY   STATUS   RESTARTS   AGE
    test-pod   0/1     Error    0          26s
    ```  
  
- `postStart` 에서 실행이 실패하는 경우 
  - `.lifecycel.postStart.exec.command` 를 아래와 같이 실패하도록 수정 한다.  
  
    ```yaml
    spec:
      containers:
        - name: test-pod-con
          lifecycle:
            postStart:
              exec:
                command: command: ["/bin/bash", "-c", "nocommand"] 
    ```  
  
    ```bash
    kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.containerStatuses[].state'
    pod/test-pod created
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "waiting": {
        "reason": "PodInitializing"
      }
    }
    {
      "terminated": {
        "containerID": "docker://0b9d69c963f061ed336c16d4898240b14fff5acbe53a633cbee91efd541dc8e7",
        "exitCode": 0,
        "finishedAt": "2022-02-20T18:12:35Z",
        "reason": "Completed",
        "startedAt": "2022-02-20T18:12:30Z"
      }
    }
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS      RESTARTS   AGE
    test-pod   0/1     Completed   0          46s
    ```  
    
  - 특이하게 `postStart` 의 커멘드 실패는 정상 종료로 간주 된다.


### Container restart policy
`Pod` 을 구성하는 모든 컨테이너에 `.spec.restartPolicy` 를 통해 재시작 정책을 설정할 수 있다. 
`Always`, `OnFailure`, `Never` 값이 있고 기본값은 `Always` 이다. 
해당 값을 설정하면 `Pod` 의 모든 컨테이너에 동일하게 적용되고, 재시작은 동일한 노드에서만 수행된다. 
컨테이너 재시작은 최대 5분으로 제한되는 `back-off` 지연값을 10초에서 부터 2배씩 늘려가며 다시 시작하게 된다. 
만약 10분 동안 문제없이 지속되면 `back-off` 타이머를 재설정 한다.  

### Containers probes
`probe`(프로브) 설정된 컨테이너를 대상으로 `kubelet` 이 주기적으로 수행하는 진단이다. 
진단 수행방법은 코드 실행하거나 네트워크 요청을 전송하는 등 다양한 방법이 존재한다. 
모든 진단이 성공한 경우 `Pod Conditions` 의 `Containers Ready` 상태가 `True` 로 변하게 된다.  

방법|설명
exec|컨테이너 내에서 지정된 명령어를 실행하는 방법으로, 명령어의 종료 코드가 0(exit 0) 인경우 진단 성공으로 판별한다. 
grpc|`gRPC` 를 사용해서 원격 프로시저 호출을 수행한다. 
httpGet|지정된 포트 경로에서 해당 컨테이너의 IP주소에 대한 `HTTP GET` 요청을 수행한다. 응답 코드가 200 이상 400 미만이면 진단 성공으로 판별한다. 
tcpSocket|지정된 포트에서 컨테이너 IP주소에 대해 TCP 검사를 수행하는 방법으로 포트가 활성화 되어 있다면 진단 성공으로 판별한다. 

프로브 결과는 아래 중 하나의 값을 가지게 된다. 

값|의미
---|---
Success|컨테이너 진단 성공
Failure|컨테이너 진단 실패
Unknown|진단 자체가 실패

프로브의 종류는 아래 3가지가 있고 필요한 것만 사용할 수도 있고 모두다 활요할 수 있다.  

프로브 이름|설명
startupProbe|컨테이너 내의 애플리케이션이 시작되었는지를 진단한다. 성공 할 떄까지 나머지 프로브는 수행되지 않는다. 해당 프로브가 실패하면 컨테이너를 죽이고 컨테이너 재시작 정책에 따라 처리된다. 해당 프로브가 설정되지 않은 경우 기본값은 `Success` 이다.
livenessProbe|컨테이너의 동작 여부를 진단한다. 해당 프로브가 실패하면 컨테이너를 죽이고 재시작 정책에 따라 처리된다. 설정하지 않은 경우 기본 값은 `Succeess` 이다. 
readinessProbe|컨테이너가 요청을 처리할 수 있는 지 진단한다. 해당 프로브가 실패하면 엔드포인트 컨트롤러는 `Pod` 와 연관된 모든 서비스에서 `Pod` 의 `IP` 를 제거한다. 기본 값은 `Failure` 로 시작하고, 만약 해당 프로브가 설정되지 않았다면 `Success` 값을 갖게 된다. 


### ReadinessGates
위에서 알아본 `Containers probe` 의 경우 컨테이너 내부 상태에 대한 진단이라면, 
`readinessGates` 는 외부 상태에 대한 진단으로 `Pod Condition` 에 기본 컨디션 외에 추가 컨디션을 적용한다. 
그래서 `readinessGates` 에 설정된 컨디션까지 `True` 가 돼야, 
`Pod` 이 실제 서비스 요청을 받아 처리 할 수 있는 상태인 `Ready` 가 `True` 가 된다.  

`readinessGates` 가 필요한 이유는 외부의 로드밸런서를 예로 들 수 있다. 
`Pod` 는 모두 준비된 상태이지만, 
로드 밸런서에 아직 해당 `Pod` 에 대한 정보가 업데이트 되지 않았거나, 
구성이 완료되지 않아 해당 `Pod` 으로 트래픽 전달이 제대로 되지 않는 상태가 발생할 수 있다.  

위와 같은 상황에서 효율적으로 활용할 수 있는게 `readinessGates` 이다. 
`readinessGates` 는 `.status.conditions` 필드에 추가한 컨디션의 상태를 `True` 로 직접 설정해 주는 방법으로 상태를 변경할 수 있다. 
그러므로 로드 밸런서에 대한 설정 및 `Pod` 에 대한 모든 설정이 완료된 경우, 
`Fetch API` 를 통해 `readinessGates` 의 컨디션 값을 `True` 로 변경해주면 `Ready` 상태도 `True` 가 되면서 실제 서비스 요청을 처리할 수 있는 상태가 된다.  

`readinessGates` 는 아래와 같이 `Pod` 오브젝트에 설정할 수 있다.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  readinessGates:
  - conditionType: "start-routing"
  containers:
  - name: test-pod-con
    image: nginx:latest
```  

앞서 설명한 것처럼 `start-routing` 이라는 `readinessGates` 의 `status` 값을 `True` 로 만들어 줘야 `Pod` 으로 트래픽이 정상적으로 전달 된다. 
실제 사용 방법은 이후 설명할 `Pod Conditions` 에서 다뤄 보도록 한다.  


### Pod Conditions
`Pod` 은 앞서 설명한 단계를 거치며 실행되면서,
현재 `Pod` 상태를 검증 할 수 있는 여러 컨디션을 검사하게 된다.

값|의미
---|---
PodScheduled|`Pod` 이 특정 노드에 스케쥴 된 상태
Initialized|`initContainers` 가 모드 성공적으로 완료된 상태
ContainersReady|`Pod` 을 구성하고 있는 모든 컨테이너가 준비된 상태(`Container probes` 포함)
Ready|`Pod` 는 요청을 처리할 수 있는 상태로 매칭되는 모든 서비스의 로드 밸런서에 추가될 수 있는 상태(`readinessGates` 포함)

컨디션은 위 표에 나열한 순서로 검사하게 되고, 첫 번째 순서의 검사가 정상 이여야 이후 컨디션 검사를 수행하는 구조이다.  

각 컨디션들은 아래 필드를 가지게 된다.

필드|설명
---|---
type|컨디션 이름(PodScheduled, Initialized, ..)
status|컨디션의 검증 여부로 `True`, `False`, `Unknown` 값을 갖음
lastProbeTime|컨디션이 마지막으로 프로브된 시간
lastTransitionTime|`Pod` 이 한 상태에서 다른 상태로 전환된 마지막 시간
reason|컨디션이 마지막으로 전환된 이유
message|마지막 상태 전환에 대한 세부 정보

`Pod` 의 커디션을 확인하기 위해서는 아래 명령어를 사용할 수 있다.  

```bash
kubectl get pod <pod-name> -o json | jq '.status.conditions'
```  

#### PodScheduled
`PodScheduled` 검사가 실패하는 경우는 스케쥴링 관련으로 예시는 아래 몇가지가 있다.  

- `.resources` 에 설정한 리소스로 인해 스케쥴링에 실패한 경우
  - `.resources.requests` 설정으로 아주 큰 숫자를 아래와 같이 추가 후 실행하면 아래와 같다.

    ```yaml
    spec:
    containers:
      - name: test-pod-con
      resources:
        requests:
        cpu: "200"
        memory: "20000Gi"
    ```  
    
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
    pod/test-pod created
    [
        {
            "lastProbeTime": null,
            "lastTransitionTime": "2022-02-20T18:27:56Z",
            "message": "0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.",
            "reason": "Unschedulable",
            "status": "False",
            "type": "PodScheduled"
        }
    ]
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS    RESTARTS   AGE
    test-pod   0/1     Pending   0          37s
    ```  


- `.nodeSelector`, `.affinity` 에 의해 스케쥴링에 실패한 경우
  - `.nodeSelector` 에 존재하지 않는 노드 정보를 아래와 같이 설정한 후 실행하면 아래와 같다.

    ```yaml
    spec:
      nodeSelector:
        kubernetes.io/hostname: nonode
    ```  
   
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
    pod/test-pod created
    [
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:25:41Z",
        "message": "0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.",
        "reason": "Unschedulable",
        "status": "False",
        "type": "PodScheduled"
      }
    ]
    
    $ kubectl get pod test-pod
    NAME       READY   STATUS    RESTARTS   AGE
    test-pod   0/1     Pending   0          59s
    ```  
   

#### Initialized
`Initialized` 검사 실패는 `initContainers` 에 대한 부분으로 예시는 아래와 같다.  

- `.initContainers.image` 의 이미지를 못 찾는 경우
  
    ```yaml
    spec:
    initContainers:
      - name: init-con
        image: noimage:notag
    ```  
  
    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
    pod/test-pod created
    [
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:35:06Z",
        "message": "containers with incomplete status: [init-con]",
        "reason": "ContainersNotInitialized",
        "status": "False",
        "type": "Initialized"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:35:06Z",
        "message": "containers with unready status: [test-pod-con]",
        "reason": "ContainersNotReady",
        "status": "False",
        "type": "Ready"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:35:06Z",
        "message": "containers with unready status: [test-pod-con]",
        "reason": "ContainersNotReady",
        "status": "False",
        "type": "ContainersReady"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:35:06Z",
        "status": "True",
        "type": "PodScheduled"
      }
    ]
  
    $ kubectl get pod test-pod
    NAME       READY   STATUS                  RESTARTS   AGE
    test-pod   0/1     Init:ImagePullBackOff   0          78s
    ```  

- `.initContainers.command` 가 실패하거나 종료되지 않은 경우
  - `initContainers.command` 를 아래 2가지 중 하나로 변경해 본다.

    ```yaml
    spec:
    initContainers:
      - name: init-con
        command: ["/bin/bash", "-c", "no cmmand"]
      
        or
      
        command: ["/bin/bash", "-c", "sleep 999999999"]
    ```  

    ```bash
    $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
    pod/test-pod created
    [
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:37:40Z",
        "message": "containers with incomplete status: [init-con]",
        "reason": "ContainersNotInitialized",
        "status": "False",
        "type": "Initialized"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:37:40Z",
        "message": "containers with unready status: [test-pod-con]",
        "reason": "ContainersNotReady",
        "status": "False",
        "type": "Ready"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:37:40Z",
        "message": "containers with unready status: [test-pod-con]",
        "reason": "ContainersNotReady",
        "status": "False",
        "type": "ContainersReady"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2022-02-20T18:37:40Z",
        "status": "True",
        "type": "PodScheduled"
      }
    ]
  
    .. 계속 유지 ..
    
    .. command: ["/bin/bash", "-c", "no cmmand"] ..
    $ kubectl get pod test-pod
    NAME       READY   STATUS       RESTARTS   AGE
    test-pod   0/1     Init:Error   0          2m42s
    
    .. command: ["/bin/bash", "-c", "sleep 999999999"] ..
    $ kubectl get pod test-pod
    NAME       READY   STATUS     RESTARTS   AGE
    test-pod   0/1     Init:0/1   0          2m34s
    ```  
    
#### ContainersReady
`ContainersReady` 검사 실패는 `initContainers` 를 제외한 `Pod` 의 모든 컨테이너가 정상이 아닌 상태이므로, 예시는 아래와 같다.  

- `.containers.image` 를 못 찾는 경우

  ```yaml
  spec:
    containers:
      - name: test-pod-con
        image: noimage:notag
  ```  
  
  ```bash
  $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
  pod/test-pod created
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:14:52Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:14:45Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:14:45Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:14:45Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  $ kubectl get pod test-pod
  NAME       READY   STATUS         RESTARTS   AGE
  test-pod   0/1     ErrImagePull   0          45s
  ```  

- `.containers` 에 정의된 컨테이너 실행이 실패하는 경우

  ```yaml
  spec:
    containers:
      - name: test-pod-con
        command: [ "/bin/bash", "-c", "sleep 10; nocommand" ]
  ```  
  
  ```bash
  $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
  pod/test-pod created
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:17:02Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:16:56Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:16:56Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:16:56Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  $ kubectl get pod test-pod
  NAME       READY   STATUS   RESTARTS   AGE
  test-pod   0/1     Error    0          61s
  ```  

- `.containers.lifecycle.postStart` 가 실패하는 경우

  ```yaml
  spec:
    containers:
      - name: test-pod-con
        lifecycle:
          postStart:
            exec:
              command: ["/bin/bash", "-c", "nocommand"]
  ```  
  
  ```bash
  $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
  pod/test-pod created
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:18:54Z",
      "reason": "PodCompleted",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:18:47Z",
      "reason": "PodCompleted",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:18:47Z",
      "reason": "PodCompleted",
      "status": "False",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:18:47Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  $ kubectl get pod test-pod
  NAME       READY   STATUS      RESTARTS   AGE
  test-pod   0/1     Completed   0          82s
  ```  

- `Container probes` 가 실패하는 경우(`startupProbe`, `livenessProbe`, `readinessProbe`)


  ```yaml
  spec:
    containers:
      - name: test-pod-con
        startupProbe:
          exec:
            command: ["/bin/bash", "-c", "nocommand"]

          # or


        livenessProbe:
          exec:
            command: ["/bin/bash", "-c", "nocommand"]

          # or

         readinessProbe:
           httpGet:
             path: /noindex.no
             port: 9999

  ```  

  ```bash
  $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
  pod/test-pod created
  
  .. startupProbe, livenessProbe ..
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:21:43Z",
      "reason": "PodCompleted",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:21:36Z",
      "reason": "PodCompleted",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:21:36Z",
      "reason": "PodCompleted",
      "status": "False",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:21:36Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  
  .. readinessProbe ..
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:29:37Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:29:30Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:29:30Z",
      "message": "containers with unready status: [test-pod-con]",
      "reason": "ContainersNotReady",
      "status": "False",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:29:30Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  .. startupProbe, livenessProbe ..
  $ kubectl get pod test-pod
  NAME       READY   STATUS      RESTARTS   AGE
  test-pod   0/1     Completed   0          2m8s
  
  .. readinessProbe ..
  $ kubectl get pod test-pod
  NAME       READY   STATUS    RESTARTS   AGE
  test-pod   0/1     Running   0          74s
  ```  

#### Ready
`Ready` 실패는 `Pod` 이 실제 서비스 트래픽을 받을 수 없는 상태이므로 예시는 아래와 같다. 

- `readinessGates` 설정되었지만, `status` 값이 `False` 인 경우
  - 먼저 아래 처럼 `readinessGates` 를 설정해 준다.  
	
  ```yaml
  spec:
    readinessGates:
      - conditionType: "start-routing"
  ```  
  
  - 현재 상태에서 `Pod` 을 실행하면 `start-routing` 상태를 찾을 수 없다는 이유로 `Ready` 가 실패한다. 하지만 `kubectl get` 에서는 `Ready` 가 `1/1` 이지만, 연결된 서비스에 요청을 해도 요청을 수행되지 않는다. 
	
  ```bash
  $ kubectl apply -f test-pod.yaml; kubectl get pod -w test-pod -o json | jq '.status.conditions'
  pod/test-pod created
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:34:01Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:33:54Z",
      "message": "corresponding condition of pod readiness gate \"start-routing\" does not exist.",
      "reason": "ReadinessGatesNotReady",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:34:19Z",
      "status": "True",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:33:54Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  $ kubectl get pod test-pod
  NAME       READY   STATUS    RESTARTS   AGE
  test-pod   1/1     Running   0          45s
  ```  
  
  - `readinessGates` 는 `Kubernetes Cluster API` 를 사용해서 업데이트가 필요하므로 프록시 설정을 해준다.  
	
  ```bash
  $ kubectl proxy --port 12345 &
  Starting to serve on 127.0.0.1:12345
  ```  
  - `readinessGates` 설정을 위해 `curl` 을 사용해 `starting-routing` 컨디션을 `True` 로 만들어 준다. 
	
  ```bash
  $ curl -k \
     -H "Content-Type: application/json-patch+json" \
     -X PATCH http://localhost:12345/api/v1/namespaces/default/pods/test-pod/status \
     --data '[{ "op": "replace", "path": "/status/conditions/0",
                "value": { "type": "start-routing", "status": "True" }}]'
  ```  
  
  - 그후 다시 컨디션을 조회하면 아래와 같이 모든 상태가 `True` 로 정상인 상태인 것을 확인 할 수 있다. `readinessGates` 가 있는 경우 해당 시점 부터 `Pod` 으로 실제 트래픽이 전달되게 된다. 
	
  ```bash
  $ kubectl get pod -w test-pod -o json | jq '.status.conditions'
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": null,
      "status": "True",
      "type": "start-routing"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:34:01Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:38:48Z",
      "status": "True",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:34:19Z",
      "status": "True",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:33:54Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  
  $ kubectl get pod test-pod
  NAME       READY   STATUS    RESTARTS   AGE
  test-pod   1/1     Running   0          2m20s
  ```  

  - 다시 `readinessGates` 상태를 `False` 로 변경하고 싶다면 동일하게 `curl` 을 사용해서 `start-routing` 컨디션을 `False` 로 변경하면 `Read` 컨디션 검사까지 실패하고, `Pod` 은 더이상 트래픽을 받지 않는다. 
	
  ```bash
  $ curl -k \
     -H "Content-Type: application/json-patch+json" \
     -X PATCH http://localhost:12345/api/v1/namespaces/default/pods/test-pod/status \
     --data '[{ "op": "replace", "path": "/status/conditions/0",
                "value": { "type": "start-routing", "status": "False" }}]'
  
  $ kubectl get pod -w test-pod -o json | jq '.status.conditions'
  [
    {
      "lastProbeTime": null,
      "lastTransitionTime": null,
      "status": "False",
      "type": "start-routing"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:42:41Z",
      "status": "True",
      "type": "Initialized"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:44:45Z",
      "message": "the status of pod readiness gate \"start-routing\" is not \"True\", but False",
      "reason": "ReadinessGatesNotReady",
      "status": "False",
      "type": "Ready"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:42:59Z",
      "status": "True",
      "type": "ContainersReady"
    },
    {
      "lastProbeTime": null,
      "lastTransitionTime": "2022-02-20T19:42:34Z",
      "status": "True",
      "type": "PodScheduled"
    }
  ]
  ```  
  
### 정리

......

---
## Reference
[Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)  
[Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)  
[Ready? A Deep Dive into Pod Readiness Gates for Service Health](https://www.youtube.com/watch?v=Vw9GmSeomFg)  
[Improving Application Availability with Pod Readiness Gates](https://martinheinz.dev/blog/63)  







