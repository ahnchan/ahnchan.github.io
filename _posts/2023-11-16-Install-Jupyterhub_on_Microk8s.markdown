---
layout: post
title:  "Install JupyterHub on Microk8s"
date:   2023-11-16 16:00:00 +0900
categories: Platform
---
created : 2023-11-16, updated : 2023-11-16


# Introduction
소규모로 운영되는 데이터 분석 환경을 구성한다. 단일 사용자가 아닌 여러명이 자신의 Processor, Storage를 할당받아 Jupyter Notebook을 이용하여 분석을 한다. 

# 필요한 요소
여러명이 사용하기 위해서 필요한 요소를 정의하였다. 

- Microk8s
- JupyterHub
- Auth0

# Requirements
Microk8s의 설치는 다른 Tutorial인 ["Install Microk8s on Ubuntu(준비중)"]()를 확인하여 주기 바란다. 이 Tutorial은 Microk8s가 설치되어 있는 환경에서 JupyterHub를 설치하고 Storage를 변경하고 사용자 관리를 위해 Auth0를 연결하는 것을 설명하겠다. 

# JupyterHub 설치
Helm 을 이용하여 설치할 것이다. 아래의 스크립트를 실행하여 설치를 진행한다. sudo user에서 진행을 해야한다. 

## Helm 설치 확인
Microk8s설치시에 필요한 add-on을 enable 시키다 보면 heml이 같이 enable되기도 한다. 
먼저 아래와 같이 명령어러 microk8s에 helm이 설치되어 있는지 확인을 해보자.

```bash
microk8s status
```

현재 microk8s의 동작상태를 확인할 수 있다. 그리고 enable된 add-on도 같이 확인을 할 수 있다. 
enable에 helm이 있으면 설치가 되어 있는 것이다. 

아래의 명령으로 helm의 동작을 확인해보자
```bash
microk8s helm version
```

버전을 확인할 수 있을 것이다. Jupyterhub 문서에 보면 jupyterhub 3-1-0.dev 설치를 위해서는 helm의 버전은 3.5 이상이면 된다고 한다. 

```
version.BuildInfo{Version:"v3.9.1+unreleased", GitCommit:"7c5f0cafcf767eb766b8f99cd0aea88aaee454a2", GitTreeState:"clean", GoVersion:"go1.20.10"}
```

## Jupyterhub 설치
Helm 의 설정이 완료되면 먼저 비어 있는 config.yaml 파일을 하나 만든다. 아래의 주석을 포함하여 만들어보자. 이 파일은 이후에 JupyterHub의 설정을 추가할때 내용을 Update하여 사용할 예정이다. 

```
# This file can update the JupyterHub Helm chart's default configuration values.
#
# For reference see the configuration reference and default values, but make
# sure to refer to the Helm chart version of interest to you!
#
# Introduction to YAML:     https://www.youtube.com/watch?v=cdLNKUoMc6c
# Chart config reference:   https://zero-to-jupyterhub.readthedocs.io/en/stable/resources/reference.html
# Chart default values:     https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/HEAD/jupyterhub/values.yaml
# Available chart versions: https://hub.jupyter.org/helm-chart/
#
```

Helm 저장소에 추가하고 update한다. 
```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
```


이제 config.yaml을 만든 디렉토리에서 아래의 명령을 실행해보자. 

```
microk8s helm upgrade --cleanup-on-fail \
  --install jupyterhub jupyterhub/jupyterhub \
  --namespace jupyterhub \
  --create-namespace \
  --version=3.1.0 \
  --values config.yaml
```

namespace는 jupyterhub로 하고 version은 3.1.0으로 설정하였다. 
실행을 하면 진행되면서 

```
Release "jupyterhub" does not exist. Installing it now.
NAME: jupyterhub
LAST DEPLOYED: Tue Nov  7 08:50:05 2023
NAMESPACE: jupyterhub
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
.      __                          __                  __  __          __
      / / __  __  ____    __  __  / /_  ___    _____  / / / / __  __  / /_
 __  / / / / / / / __ \  / / / / / __/ / _ \  / ___/ / /_/ / / / / / / __ \
/ /_/ / / /_/ / / /_/ / / /_/ / / /_  /  __/ / /    / __  / / /_/ / / /_/ /
\____/  \__,_/ / .___/  \__, /  \__/  \___/ /_/    /_/ /_/  \__,_/ /_.___/
              /_/      /____/

       You have successfully installed the official JupyterHub Helm chart!

### Installation info

  - Kubernetes namespace: jupyterhub
  - Helm release name:    jupyterhub
  - Helm chart version:   3.1.0
  - JupyterHub version:   4.0.2
  - Hub pod packages:     See https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/3.1.0/images/hub/requirements.txt

### Followup links

  - Documentation:  https://z2jh.jupyter.org
  - Help forum:     https://discourse.jupyter.org
  - Social chat:    https://gitter.im/jupyterhub/jupyterhub
  - Issue tracking: https://github.com/jupyterhub/zero-to-jupyterhub-k8s/issues

### Post-installation checklist

  - Verify that created Pods enter a Running state:

      kubectl --namespace=jupyterhub get pod

    If a pod is stuck with a Pending or ContainerCreating status, diagnose with:

      kubectl --namespace=jupyterhub describe pod <name of pod>

    If a pod keeps restarting, diagnose with:

      kubectl --namespace=jupyterhub logs --previous <name of pod>

  - Verify an external IP is provided for the k8s Service proxy-public.

      kubectl --namespace=jupyterhub get service proxy-public

    If the external ip remains <pending>, diagnose with:

      kubectl --namespace=jupyterhub describe service proxy-public

  - Verify web based access:

    You have not configured a k8s Ingress resource so you need to access the k8s
    Service proxy-public directly.

    If your computer is outside the k8s cluster, you can port-forward traffic to
    the k8s Service proxy-public with kubectl to access it from your
    computer.

      kubectl --namespace=jupyterhub port-forward service/proxy-public 8080:http

    Try insecure HTTP access: http://localhost:8080
```

