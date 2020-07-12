--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 서비스(Service)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '파드에 접근할 수 있는 하나의 IP를 제공하면서, 로드벨런서 역할을 수행하는 서비스에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Service
  - ClusterIP
  - NodePort
  - ExternalName
  - Headless Service
toc: true
use_math: true
---  

## 서비스
서비스(`Service`) 는 여러 파드에 접근할 수 있는 `IP` 를 제공하는 로드밸런서 역할을 수행한다. 
다른 여러기능도 있지만, 주로 로드밸런서로 사용된다.  

쿠버네티스 클러스터는 여러 노드로 구성돼있고, 노드에서 실행되는 파드는 컨트롤러에서 관리한다. 
이는 한 파드가 `A` 노드에서 배포나 특정 동작으로 인해 `B` 라는 다른 노드로 이동할 수 있다는 의미이다. 
그리고 이런 과정에서 파드의 `IP` 가 변경되기도 한다.  

위와 같이 동적인 특성을 띄는 파드를 고정적인 방법으로 접근이 필요할 때 서비스를 사용할 수 있다. 
위에서 언급한 것과 같이 서비스는 쿠버네티스 클러스터에서 파드에 접근할 수 있는 고정 `IP` 를 제공한다. 
또한 클러스터 외부에서 접근할 수도 있다. 
그리고 서비스는 `L4` 영역에서 위와 같은 역할을 수행한다.  

### 서비스의 종류
서비스는 아래와 같은 4가지 종류가 있다. 

- `ClusterIP` : 서비스의 기본 타입으로 쿠버네티스 클러스터 내부 접근시에만 사용할 수 있다. 
클러스터에 구성된 노드나 파드에서 클러스터 `IP` 로 서비스에 연결된 파드에 접근할 수 있다. 
- `NodePort` : 클러스터 외부에서 내부로 접근할 때 가장 간단한 방법으로, 
서비스 하나에 모든 노드의 지정된 포트를 할당한다. 
서비스에 `node1:8080`, `node2:8080` 과 같이 등록하면 노드와 관계없이 파드에 접근 할 수 있다. 
이런 특성으로 `node2` 에 접근하려는 파드가 실행중이지 않더라도 `node2:8080` 을 통해 클러스터에 실행 중인 파드로 연결할 수 있다. 
클러스터 내부 뿐만아니라 클러스터 외부에서도 파드에 접근할 수 있다. 
- `LoadBalancer` : 클라우드 서비스(아마존, 구글 ..)에서 쿠버네티스를 지원하는 로드밸런서 장비에서 사용할 수 있다. 
클라우드 서비스에서 제공하는 로드벨런서와 파드를 연결해 로드벨런서 `IP` 로 클러스터 외부에서 내부 파드에 접근할 수 있다. 
- `ExternalName` : 서비스를 `.spec.externalName` 필드에 설정한 값과 연결해 클러스터 내부에서 외부로 접근할 때 사용한다. 
서비스를 통해 클러스터 외부로 접근할 때 설정된 `CNAME` 을 사용한다. 
해당 서비스를 구성할 때는 `.spec.selector` 필드가 필요하지 않다. 


### 서비스 템플릿
아래는 서비스를 구성하는 템플릿의 기본적인 예시 이다. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip
spec:
  type: ClusterIP
  clusterIP: 10.96.10.10
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```  

- `.spec.type` : 구성하는 서비스의 타입을 지정한다. 기본 타입 값은 `ClusterIP` 이다. 
- `.spec.clusterIP` : 클러스터 아이피를 직접 지정해야 할 경우, 해당 필드를 사용해서 가능하다. 
설정하지 않으면 자동으로 `IP` 를 할당한다. 
- `.spec.selector` : 서비스와 연결할 파드에 설정한 `.labels` 필드 값을 설정한다. 
- `.spec.ports[]` : 서비스에서 사용하는 포트 정보를 배열 형태로 입력한다. 

서비스를 구성하는 대략적인 템플릿의 구조는 위와 같다. 
이후 서비스를 생성하고 테스트하기 위해 먼저 서비스와 연결할 디플로이먼트를 `kubectl run` 명령을 사용해서 생성한다. 

```bash
$ kubectl run deployment-nginx \
> --image=nginx \
> --replicas=2 \
> --port=80 \
> --labels="app=deployment-nginx"
deployment.apps/deployment-nginx created
```  

서비스 테스트시에 사용하는 파드의 이름은 `deployment-nginx` 이고, 포드는 `80`, 
서비스에서 매칭시킬 레이블은 `app=deployment-nginx` 이다. 

### ClusterIP
`ClusterIP` 테스트를 위해 아래와 같이 템플릿으로 `ClusterIP` 서비스를 구성한다.  

```yaml
# service-clusterip.yaml

apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
spec:
  type: ClusterIP
  selector:
    app: deployment-nginx
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
```  

- `.spec.type` : 서비스 타입을 `ClusterIP` 로 설정한다. 
- `.spec.selector` : 앞서 생성한 디플로이먼트의 레이블인 `app=deployment-nginx` 에 맞게 입력한다. 

구성한 `ClusterIP` 서비스 템플릿을 `kubectl apply -f service-clusterip.yaml` 명령을 통해 클러스터에 적용한다. 
그리고 `kubectl get svc` 로 서비스 리스트 조회, `kubectl describe svc <서비스 이름>` 으로 해당 서비스의 상제 정보를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f service-clusterip.yaml
service/service-clusterip created
$ kubectl get svc
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP    13d
# 생성한 서비스
service-clusterip   ClusterIP   10.108.211.132   <none>        8000/TCP   4s
$ kubectl describe svc service-clusterip
Name:              service-clusterip
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"service-clusterip","namespace":"default"},"spec":{"ports":[{"port...
Selector:          app=deployment-nginx
Type:              ClusterIP
IP:                10.108.211.132
Port:              <unset>  8000/TCP
TargetPort:        80/TCP
Endpoints:         10.1.3.213:80,10.1.3.214:80
Session Affinity:  None
Events:            <none>
```  

`kubectl describe` 출력 필드에서 몇가지 부분만 설명하면 아래와 같다.
- `Name` : 해당 서비스의 이름
- `Namepsace`: 서비스가 속한 네임스페이스
- `Selector` : 서비스가 연결하는 레이블 정보
- `Endpoints` : 서비스가 실제로 연결하는 파드의 `IP`

서비스에 연결된 파드의 `IP` 를 실제로 확인해 보고 싶다면, 
`kubectl get pods -o wide` 명령으로 가능하다. 

```bash
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE             NOMINATED NODE   READINESS GATES
deployment-nginx-7578988fbd-79s7q   1/1     Running   0          11m     10.1.3.213   docker-desktop   <none>           <none>
deployment-nginx-7578988fbd-vv42w   1/1     Running   0          11m     10.1.3.214   docker-desktop   <none>           <none>
```  

위 출력 결과에서 `IP` 필드를 보면, 서비스와 연결된 파드의 아이피와 일치함을 확인 할 수 있다. 

생성한 `ClusterIP` 서비스를 사용해서 실제로 파드와 연결이 가능한지 테스트를 하기 위해 `netshoot` 이미지를 사용한다. 
`netshoot` 이미지로 클러스터안에 파드를 생성하고, `ClusterIP` 의 `IP` 를 사용해서 디플로이먼트 파드와 연결된 포트로 요청을 날려 본다. 

```bash
$ kubectl run \
> -it \
> --image nicolaka/netshoot \
> testnet \
> bash
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
bash-5.0# curl 10.108.211.132:8000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```  

위 결과와 같이 `curl <ClusterIP 서비스 아이피>:<포트>` 명령을 수행하면 `Nginx` 메시지가 출력되는 것을 확인 할 수 있다.  

생성된 `testnet` 파드는 아래와 같이 빠져나오고 삭제할 수 있다. 

```bash
bash-5.0# exit
exit
Session ended, resume using 'kubectl attach testnet-5f694fd785-5959k -c testnet -i -t' command when the pod is running
root@CS_NOTE_2:/mnt/c/Users/ckdtj/Documents/kubernetes-example/service# kubectl delete deploy testnet
deployment.apps "testnet" deleted
```  

### NodePort
`NodePort` 테스트를 위해 아래와 같이 템플릿으로 `NodePort` 서비스를 구성한다.  

```yaml
# service-nodeport.yaml

apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
spec:
  type: NodePort
  selector:
    app: deployment-nginx
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
      nodePort: 8080
```  

`ClusterIP` 템플릿과 비교해서 달라진 점은 `.metadata.name` 부분과 `.spec.ports[].nodePort` 필드의 추가이다.  

`NodePort` 는 클러스터 외부에서 내부로 접근할 수 있는 수단이기 때문에, 
외부에서 `32222` 포트를 사용해서 외부에서 접근 할 수 있다. 
`kubectl apply -f service-nodeport.yaml` 명령으로 클러스터에 등록하고, 
브라우저 또는 호스트에서 `curl` 명령으로  `localhost:32222` 으로 접속하면 아래와 같이 `Nginx` 메시지를 확인 할 수 있다. 

```bash
$ kubectl apply -f service-nodeport.yaml
service/service-nodeport created
$ curl localhost:32222
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
$ kubectl get svc
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP          13d
service-clusterip   ClusterIP   10.108.211.132   <none>        8000/TCP         133m
service-nodeport    NodePort    10.110.183.164   <none>        8000:32222/TCP   76s
```  

