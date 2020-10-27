--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Vagrant 와 Kubespray 사용해서 Kubernetes 환경 구성하기"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: 'Vagrant 를 사용해서 가상환경을 만들고 그 위에 Kubespray 를 사용해서 Kubernetes 환경을 구성하자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Practice
  - Vagrant
  - Kubespray
toc: true
use_math: true
---  

## Kubespray
`Kubespray` 는 보안성과 고가용성이 있는 쿠버네티스 클러스터를 배포할 수 있는 오픈소스 프로젝트이다. 
`Kubespray` 는 앤서블(`Ansible`) 이라는 서버 환경 설정 자동화 도구를 사용해서 클러스터 대상이되는 노드들에 쿠버네티스 환경을 구성한다. 
설정에 맞춰 다양한 쿠버네티스 환경을 구성할 수 있기 때문에, 
온프레미스 환경에서 상용 서비스로 사용할 쿠버네티스  클러스터 구성할때 용의하다. 
그리고 `ingress-nginx`, `helm`, 볼륨 플러그인 등과 같은 추가 구성 요소를 클러스터에 실행할 수 있다.  

`Kubespray` 에서 사용하는 고가용성 구조는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/kubernetes/practice-vagrant-kubespray-1.png)


각 노드 `Nginx`(`nginx proxy`) 가 리버시 프록시 역할을 수행하며, 
마스터 노드의 `kube-apiserver` 를 바라보는 구조이다. 
각 노드에서 실행 중인 쿠버네티스 컴포넌트들은 마스터 노드와 직접 통신하는 것이 아니라, 
컴포넌트가 위치하는 노드의 `nginxproxy` 을 거쳐 마스터와 통신하게 된다. 
마스터 노드의 상태는 각 노드의 `nginx proxy` 가 헬스체크를 수행하며 판별 및 처리하게 된다.  

