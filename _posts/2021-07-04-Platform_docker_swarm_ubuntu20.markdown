---
layout: post
title:  "Docker Swarm 구성하기(Ubuntu 20.04, Ansible)"
date:   2021-07-04 16:00:00 +0900
categories: Platform
---

# Introduction
Docker Swarm 을 구성해서 docker를 여러 worker node에서 서비스할수 있는 구조를 구성해본다. 
검색을 해보면 많은 구성 방법이 있으나 이 문서는 Docker 공식 Doc의 방식을 Ansible 로 구성해 봤다. 

이 문서는 VirtualBox를 이용하여 Local에 Docker Swarm을 구성해봅니다.

# Prerequisites
- Ubuntu 20.04을 VirtualBox에 설치하기
- Ansible 설치하기와 Playbook 실행하기

# Requirements
Ubuntu 20.04 VM (VirtualBox)
- Manager node 1개 와 Worker node 2개
- 메모리, 스토리지는 큰 제약은 없음. 실제 서비스를 구성하는 크기에 맞게 구성하면 될 것 같음.
- Network는 Static IP 가져야 하기에 Network adapter 1이나 2를 bridged network로 설정한다.

Ansible이 설치된 환경 (Ansible control node)
- Ansible playbook을 실행하기 위해서 필요하다.


# VM 환경 구성하기
## IP Address 설정하기
Ubuntu 20.04는 netplan을 이용하여 IP 설정을 할 수 있다. VM마다 고정 IP를 설정하는 것이 좋다. 패키지들을 인터넷을 통해 다운로드 받아 설치를 하기 때문에 인터넷이 가능한 환경으로 구성해야 한다.
일반적으로 공유기를 설정한 환경이라면, 아래와 같이 구성을 할 수 있을 것이다. IP는 중복을 피해 할당을 하기 바란다.

- Manager1 node static IP address : 192.168.0.71
- Worker1 node static IP address : 192.168.0.75
- Worker2 node static IP address :  192.168.0.76

## Host 이름변경하기
각각의 서버의 Hostname도 변경 한다.  서버들의 연결된 상태를 조회할때 사용하니 각각의 용도에 맞는 이름을 설정해주는 것이 좋다.

```
$ ssh admin@192.168.0.71
$ sudo hostnamectl set-hostname manager1
```

위와 같은 방법으로 worker1, worker2 의 hostname도 변경을 한다.


# Ansible 환경 구성하기 
Ansible을 이용하여 Manager node와 Worker node를 구성할 것이다. Ansible control node에서 수행을 한다. 

> Note. Ansible 은 미리 설치가 되어 있어야 한다.


자신의 home에 kube-cluster 라는 디렉토리를 만들어서 host 파일, playbook 파일을 관리하자.

```
$ mkdir ~/docker-swarm
$ cd ~/docker-swarm
```


```
$ nano hosts
```

텍스트 편집기를 이용하여 hosts 파일을 만든다. hosts 파일은 Ansible이 관리하는 서버들(Inventory)의 목록이다. Step 1에서 정의한 IP 주소를 입력한다.  

```
[managers]
manager1 ansible_host=192.168.0.71

[workers]
worker1 ansible_host=192.168.0.75
worker2 ansible_host=192.168.0.76

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```


[masters], [worker] 는 각각 master node, worker node의 그룹명이다. Ansible에서 그룹으로 작업을 할때 사용된다.  


# Ubuntu 계정을 생성하기
관리할 계정으로 ubuntu 계정을 생성한다. 이 계정은 비밀번호 없이 명령을 처리할 수 있는 sudoer로 등록한다. 

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

manager1, worker1, woker2 에 모두 실행한다.

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
[managers]
manager1 ansible_host=192.168.0.71 ansible_user=ubuntu

[workers]
worker1 ansible_host=192.168.0.75 ansible_user=ubuntu
worker2 ansible_host=192.168.0.76 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

ansible_user에 ubuntu 를 설정하면 앞으로 ansible-playbook을 실행할때  -u, -K를 붙이지 않아도 된다. 

# Docker 설치
Docker 공식문서의 Ubuntu 설치 순서를 그대로 진행한다. apt에서 docker를 진행해도 될 것 같지만, 일단 공식문서를 따라 진행을 해본다.