위와 같이 `kubectl get svc` 로 서비스를 확인해 보면 `NodePort` 서비스가 추가 된 것을 확인 할 수 있다. 
그리고 `NodePort` 의 `PORT` 필드를 확인해 보면, 
`8000:32222/TCP` 와 같이 노드의 `32222` 포트가 `ClusterIP` 타입 서비스의 `80` 포트와 연결된 것도 확인 가능하다. 


### LoadBalancer
`LoadBalancer` 테스트를 위해 아래와 같이 템플릿으로 `LoadBalancer` 서비스를 구성한다.

```yaml
# service-loadbalancer.yaml

apiVersion: v1
kind: Service
metadata:
  name: service-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: deployment-nginx
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
```  

`ClusterIP` 템플릿과 비교해서, 
`.metadata.name` 필드와 `.spec.type` 필드의 값이 다르다는 점을 제외하곤 동일한 설정이다.  

`kubectl apply -f service-loadbalancer.yaml` 명령으로 클러스터에 등록하고, 
`kubectl get svc` 로 조회하면 아래와 같다. 

```bash
$ kubectl apply -f service-loadbalancer.yaml
service/service-loadbalancer created
$ kubectl get svc
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP          13d
service-clusterip      ClusterIP      10.108.211.132   <none>        8000/TCP         141m
service-loadbalancer   LoadBalancer   10.100.63.63     localhost     8000:30845/TCP   3s
service-nodeport       NodePort       10.110.183.164   <none>        8000:32222/TCP   8m43s
```  

이전에 수행했던 다른 서비스들과는 달리 `LoadBalancer` 서비스에는 `EXTERNAL-IP` 필드에 `localhost` 라는 값이 있는 것을 확인 할 수 있다. 
현재 쿠버네티스 클러스터 환경은 도커 데스크탑이기 때문에, `LoadBalancer` 서비스와 연결할 외부 로드벨런서가 없어 `localhost` 가 할당 된 것이다. 
만약 연계할 수 있는 로드벨런서가 있다면 해당 로드벨런서의 장비 `IP` 로 설정 된다. 


### ExternalName
`ExternalName` 테스트를 위해 아래와 같이 템플릿으로 `ExternalName` 서비스를 구성한다.

```yaml
# service-externalname.yaml

apiVersion: v1
kind: Service
metadata:
  name: service-externalname
spec:
  type: ExternalName
  externalName: google.com
```  

- `.spec.type` : `ExternalName` 으로 설정한다. 
- `.spec.externalName` : 테스트를 위해 연결하려는 외부 도메인 `google.com` 으로 설정한다.

