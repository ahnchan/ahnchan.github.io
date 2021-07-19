---
layout: post
title:  "Ansible을 이용하여 Ubuntu 20.04에 Kubernetes 구성하기"
date:   2021-07-11 16:00:00 +0900
categories: Platform
---

# Introduction
이 문서를 찾은 사람들은 아마 Kubernetes가 무엇인지는 알고, 구성을 해보고 싶어서 검색을 했을 것이라 생각된다. 그래서 Kubernetes를 왜 사용해야하고, 장단점이 무엇인지를 논의하지는 않겠다. 간단하게 로컬에 VirtualBox를 이용하여 Kubernetes Cluster를 구성하는데 초점을 맞추겠다. 

> Note: 이 튜토리얼은 Kubernetes Cluster를 Local에 환경을 구성해보고 싶을 경우 참조할 만한 목적으로 작성이 되었다. 명령문, 용어를 익히기 위한 것으로 생각하면 좋겠다. 실제 개발 환경은 Docker Desktop의 Kubernetes를 이용하면 더 간단하게 구성이 될 것이다. 

# Prerequisites
- Ubuntu 20.04을 VirtualBox에 설치하기
- Ansible 설치하기와 Playbook 실행하기

# Requirements
Ubuntu 20.04 VM들
- VirtualBox에서 Ubuntu 20.04를 설치한다.
- 설치시 사용한 구성은 2 GB Memory, 2 vCore, 20 GB Storage 이다.
- Network Adapter 1을 Bridged Network로 선택한다. NAT인 경우 Host 및 다른 VM에서 접속이 안된다.
- master, worker1, worker2 로 3개의 VM이 필요하다.
- Sudo 권한을 가진 계정이 필요하다.

Ansible이 설치된 환경 (Ansible control node)
- Ansible playbook을 실행하기 위해서 필요하다.
- 해당 작업을 실행하는 Host(VirtualBox 입장에서)면 된다. (OS는 어떤것이든 상관이 없으나 ssh 접속이 되어야 하니 linux, macos를 추천한다.)

# Step 1 - VM 환경 설정하기
Kubernetes Cluster는 요구사항에 맞춰서 설치를 진행한다. 

> Note : 네트워크 설정이 중요하다. 본 문서는 VitualBox의 VM Network Adapter 1을 Bridged Adapter 로 설정하고 고정 IP를 할당하였다. VM은 인터넷을 통해 패키지를 다운로드 받아야 하고, 다른 VM과 IP로 통신이 이루어져야 한다. 네트워크 설정은 공유기의 설정, DHCP 서버의 환경에 따라 다를 수 있으므로 자신의 네트워크 환경에 맞게 설정하여 진행을 한다.  

     
## IP Address 설정하기
Ubuntu 20.04는 netplan을 이용하여 IP 설정을 할 수 있다. VM마다 고정 IP를 설정하는 것이 좋다. 패키지들을 인터넷을 통해 다운로드 받아 설치를 하기 때문에 인터넷이 가능한 환경으로 구성해야 한다.
일반적으로 공유기를 설정한 환경이라면, 아래와 같이 구성을 할 수 있을 것이다. IP는 중복을 피해 할당을 하기 바란다.

- Master node static IP address : 192.168.0.71
- Worker1 node static IP address : 192.168.0.75
- Worker2 node static IP address :  192.168.0.76

## Host 이름변경하기
각각의 서버의 Hostname도 변경 한다.  서버들의 연결된 상태를 조회할때 사용하니 각각의 용도에 맞는 이름을 설정해주는 것이 좋다.

```
$ ssh user1@192.168.0.71
$ sudo hostnamectl set-hostname master
```

위와 같은 방법으로 worker1, worker2 의 hostname도 변경을 한다.


# Step 2. Ansible 환경 구성하기 
Ansible을 이용하여 Kubernetes Cluster를 구성할 것이다. Ansible control node에서 수행을 한다. 

> Note. Ansible 은 미리 설치가 되어 있어야 한다.


자신의 home에 kube-cluster 라는 디렉토리를 만들어서 host 파일, playbook 파일을 관리하자.

```
$ mkdir ~/kube-cluster
$ cd ~/kube-cluster
```

```
$ nano hosts
```

텍스트 편집기를 이용하여 hosts 파일을 만든다. hosts 파일은 Ansible이 관리하는 서버들(Inventory)의 목록이다. Step 1에서 정의한 IP 주소를 입력한다. 접속할 계정(ansible_user)는 일단 ubuntu 계정으로 설정한다. ubuntu 계정은 Step 3 에서 만들 것이다.


