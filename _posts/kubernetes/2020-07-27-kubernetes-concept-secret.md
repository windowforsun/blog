--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 시크릿(Secret)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 민감한 정보를 저장하고 파드에게 제공해주는 시크릿에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Secret
toc: true
use_math: true
---  

## 시크릿
시크릿(`Secret`)은 비밀번호, `OAuth token`, `SSH key` 와 같이 보안상 민감할 수 있는 정보를 저장하는 역할을 수행한다. 
이러한 정보를 별도의 공간에 저장해 뒀다가 파드의 실제 컨테이너를 실행할 때 제공해 사용할 수 있도록 한다.  

쿠버네티스에서 시크릿은 크게 2가지 종류로 분류 할 수 있다. 
1. 내장(`built-in`) 시크릿
1. 사용자 정의 시크릿

내장 시크릿은 쿠버네티스 클러스터에서 쿠버네티스 `API` 접근 시에 사용하는 시크릿이다. 
클러스터에서 `ServiceAccount` 계정을 생성하면 자동으로 관련 시크릿을 생성하고, 
이를 통해 `ServiceAccount` 가 권한이 있는 `API` 를 호출해 동작을 수행할 수 있도록 한다.    

사용자 정의 시크릿은 사용자가 별도로 생성한 시크릿을 의미한다. 
시크릿은 `kubectl create secret` 명령으로 생성할 수 있고, 
템플릿을 사용해서 생성할 수도 있다. 

### 커맨드 라인으로 생성하기
커맨드 라인으로 시크릿을 생성하는 방법은 앞서 언급 했던 것과 같이, 
`kubectl create secret` 명령으로 가능하다.  

테스트를 위해 `echo -n ` 명령을 사용해서 사용자 이름과 비밀번호 정보를 담고있는 파일을 생성한다. 
그리고 `kubectl create secret generic <시크릿 이름>` 명령으로 시크릿을 생성한다. 

```bash
$ echo -n 'myname' > username
$ echo -n 'mypass' > password
$ ls
password  username
$ kubectl create secret generic my-secret --from-file=username --from-file=password
secret/my-secret created
```  

생성된 `my-secret` 을 `kubectl get secret my-secret -o yaml` 명령을 사용해서 `yaml` 형식으로 확인하면 아래와 같다. 

```bash
$ kubectl get secret my-secret -o yaml
apiVersion: v1
data:
  password: bXlwYXNz
  username: bXluYW1l
kind: Secret
metadata:
  creationTimestamp: "2020-07-26T15:30:14Z"
  name: my-secret
  namespace: default
  resourceVersion: "863530"
  selfLink: /api/v1/namespaces/default/secrets/my-secret
  uid: ad0417d7-0a72-44eb-91b3-c17035135452
type: Opaque
```  

`.data` 필드를 보면 파일 이름인 `username` 과 `password` 키로 `base64` 로 인코딩된 문자가 값으로 설정된 것을 확인 할 수 있다. 
이렇게 시크릿을 생성하게 되면 내용을 `base64` 로 인코딩해서 사용한다.  

`base64` 내용이 실제 값과 동일한지 확인하면 아래와 같다. 

```bash
$ echo bXlwYXNz | base64 --decode
mypass
$ echo bXluYW1l | base64 --decode
myname
```  

### 템플릿으로 생성하기
아래는 동일한 시크릿을 템플릿으로 구성한 예시이다. 

```yaml
# my-secret-yaml.yaml

apiVersion: v1
kind: Secret
metadata:
  name: my-secret-yaml
type: Opaque
data:
  username: bXluYW1l
  password: bXlwYXNz
```  

구성한 템플릿에서 `.type` 의 값으로 `Opaque` 를 설정했다. 
`.type` 의 값으로 설정할 수 있는 종류는 아래와 같다. 
- `Opaque` : `key-value` 형식으로 사용자 임의의 데이터를 설정할 수 있다.(기본 값)
- `kubernetes.io/service-account-token` : 쿠버네티스 인증 토큰을 저장한다.
- `kubernetes.io/dockerconfigjson` : 도커 저장소 인증 정보를 저장한다. 
- `kubernetes.io/tls` : `TLS` 인증서를 저장한다. 

