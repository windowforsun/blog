--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 인그레스(Ingress)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'L7 레이어에서 클러스터와 외부와 통신을 담당하는 인그레스에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Ingress
toc: true
use_math: true
---  

## 인그레스
인그레스(`Ingress`) 는 쿠버네티스 클러스터 외부에서 내부 파드로 접근 할 때 사용 한다. 
비슷한 기능이 서비스에도 있지만 차이점으로는 네트워크 레이어에 있다. 
서비스는 `L4` 영역 통신을 담당해서 해당 기능을 수행했다면, 
인그레스는 `L7` 영역 통신을 담당해 해당 기능을 수행한다.  

인그레스에 외부에서 들어오는 요청을 처리하는 규칙을 정해두면, 
`URL` 사용, 트래픽 로드밸런싱, `SSL`, 도메인 기반 가상 호스팅 등 다양한 기능을 제공한다.  

인그레스는 위와 같은 기능을 사용하기 위해 규칙을 정의하는 자원을 의미하고, 
실제로 해당 규칙을 바탕으로 동작의 수행은 인그레스 컨트롤러(`Ingress Controller`) 에서 담당한다.  

클라우드 서비스의 로드벨런서와 서비스를 연동해 인그레스를 사용할 수 있다. 
하지만 클라우드 서비스를 사용하지 않는 환경이라면 인그레스 컨트롤러를 직접 인그레스와 연동하는 작업이 필요하다. 
이후 나오는 예제에서는 `ingress-nginx` 를 사용해서 인그레스에 설정한 내용을 `Nginx` 에 적용한다. 

### 인그레스 템플릿

```yaml
# ingress-basic.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: www.basic1.com
      http:
        paths:
          - path: /path1
            backend:
              serviceName: service1
              servicePort: 80
          - path: /path2
            backend:
              serviceName: service2
              servicePort: 80
    - host: www.basic2.com
      http:
        paths:
          - backend:
              serviceName: service2
              servicePort: 80
```  

- `.metadata.annotations` : 인그레스에 인그레스 컨트롤러를 설정할 때 사용하는 필드이다. 
현재 사용하는 인그레스 컨트롤러는 `ingress-nginx` 이므로 해당하는 `nginx.ingress.kubernetes.io/rewrite-target` 을 키로 리다이렉트 경로인 `/` 을 설정했다. 
- `.spec.rules[]` : 인그레스에 규칙을 정의하는 필드이다. 
첫번째 `.spec.rules[0].host` 는 `www.basic1.com` 로 요청이 들어오면 하위 `paths` 의 설정에 따라 하위 경로를 처리한다. 
`.spec.rules[].host.paths[0].path` 에 설정된 `/path1` 로 요청이 오면 `service1` 서비스의 `80` 포트로 전달한다. 
그리고 `.spec.rules[].host.paths[1].path` 에 설정된 `/path2` 로 요청이 오면 `service1` 서비스의 `80` 포트로 전달한다. 
두번째 `.spec.rules[1].host` 는 `www.basic2.com` 로 요청이 들오면 하위 `paths` 의 설정에 따라 `service2` 의 `80` 포트로 요청을 전달한다.  

구성한 템플릿을 `kubectl apply -f ingress-basic.yaml` 로 클러스터에 적용하고, 
`kubectl describe ingress basic` 명령으로 인그레스의 자세한 내용을 조회하면 아래와 같다. 

```bash
$ kubectl apply -f ingress-basic.yaml
ingress.extensions/basic created
$ kubectl describe ingress basic
Name:             basic
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host            Path  Backends
  ----            ----  --------
  www.basic1.com
                  /path1   service1:80 (<none>)
                  /path2   service2:80 (<none>)
  www.basic2.com
                     service2:80 (<none>)

.. 생략 ..
```  

`Rules` 필드를 확인하면 템플릿에서 정의한 내용대로 인그레스가 설정된 것을 확인 할 수 있다. 


### ingress-nginx
앞서 언급한 것과 같이 인그레스 템플릿은 클러스터에 인그레스 규칙을 정의하기 위한 수단에 불과하다. 
실제로 인그레스 동작을 수행하는 것은 인그레스 컨트롤러이다.  

인그레스 컨트롤러 중 `ingress-nginx` 를 `Docker Desktop` 에 설정하고 사용하는 방법에 대해 알아본다.  
`ingress-nginx` 는 `git` 을 통해 클론 받아 구성할 수 있다. 
태그는 `nginx-0.29.0` 태그를 사용해서 클론한다. 

