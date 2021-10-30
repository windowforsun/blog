--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Kubernetes Jenkins Master Slave(Agent) 구성 및 사용하기"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Kubernetes 클러스터에서 동적으로 Jenkins Agent 를 실행하는 Jenkins Master/Slave 구조를 구성해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Jenkins
  - Jenkins Agent
  - Jenkins Master/Slave
  - Jnlp
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
$ kubectl get serviceaccount,role,rolebinding -n jenkins
NAME                     SECRETS   AGE
serviceaccount/default   1         2d19h
serviceaccount/jenkins   1         24s

NAME                                     CREATED AT
role.rbac.authorization.k8s.io/jenkins   2021-10-28T14:45:49Z

NAME                                            ROLE           AGE
rolebinding.rbac.authorization.k8s.io/jenkins   Role/jenkins   24s
```  

### Jenkins 플러그인 추가 및 설정
`Kubernetes` 클러스터에 `Master/Slave` 를 적용하기 위해서는 `Kubernetes` 플러그인이 필요한데, `Jenkins 관리 > 플러그인 관리 > 설치 가능` 을 눌러 검색하고 설치해 준다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-1.png)  
![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-1-1.png)  

설치가 완료되면 `Jenkins 관리 > 시스템 설정 > Cloud(가장 아래쪽)` 으로 가서 아래 버튼을 눌러 주고 사진을 따라 설정을 해준다. 

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-2.png)  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-3.png)  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-4.png)  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-5.png)  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-6.png)  

이제 `Save` 혹은 `Apply` 를 눌러 설정을 적용해 준다.  

### Kubernetes Jenkins Slave(Agent) 생성하기 
`새로운 Item` 을 눌러 `test-agent` 라는 `Pipeline` 아이템을 생성한다. 

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-7.png)  

이제 아래 `Script` 에 `Agent` 로 실행할 템플릿을 작성해 주면된다. 
템플릿 작성 관련해서는 [여기](https://github.com/jenkinsci/kubernetes-plugin#configuration-reference)
를 참고할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-8.png)  

#### Simple Template
가장 먼저 기본으로 실행하는 `jnlp` 컨테이너만 실행하고 간단한 `echo` 명령을 수행하는 템플릿을 테스트해 본다.  

```groovy
 podTemplate(
         containers: [
             containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:4.10-2-jdk11')
         ]
     )
 {
     node(POD_LABEL) {
         stage('Run shell') {
             sh 'hostname'
             sh 'echo hello world'
         }
     }
 }
```  

기본으로 실행하는 `jnlp` 컨테이너 이미지나 관련 설정 변경이 필요한 경우 위와 같이 `containerTemplate` 을 사용해서 수정 할 수 있다.  
`Build Now` 를 눌러 빌드를 실행하면, `Jenkins 관리 > 노드 관리` 에 들어가면 아래와 같이 `test-agent` 로 시작하는 `Agent` 가 실행 된것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-9.png)  

`Kubernetes` 클러스터에서도 실행된 `Pod` 을 확인 할 수 있다.  

```bash
$ kubectl get pod -n jenkins
NAME                             READY   STATUS    RESTARTS   AGE
jenkins-0                        1/1     Running   0          71m
test-agent-1-0zb9x-8sh3j-kfdrk   1/1     Running   0          22s
```  

실행된 `Agent Pod` 는 작업을 완료하고 종료된다. 
그리고 `Build History` 에서 로그를 확인하면 아래와 같이, `jnlp` 컨테이너에서 템플릿에 작성한 명령이 수행된 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-10.png)  


#### Multiple Container Template
이번에는 기본 컨테이너인 `jnlp` 외에 빌드 혹은 배치 작업을 수행하는 용도로 사용할 수 있는 추가 컨테이너를 실행하는 템플릿을 작성해 본다.  

```groovy
 podTemplate(yaml: '''
     apiVersion: v1
     kind: Pod
     metadata:
       labels: 
         some-label: some-label-value
     spec:
       containers:
       - name: busybox
         image: busybox
         command:
         - sleep
         args:
         - 99d
       - name: ubuntu
         image: ubuntu
         command:
         - sleep
         args:
         - 99d
     '''
     ,containers: [
         containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:4.10-2-jdk11')
     ]) {
     node(POD_LABEL) {
       container('busybox') {
         echo POD_CONTAINER // displays 'busybox'
         sh 'hostname'
         echo 'Im busybox'
       }
        container('ubuntu') {
            echo POD_CONTAINER // displays 'ubuntu'
            sh 'hostname'
            echo 'Im ubuntu'
        }
     }
 }
