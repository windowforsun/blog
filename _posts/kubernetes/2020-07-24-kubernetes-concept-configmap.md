--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨피그맵(ConfigMap)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 실행되는 컨테이너와 환경을 분리해서 관리할 수 있는 컨피그맵에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - ConfigMap
toc: true
use_math: true
---  

## 컨피그맵
컨피그맵(`ConfigMap`) 은 쿠버네티스 클러스터에 구성되는 컨테이너에 필요한 환경 설정을 분리해서 관리 할 수 있도록 제공하는 기능이다. 
실환경에 배포되는 애플리케이션은 개발환경에서 완료된 코드를 바탕으로 만들어진다. 
이러한 흐름에서 중요한 것은 개발환경과 실 환경에서 사용하는 컨테이너가 같아야 한다는 점이다. 
컨테이너가 같아야 실환경에 배포했을 때 잠재적인 문제를 최소화 할 수 있기 때문이다.   

하지만 필연적으로 개발환경과 실환경에는 달라야 하는 여러 설정 값들이 존재한다. 
데이터베이스 설정, 로그 설정 등 구성에 따라 다양한 부분이 존재할 수 있다. 
이런 상황에서 컨피그맵을 사용해서 컨피그맵만 환경 별로 분리하면, 
같은 컨테이너를 개발환경에서 부터 실환경까지 동일하게 사용 할 수 있다.  


## 컨피그맵 템플릿
아래는 컨피그맵을 구성할 수 있는 템플릿의 예시이다. 

```yaml
# configmap-dev.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: env-dev
  namespace: default
data:
  DB_HOST: localhost
  DB_USER: dev
  DB_PASS: dev-pass
  DEBUG_INFO: debug
```  

- `.data` : 하위에 해당 환경에 필요한 값을 `key-value` 구조로 설정한다. 

`kubectl apply -f configmap.dev.yaml` 명령으로 클러스터에 적용하고, 
`kubectl describe configmap <컨피그맵 이름>` 명령으로 상세 정보를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f configmap-dev.yaml
configmap/env-dev created
$ kubectl describe configmap env-dev
Name:         env-dev

.. 생략 ..

Data
====
DB_PASS:
----
dev-pass
DB_USER:
----
dev
DEBUG_INFO:
----
debug
DB_HOST:
----
localhost
Events:  <none>
```  

`Data` 필드에 템플릿에서 설정한 값들이 저장된 것을 확인 할 수 있다.  

이렇게 적용한 컨피그맵은 다양한 방식으로 불러와 실제 컨테이너에서 실제 컨테이너에서 사용할 수 있다. 
그 방법은 아래 3가지 정도로 정리 할 수 있다. 
- 컨피그맵 일부만 불러와서 사용
- 컨피그맵 전체를 불러와서 사용
- 컨피그맵을 볼륨으로 로드해서 사용

### 애플리케이션
컨피그맵 테스트를 위해 컨테이너에서 실행하는 애플리케이션에 대해 간단하게 설명한다. 
`Spring` 을 사용한 간단한 웹 애플리케이션으로 아래와 같은 하나의 컨트롤러만 있다. 

```java
@RestController
public class EnvController {
    @GetMapping("/env")
    public Map<String, String> env() {
        return System.getenv();
    }

    @GetMapping("/volume-env")
    public Map<String, String> volumeEnv(@PathParam("path") String path) throws Exception {
        Process p = Runtime.getRuntime().exec("cat " + path);
        BufferedReader reader = new BufferedReader(new InputStreamReader(p.getInputStream()));
        StringBuilder strBuilder = new StringBuilder();
        String line;

        while((line = reader.readLine()) != null) {
            strBuilder.append(line);
        }

        Map<String, String> envMap =  new HashMap<String, String>();
        envMap.put(path, strBuilder.toString());

        return envMap;
    }
}
``` 

- `/env` : `HTTP GET` 요청으로 요청하면 현재 컨테이너에 설정된 모든 환경 변수를 `json` 형식으로 응답한다. 
- `/volume-env` : `HTTP GET` 요청으로 `path` 를 이름으로 경로를 요청 파라미터로 넘겨 주면, 
경로에 해당하는 파일을 읽어 출력한 결과를 `filename:filecontent` 구조의 `json` 으로 응답한다. 

해당 애플리케이션은 `Docker` 이미지로 빌드 되었고, 
`windowforsun/config:latest` 이미지 이름으로 사용 할 수 있다. 

### 일부만 불러오기
아래는 컨피그맵 중 일부의 값만 불러와서 사용하는 예시 템플릿이다. 
디플로이먼트, 서비스가 한 파일에 구성 돼있다. 
구성하는 템플릿에서는 위에서 만든 `configmap-dev.yaml` 의 설정 값을 사용한다.

```yaml
# configmap-each.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-each-app
  labels:
    app: load-each-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-each-app
  template:
    metadata:
      labels:
        app: load-each-app
    spec:
      containers:
        - name: load-each-app
          image: windowforsun/configmap-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: DEBUG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: env-dev
                  key: DEBUG_INFO

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: load-each-app
  name: load-each-app-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: load-each-app
  ports:
    - nodePort: 32222
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