`Kubespray` 에서 지원하는 네트워크 플러그인은 [여기](https://github.com/kubernetes-sigs/kubespray#network-plugins)
에서 확인할 수 있다. 
위와 같은 네트워크 플러그인을 통해 간단하게 네트워크를 자동으로 구성할 수 있다. 
현재 실습에서는 기본 네트워크 플러그인인 `calico` 를 사용해서 구성한다.  

실습에서 사용할 `Kubespray` 버전은 `v2.11.0` 이고, 
해당 버전에서 설치되는 요소와 버전은 아래와 같다. 
- 클러스터 : `Kubernetes v1.16.3`, `etcd v3.3.10`, `Docker v18.09.7`
- 네트워크 플러그인 : `calico v3.7.3`
- 추가 애플리케이션 : `coredns v1.6.0`, `ingress-nginx v0.25.1`


## Kubespray 환경 구성
`Kubespray` 를 사용해서 `Kubernetes` 환경을 구성하기 위해 필요한 환경과 툴은 아래와 같다. 
- `Kubespray`(v2.11.0)
- `Vagrant`(2.2.0 이상)
- `Hyper-V` 혹은 `VirtualBox`

`Vagrant` 라는 가상 시스템 환경 관리도구를 사용해서, 
필요한 시스템 구성을 위한 스크립트를 작성해서 관리와 재사용성을 높인다. 
그리고 `Vagrant` 에서 사용하는 가상 시스템은 `Hyper-V` 를 사용한다. 
필요에 따라 `VirtualBox` 를 사용해서 동일한 가상 시스템을 구축할 수도 있다. 
`Kubespray` 를 사용해서 생성한 시스템에 `Kubernetes` 환경을 구축한다.  

생성할 시스템의 수는 총 5로(`node-1 ~ 5`) 구성하고, 
`CentOS 7` 을 사용한다. 
`Kubespray` 런타임은 하나의 노드에만 구축하고 명령어 실행으로 `Kubernetes` 환경을 구성한다. 
모든 스크립트 및 명령은 `Vagrant` 를 통해 작성 및 실행된다.  

구성에 필요한 파일은 아래와 같다. 

```bash
k8s-centos
├── Vagrantfile
├── common-setup.sh
├── inventory-setup.sh
├── inventory.ini
├── k8s-ansible-setup.sh
├── nginx-test.yaml
└── setup.sh
```  


### Vagrantfile
가상 시스템을 구성하는 `Vagrant` 의 스크립트인 `Vagrantfile` 내용은 아래와 같다. 

```
BOX_IMAGE = "centos/7"
Vagrant.require_version ">= 2.2.0"

# 사용할 Kubespray 버전
KUBESPRAY_VERSION = "v2.11.0"
# Kubespray 설치 상위 절대경로
KUBESPRAY_ROOT_PATH = "/home/vagrant"

# VM 구성에 필요한 정보를 작성하는 Map(VM이름Prefix => 정보)
VM_INFO = {
    "node" => {
        "count" => 5,
        "cpus" => 1,
        "memory" => 2048
    }
}

Vagrant.configure("2") do |config|
    # VM 이미지 설정
    config.vm.box = BOX_IMAGE
    # VM 네트워크 설정
    config.vm.network "public_network", bridge: "Default Switch"
    # 현재 경로 VM의 /vagrant 경로에 마운트
    config.vm.synced_folder ".", "/vagrant"

    # VM_INFO 에 작성된 키 순서대로 실행
    VM_INFO.each.with_index do |(vm_name, info), index|
        count = info['count']
        cpus = info['cpus']
        memory = info['memory']
        scripts = info['scripts']

        # VM 생성은 내림차순으로 한다. (..3,2,1)
        (1..count).reverse_each { |i|
            # 노드 이름
            name = "#{vm_name}-#{i}"

            # name 에 해당하는 VM 생성
            config.vm.define "#{name}" do |vm_name|
                vm_name.vm.hostname = "#{name}"

                # VM_INFO 에 작성한 VM 스펙 설정
                vm_name.vm.provider "hyperv" do |hv|
                    # Customize the amount of memory on the VM:
                    hv.cpus = cpus
                    hv.memory = memory
                end

                # Kubespray 환경을 구성하고 Kubernetes 를 구성하는 VM 설정
                if index == (VM_INFO.count - 1) && i == 1
                    vm_name.vm.provision "shell",
                        privileged: false, path: "setup.sh"

                    vm_name.vm.provision "shell",
                        privileged: false,
                        path: "k8s-ansible-setup.sh",
                        :args => ["#{KUBESPRAY_VERSION}", "#{KUBESPRAY_ROOT_PATH}"]

                    VM_INFO.each do |(vm_name2, info2)|
                        vm_name.vm.provision "shell",
                            privileged: false,
                            path: "inventory-setup.sh",
                            :args => ["#{vm_name2}", info2['count'], "#{KUBESPRAY_ROOT_PATH}/kubespray/inventory/mycluster/inventory.ini"]
                    end

                    # Kubernetes 환경 구성 명령어
                    vm_name.vm.provision "shell", privileged: false, inline: <<-SHELL
                       cd #{KUBESPRAY_ROOT_PATH}/kubespray
                       ansible-playbook -i inventory/mycluster/inventory.ini -v --become --become-user=root cluster.yml
                    SHELL
                end
            end
        }
    end

    # 모든 VM 에 필요한 공통 환경 구성 스크립트 실행
    config.vm.provision "shell", privileged: true, path: "common-setup.sh"
end
```  

`KUBESPRAY_VERSION` 로 사용할 `Kubespray` 의 버전을 사용할 수 있고, 
`KUBESPRAY_ROOT_PATH` 를 사용해서 `Kubepsray` 를 `git clone` 할 부모 경로를 설정할 수 있다.  

`VM_INFO` 은 `Map` 구조를 사용해서 키에 `VM` 의 이름으로 사용할 `Prefix` 를 설정해 주고, 
값에는 `count`(`VM` 수), `cpus`(코어 수), `memory`(메모리 용량) 을 설정하면 아래 스크립트에서 환경에 맞게 자동으로 구성한다. 
`master` 라는 이름을 `Prefix` 로 갖는 `VM` 을 추가하면 아래와 같다.  

```
VM_INFO = {
    "node" => {
        "count" => 3,
        "cpus" => 1,
        "memory" => 2048
    },
    "master" => {
        "count" => 3,
        "cpus" => 1,
        "memory" => 2028
    }
}
```  

`VM_INFO.each.with_index` 를 통해 `VM_INFO` 를 작성한 키 순으로 순회한다. (`node`, `master`)
그리고 `(1..count).reverse_each` 에서는 `VM_INFO` 의 `count` 값을 사용해서 내림차순으로 순회한다. (`count`, .., 2, 1)
`master` 이름이 추가된 `VM_INFO` 를 예시로 생성되는 `VM` 의 순서는 아래와 같다. 
1. node-3
1. node-2
1. node-1
1. master-3
1. master-2
1. master-1

그리고 스트립트에서는 `if index == (VM_INFO.count - 1) && i == 1` 조건으로, 
가장 마지막에 생성되는 `VM` 에 `Kubespray` 환경을 구성한다.  


### 스크립트 실행 순서
`Vagrantfile` 에서는 각 `VM` 을 실행하며 정해진 쉘 스크립트 혹은 명령을 실행하며 필요한 환경을 구성한다. 
먼저 일반적은 `VM` 를 구성할때는 `common-setup.sh` 쉘 스크립트 파일을 실행해 환경을 구성한다. 
그리고 `Kubespray` 환경이 구성되는 `VM` 에서 실행 순서는 아래와 같다.
1. `common-setup.sh`
1. `setup.sh`
1. `k8s-ansible-setup.sh`
1. `inventory-setup.sh`
1. `ansible-playbook` 명령어 

`common-setup.sh` 는 공통적인 설정으로 `root` 비밀번호 설정, `ssh` 설정, 필요 툴 설치 `kubectl` 명령어 등록 등을 수행한다. 

```shell
#!/bin/bash

echo "common setup"

echo -e "root\nroot" | passwd
sed  -i 's/#PermitRootLogin yes/PermitRootLogin yes/g' /etc/ssh/sshd_config;
sed  -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config;
systemctl restart sshd
yum install -y net-tools bind-utils sshpass
echo "alias kubectl='/usr/local/bin/kubectl'" >> ~/.bashrc
```  

`setup.sh` 는 `Kubespray` 가 구성되는 `VM` 에 필요한 기본적인 설정으로 `ssh` 키를 생성한다. 

```shell
#!/bin/bash

echo "setup"

echo -e "\n\n\n\n" | ssh-keygen -t rsa
```  

`k8s-ansible-setup.sh` 는 `Kubespray` 설치를 위한 구성을 수행하고, 
`Kubespray` 구성은 `git clone` 으로 수행한다. 
그리고 `Kubespray` 에서 `Kubernetes` 노드 구성과 관리를 위해 `Ansible` 을 사용한다. 
`Ansible` 에는 `inventory.ini` 라는 클러스터 노드에 대한 정보가 필수로 구성되야 하는데, 
미리 작성된 `inventory.ini` 의 템플릿 파일을 사용할 클러스터 경로에 복사한다. 
마지막으로 `requirements.txt` 에 작성된 `Kubespray` 의 필요 환경을 설치한다.  

`inventory-setup.sh` 는 `inventory.ini` 에 작성돼야 할 `VM` 노드의 아이피 정보를 기입하고, 
`Kubespray` 노드에서 다른 노드로 `ssh` 접근을 할 수 있도록 설정한다. 

```shell
#!/bin/bash

prefix=$1
count=$2
src=$3
target=$4

if [ -z ${target} ]
then
  target=$src
else
  cp -f ${src} ${target}
fi

function iplookup() {
  local domain=$1
  echo `nslookup ${domain} | sed -ne 's/Address: \([0-9.]*\).*/\1/p;'`
}

for ((i=1;i<=${count};i++))
do
  name="${prefix}-${i}"
  ssh-keyscan ${name} >> ~/.ssh/known_hosts
  sshpass -p "vagrant" ssh-copy-id vagrant@${name}
  ip=$(iplookup ${name})

  new_ini=`sed -i "s/${name}/${ip}/g" ${target}`
done
```  

`Kubespray` 에서 사용하는 `Ansible` 은 `inventory.ini` 파일에 클러스터를 구성하는 노드의 실제 아이피가 기입되야한다. 
하지만 현재 `Vagrant` 에서 `Hyper-V` 를 사용해서 `VM` 를 생성할 경우 `Static IP` 설정이 불가능하다. 
이러한 이유로 `inventory-setup.sh` 에서 노드에 대한 아이피 정보를 파싱하고, 
그 결과를 `inventory-setup.sh` 에서 노드 이름에 해당하는 부분에 덮어 쓰는 방식으로 내용을 완성한다.  


마지막으로 `ansible-playbook -i inventory/mycluster/inventory.ini -v --become --become-user=root cluster.yml` 
명령으로 실행하면서 `inventory.ini` 에 설정된 노드에 `Ansible` 을 사용해서 `Kubernetes` 환경을 구성한다.  


### 노드 구성 설정
지금 `Vagrantfile` 는 5개(`node-1 ~ 5`) `VM`을 실행한다. 
앞서 `Kubespray` 는 `Ansible` 이라는 소프트웨어 프로비저닝 툴을 사용한다고 했었다. 
`Vagrnatfile` 로 생성한 5개 `VM` 에 모두 `Kubernetes` 를 구성하기 위해서는 
`inventory.ini` 파일에 원하는 구성으로 작성해주는 작업이 필요하다. 
작성된 `inventory.ini` 파일 내용은 아래와 같다. 

```ini
[all]
node1 ansible_host=node-1 ip=node-1 etcd_member_name=etcd1
node2 ansible_host=node-2 ip=node-2 etcd_member_name=etcd2
node3 ansible_host=node-3 ip=node-3 etcd_member_name=etcd3
node4 ansible_host=node-4 ip=node-4
node5 ansible_host=node-5 ip=node-5

[kube-master]
node1
node2
node3

[etcd]
node1
node2
node3

[kube-node]
node4
node5

[calico-rr]

[k8s-cluster:children]
kube-master
kube-node
calico-rr
```  
`[all]` 에는 사용하는 노드의 정보를 작성해주는 부분이다. 
`node1~5` 까지 `ansible_host=<node ip> ip=<node ip> etcd_member_name=<etcd 이름>`
내용을 각 노드에 맞는 값으로 작성해야 한다. 
`<node ip>` 의 경우 `Hyper-V` 를 사용하면 동적으로 설정되기 때문에, 
파일에는 우선 노드의 이름으로 기입한다. (`node-1~5`)
그러면 앞서 설명한 `inventory-setup.sh` 스크립트가 해당하는 노드 이름의 아이피를 파싱하고, 
`inventory-setup.sh` 파일에 적힌 노드 이름부분을 아이피로 덮어쓰게 된다.  

`inventory-setup.sh` 의 결과로 노드의 이름이 아이피로 덮어써진 `inventory.ini` 의 일부 내용은 아래와 같다. 

```
[all]
node1 ansible_host=172.31.201.51 ip=172.31.201.51 etcd_member_name=etcd1
node2 ansible_host=172.31.202.95 ip=172.31.202.95 etcd_member_name=etcd2
node3 ansible_host=172.31.195.184 ip=172.31.195.184 etcd_member_name=etcd3
node4 ansible_host=172.31.200.144 ip=172.31.200.144
node5 ansible_host=172.31.202.83 ip=172.31.202.83
```  

그리고 `[kube-master]` 은 `Kubenetes` 의 `Master` 노드 , 
`[etcd]` 는 `Kubernetes` 에서 `etc` 가 설치되는 노드, 
`[kube-node]` 는 `Kubenetes` 의 `Worker` 에 해당하는 노드들을 `[all]` 에 작성한 노드의 이름으로 기입해 주면된다. 
작성된 정보로 보면 `node-1~3` 은 `kube-master` 에 해당하고, `node4~5` 는 일반 노드에 해당한다.  


### 환경 구성하기
`Vagrantfile` 및 사용하는 쉘 스크립트는 정해진 규칙에 맞춰 `VM` 을 생성하기 때문에, 
`Vagrantfile` 의 `VM_INFO` 에 맞춰 `inventory.ini` 파일의 내용을 수정해주면 
`Vagrant` 명령어를 통해 `Kubenetes` 클러스터를 구성할 수 있다.  

환경을 구성하는 명령어는 `Vagrantfile` 이 위치한 경로에서 `vagrant up` 또는 
`vagrant up --provider hyperv` 로 가능하다. 

```bash
> vagrant up --provider hyperv
Bringing machine 'node-5' up with 'hyperv' provider...
Bringing machine 'node-4' up with 'hyperv' provider...
Bringing machine 'node-3' up with 'hyperv' provider...
Bringing machine 'node-2' up with 'hyperv' provider...
Bringing machine 'node-1' up with 'hyperv' provider...
==> node-5: Verifying Hyper-V is enabled...
==> node-5: Verifying Hyper-V is accessible...
==> node-5: Importing a Hyper-V instance
    node-5: Creating and registering the VM...

.. 생략 ..

    node-1: PLAY RECAP *********************************************************************
    node-1: localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    node-1: node1                      : ok=585  changed=132  unreachable=0    failed=0    skipped=1095 rescued=0    ignored=0
    node-1: node2                      : ok=503  changed=117  unreachable=0    failed=0    skipped=938  rescued=0    ignored=0
    node-1: node3                      : ok=505  changed=118  unreachable=0    failed=0    skipped=936  rescued=0    ignored=0
    node-1: node4                      : ok=354  changed=84   unreachable=0    failed=0    skipped=558  rescued=0    ignored=0
    node-1: node5                      : ok=354  changed=84   unreachable=0    failed=0    skipped=557  rescued=0    ignored=0
    node-1:
    node-1: Friday 23 October 2020  14:10:43 +0000 (0:00:00.545)       0:48:06.916 ********
    node-1: ===============================================================================
    node-1: download : download_container | Download image if required ------------ 203.17s
    node-1: download : download_container | Download image if required ------------ 191.08s
    node-1: download : download_container | Download image if required ------------ 158.10s
    node-1: container-engine/docker : ensure docker packages are installed -------- 134.83s
    node-1: download : download_container | Download image if required ------------- 95.80s
    node-1: download : download_container | Download image if required ------------- 95.75s
    node-1: kubernetes/master : Joining control plane node to the cluster. --------- 93.76s
    node-1: download : download_file | Download item ------------------------------- 93.53s
    node-1: download : download_container | Download image if required ------------- 92.49s
    node-1: kubernetes/master : kubeadm | Initialize first master ------------------ 91.98s
    node-1: download : download_container | Download image if required ------------- 90.67s
    node-1: download : download_container | Download image if required ------------- 89.29s
    node-1: download : download_container | Download image if required ------------- 86.15s
    node-1: download : download_container | Download image if required ------------- 81.50s
    node-1: download : download_container | Download image if required ------------- 80.26s
    node-1: download : download_container | Download image if required ------------- 77.84s
    node-1: download : download_file | Download item ------------------------------- 74.25s
    node-1: download : download_container | Download image if required ------------- 54.80s
    node-1: download : download_file | Download item ------------------------------- 53.30s
    node-1: etcd : Gen_certs | Write etcd master certs ----------------------------- 21.41s
```  

총 5개의 `VM` 생성과 `Kubernetes` 환경 구성을 수행하며 머신마다 다르겠지만, 
40분 ~ 60분 가량 소요된다.  

만약 실패한다면 `vagrant destroy -f ` 명령으로 생성한 `VM` 을 모두 강제로 지우고 다시 실행하는 방법으로 시도해볼 수 있다. 
그리고 `vagrant up --provider <VM이름>`, `vagrant destory -f <VM이름>` 등으로 특정 `VM` 만 실행하거나 삭제하는 방법도 있다. 
실행 후 삭제하지 않고 `VM` 을 중지하는 명령어는 `vagrant halt` 명령이다.  

모든 수행이 성공했으면 `vagrant ssh node-1` 명령으로 `node-1` 에 접속 한 후, 
`kubectl get node` 명령으로 구성된 `Kubernetes` 의 노드를 확인하면 아래와 같다. 

```bash
> vagrant ssh node-1
Last login: Mon Oct 23 14:20:43 2020 from 172.31.192.1
[vagrant@node-1 ~]$ sudo su
[root@node-1 vagrant]# kubectl get node
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   1h   v1.19.3
node2   Ready    master   1h   v1.19.3
node3   Ready    master   1h   v1.19.3
node4   Ready    <none>   1h   v1.19.3
node5   Ready    <none>   1h   v1.19.3
```  

### 테스트
`Vagrant` 와 `Kubespray` 로 구성한 `Kubernetes` 클러스터가 정상적으로 동작하는지, 
간단한 디플로이먼트와 서비스 템플릿으로 테스트를 진행해 본다. 
사용할 템플릿 내용은 아래와 같다. 

```yaml
# nginx-test.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nginx
  labels:
    app: deployment-nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: deployment-nginx
  template:
    metadata:
      name: nginx-pod
      labels:
        app: deployment-nginx
    spec:
      containers:
        - name: nginx-app
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

`deployment-nginx` 디플로이먼트는 `Nginx` 컨테이너를 실행하는 파드를 5개 관리한다. 
그리고 외부와 통신을 위해 `NodePort` 타입의 서비스인 `service-nodeport` 를 추가한다. 
`service-nodeport` 는 `deploynent-nginx` 의 파드에서 80 포트를 외부 포트인 32222 포트로 포워딩 하는 역할을 한다.  

`kubectl apply -f nginx-test.yaml` 명령으로 클러스터에 적용하고, 
`kubectl get svc,deploy,pod -0 wide` 명령으로 조회하면 아래와 같다. 

```bash
[root@node-1 vagrant]# kubectl get svc,deploy,pod -o wide
NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE     SELECTOR
service/kubernetes         ClusterIP   10.233.0.1      <none>        443/TCP          2d21h   <none>
service/service-nodeport   NodePort    10.233.30.124   <none>        8000:32222/TCP   66s     app=deployment-nginx

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
deployment.apps/deployment-nginx   5/5     5            5           66s   nginx-app    nginx:latest   app=deployment-nginx

NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE    NOMINATED NODE   READINESS GATES
pod/deployment-nginx-855648744-4b9lf   1/1     Running   0          66s   10.233.70.15    node5   <none>           <none>
pod/deployment-nginx-855648744-82b78   1/1     Running   0          66s   10.233.70.14    node5   <none>           <none>
pod/deployment-nginx-855648744-c95r9   1/1     Running   0          66s   10.233.70.13    node5   <none>           <none>
pod/deployment-nginx-855648744-t7gwc   1/1     Running   0          66s   10.233.105.17   node4   <none>           <none>
pod/deployment-nginx-855648744-xptlx   1/1     Running   0          66s   10.233.105.16   node4   <none>           <none>
```  

호스트에서 요청을 보내기위해 `kubectl get node -o wide` 명령으로 아이피 및 상세 정보까지 출력한다. 

```bash
[root@node-1 vagrant]# kubectl get node -o wide
NAME    STATUS   ROLES    AGE  VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
node1   Ready    master   1h   v1.19.3   172.31.201.51    <none>        CentOS Linux 7 (Core)   3.10.0-1127.19.1.el7.x86_64   docker://19.3.13
node2   Ready    master   1h   v1.19.3   172.31.202.95    <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64        docker://19.3.13
node3   Ready    master   1h   v1.19.3   172.31.195.184   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64        docker://19.3.13
node4   Ready    <none>   1h   v1.19.3   172.31.200.144   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64        docker://19.3.13
node5   Ready    <none>   1h   v1.19.3   172.31.202.83    <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64        docker://19.3.13
```  

호스트의 브라우저 혹은 `cli` 에서 `curl` 명령을 사용해서 클러스터 중 아무 아이피로 요청을 하면 `Nginx` 기본 페이지가 출력된다. 

```bash
> curl http://172.31.201.51:32222


StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html>
                    <html>
                    <head>
                    <title>Welcome to nginx!</title>
                        body {
                            width: 35em;
                            margin: 0 auto;

> curl http://172.31.202.83:32222


StatusCode        : 200
StatusDescription : OK
Content           : <!DOCTYPE html>
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
                    <...
```  


`kubectl logs -f deployment/<디플로이먼트이름>` 명령으로 디플로이먼트 중 하나의 파드의 로그를 모니터링 해놓고, 
계속해서 `htt://<클러스터 노드 아이피>:32222` 로 요청을 보내다 보면 모니터링 중인 파드가 요청을 처리하며 생성된 `Nginx` 로그를 확인해볼 수 있다. 

```bash
[root@node-1 vagrant]# kubectl logs -f deployment/deployment-nginx
Found 5 pods, using pod/deployment-nginx-855648744-xptlx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
10.233.70.0 - - [23/Oct/2020:14:46:00 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:00 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:00 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:01 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:01 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:01 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
10.233.70.0 - - [23/Oct/2020:14:46:01 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36" "-"
```  



---
## Reference
[kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)  
[A Beginner’s Guide and Tutorial to Ansible Playbooks, Variables, and Inventory Files](https://linuxhint.com/begineers_guide_tutorial_ansible/)
[HA endpoints for K8s](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md)