`Opaque` 타입으로 설정할 때, `base64` 로 인코딩된 문자를 사용해서 값으로 설정해야 한다. 
`base64` 문자열은 아래와 같이 `echo -n` 명령으로 생성할 수 있다. 

```bash
$ echo -n 'myname' | base64
bXluYW1l
$ echo -n 'mypass' | base64
bXlwYXNz
```  

`kubectl apply -f my-secret-yaml.yaml` 명령으로 클러스터에 적용하고, 
`kubectl get secret my-secret-yaml -o yaml` 명령으로 생성된 시크릿을 확인하면 아래와 같다. 

```bash
$ kubectl apply -f my-secret-yaml.yaml
secret/my-secret-yaml created
$ kubectl get secret my-secret-yaml -o yaml
apiVersion: v1
data:
  password: bXlwYXNz
  username: bXluYW1l
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"bXlwYXNz","username":"bXluYW1l"},"kind":"Secret","metadata":{"annotations":{},"name":"my-secret-yaml","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2020-07-26T15:37:36Z"
  name: my-secret-yaml
  namespace: default
  resourceVersion: "864360"
  selfLink: /api/v1/namespaces/default/secrets/my-secret-yaml
  uid: 486d28d4-b604-4edf-a8d2-040a354dcc46
type: Opaque
```  

### 임의의 시크릿 데이터 사용하기 
시크릿을 파드의 환경 변수로 설정해서 사용하는 방법에 대해 알아본다. 
아래 템플릿은 생성한 `my-secret-yaml` 시크릿을 파드의 환경 변수로 설정해서 디플로이먼트를 구성하는 예시 템플릿이다. 
컨테이너 이미지는 [ConfigMap]({{site.baseurl}}{% link _posts/kubernetes/2020-07-24-kubernetes-concept-configmap.md %})
와 동일한 이미지를 사용한다. 


```yaml
# deploy-secret-env.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-env-app
  labels:
    app: secret-env-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-env-app
  template:
    metadata:
      labels:
        app: secret-env-app
    spec:
      containers:
        - name: env-app
          image: windowforsun/configmap-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: SECRET_USERNAME
              valueFrom:
                secretKeyRef:
                  name: my-secret-yaml
                  key: username
            - name: SECRET_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-secret-yaml
                  key: password

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: secret-env-app
  name: secret-env-app-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: secret-env-app
  ports:
    - nodePort: 32222
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

`.spec.template.spec.containers[].env[]` 하위에서 `my-secret-yaml` 시크릿을 사용해서 환경변수를 설정하고 있다.
- `.name[0]` : `SECRET_USERNAME` 환경 변수는 `.valueFrom` 과 `.secretKeyRef` 필드를 사용해서, 
`my-secret-yaml` 시크릿 중 `username` 키의 값으로 설정 했다. 
- `.name[1]` : `SECRET_PASSWORD` 환경 변수도 `.valueFrom` 과 `.secretKeyRef` 필드를 사용해서,
`my-secret-yaml` 시크릿 중 `password` 키의 값으로 설정 했다.

`kubectl apply -f deploy-secret-env.yaml` 명령으로 클러스터에 적용한 후, 
브라우저 혹은 `curl` 명령으로 `localhost:32222/env` 에 요청을 하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ curl localhost:32222/env
{
  "SECRET_ENV_APP_SVC_PORT_8080_TCP": "tcp://10.98.101.95:8080",
  "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin",
  "SECRET_ENV_APP_SVC_PORT_8080_TCP_ADDR": "10.98.101.95",
  "KUBERNETES_PORT": "tcp://10.96.0.1:443",
  "JAVA_HOME": "/usr/lib/jvm/java-1.8-openjdk/jre",

  # 시크릿 환경 변수
  "SECRET_USERNAME": "myname",
  "SECRET_ENV_APP_SVC_SERVICE_PORT": "8080",
  "LANG": "C.UTF-8",
  "KUBERNETES_SERVICE_HOST": "10.96.0.1",
  "LD_LIBRARY_PATH": "/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64/server:/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64:/usr/lib/jvm/java-1.8-openjdk/jre/../lib/amd64",
  "JAVA_VERSION": "8u212",
  "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
  "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",

  # 시크리 환경 변수
  "SECRET_PASSWORD": "mypass",
  "SECRET_ENV_APP_SVC_PORT": "tcp://10.98.101.95:8080",
  "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
  "KUBERNETES_SERVICE_PORT": "443",
  "SECRET_ENV_APP_SVC_PORT_8080_TCP_PORT": "8080",
  "SECRET_ENV_APP_SVC_SERVICE_HOST": "10.98.101.95",
  "SECRET_ENV_APP_SVC_PORT_8080_TCP_PROTO": "tcp",
  "HOSTNAME": "secret-env-app-5c5844879f-2584z",
  "JAVA_ALPINE_VERSION": "8.212.04-r0",
  "KUBERNETES_PORT_443_TCP_PORT": "443",
  "KUBERNETES_SERVICE_PORT_HTTPS": "443",
  "HOME": "/root"
}
```  

