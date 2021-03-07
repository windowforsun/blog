--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Prometheus 를 활용한 모니터링 구성 및 Metric 수집"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Kubernetes Cluster 에서 Prometheus 를 사용해서 모니터링과 Metric 수집을 수행해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Prometheus
  - Monitoring
  - kube-state-metrics
toc: true
use_math: true
---  
## 환경
- `Docker desktop for Windows` 
    - `docker v20.10.2` 
    - `kubernetes v1.19.3`

## Prometheus
`kubernetes Cluster` 에서 `Prometheus` 를 사용해서 클러스터 및 서버 환경을 모니터링 할 수 있는 `Metric` 을 수집하는 방법에 대해 알아본다. 
`Prometheus` 는 모니터링 두고로 주목을 받는 도구 이다. 
`Prometheus` 는 사운드클라우드(`SoundCloud`) 에서 처음 개발되었고 지금은 `CNCF` 재단에서 관리하는 프로젝트이다.  

`Prometheus` 는 시계열(`Time series`) 데이터를 저장할 수 있는 다차원 데이터 모델과 데이터 모델을 활용 할 수 있는 `PromQL` 이라는 쿼리를 사용한다. 
그리고 모니터링에 필요한 `Metric` 수집은 `Pull` 구조를 사용한다. 
수집 대상은 정적, 서비스 디스커버리를 사용해서 동적으로 설정할 수 있다. 
필요한 경우 외부에서 직접 `Push` 한 `Metric` 을 푸시 게이트웨이로 받아 저장하는 방식도 사용할 수 있다.  
모니터링 데이터는 로컬 스토리지에 저장되고 외부 스토리지에 저장 할 수도 있다. 
또한 시각화의 경우 내장된 `UI` 를 사용할 수도 있지만, 대부분 `Grafana` 와 연동해 시각화를 구성한다.  

![그림 1]({{site.baseurl}}/img/kubernetes/practice_prometheus_1.png)  

위 그림은 `Prometheus` 를 사용한 아키텍쳐의 예시이고, 아래와 같은 컴포넌트로 구성된다. 
- `Prometheus Server` : 시계열 데이터를 수집해서 저장한다. 
- `Client Library` : 애플리케이션 개발할 때 `Prometheus` 에서 데이터를 수집할 수 있또록 만든 라이브러리이다. 
- `Push Gateway` : 클라이언트에서 직접 `Proemtheus` 로 데이터를 보낼 때 받는 역할을 한다. 
- `Exporter` : `Prometheus` 클라이언트 라이브러리를 내장해서 만들지 않은 애플리케이션에서 데이터를 수집하는 역할을 한다. 
100개 가량의 다양한 종류가 있어서 거의 대부분의 애플리케이션에서 `Metric` 수집이 가능하다. 
- `Alertmanager` : 알람을 보낼 때 중복 처리, 그룹화 등 알람을 보내는 관리를 한다. 

`Prometheus` 는 `Server-Client` 구조에서 `Exporter` 는 어떤 `Metric` 를 수집할지 정의된 상태에서 
정해진 주기마다 `Metric` 을 수집한다. 
`Prometheus Server` 는 `Node Exporter` 목록을 가지고 있으면서 주기적으로 `Exporter` 에 요청을 보내 `Pull` 방식으로 `Metrics` 을 수집한다.  

`kubernetes Cluster` 에 `Prometheus` 는 `helm` 을 사용해서 간편하게 설치할 수 있지만, 
구성하는 구성요소들을 알아보며 설치하기 위해 직접 템플릿을 구성해서 설치를 진행한다.  

## Kubernetes Metric
`Kubernetes` 는 클러스터의 상태를 자동으로 모니터링해서 보여주지 않는다. 
즉 모니터링을 위해서는 별도로 클러스터에 구성을 해주어야 한다. 
관련해서 검색하면 아래 2가지 결과를 얻을 수 있다. 
- `metric-server`
- `kube-state-metrics` 