```bash
$ git clone -b nginx-0.29.0 https://github.com/kubernetes/ingress-nginx.git
Cloning into 'ingress-nginx'...
remote: Enumerating objects: 48, done.

.. 생략 ..

Updating files: 100% (4769/4769), done.
```  

`ingress/deploy/baremetal` 경로로 이동한 뒤, 
`kubectl create namespace ingress-nginx` 명령으로 네임스페이스를 생성해 준다. 
그리고 해당 경로에서 `kubectl apply -k .` 명령으로 `ingress-nginx` 컨트롤러의 구성을 수행한다. 

```bash
$ cd ingress-nginx/deploy/baremetal/
$ kubectl create namespace ingress-nginx
namespace/ingress-nginx created
$ kubectl apply -k .
serviceaccount/nginx-ingress-serviceaccount created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
service/ingress-nginx created
deployment.apps/nginx-ingress-controller created
limitrange/ingress-nginx created
```  

`kubectl get deploy,svc -n ingress-nginx` 명령으로 구성된 컨트롤러와 서비스를 확인해보면 아래와 같다. 

```bash
$ kubectl get deploy,svc -n ingress-nginx
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   1/1     1            1           84s

NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx   NodePort   10.111.63.108   <none>        80:31769/TCP,443:32204/TCP   84s
```  

`nginx-ingress-controller` 라는 디플로이먼트, `ingress-nginx` 라는 `NodePort` 타입의 서비스가 실행 중인 것을 확인 할 수 있다.  

`ingress-nginx` 서비스의 `PORT` 필드에 있는 `31769` 로 호스트에서 요청을 보내면 아래와 같은 `404 Not Found` 결과를 확인 할 수 있다. 

```bash
$ curl localhost:31769
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
```  