다음으로는 볼륨을 사용해서 파드에 시크릿을 제공하는 방법에 대해 알아본다. 
환경 변수를 사용해서 제공할 때는 `.spec.template.spec.containers[].env[]` 필드를 사용했지만, 
볼륨을 사용해서 제공할 때는 `.spec.template.spec.containers[].volumes[]` 필드를 사용한다. 
템플릿 예시는 아래와 같다.

```yaml
# deploy-secret-volume.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-volume-app
  labels:
    app: secret-volume-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-volume-app
  template:
    metadata:
      labels:
        app: secret-volume-app
    spec:
      containers:
        - name: env-app
          image: windowforsun/configmap-app:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: secret-volume
              mountPath: /etc/secret-volume
              readOnly: true
      volumes:
        - name: secret-volume
          secret:
            secretName: my-secret-yaml

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: secret-volume-app
  name: secret-volume-app-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: secret-volume-app
  ports:
    - nodePort: 32223
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

`.spec.template.spec.containers[].volumeMounts[]` 필드 하위에 `secret-volume` 에 대한 설정이 있다.
- `.mountPath` : 시크릿 내용을 저정할 경로를 설정한다.
- `.readOnly` : 해당 볼륨을 읽기 전용으로 설정한다. 

그리고 `.spec.template.spec.volumes[]` 필드 하위에 `.secretName` 을 사용해서 설정한 볼륨에 `my-secret-yaml` 을 설정한다.  

`kubectl apply -f deploy-secret-volume.yaml` 명령으로 클러스터에 적용하고, 
브라우저나 `curl` 명령으로 `localhost:32223/volume-env?path=/etc/secret-volume/username` 을 요청하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ curl localhost:32223/volume-env?path=/etc/secret-volume/username
{
  "/etc/secret-volume/username": "myname"
}
```  

`kubectl get pod` 로 실행 중인 파드중 `secret-volume-app` 의 파드 이름을 확인 하고, 
`kubectl exec -it <파드 이름> sh` 명령으로 파드의 컨테이너에 접근해서 경로를 확인하면 아래와 같다. 

```bash
$ kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
secret-env-app-5c5844879f-2584z      1/1     Running   0          68m
secret-volume-app-6ddb88bc96-9zfsc   1/1     Running   0          5m22s
$ kubectl exec -it secret-volume-app-6ddb88bc96-9zfsc sh
/ # ls /etc/secret-volume/
password  username
/ # cat /etc/secret-volume/username
myname
/ # cat /etc/secret-volume/password
mypass
```  

`my-secret-yaml` 시크릿이 파드에 적용 될 때는, `base64` 문자 인코딩값을 디코딩해서 적용되는 것을 확인 할 수 있다. 