```  

결과는 아래와 같이 추가한 `busybox`, `ubuntu` 컨테이너에서 정의된 명령이 수행된 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-11.png)  


#### Multiple Stage Template
지금까지는 단일 `Stage` 로 구성된 템플릿을 사용했는데, 아래와 같이 여러 스테이지를 정의해서 파이프라인을 구성할 수 있다.  

```groovy
 podTemplate(yaml: '''
     apiVersion: v1
     kind: Pod
     metadata:
       labels: 
         some-label: some-label-value
     spec:
       containers:
       - name: busybox
         image: busybox
         command:
         - sleep
         args:
         - 99d
       - name: ubuntu
         image: ubuntu
         command:
         - sleep
         args:
         - 99d
     '''
    ,containers: [
    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:4.10-2-jdk11')
]) {
    node(POD_LABEL) {
        stage('Run Busybox') {
            container('busybox') {
                echo POD_CONTAINER // displays 'busybox'
                sh 'hostname'
                echo 'Im busybox stage'
            }
        }
        stage('Run Ubuntu') {
            container('ubuntu') {
                echo POD_CONTAINER // displays 'ubuntu'
                sh 'hostname'
                echo 'Im ubuntu stage'
            }
        }
    }
}
```  


결과는 아래와 같이 추가한 `busybox`, `ubuntu` 컨테이너가 각 스테이지에서 명령이 수행된 것을 확인 할 수 있다.

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-12.png)  

그리고 아래처럼 구성한 스테이지와 각 스테이지의 소요 시간도 확인할 수 있다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-13.png)  


#### Execute Docker Image
이번에는 배치 잡처럼 주기적으로 특정 동작을 `Docker Image` 를 통해 수행하는 방법에 대해서 알아본다. 
아래는 배치 1 ~ 10까지 차례대로 출력하는 간단한 배치잡 테스트용 `Dockerfile` 이다.  

```dockerfile
FROM centos:7

ENV param=default

RUN echo "#!/bin/bash" >> /exec.sh
RUN echo "for ((i=1; i<= 10; i++)) do" >> /exec.sh
RUN echo "echo \"Running \$i \$param\";" >> /exec.sh
RUN echo "sleep 1" >> /exec.sh
RUN echo "done" >> /exec.sh

RUN chmod 755 /exec.sh
ENTRYPOINT ["/bin/bash", "-c", "/exec.sh"]
```  

`jenkins-job` 이라는 태그로 빌드하고 테스트를 위해 `Docker Hub` 에 푸시해준다.  

```bash
$ docker build -t windowforsun/jenkins-job .
Sending build context to Docker daemon  3.072kB
Step 1/8 : FROM centos:7
 ---> eeb6ee3f44bd
 
 .. 생략 ..
 
Step 8/8 : ENTRYPOINT ["/bin/bash", "-c", "/exec.sh"]
 ---> Running in 31fd2207a339
Removing intermediate container 31fd2207a339
 ---> 0697e3e2161e
Successfully built 0697e3e2161e
Successfully tagged windowforsun/jenkins-job:latest

$ docker push windowforsun/jenkins-job
Using default tag: latest
The push refers to repository [docker.io/windowforsun/jenkins-job]

.. 생략 ..

latest: digest: sha256:b270c80616337cefd4c8a6368f5afd884d9b4e360ebb3ba7322349fcc5098f78 size: 1771
```  

아래는 `jenkins-job` 이라는 이미지를 `Jenkins Agent` 로 실행하는 파이프라인 템플릿이다.  

```groovy
def LABEL = "agent-${UUID.randomUUID().toString()}"
def IMAGE = "windowforsun/jenkins-job:latest"
def PARAM = "Jenkins Test!"

podTemplate(
        label: LABEL,
        containers: [
            containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:4.10-2-jdk11'),
            containerTemplate(name: 'docker', image: 'docker', commant: 'cat', ttyEnabled: true)
        ],
        volumes: [
            hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
        ]
)
{
    node(LABEL) {
        try {
            stage('Pull Docker Image') {
                container('docker') {
                    sh """
                        docker pull ${IMAGE}
                    """
                }
            }
            
            stage('Run Batch') {
                container('docker') {
                    sh """
                        docker run -e param=${JOB} --rm ${IMAGE}

                    """
                }
            }
            
            currentBuild.result = 'SUCCESS'
        } catch(err) {
            currentBuild.result = 'FAILURE'
        } finally {
            // if(currentBuild.getPreviousBuild().result == 'FAILURE' && currentBuild.result == 'SUCCESS') {
            //     currentBuild.result = 'FIX'
            // }
        }
    }        
}
```  

`Jenkins Agent Pod` 에 `docker` 를 사용할 수 있는 컨테이너를 추가로 올리고 해당 컨테이너와 호스트의 `Docker Daemon` 을 마운트한다. 
그리고 마운트한 `docker` 컨테이너에서 배치 잡으로 사용할 이미지를 도커 컨테이너로 다시 실행하는 방법으로 수행된다. 
실행된 배치의 출력 로그와 결과는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-14.png)  

![그림 1]({{site.baseurl}}/img/kubernetes/practice-jenkins-master-slave-15.png)  


---
## Reference
[HOW TO INSTALL JENKINS ON A KUBERNETES CLUSTER](https://admintuts.net/server-admin/how-to-install-jenkins-on-a-kubernetes-cluster/)  
[jenkinsci/kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin)  







