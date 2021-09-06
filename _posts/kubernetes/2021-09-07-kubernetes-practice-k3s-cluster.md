--- 
layout: single
classes: wide
title: "[Kubernetes 실습] K3S 소개와 클러스터 구성"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Lightweight Kubernetes 도구인 K3S에 대해 알아보고 클러스터를 구성하는 방법에 대해서도 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - K3S
  - Cluster
toc: true
use_math: true
---  

## 환경
- `Ubuntu 20.04.3 LTS`
- `k3s version v1.21.4+k3s1 (3e250fdb)`

## K3S
`K3S` [공식 홈페이지](https://k3s.io/)
에 접속하면 볼 수 있는 `Lightweight Kubernetes` 라는 글자와 같이 가벼운 `Kubernetes` 이다.  
컨테이너 관련 기술을 주로 개발하는 `Rancher Labs` 에서 개발한 `Kubernetes` 의 다른 버전이라고 할 수 있다. 
`Kubernetes` 처럼 컨테이너 오케이스트레이션을 목적으로 만들어 진것을 동일하지만 몇가지 차이점이 있다. 
`Kubernetes` 클러스터 구성을 위해서는 툴을 설치하고 이들간의 연동하는 복잡한 작업이 필요하다. 
이러한 이유로 `Kubespray`, `Kubeadm` 와 같은 도구를 제공하지만, 
구성하는데 오랜 시간이 걸리고 문서대로 진행하더라도 항상 에러를 만나게 되는 불편함이 있다. 
완전한 `Kubernetes` 클러스터 구성은 아니겠지만, 클러스터 구성, 테스트 더나아가 서비스까지 경험해 볼 수 있는 간편한 도구라고 생각한다.  

몇가지 `K3S` 의 특징을 나열하면 아래와 같다. 

- `100MB` 정도 되는 바이너리 파일 하나로 간단한 구성이 가능하다. 
- `sqlite3` 를 기반으로하는 가벼운 저장소 메커니즘을 통해 경량 저장소 백엔드를 사용한다. 
  원한다면 `etcd3`, `MySQL`, `Postgres` 등을 저장소로 사용할 수 있다.
- `TLS` 와 옵션의 복잡성을 단순하게 처리하는 런쳐를 제공한다. 
- 경량 환경에 알맞는 적합한 보안을 기본으로 제공한다. 
- 로컬 저장소, 서비스 로드 밸런서, `Helm`, `Treafix ingress` 등을 제공한다. 
- `Kubernetes` 의 `Control plane` 에 대한 동작은 모두 캡슐화 되어 하나의 바이너리 파일과 프로세스로 가능하다. 
이로 인해 복잡한 클러스터에 대한 관리 동작을 `K3S` 가 자동으로 수행해 준다.
- 최소화된 외부 의존성으로 구성된다. `K3S` 에서 요구하는 의존성은 아래와 같다. 
  - containerd(`Docker` 도 가능)
  - Flannel
  - CoreDNS
  - CNI
  - Host utilities(iptables, socat, etc)
  - Ingress controller(Traefik)
  - Embedded service loadbalancer
  - Embedded network policy controller
	

## Cluster
`K3S` 는 `Kubernetes` 의 경량화 버전이기 때문에 멀티노드를 바탕으로하는 클러스터 구성을 제공한다.
각 노드의 역할을 나누면 아래와 같다. 

Node Type|Desc
---|---
k3s server|`k8s` 의 `master` 혹은 `control plane` 역할을 하는 클러스터의 제어부
k3s agent|`k3s server` 의 명령을 받아 동작을 수행하는 `Worker` 역할로 일반 `k8s` 노드 역할

이후 부터는 `K3S` 에서 어떠한 클러스터 구성을 제공하는지에 대해서 알아본다.  

### Single-Server with Embedded DB

![그림 1]({{site.baseurl}}/img/kubernetes/practice-k3s-cluster-1.png)  

가장 간단하게 구성할 수 있는 `K3S Cluster` 구성으로 하나의 `k3s server` 를 두고 `N` 개의 `k3s agent` 를 통해 클러스터를 확장하는 구조이다. 
모든 클러스터에 대한 정보나 구성되는 노드에 대한 정보는 `k3s server` 에 존재하는 `SQLite` 데이터베이스에 저장되고 관리된다. 
간편하게 구성할 수 있고, 관리하기도 용이하지만 클러스터를 관리하고 관제하는 노드가 1개이기 때문에 안전성 측면에서 좋지 않다는 단점이 있다. 


### High-Availability Server with External DB

![그림 1]({{site.baseurl}}/img/kubernetes/practice-k3s-cluster-2.png)  

`Single-Server` 구조에서 치명적인 안정성에 대한 부분을 보완할 수 있는 구조이다. 
`N` 개의 `k3s server` 와 `N` 개의 `k3s agent` 를 통해 클러스터를 구성할 수 있다. 
클러스터를 관리하고 관제하는 `k3s server` 가 `N` 개이기 때문에 좀 더 고가용성 측면에서 안전성이 확보된 구조라고 할 수 있다. 
한가지 더 `Single-Server` 와 차이점은 클러스터 정보를 저장하는 데이터베이스가 `SQLite` 가 아닌 다른 저장소를 사용해야 한다. 
외부에 존재하는 `MySQL`, `Postgres` 를 사용할 수도 있고, 기존 `k8s` 와 동일하게 `etcd` 를 설치해서 사용할 수도 있다. 


## 예제
총 3대의 테스트 노드를 준비해서 `k3s` 를 설치하고 클러스터를 구성하는 예제를 진행해 본다. 

Node Name|OS|IP
---|---|---
node-01|Ubuntu 20.04.3 LTS|192.168.0.101
node-02|Ubuntu 20.04.3 LTS|192.168.0.102
node-03|Ubuntu 20.04.3 LTS|192.168.0.103

예제는 `Ubuntu` 이미지가 설치된 직후의 상황에서 수행된다. 
`k3s` 설치후 잘 구성됐는지 테스트가 필요한 경우, 
아래 `Nginx` 를 바탕으로 구성된 테스트 템플릿인 `test-nginx.yaml` 을 실행한다.  

```yaml
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

---
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
      nodePort: 32222
```  

```bash
$ kubectl apply -f test-nginx.yaml
$ kubectl scale --replicas=10 deploy/deployment-nginx
$ kubectl delete -f test-nginx.yaml
```  

### K3S Server 설치<a id="install"></a>
먼저 `node-01` 노드를 사용해서 `k3s` 를 설치하는 예제를 진행해본다.  

> 참고로 예제에서는 `k3s` 에서 사용하는 기본 컨테이너환경인 `containerd` 를 사용하지 않고, 
> `Docker` 를 사용해서 설치하는 방법에 대해서 알아본다. 

`OS` 가 막 설치된 상황이므로 패키지 업데이트 먼저 수행해 준다.  

```bash
$ sudo apt -y update; sudo apt -y upgrade
```  

`/boot/firmware/cmdline.txt` 에서 `cgroups` 에 대한 활성화가 돼있는지 확인 후, 
돼있지 않다면 아래 명령어로 파일의 마지막 부분에 `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory` 내용을 추가해 준다. 

```bash
$ origin=`cat /boot/firmware/cmdline.txt`; echo "${origin} cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory" | sudo tee /boot/firmware/cmdline.txt
```  

`Docker` 를 설치해 준다.  

```bash
$ curl https://releases.rancher.com/install-docker/19.03.sh | sudo sh
.. 생략 ..

$ sudo docker version
Client: Docker Engine - Community
 Version:           19.03.15
 API version:       1.40
 Go version:        go1.13.15
 Git commit:        99e3ed8
 Built:             Sat Jan 30 03:17:52 2021
 OS/Arch:           linux/arm64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.15
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       99e3ed8
  Built:            Sat Jan 30 03:16:22 2021
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.4.9
  GitCommit:        e25210fe30a0a703442421b0f60afac609f950a3
 runc:
  Version:          1.0.1
  GitCommit:        v1.0.1-0-g4144b63
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```  

이후 `K3S Cluster` 구성에서 필요한 부분이지만, 
`6443`, `443` 포트에 대한 방화벽 허용 설정을 해준다. 

```bash
sudo ufw allow 6443/tcp
sudo ufw allow 443/tcp
```  

이제 `k3s` 를 설치할 모든 준비가 됐으므로, 
`k3s` 설치 명령을 아래와 같이 수행하면 설치가 완료된다. 

> `Docker` 컨테이너 환경을 사용하지 않다면 `--docker` 옵션은 필요하지 않다. 

```bash
$ curl -sfL https://get.k3s.io | sudo sh -s - --docker
.. 생략 ..
```  

정상적으로 모두 설치 됐는지 확인해 보면 아래와 같다.  

```bash
$ sudo systemctl status k3s
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-09-06 18:18:41 UTC; 14s ago
       Docs: https://k3s.io
    Process: 983141 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service (code=exited, status=0/SUCCESS)
    Process: 983143 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 983144 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 983145 (k3s-server)
      Tasks: 25
     Memory: 643.0M
     CGroup: /system.slice/k3s.service
             └─983145 /usr/local/bin/k3s server

Sep 06 18:18:55 node-01 k3s[983145]: W0906 18:18:55.043472  983145 empty_dir.go:520] Warning: Failed to clear quota on /var/lib/kubelet/pods/3d55ff0>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.044293  983145 operation_generator.go:829] UnmountVolume.TearDown succeeded for volume "kubernet>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.145870  983145 operation_generator.go:829] UnmountVolume.TearDown succeeded for volume "kubernet>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.150880  983145 reconciler.go:319] "Volume detached for volume \"values\" (UniqueName: \"kubernet>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.150997  983145 reconciler.go:319] "Volume detached for volume \"content\" (UniqueName: \"kuberne>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.151045  983145 reconciler.go:319] "Volume detached for volume \"kube-api-access-w84ht\" (UniqueN>
Sep 06 18:18:55 node-01 k3s[983145]: I0906 18:18:55.192507  983145 pod_container_deletor.go:79] "Container not found in pod's containers" containerI>
Sep 06 18:18:55 node-01 k3s[983145]: E0906 18:18:55.307704  983145 available_controller.go:508] v1beta1.metrics.k8s.io failed with: failing or missi>
Sep 06 18:18:56 node-01 k3s[983145]: I0906 18:18:56.084144  983145 event.go:291] "Event occurred" object="kube-system/traefik" kind="Deployment" api>
Sep 06 18:18:56 node-01 k3s[983145]: I0906 18:18:56.175357  983145 event.go:291] "Event occurred" object="kube-system/traefik-97b44b794" kind="Repli>

$ sudo kubectl get node -o wide
NAME      STATUS   ROLES                  AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,master   70s   v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15

$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
5bc9f685fef9        8700db2e97e2        "entry"                  42 seconds ago       Up 41 seconds                           k8s_lb-port-443_svclb-traefik-clcps_kube-system_d1a70fb6-47aa-4791-a293-3d442b4ae598_0
ed1df9b3e52f        8700db2e97e2        "entry"                  43 seconds ago       Up 42 seconds                           k8s_lb-port-80_svclb-traefik-clcps_kube-system_d1a70fb6-47aa-4791-a293-3d442b4ae598_0
233e1b93e9a2        78580c41697c        "/entrypoint.sh --gl…"   44 seconds ago       Up 43 seconds                           k8s_traefik_traefik-97b44b794-b5kr4_kube-system_8ea662d4-c617-454e-9a30-cf8f3c08d070_0
d21c46edf7d7        rancher/pause:3.1   "/pause"                 46 seconds ago       Up 44 seconds                           k8s_POD_svclb-traefik-clcps_kube-system_d1a70fb6-47aa-4791-a293-3d442b4ae598_0
83e86bd1a950        rancher/pause:3.1   "/pause"                 47 seconds ago       Up 44 seconds                           k8s_POD_traefik-97b44b794-b5kr4_kube-system_8ea662d4-c617-454e-9a30-cf8f3c08d070_0
b8ac73907091        f9499facb1e8        "/metrics-server"        59 seconds ago       Up 58 seconds                           k8s_metrics-server_metrics-server-86cbb8457f-qz2rn_kube-system_9e02ce96-6f4f-4dd5-a45b-b84e7a2d772a_0
3d1d3bf9dae0        5847bbe01541        "/coredns -conf /etc…"   59 seconds ago       Up 58 seconds                           k8s_coredns_coredns-7448499f4d-7q2qq_kube-system_66b81209-354a-46d9-b534-e5ee5a3dc333_0
6d1ee8a49b98        72e75b6d97fe        "local-path-provisio…"   59 seconds ago       Up 58 seconds                           k8s_local-path-provisioner_local-path-provisioner-5ff76fc89d-8r4hf_kube-system_40918f6f-925e-4bc8-9212-a839cad18155_0
4e5aef16cbca        rancher/pause:3.1   "/pause"                 About a minute ago   Up 59 seconds                           k8s_POD_coredns-7448499f4d-7q2qq_kube-system_66b81209-354a-46d9-b534-e5ee5a3dc333_0
7e0d94692806        rancher/pause:3.1   "/pause"                 About a minute ago   Up 59 seconds                           k8s_POD_metrics-server-86cbb8457f-qz2rn_kube-system_9e02ce96-6f4f-4dd5-a45b-b84e7a2d772a_0
acc939ea4005        rancher/pause:3.1   "/pause"                 About a minute ago   Up About a minute                       k8s_POD_local-path-provisioner-5ff76fc89d-8r4hf_kube-system_40918f6f-925e-4bc8-9212-a839cad18155_0
```  

### 삭제
설치만큼 중요한 작업이 삭제인 만큼 메뉴얼대로 삭제하는 방법에 대해 알아본다. 

아래 `/usr/local/bin` 경로를 조회하면 아래와 같이 `k3s` 관련 스크립트가 존재하는 것을 확인 할 수 있다.  

```bash
$ ll /usr/local/bin/k3s-*
-rwxr-xr-x 1 root root 1813 Sep  5 18:43 /usr/local/bin/k3s-killall.sh*
-rwxr-xr-x 1 root root 1047 Sep  5 18:43 /usr/local/bin/k3s-uninstall.sh*
```  

> 만약 `k3s agent` 라면 `k3s-agent-uninstall.sh` 와 같은 파일이 존재한다.  

파일이 존재한다면 삭제는 아래 명령어를 차례대로 실행하는 방법으로 진행해 준다.  

```bash
$ sudo /usr/local/bin/k3s-killall.sh
$ sudo /usr/local/bin/k3s-uninstall.sh
.. 생략 ..

$ sudo rm -rf /var/lib/rancher
$ sudo docker stop `sudo docker ps -qa`
$ sudo docker rm `sudo docker ps -qa`
```  


### Single-Server 클러스터 구성하기
`node-01` ~ `node-03` 을 사용해서 `Single-Server` 클러스터를 구성해 본다. 

Node Name|Node Type
---|---
node-01|k3s server
node-02|k3s agent
node-03|k3s agent

`node-01` 는 [설치](#install)를 모두 진행해 준다. 
그리고 아래 명령어를 통해 `node-01` 에서 `k3s server` 의 토큰 값을 조회할 수 있다.  

```bash
.. node-01 ..
$ sudo cat /var/lib/rancher/k3s/server/node-token
K10d87c2a2e4fe8d1087823341bf8894786d412b2f694f196dfbfa9516cc87f9436::server:3b9b4c87ea98b9559f5bd633592d6228
```  

이제 위 토큰 값을 사용해서 `node-02`, `node-03` 에 `k3s agent` 구성을 진행한다.
[설치](#install)에서 `k3s` 설치 명령어 실행 전인 `Docker` 설치까지만 진행한다. 
`k3s agent` 구성을 위해서는 `k3s server` 설치 명령과는 차이가 있는데 구성은 아래와 같다.  

```
.. sudo 명령어 없이 실행 한다 ..
curl -sfL https://get.k3s.io | K3S_URL=https://<K3S_SERVER_IP>:6443 K3S_TOKEN=<TOKEN> sh -s - --docker
```  

```bash
.. node-02, node-03 ..
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.101:6443 K3S_TOKEN=K10d87c2a2e4fe8d1087823341bf8894786d412b2f694f196dfbfa9516cc87f9436::server:3b9b4c87ea98b9559f5bd633592d6228 sh -s - --docker
```  

설치 확인을 하면 아래와 같다.  

```bash
.. node-02, node-03 동일 ..
$ sudo systemctl status k3s-agent
● k3s-agent.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s-agent.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-09-06 18:25:16 UTC; 36s ago
       Docs: https://k3s.io
    Process: 658196 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service (code=exited, status=0/SUCCESS)
    Process: 658198 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 658199 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 658205 (k3s-agent)
      Tasks: 18
     Memory: 216.3M
     CGroup: /system.slice/k3s-agent.service
             └─658205 /usr/local/bin/k3s agent

Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.179075  658205 iptables.go:160] Adding iptables rule: ! -s 10.42.0.0/16 -d 10.42.0.0/16 -j MASQU>
Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.829862  658205 kubelet_getters.go:300] "Path does not exist" path="/var/lib/kubelet/pods/9242fd4>
Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.830096  658205 kubelet_getters.go:300] "Path does not exist" path="/var/lib/kubelet/pods/dbfc6af>
Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.830207  658205 kubelet_getters.go:300] "Path does not exist" path="/var/lib/kubelet/pods/e2531c7>
Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.830345  658205 kubelet_getters.go:300] "Path does not exist" path="/var/lib/kubelet/pods/83c46ae>
Sep 06 18:25:28 node-02 k3s[658205]: I0906 18:25:28.830445  658205 kubelet_getters.go:300] "Path does not exist" path="/var/lib/kubelet/pods/749601d>
Sep 06 18:25:29 node-02 k3s[658205]: I0906 18:25:29.064071  658205 pod_container_deletor.go:79] "Container not found in pod's containers" containerI>
Sep 06 18:25:29 node-02 k3s[658205]: time="2021-09-06T18:25:29.136891465Z" level=info msg="labels have been set successfully on node: node-02"
Sep 06 18:25:29 node-02 k3s[658205]: I0906 18:25:29.295908  658205 network_policy_controller.go:144] Starting network policy controller
Sep 06 18:25:29 node-02 k3s[658205]: I0906 18:25:29.492705  658205 network_policy_controller.go:154] Starting network policy controller full sync go>
```  

`k3s agent` 에서는 `kubectl` 명령이 불가능하다. 
`k3s server` 인 `node-01` 에서 클러스터 노드를 조회하면 아래와 같다.  

```bash
$ sudo kubectl get node -o wide
NAME      STATUS   ROLES                  AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,master   7m55s   v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready    <none>                 49s     v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-03   Ready    <none>                 57s     v1.21.4+k3s1   192.168.0.103   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
```  

`node-01` 에서 `test-nginx.yaml` 을 적용하고 `scale` 을 20으로 설정하면 `Pod` 가 3개의 노드에 골고루 분포돼서 실행되는 것을 확인할 수 있다.  

```bash
$ sudo kubectl apply -f test-nginx.yaml
deployment.apps/deployment-nginx created
service/service-nodeport created
$ sudo kubectl scale --replicas=20 deploy/deployment-nginx
deployment.apps/deployment-nginx scaled
$ sudo kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
deployment-nginx-5bcf797978-8nqpq   1/1     Running   0          51s   10.42.2.3    node-02   <none>           <none>
deployment-nginx-5bcf797978-gcv8m   1/1     Running   0          51s   10.42.1.3    node-03   <none>           <none>
deployment-nginx-5bcf797978-m7m6b   1/1     Running   0          51s   10.42.2.4    node-02   <none>           <none>
deployment-nginx-5bcf797978-mk2db   1/1     Running   0          28s   10.42.2.6    node-02   <none>           <none>
deployment-nginx-5bcf797978-68tfp   1/1     Running   0          28s   10.42.1.4    node-03   <none>           <none>
deployment-nginx-5bcf797978-4sdkw   1/1     Running   0          28s   10.42.2.5    node-02   <none>           <none>
deployment-nginx-5bcf797978-qprk9   1/1     Running   0          28s   10.42.1.5    node-03   <none>           <none>
deployment-nginx-5bcf797978-rpqj5   1/1     Running   0          28s   10.42.1.6    node-03   <none>           <none>
deployment-nginx-5bcf797978-kxtcc   1/1     Running   0          28s   10.42.2.7    node-02   <none>           <none>
deployment-nginx-5bcf797978-mfgjx   1/1     Running   0          28s   10.42.0.9    node-01   <none>           <none>
deployment-nginx-5bcf797978-4ncwt   1/1     Running   0          28s   10.42.1.7    node-03   <none>           <none>
deployment-nginx-5bcf797978-cs8ns   1/1     Running   0          28s   10.42.2.8    node-02   <none>           <none>
deployment-nginx-5bcf797978-lmxzt   1/1     Running   0          28s   10.42.0.10   node-01   <none>           <none>
deployment-nginx-5bcf797978-cztbl   1/1     Running   0          28s   10.42.2.9    node-02   <none>           <none>
deployment-nginx-5bcf797978-zq64t   1/1     Running   0          28s   10.42.0.11   node-01   <none>           <none>
deployment-nginx-5bcf797978-vhztd   1/1     Running   0          28s   10.42.1.8    node-03   <none>           <none>
deployment-nginx-5bcf797978-d9wrv   1/1     Running   0          28s   10.42.2.10   node-02   <none>           <none>
deployment-nginx-5bcf797978-2jb8c   1/1     Running   0          28s   10.42.1.9    node-03   <none>           <none>
deployment-nginx-5bcf797978-4rjcd   1/1     Running   0          28s   10.42.0.12   node-01   <none>           <none>
deployment-nginx-5bcf797978-jx9m9   1/1     Running   0          28s   10.42.0.13   node-01   <none>           <none>
```  


### High-Availability Server 클러스터 구성하기
`node-01` ~ `node-03` 을 사용해서 `High-Availability Server` 클러스터를 구성해 본다.  

Node Name|Node Type
---|---
node-01|k3s server
node-02|k3s server
node-03|k3s server

먼저 `node-01` ~ `node-03` 각각 [설치](#install)를 모두 진행해 준다. 
`High-Availability Server` 클러스터 구성을 위해서는 기준이 되는 하나의 노드와 그외 노드의 구성 명령이 다르다. 
기준이 되는 노드를 `node-01` 로 하고 그외 노드는 `node-02`, `node-03` 으로 한다. 
그리고 명령어 수행은 `sudo su` 를 통해 `root` 로 전환 후 수행한다.  

먼저 `node-01` 노드에서 토큰값을 알아낸후, `6443` 포트를 사용하는 프로세스를 죽이고 나서 토큰을 사용해서 클러스터 설정 명령을 수행한다.  

```bash
node-01# cat /var/lib/rancher/k3s/server/node-token
K10fe1d33e1017abe5ac4415a741122d00606fd09c23818458558449aa5d3256491::server:0e06810bca0ffec166240969f7779458
node-01# sudo kill -9 $(sudo lsof -i :6443 | grep LISTEN | awk '{print $2}')
node-01# K3S_TOKEN=K10fe1d33e1017abe5ac4415a741122d00606fd09c23818458558449aa5d3256491::server:0e06810bca0ffec166240969f7779458 sudo k3s server --cluster-init
.. wait some minutes ..
```  

>만약 아래와 같은 에러 메시지가 뜬다면 다시 `6443` 프로세스를 죽이고 클러스터 구성 명령을 실행한다.
> 
> ```bash
> FATA[2021-09-06T19:08:58.293645221Z] starting kubernetes: preparing server: init cluster datastore and https: listen tcp :6443: bind: address already in use
> ```  

명령을 실행하고 몇분 정도 기다려 준다. 
대략 3 ~ 5분 정도 기다리면 특정 메시지(에러 메시지 포함)세트가 반복돼서 출력되거나 출력이 멈춘것을 확인 할 수 있다. 
이때 `Ctrl + C` 를 눌러 강제 종료 시키고 아래 명령으로 `k3s` 서비스를 재시작 한다.  

```bash
node-01# systemctl restart k3s
```  

그리고 클러스터 노드정보를 조회하면 기존 노드정보와 달리 `etcd` 가 추가된 것을 확인 할 수 있다.  

```bash
node-01# kubectl get node -o wide
NAME      STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,etcd,master   3m39s   v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
```  

이제 `node-02`, `node-03` 차례로 `k3s` 관련 명령에만 차이가 있고, 대부분 비슷하다. 

```bash
node-02,node-03# kill -9 $(sudo lsof -i :6443 | grep LISTEN | awk '{print $2}')
node-02,node-03# K3S_TOKEN=K10fe1d33e1017abe5ac4415a741122d00606fd09c23818458558449aa5d3256491::server:0e06810bca0ffec166240969f7779458 k3s server --server https://192.168.0.101:6443
.. wait some minutes ..
```  

`node-01` 과 동일하게 3 ~ 5분 정도 기다린 후, 
`Ctrl + C` 명령으로 강제 종료 시킨다. 
그리고 아래 명령으로 `k3s` 서비스를 재시작 한다.  

```bash
node-02,node-03# systemctl restart k3s
```  

그리고 `node-01` ~ `node-03` 에서 모드 클러스터 노드 정보를 조회하면 아래와 같이, 
모든 노드가 `k3s server` 로 클러스터에 등록됐고, `etcd` 까지 설치된 것을 확인 할 수 있다.  

```bash
node-01# kubectl get node -o wide
NAME      STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,etcd,master   15m     v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready    control-plane,etcd,master   2m54s   v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-03   Ready    control-plane,etcd,master   2m38s   v1.21.4+k3s1   192.168.0.103   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15

node-02# kubectl get node -o wide
NAME      STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,etcd,master   16m     v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready    control-plane,etcd,master   3m24s   v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-03   Ready    control-plane,etcd,master   3m8s    v1.21.4+k3s1   192.168.0.103   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15

node-03# kubectl get node -o wide
NAME      STATUS   ROLES                       AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,etcd,master   16m     v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready    control-plane,etcd,master   3m46s   v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-03   Ready    control-plane,etcd,master   3m30s   v1.21.4+k3s1   192.168.0.103   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
```  

테스트를 위해 `test-nginx.yaml` 적용하고 `scale` 을 20으로 조정하면 모든 `k3s server` 에 골고루 `Pod` 가 분배된 것을 확인 할 수 있다. 
`kubectl` 명렁은 모든 노드에서 다 사용할 수 있다.  

```bash
node-02# kubectl apply -f test-nginx.yaml
deployment.apps/deployment-nginx created
service/service-nodeport created
node-02# kubectl scale --replicas=20 deploy/deployment-nginx
deployment.apps/deployment-nginx scaled
node-02# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
deployment-nginx-5bcf797978-2l574   1/1     Running   0          47s   10.42.2.4    node-03   <none>           <none>
deployment-nginx-5bcf797978-57sss   1/1     Running   0          30s   10.42.0.24   node-01   <none>           <none>
deployment-nginx-5bcf797978-5vqph   1/1     Running   0          31s   10.42.1.7    node-02   <none>           <none>
deployment-nginx-5bcf797978-97xlr   1/1     Running   0          31s   10.42.2.5    node-03   <none>           <none>
deployment-nginx-5bcf797978-996nb   1/1     Running   0          30s   10.42.2.9    node-03   <none>           <none>
deployment-nginx-5bcf797978-b9hps   1/1     Running   0          30s   10.42.0.23   node-01   <none>           <none>
deployment-nginx-5bcf797978-cnkjm   1/1     Running   0          31s   10.42.2.6    node-03   <none>           <none>
deployment-nginx-5bcf797978-cztt6   1/1     Running   0          47s   10.42.1.3    node-02   <none>           <none>
deployment-nginx-5bcf797978-glk59   1/1     Running   0          47s   10.42.2.3    node-03   <none>           <none>
deployment-nginx-5bcf797978-gpcgw   1/1     Running   0          30s   10.42.0.21   node-01   <none>           <none>
deployment-nginx-5bcf797978-gv22t   1/1     Running   0          30s   10.42.1.8    node-02   <none>           <none>
deployment-nginx-5bcf797978-kdx9b   1/1     Running   0          30s   10.42.1.10   node-02   <none>           <none>
deployment-nginx-5bcf797978-kr6cs   1/1     Running   0          31s   10.42.1.6    node-02   <none>           <none>
deployment-nginx-5bcf797978-l9wss   1/1     Running   0          31s   10.42.1.5    node-02   <none>           <none>
deployment-nginx-5bcf797978-lbvpl   1/1     Running   0          30s   10.42.2.8    node-03   <none>           <none>
deployment-nginx-5bcf797978-q4ppb   1/1     Running   0          30s   10.42.2.10   node-03   <none>           <none>
deployment-nginx-5bcf797978-r2r9x   1/1     Running   0          30s   10.42.1.4    node-02   <none>           <none>
deployment-nginx-5bcf797978-v2cg2   1/1     Running   0          30s   10.42.0.22   node-01   <none>           <none>
deployment-nginx-5bcf797978-wbjrd   1/1     Running   0          31s   10.42.2.7    node-03   <none>           <none>
deployment-nginx-5bcf797978-xx7rf   1/1     Running   0          31s   10.42.1.9    node-02   <none>           <none>
```  


### 노드 삭제하기
노드 삭제는 `k3s server`, `k3s agent` 모두 동일하다. 
`node-03` 을 삭제한다고 가정하고 예제를 진행한다. 
현재 클러스터에는 `test-nginx.yaml` 템플릿 구성의 `Pod` 이 총 20개 실행 중인 상태이고, `node-03` 에서 몇개의 `Pod` 이 실행 중이다.  

```bash
node-01# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
deployment-nginx-5bcf797978-2l574   1/1     Running   0          47s   10.42.2.4    node-03   <none>           <none>
deployment-nginx-5bcf797978-57sss   1/1     Running   0          30s   10.42.0.24   node-01   <none>           <none>
deployment-nginx-5bcf797978-5vqph   1/1     Running   0          31s   10.42.1.7    node-02   <none>           <none>
deployment-nginx-5bcf797978-97xlr   1/1     Running   0          31s   10.42.2.5    node-03   <none>           <none>
deployment-nginx-5bcf797978-996nb   1/1     Running   0          30s   10.42.2.9    node-03   <none>           <none>
deployment-nginx-5bcf797978-b9hps   1/1     Running   0          30s   10.42.0.23   node-01   <none>           <none>
deployment-nginx-5bcf797978-cnkjm   1/1     Running   0          31s   10.42.2.6    node-03   <none>           <none>
deployment-nginx-5bcf797978-cztt6   1/1     Running   0          47s   10.42.1.3    node-02   <none>           <none>
deployment-nginx-5bcf797978-glk59   1/1     Running   0          47s   10.42.2.3    node-03   <none>           <none>
deployment-nginx-5bcf797978-gpcgw   1/1     Running   0          30s   10.42.0.21   node-01   <none>           <none>
deployment-nginx-5bcf797978-gv22t   1/1     Running   0          30s   10.42.1.8    node-02   <none>           <none>
deployment-nginx-5bcf797978-kdx9b   1/1     Running   0          30s   10.42.1.10   node-02   <none>           <none>
deployment-nginx-5bcf797978-kr6cs   1/1     Running   0          31s   10.42.1.6    node-02   <none>           <none>
deployment-nginx-5bcf797978-l9wss   1/1     Running   0          31s   10.42.1.5    node-02   <none>           <none>
deployment-nginx-5bcf797978-lbvpl   1/1     Running   0          30s   10.42.2.8    node-03   <none>           <none>
deployment-nginx-5bcf797978-q4ppb   1/1     Running   0          30s   10.42.2.10   node-03   <none>           <none>
deployment-nginx-5bcf797978-r2r9x   1/1     Running   0          30s   10.42.1.4    node-02   <none>           <none>
deployment-nginx-5bcf797978-v2cg2   1/1     Running   0          30s   10.42.0.22   node-01   <none>           <none>
deployment-nginx-5bcf797978-wbjrd   1/1     Running   0          31s   10.42.2.7    node-03   <none>           <none>
deployment-nginx-5bcf797978-xx7rf   1/1     Running   0          31s   10.42.1.9    node-02   <none>           <none>
```  

가장 먼저 삭제할 노드인 `node-03` 를 클러스터 스케쥴링에서 제외하고, 
실행 중인 `Pod` 들을 다른 클러스터의 노드로 옮기기 위해 `drain` 명령을 수행해 준다.  

```bash
node-01# kubectl drain node-03 --ignore-daemonsets --delete-emptydir-data
```  

`drain` 명령이 실행된 후 클러스터으 노드 정보를 조회하면 아래와 같이 `node-03` 에는 더이상 스케쥴링이 되지 않는 것을 확인 할 수 있다.  

```bash
node-01# kubectl get node -o wide
NAME      STATUS                     ROLES                       AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready                      control-plane,etcd,master   26m   v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready                      control-plane,etcd,master   13m   v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-03   Ready,SchedulingDisabled   control-plane,etcd,master   13m   v1.21.4+k3s1   192.168.0.103   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
```  

그리고 `Pod` 을 조회하면 `node-03` 에서 실행 중인 `Pod` 들이 모두 다른 노드로 옮겨진 것을 확인 할 수 있다.  

```bash
node-01# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE      NOMINATED NODE   READINESS GATES
deployment-nginx-5bcf797978-4vr5x   1/1     Running   0          84s     10.42.1.12   node-02   <none>           <none>
deployment-nginx-5bcf797978-57sss   1/1     Running   0          7m37s   10.42.0.24   node-01   <none>           <none>
deployment-nginx-5bcf797978-5vqph   1/1     Running   0          7m38s   10.42.1.7    node-02   <none>           <none>
deployment-nginx-5bcf797978-5wn4p   1/1     Running   0          84s     10.42.0.28   node-01   <none>           <none>
deployment-nginx-5bcf797978-7qf6r   1/1     Running   0          84s     10.42.1.11   node-02   <none>           <none>
deployment-nginx-5bcf797978-b9hps   1/1     Running   0          7m37s   10.42.0.23   node-01   <none>           <none>
deployment-nginx-5bcf797978-cztt6   1/1     Running   0          7m54s   10.42.1.3    node-02   <none>           <none>
deployment-nginx-5bcf797978-gpcgw   1/1     Running   0          7m37s   10.42.0.21   node-01   <none>           <none>
deployment-nginx-5bcf797978-gv22t   1/1     Running   0          7m37s   10.42.1.8    node-02   <none>           <none>
deployment-nginx-5bcf797978-kdx9b   1/1     Running   0          7m37s   10.42.1.10   node-02   <none>           <none>
deployment-nginx-5bcf797978-kr6cs   1/1     Running   0          7m38s   10.42.1.6    node-02   <none>           <none>
deployment-nginx-5bcf797978-l9wss   1/1     Running   0          7m38s   10.42.1.5    node-02   <none>           <none>
deployment-nginx-5bcf797978-nd6wc   1/1     Running   0          84s     10.42.0.25   node-01   <none>           <none>
deployment-nginx-5bcf797978-qgcr7   1/1     Running   0          84s     10.42.0.26   node-01   <none>           <none>
deployment-nginx-5bcf797978-r2494   1/1     Running   0          84s     10.42.1.13   node-02   <none>           <none>
deployment-nginx-5bcf797978-r2r9x   1/1     Running   0          7m37s   10.42.1.4    node-02   <none>           <none>
deployment-nginx-5bcf797978-stqgx   1/1     Running   0          84s     10.42.1.14   node-02   <none>           <none>
deployment-nginx-5bcf797978-v2cg2   1/1     Running   0          7m37s   10.42.0.22   node-01   <none>           <none>
deployment-nginx-5bcf797978-vz4ng   1/1     Running   0          84s     10.42.0.27   node-01   <none>           <none>
deployment-nginx-5bcf797978-xx7rf   1/1     Running   0          7m38s   10.42.1.9    node-02   <none>           <none>
```  

마지막으로 `node-03` 를 클러스터에서 삭제해주는 것으로 노드 삭제를 완료 할 수 있다.  

```bash
node-01# kubectl delete node node-03
node "node-03" deleted
node-01# kubectl get node -o wide
NAME      STATUS   ROLES                       AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node-01   Ready    control-plane,etcd,master   28m   v1.21.4+k3s1   192.168.0.101   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
node-02   Ready    control-plane,etcd,master   15m   v1.21.4+k3s1   192.168.0.102   <none>        Ubuntu 20.04.3 LTS   5.4.0-1042-raspi   docker://19.3.15
```  


---
## Reference
[K3S Document](https://rancher.com/docs/k3s/latest/en/)  