```
$ nano manager.yml
```

```
- hosts: all
  become: yes
  tasks:
  tasks:
    - name: Install aptitude using apt
      apt: name=aptitude state=latest update_cache=yes force_apt_get=yes

    - name: Install required system packages
      apt: name={{ item }} state=latest update_cache=yes
      loop: ['apt-transport-https', 'ca-certificates', 'curl', 'gnupg', 'lsb-release']

    - name: Add Docker's official GPG key 
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        keyring: /usr/share/keyrings/docker-archive-keyring.gpg
        state: present

    - name: Add specified repository into sources list
      apt_repository:
        repo: deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable
        state: present
        filename: docker

    - name: Install Docker Engine
      apt: name={{ item }} state=latest update_cache=yes
      loop: [ 'docker-ce', 'docker-ce-cli', 'containerd.io']

    - name: Enable docker service
      systemd: enabled=yes name=docker

    - name: Start docker service
      systemd: name=docker state=started       
```


- Docker 설치를 위해 필요한  기본 패키지를 설치한다. (apt-transport-https, ca-certificates, curl, gnupg, lsb-release)
- Apt key와 Repository를 추가한다.
- Docker 를 설치한다. (docker-ce, docker-ce-cli, containerd.io)

manager.yml playbook을 실행한다. 

```
$ ansible-playbook manager.yml -i hosts
```

Docker가 설치가 잘되었는지 확인을 하기 위해 Manager1 에 접속한다.

```
$ ssh ubuntu@192.168.071
```

docker가 설치가 되었는지를 확인한다.
```
$ sudo docker ps 
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```



> Note. docker  명령문을 sudo로 사용하지 않으려면 링크를 참조 하여 설정을 추가 한다. [링크](https://docs.docker.com/engine/install/linux-postinstall/)

# Manager Node 설정하기
이제 manager1에 swarm 을 생성한다. 

```
$ nano manager.yml
```

```
- hosts: manager1
  become: yes
  tasks:
    - name: Create a new swarm
      shell:  docker swarm init --advertise-addr 192.168.0.71 >> swarm_initialized.txt
      args:
        chdir: $HOME
        creates: swarm_initialized.txt
```

manager1 에서 swarm 을 생성한다.

manager1.yml을 실행한다.
```
$ ansible-playbook manager.yml -i hosts
```

manager1에 접속하여 설정이 된것을 확인한다.
```
$ ssh ubuntu@192.168.0.71
```

```
$ sudo docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
pqtmqmkgcir67743dz5wxq6qv *   manager1   Ready     Active         Leader           20.10.7
```



# Worker Node 설정하기
worker1, worker2를 설정해보자. 

```
$ nano worker.yml
```

```
- hosts: manager1
  become: yes
  gather_facts: false
  tasks:
    - name: get join command
      shell: docker swarm join-token worker
      register: join_command_raw

    - name: set join command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[2] }}"


- hosts: workers
  become: yes
  tasks:
    - name: join cluster
      shell: "{{ hostvars['manager1'].join_command }} >> worker_joined.txt"
      args:
        chdir: $HOME
        creates: worker_joined.txt
```

Manager1에서 swarm 에 join 할 정보를 가져온다.
위에서 가져온 정보를 가지고 manager에 Join을 한다. 


worker.yml을 실행한다.
```
$ ansible-playbook worker.yml -i hosts
```

# 연결 확인하기
manager1에 접속하여 worker1, worker2가 연결된 것을 확인한다.

```
$ ssh ubuntu@192.168.0.71
```

```
$ sudo docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
pqtmqmkgcir67743dz5wxq6qv *   manager1   Ready     Active         Leader           20.10.7
z2leq18cjnhahz282yhkkq5a7     worker1    Ready     Active                          20.10.7
5ohojxn35j54upjbiinekef8s     worker2    Ready     Active                          20.10.7
```


# References
[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

[Getting started with swarm mode](https://docs.docker.com/engine/swarm/swarm-tutorial/)

[Create a swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)

[Add nodes to the swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/)

[Configuration Management 101: Writing Ansible Playbooks](https://www.digitalocean.com/community/tutorials/configuration-management-101-writing-ansible-playbooks)

[Install Ansible on Macos](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-macos)