Pod의 구동을 확인해보자.
```
$ microk8s kubectl get pods --namespace jupyterhub
NAME                              READY   STATUS    RESTARTS   AGE
continuous-image-puller-n7mmg     1/1     Running   0          6m44s
user-scheduler-6d46b8dfc9-6jbsp   1/1     Running   0          6m44s
proxy-b4d5b4f68-gqs7c             1/1     Running   0          6m44s
user-scheduler-6d46b8dfc9-64mf9   1/1     Running   0          6m44s
hub-5cb8bc6d-d4966                1/1     Running   0          6m44s
```

먼저 간단하게 Jupyter의 접속을 확인하기 위해서는 아래와 같이 Port를 Forward하고 브라우저로 접속하여 확인할 수 있다. 
```bash
microk8s kubectl --namespace=jupyterhub port-forward service/proxy-public 8080:http
```

하지만 우리는 사용자를 인증하고 사용해야하기 때문에 몇가지 더 진행을 하도록 할 것이다. 
아래의 내용 일부는 Domain의 https 접속을 인증해야하기에 간단하게 단계만 설명을 하겠다. 
config.yaml에 정보를 추가하여 아래의 정보들을 입력한다. 
- Loadbalancer의 IP 설정
- TLS 인증서 설정
- domain 설정

config.yaml을 변경하면 아래의 명령으로 반영을 한다. 
```
microk8s helm upgrade --cleanup-on-fail \
  jupyterhub jupyterhub/jupyterhub \
  --namespace jupyterhub \
  --version=3.1.0 \
  --values config.yaml
```

## Mcirok8s에서 Storage 위치 바꾸기 (Optional)
microk8s가 설치되면 default storage 가 설정이 되고 Jupyterhub의 작업공간이 그곳에 할당된다. 일반적은 k8s에서는 외부의 Storage 영역을 사용하나 microk8s는 1대에서 구동이 가능하기에 일반적으로 Local의 Storage를 사용하게 될 것이다. 이럴때 Partition을 따로 구성해놨으면 default-storage의 path를 변경할 필요가 있다. 

```bash
microk8s kubectl -n kube-system edit deploy hostpath-provisioner
```

hostpath-provisioner의 deployment의 설정을 변경할 수 있다. 여기서 PV_DIR 에 설정된 value나 hostPath, mountPath를 원하는 위치로 변경하여 수정할 수 있다. 

# Auth0 연결하기
사용자 관리하는 방법은 여러개가 있다. 다양한 방법으로 사용이 가능하고 그것들을 공식 Reference에서도 설명을 해주고 있다. 
여기 Tutorial에서는 Auth0라는 서비스를 이용하여 Application을 등록하여 연결하고 Auth0에 등록된 사용자로 로그인이 가능하게 할 것이다.

Auth0를 설정하기 전에는 아래와 같은 화면으로 Signin이 되게 되어 있다. JupyterHub에 어떠한 설정도 하지 않았기 때문에 아무 ID나 넣고, Password는 비워 놓으면 Jupyter Notebook 이 생성된다.

![Defatult Signin](/posts/assets/platform/jupyterhub_microk8s/jupyterhub_singin_default.png){: width="200"}

이 Signin 부분을 Auth0와 연결하여 인증된 사용자만 접속이 가능하게 변경할 것이다. 