- `.spec.template.spec.containers[].env[].name` : 컨테이너에서 사용할 환경 변수의 이름을 설정한다.
- `.spec.template.spec.containers[].env[].valueFrom` : 환경 변수의 값을 가져오는 곳을 설정한다. 
- `.spec.template.spec.containers[].env[].valueFrom.name` : 앞서 설정한 컨피그맵에서 가져오기 위해 해당 이름을 설정한다.  
- `.spec.template.spec.containers[].env[].valueFrom.key` : 컨피그맵에서 구성한 필드 중 값을 가져올 필드의 이름을 설정한다.

결과적으로 디플로이먼트에서 실행되는 컨테이너의 `DEBUG_LEVEL` 환경 변수에는 `debug` 라는 값이 들어가게 된다.  

`kubectl apply -f configmap-each.yaml` 명령으로 구성을 클러스터에 적용하고, 
브라우저 혹은 `curl` 명령으로 `localhost:32222/env` 요청을 보내 결과를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f configmap-each.yaml
deployment.apps/load-each-app created
service/load-each-app-service created
$ curl localhost:32222/env
{
  "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
  "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin",

  # 템플릿에서 추가한 환경 변수
  "DEBUG_LEVEL": "debug",
  "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP": "tcp://10.107.150.219:8080",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_ADDR": "10.107.150.219",
  "KUBERNETES_PORT": "tcp://10.96.0.1:443",
  "JAVA_HOME": "/usr/lib/jvm/java-1.8-openjdk/jre",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PROTO": "tcp",
  "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
  "LANG": "C.UTF-8",
  "KUBERNETES_SERVICE_HOST": "10.96.0.1",
  "KUBERNETES_SERVICE_PORT": "443",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PORT": "8080",
  "HOSTNAME": "load-each-app-6dfd757dd5-gmf4n",
  "JAVA_ALPINE_VERSION": "8.212.04-r0",
  "LOAD_EACH_APP_SERVICE_SERVICE_HOST": "10.107.150.219",
  "LOAD_EACH_APP_SERVICE_SERVICE_PORT": "8080",
  "LD_LIBRARY_PATH": "/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64/server:/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64:/usr/lib/jvm/java-1.8-openjdk/jre/../lib/amd64",
  "LOAD_EACH_APP_SERVICE_PORT": "tcp://10.107.150.219:8080",
  "KUBERNETES_PORT_443_TCP_PORT": "443",
  "JAVA_VERSION": "8u212",
  "KUBERNETES_SERVICE_PORT_HTTPS": "443",
  "HOME": "/root"
}
```  

### 전체 불러오기
이번에는 앞에서 구성했던 컨피그맵의 데이터 전체를 불러와 컨테이너에 설정하는 예제에 대해 살펴 본다. 

```yaml
# configmap-all.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-all-app
  labels:
    app: load-all-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-all-app
  template:
    metadata:
      labels:
        app: load-all-app
    spec:
      containers:
        - name: load-all-app
          image: windowforsun/configmap-app:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: env-dev

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: load-all-app
  name: load-all-app-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: load-all-app
  ports:
    - nodePort: 32223
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

- `.spec.template.spec.containers[].envFrom[]` : 환경 변수를 가져올 곳을 설정한다.
- `.spec.template.spec.containers[].envFrom[].configMapRef.name` : 환경 변수를 설정된 컨피그맵에서 가져오기 위해 해당 컨피그맵의 이름을 설정한다. 

결과적으로 `env-dev` 에 설정된 모든 값들(`DB_HOST`, `DB_USER`, `DB_PASS`, `DEBUG_INFO`) 이 환경 변수로 컨테이너에 설정된다.  