### 프라이빗 도커 이미지 사용하기
지금까지 사용한 도커 이미지는 공개된(`public`) 이미지를 사용했다. 
하지만 실제 서비스를 구성하고 제공하기위해서는 프라이빗 이미지를 사용하는 경우가 생길 수 있다. 
이런 경우 쿠버네티스 클러스터에 시크릿을 사용해서 인증정보를 저장한 후 사용할 수 있다.  

도커 저장소에 대한 시크릿은 `kubectl create secret docker-registry` 명령을 사용해서 생성할 수 있다. 
그리고 생성할 때 설정할 수 있는 값은 아래와 같은 것들이 있다. 
- `--docker-server` : 도커 이미지 저장소의 `url` 를 설정한다. 
`Docker Hub` 를 사용한다면 `https://index.docker.io/v1/` 으로 설정하고, 
별도의 저장소의 경우 해당하는 `url` 로 설정한다. 
- `--docker-username` : 사용할 계정 이름을 설정한다. 
- `--docker-password` : 사용하는 계정의 비밀번호를 설정한다.
- `--docker-email` : 사용하는 계정의 이메일을 설정한다.

```bash
$ kubectl create secret docker-registry my-dockersecret --docker-server=https://index.docker.io/v1/ --docker-username=<계정이름> --docker-password=<계정 비밀번호> --docker-email=<계정 이메일>
secret/my-dockersecret created
```  

생성된 `docker-registry` 시크릿을 `kubectl get secret my-dockersecret -o yaml` 명령으로 확인하면 아래와 같다. 

```bash
$ kubectl get secret my-dockersecret -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJ3aW5kb3dmb3JzdW4iLCJwYXNzd29yZCI6IlRoc21sKjEwMDEiLCJlbWFpbCI6IndpbmRvd19mb3Jfc3VuQG5hdmVyLmNvbSIsImF1dGgiOiJkMmx1Wkc5M1ptOXljM1Z1T2xSb2MyMXNLakV3TURFPSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: "2020-07-26T17:16:48Z"
  name: my-dockersecret
  namespace: default
  resourceVersion: "875626"
  selfLink: /api/v1/namespaces/default/secrets/my-dockersecret
  uid: dd44b3ec-e344-4219-94ce-fd227981d96c
type: kubernetes.io/dockerconfigjson
```  

`.data` 하위 `.dockerconfigjson` 필드에 도커 인증 정보 값이 설정된 것을 확인 할 수 있다. 
그리고 `.type` 값이 `kubernetes.io/dockerconfigjson` 인 것을 확인 할 수 있다.  

아래는 프라이빗 이미지를 사용하는 디플로이먼트와 서비스 템플릿의 예시이다. 

```yaml
# deploy-secret-privateimage.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-privateimage-app
  labels:
    app: secret-privateimage-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-privateimage-app
  template:
    metadata:
      labels:
        app: secret-privateimage-app
    spec:
      containers:
        - name: private-app
          image: windowforsun/private-app:latest
          ports:
            - containerPort: 8080
      imagePullSecrets:
        - name: my-dockersecret

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: secret-privateimage-app
  name: secret-privateimage-app-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: secret-privateimage-app
  ports:
    - nodePort: 32224
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

- `.spec.template.spec.containers[].image` : 설정된 이미지는 저장소에서 프라이빗으로 설정된 이미지이다.
- `.spec.template.spec.imagePullSecrets[].name` :  앞서 설정한 `Docker Hub` 시크릿을 설정한다. 

먼저 `.spec.template.spec.imagePullSecrets[].name` 부분을 주석 처리한 후, 
`kubectl apply -f deploy-secret-privateimage.yaml` 명령으로 클러스터에 적용 한다. 
그리고 `kubectl get pod` 명령으로 디플로이먼트에 해당하는 파드를 확인하면 이미지를 가지고 오지 못해 파드가 실행되지 못한 것을 확인 할 수 있다. 

```bash
$ kubectl apply -f deploy-secret-privateimage.yaml
deployment.apps/secret-privateimage-app created
service/secret-privateimage-app-svc created
$ kubectl get pod
NAME                                       READY   STATUS         RESTARTS   AGE
secret-privateimage-app-775cb66584-6rbn4   0/1     ErrImagePull   0          6s
```  

템플릿에서 `.spec.template.spec.imagePullSecrets[].name` 부분의 주석을 제거하고, 
다시 `kubectl apply -f` 명령으로 템플릿을 적용한다. 
그리고 `kubectl get pod` 로 해당 파드를 확인하면 정상적으로 파드가 실행 중인 것을 확인 할 수 있다. 

```bash
$ kubectl apply -f deploy-secret-privateimage.yaml
deployment.apps/secret-privateimage-app configured
service/secret-privateimage-app-svc unchanged
$ kubectl get pod
NAME                                       READY   STATUS    RESTARTS   AGE
secret-privateimage-app-6ccfb4879c-tlcvj   1/1     Running   0          7s
```  

### TLS 인증서 사용하기
`HTTPS` 통신에 필요한 `TLS` 인증서를 저장하는 용도로 시크릿을 사용할 수 있다. 
테스트 용으로 직접 인증서를 생성해 진행한다. 
인증서 생성은 아래와 같은 명령으로 수행한다. 

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=my-tls.com"
Generating a RSA private key
..............................................+++++
....................................................+++++
writing new private key to 'tls.key'
-----
$ ls | grep tls
tls.crt
tls.key
```  

