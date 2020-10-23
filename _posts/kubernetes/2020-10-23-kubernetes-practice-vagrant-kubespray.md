--- 
layout: single
classes: wide
title: "[Kubernetes 실습] Vagrant 와 Kubespray 사용해서 Kubernetes 환경 구성하기"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: ''
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

```vagrantfile
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

```shell script
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

```shell script
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

```shell script
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






 

















































---
## Reference