```
[masters]
master ansible_host=192.168.0.71

[workers]
worker1 ansible_host=192.168.0.75
worker2 ansible_host=192.168.0.76

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

* [masters], [worker] 는 각각 master node, worker node의 그룹명이다. Ansible에서 그룹으로 작업을 할때 사용된다.  


# Step 3. Ubuntu 계정을 생성하기
Kubernetes 를 관리할 계정으로 ubunt 계정을 생성한다. 이 계정은 비밀번호 없이 명령을 처리할 수 있는 sudoer로 등록한다. 

이 작업을 하기 전에 ssh key가 만들어서 설치될 서버에 있어야 한다. 간단하게 아래의 명령을 입력하여 접속 계정의 ssh key 를 만들수 있다. 

```
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/USER_NAME/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/USER_NAME/.ssh/id_rsa
Your public key has been saved in /home/USER_NAME/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:JrSP0WnLLb/uMnQJhybvEtXhqnxFePFL7Kcr12hTaWg USER_NAME@HOSTNAME
The key's randomart image is:
...
```


이 작업이 완료되면 home directory에 .ssh 에 id_rsa 파일이 생성되어 있을 것이다. 
설치할 서버의 관리 계정으로 ssh key를 복사해야 한다.

```
$ ssh-copy-id admin@192.168.0.71 
```

master, worker1, woker2 에 모두 실행한다.

이제 ubuntu 계정을 서버에 생성하기 위한 playbook을 만들어 보자

```
$ nano initial.yml
```

텍스트 편집기를 이용하여 아래의 내용을 복사하여 넣는다. 

```
- hosts: all
  become: yes
  tasks:
    - name: create the 'ubuntu' user
      user: name=ubuntu append=yes state=present createhome=yes shell=/bin/bash

    - name: allow 'ubuntu' to have passwordless sudo
      lineinfile:
        dest: /etc/sudoers
        line: 'ubuntu ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: set up authorized keys for the ubuntu user
      authorized_key: user=ubuntu key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
```

* hosts: all 모든 서버에서 실행한다.
* ubuntu 사용자를 생성한다. 
* ubuntu 사용자를 sudo 권한을 주면서, 비밀번호 입력없이 사용할 수 있게 한다.
* ssh접속을 위해 ansible control node에서 생성한 ssh key를 복사한다.

생성한 Playbook 을 실행을 한다.

```
$ ansible-playbook initial.yml -i hosts -u admin -K
```

Ansible을 이용하여 initial.yml 을 실행한다. 이 Playbook 은 hosts에 등록한 ansible_user인 ubuntu 계정을 생성하는 것이기에 추가 Parameter를 이용하여 관리계정(admin)을 이용한다.

* -u admin  관리계정(admin) 으로 접속을 한다. 
* -K  관리계정은 비밀번호 입력이 없는 계정이 아니기에(기본설정) 암호를 입력받는다.

ubuntu계정이 생성되면 이후부터는 ubuntu 게정으로 진행하기위해 hosts 파일을 수정한다.

```
[masters]
master ansible_host=192.168.0.71 ansible_user=ubuntu

[workers]
worker1 ansible_host=192.168.0.75 ansible_user=ubuntu
worker2 ansible_host=192.168.0.76 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

- ansible_user에 ubuntu 를 설정하면 앞으로 ansible-playbook을 실행할때  -u, -K를 붙이지 않아도 된다. 




# Step 4. Kubernetes Package 설치하기

이제 Kubernetes Cluster를 구성하기 위한 준비작업이 끝났다. Kubernetes Cluster 구성을 위한 패키지를 설치해보자. 
master와 worker1, worker2 모두 설치를 한다. 

```
$ nano kube-package.yml
```

텍스트 편집기를 이용하여 아래의 텍스트를 복사하여 넣는다.

```
- hosts: all
  become: yes
  tasks:
   - name: Update and upgrade apt packages
     apt:
       upgrade: yes
       update_cache: yes
       cache_valid_time: 86400 #One day

   - name: install Docker
     apt:
       name: docker.io
       state: present
       update_cache: true

   - name: Enable docker service
     systemd: enabled=yes name=docker

   - name: Start docker service
     systemd: name=docker state=started       

   - name: install APT Transport HTTPS
     apt:
       name: apt-transport-https
       state: present

   - name: add Kubernetes apt-key
     apt_key:
       url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
       state: present

   - name: add Kubernetes' APT repository
     apt_repository:
      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
      state: present
      filename: 'kubernetes'

   - name: install kubeadm
     apt:
       name: kubeadm
       state: present
       update_cache: true

   - name: install kubelet
     apt:
       name: kubelet
       state: present
       update_cache: true

   - name: install kubectl
     apt:
       name: kubelet
       state: present
       update_cache: true
```

* hosts: all 로 모든 서버에서 실행한다.
* apt 의 update, upgrade를 진행하여 최신상태로 만든다.
* Docker 를 설치한다.
* k8s 설치를 위한 apt-key, apt-repository 를 등록한다.
* kubeadm, kubelet, kubectl 을 설치한다. 


생성한 Playbook 을 실행을 한다.

```
$ ansible-playbook kube-pacakage.yml -i hosts
```

package설치는 ubuntu 계정으로 사용하기에 -u parameter는 사용하지 않는다.

# Step 5. Master Node 구성하기 (Kubernetes Cluster 생성하기)

