--- 
layout: single
classes: wide
title: "[Kubernetes 실습] ConfigMap 으로 애플리케이션 설정파일 구성하기"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'ConfigMap 으로 컨테이너 런타임에 필요한 설정 파일을 마운트해서 사용하는 법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Prometheus
  - ConfigMap
toc: true
use_math: true
---  
## 환경
- `Docker desktop for Windows` 
    - `docker v20.10.2` 
    - `kubernetes v1.19.3`

## ConfigMap
[여기]({{site.baseurl}}{% link _posts/kubernetes/2020-07-24-kubernetes-concept-configmap.md %}) 
에서 `ConfigMap` 에 대한 기본적인 동작과 특징에 대해서 알아봤다. 
`ConFigMap` 은 `kubernetes Cluster` 에서 `Container` 의 런타임에 필요한, 
설정 파일이나 환경변수를 사용할 수 있도록하는 역할을 한다. 
이번 포스트에서는 `nginx.conf` 와 같은 특정 애플리케이션 설정파일을 컨테이너에 마운트해서 사용하는 방법에 대해서 알아본다.  

## 실습
먼저 `ConfigMap` 은 아래와 같이 템플릿을 사용해서 `nginx.conf` 와 같은 파일이름과 내용을 작성할 수 있다. 

```yaml
# template-configmap.yaml 

apiVersion: v1
kind: ConfigMap
metadata:
  name: template-configmap
  namespace: default
data:
  nginx-1.conf: |
    http {
        server {
            listen 80;
            server_name localhost;
        }
    }

  nginx-2.conf: |
    http {
        server {
            listen 90;
            server_name localhost;
        }
    }
```  

```bash
$ kubectl apply -f template-configmap.yaml
configmap/template-configmap created
$ kubectl get configmap | grep template
template-configmap           2      16s
```  

그리고 설정파일이 파일로 존재하는 경우 아래와 같이 파일을 사용해서 `ConfigMap` 을 바로 생성할 수 있다. 

```bash
file-configmap
├── nginx-3.conf
└── nginx-4.conf

$ cat file-configmap/nginx-3.conf
http {
    server {
        listen 100;
        server_name 127.0.0.1;
    }
}
$ cat file-configmap/nginx-4.conf
http {
    server {
        listen 110;
        server_name 127.0.0.1;
    }
}
```  

```bash
$ kubectl create configmap file-configmap --from-file=./file-configmap
configmap/file-configmap created
$ kubectl get configmap | grep file
file-configmap               2      6s
```  

테스트로 사용할 `Pod` 의 템플릿내용은 아래와 같다. 

```yaml
# pod.yaml 

apiVersion: v1
kind: Pod
metadata:
  name: configmap-test
spec:
  containers:
    - name: mountall
      image: centos:7
      command: ['/bin/bash', '-c', 'sleep 999999']
      volumeMounts:
        - name: template
          mountPath: /tmp/template
        - name: file
          mountPath: /tmp/file
    - name: mountone
      image: centos:7
      command: ['/bin/bash', '-c', 'sleep 999999']
      volumeMounts:
        - name: template-one
          mountPath: /tmp/template-one/nginx-template-one.conf
          subPath: nginx-template-one.conf
        - name: file-one
          mountPath: /tmp/file-one/nginx-file-one.conf
          subPath: nginx-file-one.conf
  volumes:
    - name: template
      configMap:
        name: template-configmap
    - name: file
      configMap:
        name: file-configmap
    - name: template-one
      configMap:
        name: template-configmap
        items:
          - key: nginx-1.conf
            path: nginx-template-one.conf
    - name: file-one
      configMap:
        name: file-configmap
        items:
          - key: nginx-3.conf
            path: nginx-file-one.conf
```  

`Pod` 는 2개의 `Container` 로 구성된다. 
- `mountall` : `template-configmap`, `file-configmap` 전체를 `/tmp/template`, `/tmp/file` 경로에 마운트 하는 컨테이너
- `mountone` : `template-config` 의 `nginx-1.conf`, `file-configmap` 의 `nginx-3.conf` 를 `/tmp/template-one`, `/tmp/file-one` 경로에 마운트 하는 컨테이너

`Pod` 를 클러스터에 적용하고 컨테이너에 마운트한 설정파일 내용을 각각 확인하면 아래와 같다. 

```bash
$ kubectl apply -f pod.yaml
pod/configmap-test created

.. mountall ..
$ kubectl exec -it configmap-test -c mountall -- /bin/bash
[root@configmap-test /]# cat /tmp/template/nginx-1.conf
http {
    server {
        listen 80;
        server_name localhost;
    }
}
[root@configmap-test /]# cat /tmp/template/nginx-2.conf
http {
    server {
        listen 90;
        server_name localhost;
    }
}
[root@configmap-test /]# cat /tmp/file/nginx-3.conf

http {
    server {
        listen 100;
        server_name 127.0.0.1;
    }
}
[root@configmap-test /]# cat /tmp/file/nginx-4.conf

http {
    server {
        listen 110;
        server_name 127.0.0.1;
    }
}

.. mountone ..
$ kubectl exec -it configmap-test -c mountone -- /bin/bash
[root@configmap-test /]# cat /tmp/template-one/nginx-template-one.conf
http {
    server {
        listen 80;
        server_name localhost;
    }
}
[root@configmap-test /]# cat tmp/file-one/nginx-file-one.conf

http {
    server {
        listen 100;
        server_name 127.0.0.1;
    }
}
```  

위와 같이 다양한 방법과 구성으로 컨테이너 런타임에 필요한 설정 파일 등을 마운트해서 애플리케이션을 실행 시킬 수 있다.  


---
## Reference