생성한 인증서 파일을 사용해서 `kubectl create secret tls <시크릿 이름> --key tls.key --cert tls.crt` 명령으로 시크릿을 생성한다. 

```bash
$ kubectl create secret tls my-tls --key tls.key --cert tls.crt
secret/my-tls created
```  

생성한 시크릿을 `kubectl get secret my-tls -o yaml` 명령으로 확인하면 아래와 같다. 

```bash
$ kubectl get secret my-tls -o yaml
apiVersion: v1
data:
  tls.crt: LS0tLS1CR .. 생략 .. VElGSUNBVEUtLS0tLQo=
  tls.key: LS0tLS1CR .. 생략 .. SVZBVEUgS0VZLS0tLS0K
kind: Secret
metadata:
  creationTimestamp: "2020-07-26T17:44:40Z"
  name: my-tls
  namespace: default
  resourceVersion: "878898"
  selfLink: /api/v1/namespaces/default/secrets/my-tls
  uid: 3ecc8fdc-b22a-44b9-af23-5d0a7b76fa19
type: kubernetes.io/tls
```  

`.data` 하위에 `tls.key` 와 `tls.crt` 필드와 해당 하는 값이 설정 된 것을 확인 할 수 있다. 
그리고 `.type` 은 `kubernetes.io/tls` 인 것을 확인 할 수 있다. 
생성한 `TLS` 스크릿은 이후 인그레스와 연결해 `HTTPS` 통신을 위핸 인증서로 사용 할 수 있다. 

### 시크릿 관리
시크릿은 `kube-apiserver`, `kubelet` 에 저장되고 사용되기 때문에 해당 노드의 메모리를 차지하게 된다. 
이러한 이유로 시크릿 하나의 최대 용량은 `1MB` 이다. 그리고 이후에는 전체 시크릿의 용량을 제한하는 기능도 생길 수도 있다.  

그리고 시크릿의 데이터는 `etcd` 에 암호화 되지 않은 상태로 저장된다. 
특정 사용자가 `etcd` 접근한다면 저장된 시크릿의 내용을 확인해 볼 수가 있다. 
이 때문에 `etcd` 에 대한 접근을 제한할 필요가 있다. 
기본 적으로 `etcd` 명령을 사용할 수 있는 `API` 는 `TLS` 기반으로 인증된 사용자만 관련 명령을 수행할 수 있다.  

또한 `etcd` 가 실행 중인 마스터 노드에 대한 접근을 제한하기 위해 계정 혹은 `IP` 기반으로 제한을 할 수 있다. 
추가로 `etcd` 에 저장되는 시크릿 데이터를 암호화를 수행할 수도 있다. 
관련 자세한 내용은 [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
에서 확인 가능하다. 

---
## Reference