먼저 `metric-server` 는 `Kubernetes` 의 `Node`, `Pod` 등에 대한 자원을 모니터링해주는 도구 이다. 
모니터링이 가능하기 때문에 `metric-server` 에서 수집하는 `Metric` 을 기반으로 `HPA` 동작도 설정할 수 있다. 
또한 `Kubernetes` 의 `Health` 상태도 함께 체크를 해주기 때문에, 
모든 `Metric` 의 정보가 `Unknown` 이라면 `Kubernetes` 에 문제가 발생 했음을 인지 할 수도 있다. 
하지만 `metric-server` 의 `Metirc` 은 `Prometheus` 에서 수집할 수 없다는 점이 있다.  

`kube-state-metrics` 는 `metric-server` 와 달리 `Kubernetes` 의 `Health` 상태를 체크해주지 않지만, 
`Prometheus` 를 이용한 모니터링 구축이 가능하다. 

## kube-state-metrics Install
`kube-state-metrics` 는 `Kubernetes Cluster` 에서 `Pod` 의 수, 리소스, 네트워크 등등의 다양한 `Metric` 을 제공해 준다. 
`kube-state-metrics` 를 설치하지 않고 `Prometheus` 구성이 불가능한 것은 아니지만, 
`Kubernetes Cluster` 를 `Prometheus` 로 모니터링하기 위해서는 설치가 필요하다.  

