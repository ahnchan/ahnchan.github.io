---
layout: post
title:  "Install microk8s on RedHat Enterprise 8"
date:   2023-10-18 16:00:00 +0900
categories: Platform
---
created : 2023-10-18, updated : 2023-10-18


# Introduction
다양한 Application을 구동하고 관리하는 부분은 쉽지가 않다. 라이브러간의 의존성도 문제이고 OS와 호환 문제도 있다. 그래서 이를 해결하기 위해 Container 기술을 사용할 수 있다. 하지만 Container만을 사용하여도 외/내부 Network, DNS등 고민할 것이 많다. 이런 상황을 해결하는 하나로 Kubernetes(k8)가 활용되고 있다. 하지만 K8s는 여러 서버를 클러스터로 묶어서 운영에 최적화 되어 있다. 소규모 서버나 저비용으로 구성을 위한 니즈도 나와서 이제는 한대의 서버나 소규모 서버를 관리할 수 있게 다양한 배포판(minkube, microk8s 등)이 나오고 있다. 이 튜토리얼에서는 Microk8s를 이용해서 구성을 해보도록 하겠다. 

# Microk8s
Server Cluster를 하나의 논리적인 형태로 관리가 가능하고 Node를 추가하여 확장을 할수 있다. 또한 Helm 을 이용하여 Application 을 한꺼번에 설치, 설정이 가능하다. 
Kubernetes(k8s)는 대용량의 서버군을 관리하기에 적합하여 Microk8s는 Kubernetes의 환경을 상당히 유사하게 소규모 서버에서 사용할 수 있다고 판단하여 사용하기로 하였다. 

[Microk8s, k3s, minikube 비교자료 링크](https://microk8s.io/compare)

이 튜토리얼에서는 RedHat Enterprise 8에서 설치를 해보도록 하겠다. Microk8s는 Canonical(Ubuntu)에서 서비스를 하고 있어서 Ubuntu계열에서는 쉽게 설치가 가능하다. 하지만 다양한 환경에서 잘 설치되는지 확인해보기 위해 RedHat Enterise 8에서 작업을 해보았다. 

# Microk8s 설치
Microk8s 설치시 snaps를 사용하면 필요한 라이브러들의 설치도 같이 진행되기 때문에 편리하다. 향후 Upgrade나 update시에도 간단하게 진행을 할 수 있다. 

## snap 설치
EPEL(Extra Packages for Enterprise Linux) 저장소를 추가한다.

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf upgrade
```

Optional, Extra 저장소의 추가를 합니다.
```bash
sudo subscription-manager repos --enable "rhel-*-optional-rpms" --enable "rhel-*-extras-rpms"
sudo yum update
```

이제 snap을 설치합니다.
```bash
sudo yum install snapd
```

설치가 완료되면 시스템에 등록을 한다. 
```bash
sudo systemctl enable --now snapd.socket
```

snap을 Symbolic link로 연결하여 path없이 실행이 가능하게 한다.
```bash
sudo ln -s /var/lib/snapd/snap /snap
```

## Microk8s 설치
snap이 정상적으로 설치가 되었으면 snap을 이용하여 microk8s를 설치한다. snap으로는 간단하게 설치할 수 있다. 특정 버전으로 설치를 원하거나 하면 Refenece를 확인해보자. 여기서는 최신 버전으로 설치를 한다.

```bash
sudo snap install microk8s --classic
```

설치가 잘되고 구동이 되었는지 확인하기 위해서는 아래와 같이 입력하면 구동이 완료되면 microk8s의 정보를 보여준다. 현재 구동 상태와 add-on으로 enable된 리스트를 보여준다.
```bash
microk8s status --wait-ready
```

microk8s 명령을 실행하기 위해서는 microk8s의 Group에 해당 사용자를 추가해야한다.
```bash
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

## Add-on 설치
이제 기본 설정은 완료되었고 일반적으로 필요한 add-on을 enable하면 서비스를 설치를 해준다.

```bash
microk8s enable hostpath-storage
microk8s enable observability
microk8s enable ingress
microk8s enable metallb
```

> metallb 는 Loadbalancer이며 enable시 IP Range를 물어봄. IP가 여러개가 나이면 해당 서버에 할당된 IP를 입력하면된다. 예> 192.168.1.10-192.168.1.10

## Default Storage Class의 Path 변경하기 (Option)
기본으로 설치된 Storage의 디렉토리의 변경이 필요할 경우 사용한다. 

```bash
microk8s kubectl -n kube-system edit hostpath-provisioner
```

edit화면이 나타난다. 기본 yaml이 보여지고 여기서 mount되는 위치를 변경하면 된다. 변경이 완료되어 저장이 되면 deployment에 적용이 된다.


# Conclusions
RedHat Enterpise 8 에서도 동작은 잘 되었다. 일전에 RedHat에 Docker를 깔기 위해 라이브러리를 수정해서 다른 제품이 안돌아가서 문제가 있었는데, Minik8s를 설치하여 컨테이너 관리를 해결할 수 있었다. 


# References
[Install microk8s on Red Hat Enterprise Linux](https://snapcraft.io/install/microk8s/rhel)

[Tutorial Get started](https://microk8s.io/docs/getting-started)