`kubectl apply -f configmap-all.yaml` 로 클러스터에 적용하고, 
브라우저 혹은 `curl` 명령을 사용해서 `localhost:32223/env` 로 요청을 보내면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ kubectl apply -f configmap-all.yaml
deployment.apps/load-all-app created
service/load-all-app-service created
$ curl localhost:32223/env
{
  "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin",

  # 설정된 환경 변수
  "DEBUG_INFO": "debug",
  "KUBERNETES_PORT": "tcp://10.96.0.1:443",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_ADDR": "10.107.150.219",

  # 설정된 환경 변수
  "DB_USER": "dev",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_PORT": "8080",
  "JAVA_HOME": "/usr/lib/jvm/java-1.8-openjdk/jre",
  "LANG": "C.UTF-8",
  "LOAD_ALL_APP_SERVICE_SERVICE_PORT": "8080",
  "KUBERNETES_SERVICE_HOST": "10.96.0.1",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PORT": "8080",
  "LOAD_EACH_APP_SERVICE_SERVICE_PORT": "8080",

  # 설정된 환경 변수
  "DB_PASS": "dev-pass",
  "LD_LIBRARY_PATH": "/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64/server:/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64:/usr/lib/jvm/java-1.8-openjdk/jre/../lib/amd64",
  "JAVA_VERSION": "8u212",
  "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
  "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP": "tcp://10.107.150.219:8080",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PROTO": "tcp",
  "KUBERNETES_PORT_443_TCP_PROTO": "tcp",

  # 설정된 환경 변수
  "DB_HOST": "localhost",
  "KUBERNETES_SERVICE_PORT": "443",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP": "tcp://10.109.128.80:8080",
  "LOAD_ALL_APP_SERVICE_SERVICE_HOST": "10.109.128.80",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_PROTO": "tcp",
  "HOSTNAME": "load-all-app-569dd68657-mhfjm",
  "JAVA_ALPINE_VERSION": "8.212.04-r0",
  "LOAD_EACH_APP_SERVICE_SERVICE_HOST": "10.107.150.219",
  "LOAD_ALL_APP_SERVICE_PORT": "tcp://10.109.128.80:8080",
  "LOAD_EACH_APP_SERVICE_PORT": "tcp://10.107.150.219:8080",
  "KUBERNETES_PORT_443_TCP_PORT": "443",
  "KUBERNETES_SERVICE_PORT_HTTPS": "443",
  "HOME": "/root",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_ADDR": "10.109.128.80"
}
```  

이번에는 아래와 같은 새로운 컨피그맵 템플릿을 구성한다. 

```yaml
# configmap-prod.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: env-prod
  namespace: default
data:
  DB_HOST: pordhost
  DB_USER: prod
  DB_PASS: prod-pass
  DEBUG_INFO: prod_debug
```  

`kubectl apply -f` 명령으로 해당 템플릿을 적용하고, 
`configmap-all.yaml` 템플릿의 디플로이먼트에서 `.spec.template.spec.containers[].envFrom[].configMapRef.name` 의 값을 새롭게 설정한 컨피그맵의 이름으로 설정한다. 
그리고 `kubectl apply -f` 명령으로 다시 클러스터에 적용하고, 요청을 보내 확인하면 아래와 같다. 

```bash
$ kubectl apply -f configmap-prod.yaml
configmap/env-prod created
$ vi configmap-all.yaml

spec:
    spec:
      containers:
          envFrom:
            - configMapRef:
                name: env-prod