maste구성을 위한 파일을 생성한다. 
```
$ nano master.yml
```

내용을 복사하여 넣는다.

```
- hosts: master
  become: yes
  tasks:
    - name: disable swap
      shell: "swapoff -a"

    - name: initialize the cluster
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

    - name: create .kube directory
      become_user: ubuntu
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copy admin.conf to user's kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ubuntu/.kube/config
        remote_src: yes
        owner: ubuntu

    - name: install Pod network
      become: yes
      become_user: ubuntu
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml >> pod_network_setup.txt
      args:
        chdir: $HOME
        creates: pod_network_setup.txt
```

* hosts: master 로 master 서버에서만 실행을 한다.
* swap을 disable 한다.
* Kubernetes cluster를 초기화 한다. 결과 파일은 /root/cluster_initialized.txt 에서 확인할 수 있다.
* ubuntu 계정에 .kube 디렉토리를 만들고, cluster 초기에서 생성된 정보를 복사한다.
* Pod network를 구성한다.


생성한 Playbook 을 실행을 한다.

```
$ ansible-playbook master.yml -i hosts
```


Master node에 Kubernetes Cluster를 초기화 하고, Pod Network 까지 구성하였다.

```
$ ssh ubuntu@192.168.0.71
```

master node에 ubuntu 계정으로 접속을 한다. 아래의 명령문으로 Kubernetes Service가 구성된 것을 확인할 수 있다. 

```
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   73m
```

```
$ kubectl get pods --all-namespace
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-558bd4d5db-cjmbp         1/1     Running   0          73m
kube-system   coredns-558bd4d5db-cmp2w         1/1     Running   0          73m
kube-system   etcd-ubuntu                      1/1     Running   0          73m
kube-system   kube-apiserver-ubuntu            1/1     Running   0          73m
kube-system   kube-controller-manager-ubuntu   1/1     Running   0          73m
kube-system   kube-flannel-ds-sr98g            1/1     Running   0          69m
kube-system   kube-proxy-5tnd4                 1/1     Running   0          73m
kube-system   kube-scheduler-ubuntu            1/1     Running   0          73m
```


# Step 6. Worker nodes 설정하기

이제 worker를 설정해보자

```
$ nano worker.yml
```

내용을 복사하여 넣는다.

```
- hosts: master
  become: yes
  gather_facts: false
  tasks:
    - name: get join command
      shell: kubeadm token create --print-join-command
      register: join_command_raw

    - name: set join command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[0] }}"


- hosts: workers
  become: yes
  tasks:
    - name: disable swap
      shell: "swapoff -a"
        
    - name: join cluster
      shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```


* master 에서 Cluster에 연결을 할 정보를 가져와 join_command_raw 에 저장을 한다.
* hosts: workers : workers 그룹의 서버들에서 실행을 한다.
* swap 을 disable 한다.
* cluster에 연결(join)을 한다.

생성한 Playbook 을 실행을 한다.

```
$ ansible-playbook worker.yml -i hosts 
```


Kubernetes Cluster의 구성을 완료하였다. 이제 잘 연결이 되었는지를 확인해보자.

```
$ ssh ubuntu@192.168.0.71
```

Master에 접속을 하여 명령을 실행해본다.

```
$ kubectl get nodes
NAME      STATUS   ROLES                  AGE     VERSION
master    Ready    control-plane,master   4d20h   v1.21.2
worker1   Ready    <none>                 4d19h   v1.21.2
worker2   Ready    <none>                 4d19h   v1.21.2
```
 
master, worker1, worker2가 연결되어 있는 것을 확인할 수 있다.



# Conclusions
Ansible을 이용하여 Kubernetes Cluster를 구성하였다.  Kubernetes Cluster 구성시 어떤 것들이 필요한지 알수 있었을 것이며, Ansible도 간단하게 사용해볼 수 있었다. 

Kubernetes Cluster를 구성만해서는 큰의미가 없다. Application을 운영해봐야한다. 다음 튜토리얼에서는 Kubernetes Documents의 Example을 구동해보도록 하겠다.



# References
[How to create a Kubernetes cluster with Kubeadm and Ansible on Ubuntu 20.04](https://www.arubacloud.com/tutorial/how-to-create-kubernetes-cluster-with-kubeadm-and-ansible-ubuntu-20-04.aspx)

[How to Install Kubernetes(K8s) and Docker on Ubuntu 20.04](https://www.letscloud.io/community/how-to-install-kubernetesk8s-and-docker-on-ubuntu-2004)

[How To Create a Kubernetes Cluster Using Kubeadm on Ubuntu 18.04]( https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04)

[Configuration Management 101: Writing Ansible Playbooks](https://www.digitalocean.com/community/tutorials/configuration-management-101-writing-ansible-playbooks)

[Install Ansible on Macos](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-macos)

[Stopping and starting a Kubernetes cluster and pods](https://www.ibm.com/docs/en/fci/1.0.2?topic=SSCKRH_1.0.2/platform/t_start_stop_kube_cluster.html)