`kube-state-metrics` 설치는 [여기](https://github.com/kubernetes/kube-state-metrics) 저장소를 클론 하는 방법으로 가능하다. 
본 포스트에서는 현재 `master` 브랜치인 `v2.0.0-rc.0` 을 설치한다.  

```bash
$ git clone https://github.com/kubernetes/kube-state-metrics.git
$ cd kube-state-metrics
$ kubectl apply -f examples/standard
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
service/kube-state-metrics created
$ kubectl get pod -n
kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-bq2t8                  1/1     Running   3          14d
coredns-f9fd979d6-f7kj6                  1/1     Running   3          14d
etcd-docker-desktop                      1/1     Running   3          14d
kube-apiserver-docker-desktop            1/1     Running   3          14d
kube-controller-manager-docker-desktop   1/1     Running   3          14d
kube-proxy-c979r                         1/1     Running   3          14d
kube-scheduler-docker-desktop            1/1     Running   7          14d
kube-state-metrics-84f97ffcb4-7z6s4      1/1     Running   0          99s
metrics-server-5b78d5f9c6-txh2q          1/1     Running   5          14d
storage-provisioner                      1/1     Running   9          14d
vpnkit-controller                        1/1     Running   3          14d
```  

## Prometheus Install
앞서 언급했던 것처럼 `Prometheus` 는 `helm` 을 이용하면 이미 구성된 템플릿을 사용해서, 
관련 설정값만 현재 환경에 맞게 변경해서 설치할 수 있지만 직접 템플릿을 구성하며 진행해 본다.  

먼저 `Prometheus` 를 구성할 별도의 `Namespace` 를 만들어 진행 한다. 
`Prometheus` 는 크게 설정파일, 데이터 저장 파일로 나뉘는데 아래와 같다. 
- `prometheus.yaml` : 설정 파일
- `tsdb` : 데이터 저장 파일

`prometheus.yaml` 파일은 `Prometheus` 를 설정하는 파일로 `Configmap` 을 사용해서 구성한다. 
그리고 `tsdb` 는 수집된 `Metric` 이 저장되는 파일로 영구적인 저장이 필요하기 때문에 `PV-PVC` 를 사용해서 구성할 것이다.  

### monitoring namespace
`Prometheus` 를 구성할 네임스페이스를 아래 템플릿 파일을 사용해서 구성해 준다. 

```yaml
# prometheus-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```  

```bash
$ kubectl apply -f prometheus-namespace.yaml
namespace/monitoring created

.. 아래 명령으로도 가능하다 ..
$ kubectl create namespace monitoring

$ kubectl get ns
NAME              STATUS   AGE
default           Active   14d
kube-node-lease   Active   14d
kube-public       Active   14d
kube-system       Active   14d
monitoring        Active   94s
```  

### ClusterRole
`Kubernetes` 의 자원에 대한 접근은 `kubernetes API` 를 통해 이뤄진다. 
`Prometheus` 가 `Kubernetes API` 에 접근해 자원에 대한 메트릭 생성을 해야하기 때문에 `ClusterRole` 을 사용해서 접근 권한을 설정해 준다. 
`ClusterRole` 으로 권한에 대한 설정을 명시하고, 
`Prometheus` 가 사용할 `ServiceAccount` 를 생성해준 다음, 
`ClusterRoleBinding` 으로 `ClusterRole` 과 `ServiceAccount` 를 바인딩 해준다.  

```yaml
# prometheus-cluster-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups:
      - extensions
    resources:
      - ingress
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
```  

```bash
$ kubectl apply -f prometheus-cluster-role.yaml
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```  

### PV-PVC
`Prometheus` 의 `Metric` 이 저장되는 영구 저장소를 설정한다. 
`PV` 를 사용해서 사용할 볼륨에 대한 설정을 한다. 
그리고 `PVC` 를 사용해서 설정한 볼륨 `PV` 에서 사용할 용량과 함께 이후 `Prometheus Pod` 에서 사용할 수 있도록 설정해 준다.  

```yaml
# prometheus-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
  namespace: monitoring
  labels:
    type: local
    app: prometheus
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /run/desktop/mnt/host/c/k8s/prometheus
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - docker-desktop
``` 

>여기서 `.spec.hostPath.path` 가 `/run/desktop/mnt/host/c/k8s/prometheus` 로 되어 있는 것을 확인 할 수 있다. 
>이는 지금 `docker desktop for windows` 를 `wsl` 환경에서 사용하고 있기 때문이다. 
>`/run/desktop/mnt/host/c/k8s/prometheus` 경로는 실제 `Windows` 경로 `C:\k8s\prometheus` 와 매핑된다.  

```yaml
# prometheus-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  namespace: monitoring
  labels:
    type: local
    app: prometheus
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      app: prometheus
      type: local
```  

```bash
$ kubectl get pv,pvc -l app=prometheus -n monitoring
NAME                             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   REASON   AGE
persistentvolume/prometheus-pv   2Gi        RWO            Retain           Bound    monitoring/prometheus-pvc   manual                  24s

NAME                                   STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/prometheus-pvc   Bound    prometheus-pv   2Gi        RWO            manual         18s
```  

### prometheus 설정 파일
`Prometheus` 의 설정파일은 `ConfigMap` 을 사용해서 생성하고 이후 `Prometheus` 의 볼륨으로 마운트해서 사용한다. 
이를 하나의 템플릿으로 구성하면 아래와 같다. 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yaml: |
    global:
      scrape_interval: 2s
      evaluation_interval: 2s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
        - scheme: http
          static_configs:
            - targets:
                - "alertmanager.monitoring.svc:9003"

    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
          - role: node

        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-pod'

        kubernetes_sd_configs:
          - role: pod

        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path_
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name

      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc:8080']

      - job_name: 'kubernetes-cadvisor'
        scrape_interval: 5s
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
          - role: node

        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
          - role: endpoints

        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: kubernetes_name

  prometheus.rules: |
    groups:
    - name: container memory alert
      rules:
      - alert: container memory usage rate is very high( > 55%)
        expr: sum(container_memory_working_set_bytes{pod!="", name=""}) / sum (kube_node_status_allocatable_memory_bytes) * 100 > 55
        for: 1m
        labels:
          severity: fatal
        annotations:
          summary: High Memory Usage on {{ $labels.instance }}
          identifier: "{{ $labels.instance }}"
          description: "{{ $labels.job }} Memory Usage: {{ $value }}"
    - name: container CPU alert
      rules:
      - alert: container CPU usage rate is very high( > 10%)
        expr: sum (rate (container_cpu_usage_seconds_total{pod!=""}[1m])) / sum (machine_cpu_cores) * 100 > 10
        for: 1m
        labels:
          severity: fatal
        annotations:
          summary: High Cpu Usage
```  

`prometheus.yaml` 에는 어떤 `Metric` 을 어디에서 어떻게 수집할지에 대한 기술을 하고 있다. 
지금 설정은 2초 단위로 메트릭 정보를 수집하고, `kube-state-metrics` 는 서비스 디스커버리 도메인을 사용해서 호출을 수행하고 있다. 

```bash
$ kubectl apply -f prometheus-config.yaml
configmap/prometheus-config created
```  

### Prometheus Server
`Prometheus Server` 는 `Deployment` 와 `Service` 를 사용해서 구성한다. 
그리고 앞서 먼저 적용한 `PVC`, `ConfigMap` 등을 사용해서 구성을 진행한다.  

```yaml
# prometheus-deploy.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deploy
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yaml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      nodeSelector:
        kubernetes.io/hostname: docker-desktop
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-config
        - name: prometheus-storage-volume
          persistentVolumeClaim:
            claimName: prometheus-pvc

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus-server
  name: prometheus-svc
  namespace: monitoring
spec:
  ports:
    - nodePort: 30990
      port: 9090
      protocol: TCP
      targetPort: 9090
  selector:
    app: prometheus-server
  type: NodePort
```  

`ServiceAccount` 을 사용해서 권한을 설정하고, `PVC` 볼륨을 사용해서 영구 저장소 설정을 한다. 
그리고 `ConfigMap` 을 마운트 해서 `Prometheus Server` 설정에 필요한 파일들을 마운트 한다. 
`Service` 에서는 `NodePort` 를 사용해서 `Prometheus Server` 의 `9090` 포트를 `30990` 포트로 바인딩 한다.  

```bash
$ kubectl apply -f prometheus-deploy.yaml
deployment.apps/prometheus-deploy created
service/prometheus-svc created
$kubectl get deploy,pod -l app=prometheus-server -n monitoring
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-deploy   1/1     1            1           80s

NAME                                     READY   STATUS    RESTARTS   AGE
pod/prometheus-deploy-568fc9cc69-k7mwx   1/1     Running   0          80s
```  

만약 `Pod` 의 상태가 `Error` 라면 `PVC` 볼륨 경로의 권한을 `755` 변경해 준다. 

```bash
$ chmod -R 755 /mnt/c/k8s/prometheus
```  

### Node Exporter
`Node Exporter` 는 앞서 설명한 `Prometheus` 의 `Exporter` 의 일종으로 `Kubernetes Cluster` 에서 
각 `Node` 에 대한 정보를 수집하는 역할을 한다. 
각 `Node` 마다 `Pod` 1개만 동작하면 되기 때문에 `DaemonSet` 을 사용해서 구성한다. 

```yaml
# prometheus-node-export.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
        - image: prom/node-exporter
          name: node-exporter
          ports:
            - containerPort: 9100
              protocol: TCP
              name: http

---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  ports:
    - name: http
      port: 9100
      nodePort: 31672
      protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```  

그리고 다른 `Pod` 에서 접근해서 `Metric` 을 가져갈 수 있도록 하기 위해서 `Service` 사용해서, 
`node-export` 의 `9100` 포트를 `31672` 포트로 바인딩 해준다.  

```bash
$ kubectl apply -f prometheus-node-exporter.yaml
daemonset.apps/node-exporter created
service/node-exporter created
```  

### 테스트 
웹 브라우저를 사용해서 `localhost:30990` 으로 접속하면 `Prometheus` 의 기본 모니터링 UI를 확인 할 수 있다.  


---
## Reference