$ kubectl apply -f configmap-all.yaml
deployment.apps/load-all-app configured
service/load-all-app-service unchanged
$ curl localhost:32223/env
{
  "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin",

  # 설정된 환경 변수
  "DEBUG_INFO": "prod_debug",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_ADDR": "10.107.150.219",
  "KUBERNETES_PORT": "tcp://10.96.0.1:443",

  # 설정된 환경 변수
  "DB_USER": "prod",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_PORT": "8080",
  "JAVA_HOME": "/usr/lib/jvm/java-1.8-openjdk/jre",
  "LANG": "C.UTF-8",
  "LOAD_ALL_APP_SERVICE_SERVICE_PORT": "8080",
  "KUBERNETES_SERVICE_HOST": "10.96.0.1",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PORT": "8080",
  "LOAD_EACH_APP_SERVICE_SERVICE_PORT": "8080",

  # 설정된 환경 변수
  "DB_PASS": "prod-pass",
  "LD_LIBRARY_PATH": "/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64/server:/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64:/usr/lib/jvm/java-1.8-openjdk/jre/../lib/amd64",
  "JAVA_VERSION": "8u212",
  "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
  "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP": "tcp://10.107.150.219:8080",
  "LOAD_EACH_APP_SERVICE_PORT_8080_TCP_PROTO": "tcp",
  "KUBERNETES_PORT_443_TCP_PROTO": "tcp",

  # 설정된 환경 변수
  "DB_HOST": "pordhost",
  "KUBERNETES_SERVICE_PORT": "443",
  "LOAD_ALL_APP_SERVICE_SERVICE_HOST": "10.109.128.80",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP": "tcp://10.109.128.80:8080",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_PROTO": "tcp",
  "HOSTNAME": "load-all-app-c8d7476cc-b96kw",
  "JAVA_ALPINE_VERSION": "8.212.04-r0",
  "LOAD_EACH_APP_SERVICE_SERVICE_HOST": "10.107.150.219",
  "LOAD_EACH_APP_SERVICE_PORT": "tcp://10.107.150.219:8080",
  "LOAD_ALL_APP_SERVICE_PORT": "tcp://10.109.128.80:8080",
  "KUBERNETES_PORT_443_TCP_PORT": "443",
  "KUBERNETES_SERVICE_PORT_HTTPS": "443",
  "HOME": "/root",
  "LOAD_ALL_APP_SERVICE_PORT_8080_TCP_ADDR": "10.109.128.80"
}
```  

새로 설정된 `env-prod` 의 값으로 환경 변수의 값들이 변경 된 것을 확인 할 수 있다. 

### 볼륨으로 불러오기
지금까지는 환경 변수로 컨피그맵의 값을 불러와 설정했다. 
여기서 환경 변수가 아닌 컨테이너 볼륨에 컨피그맵의 데이터를 저장해서 파일로 제공할 수도 있다. 


```yaml
# configmap-volume.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-volume-app
  labels:
    app: load-volume-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-volume-app
  template:
    metadata:
      labels:
        app: load-volume-app
    spec:
      containers:
        - name: load-volume-app
          image: windowforsun/configmap-app:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: configmap-volume
              mountPath: /etc/configmap
          volumes:
            - name: configmap-volume
              configMap:
                name: env-dev

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: load-volume-app
  name: load-volume-app-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: load-volume-app
  ports:
    - nodePort: 32224
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

- `.spec.template.spec.containers[].volumeMounts[]` : 컨테이너 하위 설정에 컨피그맵 데이터를 저장하는 용도로 사용할 볼륨 마운트 설정을 한다. 
- `.spec.template.spec.volumes[]` : 컨테이너에서 설정한 볼륨에 컨피그맵 데이터를 파일로 저장한다. 

결과적으로 컨테이너의 `/etc/configmap` 경로에는 `DB_HOST`, `DB_USER`, `DB_PASS`, `DEBUG_INFO` 라는 파일이 생성되고, 
각 파일에는 컨피그맵에 설정된 값이 들어가게 된다. 

`kubectl apply -f configmap-volume.yaml` 명령으로 클러스터에 적용한다. 
먼저 `kubeclt get pod` 명령으로 위 템플릿에서 실행 중인 파드의 정보를 얻는다. 
그리고 `kubectl exec -it <파드 이름> sh` 으로 해당 파드에 접속해 볼륨 경로를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f configmap-volume.yaml
deployment.apps/load-volume-app created
service/load-volume-app-service created
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
load-all-app-c8d7476cc-b96kw     1/1     Running   0          13m
load-each-app-6dfd757dd5-gmf4n   1/1     Running   0          36m
load-volume-app-56845574-n6nch   1/1     Running   0          7m10s
$ kubectl exec -it load-volume-app-56845574-n6nch sh
/ # ls /etc/configmap
DB_HOST     DB_PASS     DB_USER     DEBUG_INFO
/ # cat /etc/configmap/DB_USER
dev
```  

설명한 것과 같이 볼륨 경로에 파일이 생성되고 각 파일에는 값이 알맞게 들어가 있는 것을 확인 할 수 있다.
마지막으로 `localhost:32224/volume-env?path=/etc/configmap/DB_USER` 요청을 보내면 아래와 같다. 

```bash
$ localhost:32224/volume-env?path=/etc/configmap/DB_USER
{
  "/etc/configmap/DB_USER": "dev"
}
```  

---
## Reference