현재는 인그레스 컨트롤러에서 어떤 규칙으로 외부 요청을 처리할지를 정의하지 않았기 때문에 발생하는 결과이다. 
이전에 사용 했었던 [Deployment 템플릿](https://windowforsun.github.io/blog/kubernetes/kubernetes-concept-controller-deployment/#%EB%94%94%ED%94%8C%EB%A1%9C%EC%9D%B4%EB%A8%BC%ED%8A%B8-%ED%85%9C%ED%94%8C%EB%A6%BF)
을 사용해서 디플로이먼트를 클러스터에 등록하고, 앞서 구성했던 인그레스의 서비스와 연결한다. 

```bash
$ kubectl apply -f deployment-nginx.yaml
deployment.apps/deployment-nginx created
$ kubectl expose deploy deployment-nginx --name service1
service/service1 exposed
```  

앞서 구성한 인그레스 설정에서 `service1` 은 `www.basic1.com` 도메인과 설정됐기 때문에, 
`/etc/hosts` 파일 혹은 `C:\Windows\System32\drivers\etc\hosts` 파일에 `127.0.0.1   www.basic1.com` 라인을 추가한다. 

```bash
$ vi /etc/hosts

127.0.0.1   www.basic1.com
```  

인그레스 설정에서 `www.basic1.com` 에 `service1` 과 연결된 경로는 `/path1` 이기 때문에, 
`www.basic1.com/path1:31769` 로 요청을 보내면 `Nginx` 메시지가 출력되는 것을 확인 할 수 있다.

```bash
$ curl www.basic1.com:31769/path1
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

현재 구성을 살펴보면 `ingress-nginx` 또한 쿠버네티스 클러스터 안에서 동작하기 때문에, 
외부 요청을 처리하기 위해 `NodePort` 타입의 서비스를 생성한 것을 확인 할 수 있다. 
성능적인 측면이 우선시 된다면, 
인그레스 컨트롤러의 네트워크를 호스트 모드로 설정하게 되면 별도의 서비스를 거치지 않고 클러스터 내의 파드에 접근할 수 있어 성능적으로 이점이 있다. 


### SSL
외부의 요청을 처리하는 인그레스를 통해 `HTTPS` 요청을 처리할 수 있는 `SSL` 설정에 대해 알아본다. 
인그레스에 `SSL` 설정을 하고 파드를 연결하면, 각 파드에 `SSL` 설정을 하지 않아도 `HTTPS` 통신이 가능하다. 
또한 인증서 만료시에도 인그레스의 인증서만 교체해 주면된다.  

그림(인그레스를 이용한 SSL 관리 구조)  

서비스를 제공하는 도메인이라면 공인인증기관에서 `SSL` 인증서를 발급받지만, 
테스트가 목적이기 때문에 `OpenSSL` 로 직접 인증서를 생성한다. 

그림(SSL 인증서의 인증 구조)  

인증서를 생성하기 위해 먼저 `ingress-nginx` 디렉토리가 위치한 같은 경로에 `ssl` 이라는 디렉토리를 생성한다. 
그리고 아래 명령어를 수행해 인증서를 생성한다. 

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=my-ssl"
Generating a RSA private key
......................+++++
.....................................................+++++
writing new private key to 'tls.key'
-----
$ ls
tls.crt  tls.key
```  

츨력과 같이 `tls.crt`, `tls,key` 파일이 생생된것을 확인할 수 있다. 
생성된 파일을 사용해서 인증서용 시크릿을 생성해 쿠버네티스 내부에서 보안이 필요한 설정을 사용할 때 사용할 수 있다. 
명령은 다음과 같다. `kubectl create secret tls <시크릿 이름> --key tls.key --cert tls.crt` 
그리고 `kubectl describe secret <시크릿 이름>` 명령으로 인증서용 시크릿 상태를 확인 할 수 있다. 

```bash
$ kubectl create secret tls my-https --key tls.key --cert tls.crt
secret/my-https created
$ kubectl describe secret my-https
Name:         my-https
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1123 bytes
tls.key:  1704 bytes
```  

`ssl` 인증서를 적용한 인그레스 템플릿은 아래와 같다. 

```yaml
# ingress-ssl.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic-ssl
spec:
  tls:
    - hosts:
      - www.basic-ssl.com
      secretName: my-https
  rules:
    - host: www.basic-ssl.com
      http:
        paths:
          - path: /
            backend:
              serviceName: s1
              servicePort: 80
```  

- `.spec.tls[].hosts[]` : `ssl` 인증서 관련 설정 중 사용할 호스트 이름을 설정하는 필드이다. 
- `.spec.tls[].secretName` : 사용할 `ssl` 인증서를 등록하는 곳으로 앞서 생성한 인증서 이름을 설정한다. 

`kubectl apply -f ingress-ssl.yaml` 명령으로 클러스터에 적용 한 후, 
`kubectl get svc -n ingress-nginx` 로 서비스를 조회하면 아래와 같다. 

```bash
kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.96.77.73   <none>        80:32339/TCP,443:31379/TCP   2m
```  

`ingress-nginx` 서비스의 `PORT` 필드에서 확인 할 수 있듯이, 
`HTTPS` 요청을 담당하는 `443` 포트는 `31379` 로 매핑 됐기 때문에 해당 포트를 사용한다. 
그리고 사용하는 호스트 이름이 `www.basic-ssl.com` 이기 때문에 `/etc/hosts` 파일에도 `www.basic1.com` 과 동일하게 추가해 준다. 

```bash
$ vi /etc/hosts

127.0.0.1   www.basic-ssl.com
```

현재 생성한 `ssl` 인증서는 인증된 인증서가 아니기 때문에 테스트 시에 `curl -k` 명령어와 옵션을 사용해서 무시하도록 한다. 
`curl -vI -k https://<호스트>:<포트>/` 를 사용해서 테스트를 수행한다. 

```bash
$ curl -vI -k https://www.basic-ssl.com:31379
*   Trying 127.0.0.1:31379...
* TCP_NODELAY set

.. 생략 ..

> Host: www.basic-ssl.com:31379
> user-agent: curl/7.68.0
> accept: */*

.. 생략 ..

* Connection #0 to host www.basic-ssl.com left intact
```  

위 출력 결과와 같이 `Host` 에 설정한 호스트와 포트가 출력 된다면, 
인증서 설정이 정상적으로 수행 된 것이다. 

```bash
$ curl -vI -k https://www.basic-ssl.com:32339
*   Trying 127.0.0.1:32339...
* TCP_NODELAY set
* Connected to www.basic-ssl.com (127.0.0.1) port 32339 (#0)

.. 생략 ..

* error:1408F10B:SSL routines:ssl3_get_record:wrong version number
* Closing connection 0
curl: (35) error:1408F10B:SSL routines:ssl3_get_record:wrong version number
```  

위와 같이 정상적이지 않은 포트로 요청을 하게 되면 인증서 관련 에러가 발생하는 것을 확인 할 수 있다. 


---
## Reference