`kubectl apply -f service-externalname.yaml` 명령으로 클러스터에 적용시킨다. 
그리고 `kubect get svc` 로 현재 서비스를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f service-externalname.yaml
service/service-externalname created
$ kubectl get svc
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP          13d
service-clusterip      ClusterIP      10.108.211.132   <none>        8000/TCP         151m
service-externalname   ExternalName   <none>           google.com    <none>           3s
service-loadbalancer   LoadBalancer   10.100.63.63     localhost     8000:30845/TCP   10m
service-nodeport       NodePort       10.110.183.164   <none>        8000:32222/TCP   18m
```  

`ExternalName` 서비스에서 `EXTERNAL-IP` 필드는 템플릿에서 설정한 `google.com` 값이다. 
그리고 `CUSTER-IP`, `PORT` 필드 모두 `<non>` 값으로 설정 됐다.  

테스트를 위해 다시 `netshoot` 이미지를 사용한다. 
파드를 생성하고 `ExternalName` 서비스의 클러스터 내부 도메인인 `<ExternalName 이름>.default.svc.cluster.local` 을 사용해서, 
`curl` 명령으로 요청을 하면 `google.com` 의 결과를 확인 할 수 있다. 

```bash
$ kubectl run \
> -it \
> --image nicolaka/netshoot \
> testnet \
> bash
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
bash-5.0# curl service-externalname.default.svc.cluster.local
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
  <style>
    *{margin:0;padding:0}html,code{font:15px/22px arial,sans-serif}html{background:#fff;color:#222;padding:15px}body{margin:7% auto 0;max-width:390px;min-height:180px;padding:30px 0 15px}* > body{background:url(//www.google.com/images/errors/robot.png) 100% 5px no-repeat;padding-right:205px}p{margin:11px 0 22px;overflow:hidden}ins{color:#777;text-decoration:none}a img{border:0}@media screen and (max-width:772px){body{background:none;margin-top:0;max-width:none;padding-right:0}}#logo{background:url(//www.google.com/images/branding/googlelogo/1x/googlelogo_color_150x54dp.png) no-repeat;margin-left:-5px}@media only screen and (min-resolution:192dpi){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat 0% 0%/100% 100%;-moz-border-image:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) 0}}@media only screen and (-webkit-min-device-pixel-ratio:2){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat;-webkit-background-size:100% 100%}}#logo{display:inline-block;height:54px;width:150px}
  </style>
  <a href=//www.google.com/><span id=logo aria-label=Google></span></a>
  <p><b>404.</b> <ins>That’s an error.</ins>
  <p>The requested URL <code>/</code> was not found on this server.  <ins>That’s all we know.</ins>
```  

보다 자세한 확인을 위해 `DNS` 설정을 `dig <도메인>` 명령으로 확인하면, 
`<ExternalName 이름>.default.svc.cluster.local` 도메인의 `DNS` 레코드가 `CNAME` 타입 `google.com` 으로 설정 된것을 확인 할 수 있다. 

```bash
bash-5.0# dig service-externalname.default.svc.cluster.local

; <<>> DiG 9.14.8 <<>> service-externalname.default.svc.cluster.local

.. 생략 ..

;; ANSWER SECTION:
service-externalname.default.svc.cluster.local. 30 IN CNAME google.com.
google.com.             30      IN      A       172.217.27.78


.. 생략 ..
```  

### 헤드리스 서비스
서비스를 구성할 때 `.spec.clusterIP` 필드를 `None` 으로 설정하면 클러스터 `IP` 가 없는 헤드리스 서비스(`Headless Service`)를 만들 수 있다. 
이는 로드벨런싱이 필요 없거나, 단일 `IP` 가 필요하지 않을 때 사용한다.  

헤드리스 서비스에서 `.spec.selector` 필드를 설정하게 되면 쿠버네티스 API 로 확인 할 수 있는 엔드포인트가 생성되고, 
연결된 파드와 매칭되는 `DNS A` 레코드도 생성된다. 
설정하지 않으면 엔드포인트는 만들어지지 않지만, `ExternalName` 타입 서비스에서 사용할 `CNAME` 레코드는 생성된다.  

헤드리스 서비스 테스트를 위해 아래와 같이 템플릿을 구성한다. 

```yaml
# service-headless.yaml

apiVersion: v1
kind: Service
metadata:
  name: service-headless
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: deployment-nginx
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
```  

기존 `ClusterIP` 템플릿에서 `.spec.clusterIP` 필드 값이 `None` 인 부분만 차이가 있다.  

`kubectl apply -f service-headless.yaml` 으로 클러스터에 적용한 후, `kubectl get svc` 로 서비스를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f service-headless.yaml
service/service-headless created
$ kubectl get svc
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP          14d
service-clusterip      ClusterIP      10.108.211.132   <none>        8000/TCP         176m
service-externalname   ExternalName   <none>           google.com    <none>           25m
service-headless       ClusterIP      None             <none>        8000/TCP         3s
service-loadbalancer   LoadBalancer   10.100.63.63     localhost     8000:30845/TCP   35m
service-nodeport       NodePort       10.110.183.164   <none>        8000:32222/TCP   44m
```  

`service-headless` 인 부분을 보면 `TYPE` 필드는 `ClusterIP` 이지만, 
`CUSTER-IP`, `EXTERNAL-IP` 필드 모두 `<none>` 인것을 확인 할 수 있다. 
더 자세한 정보는 `kubectl describe svc service-headless` 명령을 통해 확인 할 수 있다. 

```bash
$ kubectl describe svc service-headless
Name:              service-headless
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"service-headless","namespace":"default"},"spec":{"clusterIP":"Non...
Selector:          app=deployment-nginx
Type:              ClusterIP
IP:                None
Port:              <unset>  8000/TCP
TargetPort:        80/TCP
Endpoints:         10.1.3.213:80,10.1.3.214:80
Session Affinity:  None
Events:            <none>
```  

`IP` 에는 아무 값이 할당 되지 않았지만, 
`Endpoints` 부분에 `.spec.selector` 에 설정한 파드의 아이피와 포트가 설정된 것을 확인 할 수 있다.  

추가로 `DNS A` 레코드가 생성 됐는지 확인을 위해, 
다시 `netshoot` 이미지 파드와 `dig <서비스 이름>.default.svc.cluster.local` 명령을 사용해서 확인 한다. 

```bash
$ kubectl run \
> -it \
> --image nicolaka/netshoot \
> testnet \
> bash
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
bash-5.0# dig service-headless.default.svc.cluster.local

; <<>> DiG 9.14.8 <<>> service-headless.default.svc.cluster.local

.. 생략 ..

;; ANSWER SECTION:
service-headless.default.svc.cluster.local. 30 IN A 10.1.3.213
service-headless.default.svc.cluster.local. 30 IN A 10.1.3.214

.. 생략 ..
```  


---
## Reference