## Auth0의 Application 생성 & 인증방식 선택
먼저 Auth0에 가입을 하자. ("Auth0 사이트")[https://auth0.com] 에서 계정을 생성한다. 그러면 Free 로 사용을 할 수 있다. 제약사항은 사이트((링크))[https://auth0.com/pricing]를 확인해보자. 

> Note! 계정 생성시 Default 위치를 지정하는데, 가장 가까운 일본을 선택하자.

왼쪽 메뉴에서 Applications -> Applications을 선택하고 Create Application을 선택한다.
![Auth0 - Create Application](/posts/assets/platform/jupyterhub_microk8s/auth0_create_application.jpg){: width="500"}
- Name: Application 이름을 넣는다. (my-app 으로 선택하였다.)
- Choose an application type: Regular Web Applications 를 선택한다.

이제 설정하는 화면이 나타날 것이다. 여기서는 Application의 Credential 정보를 확인할 수 있다. domain, client id, client secret 을 JupyterHub에 설정을 해야하는 정보이니 잘 복사를 해두자.

설정에서 중요한 부분은 어떤 방식으로 인증을 할 것인가와 인증한 후에 Callback를 하는 URL을 설정하는 부분이다.

먼저 인증 방식은 Connections의 Database를 보면 Username-Password-Authentication이 선택되어 있을 것이다. 가장 기본적인 ID, 비밀번호 방식으로 승인을 하는 방식이다. 나머지 인증 방식은 Diable 시켜 놓자. 
![Application Connections](/posts/assets/platform/jupyterhub_microk8s/auth0_application_connectsions.png){: width="500"}

이 인증방식은 해당 계정의 User Management에 등록된 사용자에 대해서 ID/Password로 접근을 허용한다는 이야기이다. 

## Callback URL셋팅
Callback URL은 Auth0에서 인증을 한 후에 완료되면 다시 돌아오는 URL이다. 일반적으로 https까지 설정된 domain을 입력해야한다. 
하지만, 로컬에서 사용을 한다면 127.0.0.1 과 같은 ip로 설정을 해도 된다. 인증이 완료되면, Broswer에 Redirect를 요청하는 방식이기에 문제가 없다. 

> Note. 실제로 사용을 위해서는 TLS 설정까지 되어야 하니 가능한 https 설정을 해서 사용하는 것이 좋다. Let's encrpt등을 이용하면 TLS 인증서를 Free로 사용할 수 있으니 이를 권장한다. (물론 domain은 실제로 소유를 해야한다.)

![Application Callback URL](/posts/assets/platform/jupyterhub_microk8s/auth0_application_setting_callback_urls.png){: width="500"}

- Allowed Callback URLs: https://{MY-DOMAIN.COM}/hub/oauth_callback 와 같은 형식으로 입력한다. 

## Auth0 사용자 추가
사용자를 추가해보자. User Management에서 Users를 선택하고 Create User를 선택하여 사용자를 추가한다. 로그인 화면에서 Singup을 할 수도 있다. Singup이 가능한지는 Connection의 설정에서 enable/disable할 수 있다. 

![Create User](/posts/assets/platform/jupyterhub_microk8s/auth0_create_user.png){: width="200"}

## JupyterHub 와 연결하기
Auth0의 설정이 완료되었으면 JupyterHub의 설정을 추가하여 Auth0와 연결을 설정한다. 
config.yaml 파일에 아래의 스크립트를 추가한다. 

```
...
hub:
  config:
    Auth0OAuthenticator:
      client_id: {CLIENT_ID}
      client_secret: {CLIENT_SECRET}
      oauth_callback_url: https://{MY_DOMAIN}/hub/oauth_callback
      auth0_domain: {APPLICATION URL}
      allow_all: true
      scope:
        - openid
        - email
    JupyterHub:
      authenticator_class: auth0
```
- client_id, client_secret, auth0_domain: Auth0에서 Application을 생성하면 생기는 정보를 입력한다. Application의 Setting에서 확인할 수 있다. 
- oauth_callback_url: JupyterHub를 설정한 Callback url이다. Application의 Setting에 등록한 정보를 입력한다. 

정보를 추가하고 helm upgrade를 수행하여 정보를 주입한다. 


## 실제 접속해보기
이제 설정이 완료되었으면 다시 JupyterHub에 접속을 해보자. 그럼 처음과 다르게 Auth0 버튼이 나오고 Auth0에서 Signin 화면으로 변환이 된다. 

![Auth0 singin button](/posts/assets/platform/jupyterhub_microk8s/jupyterhub_singin_auth0_0.png){: width="200"}
![Auth0 Singin](/posts/assets/platform/jupyterhub_microk8s/jupyterhub_singin_auth0_1.png){: width="200"}


# Next Step
Custom 이미지로 Tensorflow, SKLearn 등이 미리 설치되어 있는 환경을 제공하자


# References
[Zero to JupyterHub](https://z2jh.jupyter.org/en/stable/jupyterhub/installation.html)

[Install JupyterHub](https://www.run.ai/guides/machine-learning-operations/jupyterhub#Configuring-User-Environments-in-JupyterHub)

[Jupyterhub 설정 변경하여 적용하기](https://z2jh.jupyter.org/en/stable/jupyterhub/customizing/extending-jupyterhub.html)

[](https://z2jh.jupyter.org/en/stable/jupyterhub/customizing/user-management.html)

[JupyterHub의 사용자를 Auth0로 인증하기](https://z2jh.jupyter.org/en/stable/administrator/authentication.html#auth0)