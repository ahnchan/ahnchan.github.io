---
layout: post
title:  "Install Apache Airflow on Microk8s"
date:   2023-12-01 16:00:00 +0900
categories: Platform
---
created : 2023-12-01, updated : 2023-12-01


# Introduction
데이터를 주기적으로 수집하고 원하는 위치에 전달하기 위한 Workflow가 필요하다. 단순히 데이터를 수집하는 것이 아니라 수집한 데이터를 가공하고 크리닝하고 필요한 형태로 변환을 해야 한다. 스케줄에 의해서 주기적으로 수집을 해야 할 수도 있지만, 외부 요청에 의해 진행이 되어야 할 수도 있다. 이런 다양한 방법의 지원이 필요하다. 

# Airflow는?
Python 기반의 Workflow를 관리하는 Open Source이다. Data Scientist들이 Data를 수집하여 각종 전 처리 및 저장을 해야 할 때 필요하다. Airflow는 설치와 설정이 복잡하기는 하지만Data Scientist가 친숙한 Python 기반 이기 때문에 많이 사용되고 있다. 직접 개발을 해야 하는 번거로움이 있기는 하지만, 직접 개발을 하는 것이라 더 유연하게 처리할 수 있기도 하다. 
이번에는 ETL과 같이 데이터를 수집하고 변환하고 저장하기 위한 Workflow를 수행하기 위해 Airflow를 사용해보았다. Extract하고 Transform에 초점을 맞춰서 주기적으로 Data를 특정 위치에 쌓는 것을 목표로 하였다. 

# Airflow 설치
서버에 직접 설치도 가능하다. 하지만, 해당 서버를 Airflow와 다른 소프트웨어도 같이 설치하여 사용해야 하기 때문에 관리를 편하게 하기 위해 Kubernetes(이하 k8s) 환경에 설치하도록 하겠다. 

## Helm Repository 추가
K8s에 설치하기 위해서 Helm Chart를 이용하다. 이를 위해서 Apache Airflow의 Repository를 추가한다.

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update
```

## Airflow 설치하기
설치를 진행한다. namespace를 airflow로 따로 지정한다. 없을 경우 생성하는 옵션을 넣어서 설치를 진행한다. 

```bash
helm upgrade --install airflow apache-airflow/airflow --namespace airflow --create-namespace
```

설치가 완료되면 아래와 같이 메세지가 나오면서 설치된다. 
```
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/healthon/.kube/config
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: /home/healthon/.kube/config
Release "airflow" has been upgraded. Happy Helming!
NAME: airflow
LAST DEPLOYED: Wed Nov  8 13:25:34 2023
NAMESPACE: airflow
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
Thank you for installing Apache Airflow 2.7.1!

Your release is named airflow.
You can now access your dashboard(s) by executing the following command(s) and visiting the corresponding port at localhost in your browser:

Airflow Webserver:     kubectl port-forward svc/airflow-webserver 8080:8080 --namespace airflow
Default Webserver (Airflow UI) Login credentials:
    username: admin
    password: admin
Default Postgres connection credentials:
    username: postgres
    password: postgres
    port: 5432

You can get Fernet Key value by running the following:

    echo Fernet Key: $(kubectl get secret --namespace airflow airflow-fernet-key -o jsonpath="{.data.fernet-key}" | base64 --decode)     

###########################################################
#  WARNING: You should set a static webserver secret key  #
###########################################################

You are using a dynamically generated webserver secret key, which can lead to
unnecessary restarts of your Airflow components.

Information on how to set a static webserver secret key can be found here:
https://airflow.apache.org/docs/helm-chart/stable/production-guide.html#webserver-secret-key
```

오류 문구가 없이 위와 같이 표시되면 설치가 잘된 것이다. 다음에는 Webserver와 접속을 해보겠다. 

## Airflow 브라우저에서 연결하기
먼저 Port-forward로 브라우저와 연결을 해보자. 이 테스트가 완료되면 언제나 연결할 수 있게 NodePort를 구성하여 연결할 것이다. 
이전 설치 완료 문구를 보면 ID와 Password가 나오는 이를 참조하여 접속하면 된다. 설정을 변경하여 Helm upgrade를 수행해도 매번 위와 같은 메시지가 나온다. 

Local의 8080포트를 airflow의 webserver로 접속을 하고 외부접근 IP는 모든 IP를 허용하는 명령어이다. 아래를 실행하면 터미널에서 계속 해당 작업이 돌아가도록 된다. 

```
microk8s kubectl port-forward svc/airflow-werbserver 8080:8080 --namespace airflow --address 0.0.0.0
```

> 운영 환경에서는 이방식을 사용하면 매번 upgrade등이 수행되면 해당 Bash를 실행해야 할 것이니 테스트에서만 사용하자

해당 서버의 IP와 Port를 8080으로 브라우저에서 접속을 해본다. 아래 화면은 다음 장에서 이야기할 Dag를 Github 와 연결을 한 상태이고 아마 Dag가 비어있는 상태로 보일 것이다. 
[Airflow 첫 화면](/posts/assets/platform/airflow_microk8s/airflow_gui.png){width="200"}

## Airflow 설정 추가하기
Install의 마지막 문구를 보면 Webserver secret을 설정하라고 나온다. 이를 위한 URL도 표시되어 있다. 안내에 나온 (Production Guide의 Webserver secret)[https://airflow.apache.org/docs/helm-chart/stable/production-guide.html#webserver-secret-key]를 확인하면 아래와 같다. 

먼저 Key를 생성한다. 
```
python3 -c 'import secrets; print(secrets.token_hex(16))'
```

위에서 나온 Key를 아래의 같에 넣으면된다. 이를 위해 value.yaml 이라는 파일을 하나 만들겠다. 위에서 만들어진 key를 <secret_key>로 대체하여 저장한다. 

```
webserverSecretKey: <secret_key>
```

> value.yaml 은 계속 추가되어야한다. 만약에 작업할때 이전의 내용을 넣지 않으면 기존의 default(보통 null)로 대체되기 때문에 매번 해당 정보는 유지되어야 한다. 

업그레이드를 위해서는 아래와 같이 명령을 입력한다. 물론 value.yaml 파일이 있는 위치에서 실행해야 한다. 

```
microk8s helm upgrade airflow apache-airflow/airflow --namespace airflow -f values.yaml

```

문제가 없이 실행되면 다시 Install이 끝났을 때의 안내문구와 같이 나온다. Webserver Secret을 설정하였으니 마지막 문구는 빠지고 나올 것이다. 

# Dag 소스를 연동하기
Dag를 연결하기 위해서는 여러가지 방법이 있다. Airflow의 home에서 dag 디렉토리를 persistence로 연결할 수도 있고, git-sync를 이용하여 Git repository를 연결할 수도 있다. 보통 Git에서 관리하는 것이 편해서 이번에는 git으로 연결해보겠다. Git의 정보가 변경되면 바로 반영이 되어 편리하다.

> Dag에 대한 개발 방법은 따로 이야기하도록 하겠다. 본 튜토리얼에는 Aiflow의 설정에 초점을 맞추었다.

## Public git 설정 (https방식)
git에 Dags 파일/스트립트가 있다는 가정하게 진행을 하겠다. 아래와 같이 설정을 value.yaml에 추가한다. 
git-sync를 enable 시키고 git의 repo 위치를 넣어준다. branch, subPath를 입력한다. 

```
...
dags:
  gitSync:
    enabled: true
    branch: <branch-name>
    repo: https://github.com/<username>/<public-repo-name>.git
    subPath: <dag-path>

```

value.yaml 에 설정된 정보로 helm upgrade를 수행한다. 
```
microk8s helm upgrade --install airflow apache-airflow/airflow -f values.yaml
```

## Dag 연동 확인하기
브라우저로 Airflow에 접속을 해보자 그러면 Dag에 추가된 dag 가 보일 것이다. 안보여도 실망하지 말아라 이 부분은 한번에 잘 되지 않았다. 각종 정보들이 잘 연동되어야 하기 때문에 Production guide를 꼭 참조해볼 필요가 있다. 

추가로 실제로 파일이 있는지 확인을 하는 방법이다. pod에 직접 접근을 해서 디렉토리에 파일이 있는지 확인 할 수 있다. 
command: kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```
microk8s kubectl exec -it airflow-worker-0 -n airflow -- /bin/bash
```

접속이 되면 /opt/airflow 에 있게 된다. 이것이 $AIRFLOW_HOME의 위치이다. dags 디렉토리를 들어가보면 repo와 Hashkey의 디렉토리가 보인다. repo는 hashkey로 된 디렉토리를 가르치고 있다. repo를 들어가보면 git repository가 통째로 연결되어 있는 것을 확인할 수 있다.

> 만약 Git이 push되서 정보가 변경되면 바로 Hashkey가 바뀌어 있는 것을 확인 할 수 있다.

## Private git 설정 (SSH)
실제로 운영환경에서 사용하기 위해서는 Public git 보다는 Private git을 사용해야 한다. 여기서는 Git의 Private 환경에서 연결이 가능하도록 설정을 해보겠다. 

### RSA Key생성 및 Github Deploy key 등록하기
Github의 Repository에 Public Key를 Deploy Key에 등록을 하고 Airflow에 Private key를 등록을 해야한다. 
먼저 ssh키를 생성해보자.
Repository에 권한이 있는 사용자의 이메일로 RSA를 만들어보자. 마지막에 -f 를 두어 파일 위치를 지정하자. 안그러면 ~/.ssh에 등록이 되어 다른 키와 중복이 될 수 있다.

```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ./id_rsa
```

모두 Default로 enter를 눌러서 넘어가자. 완료가 되면 아래와 같이 2개의 파일이 보일것이다. 
```
$ ls -al 
...
-rw------- 1 healthon healthon 3381 11월 21 09:02 id_rsa
-rw-r--r-- 1 healthon healthon  744 11월 21 09:02 id_rsa.pub
...
```
- id_rsa : Private Key 이다. Airflow에 base64 로 변환하여 등록을 해야 한다. 
- id_rsa.pub : Public Key 이다. Github에 등록을 해야 한다. 

Github의 해당 Repository의 Settings에서 Security 세션에 Deploy keys를 선택한다. Add Deploy 버튼을 선택하면 입력창이 나오는데, 여기에 id_rsa.pub의 내용을 복사해서 넣는다. 제목은 관리하기 편한 이름으로 설정한다.

### Airflow 설정 추가하기
이제 Airflow에 등록할 키를 변환해보자. 아래의 명령문으로 base64 변환을 하자. Key.txt에 private key가 변환되어 저장이 된다.
```
base64 ./id_rsa -w 0 > key.txt
```

위에서 변환한 key를 가지고 value.yaml에 정보를 변경하자. Repo의 정보도 SSH로 접속하는 정보로 변경을 하자.
```
...
dags:
  gitSync:
    enabled: true
    repo: git@github.com:<username>/<private-repo-name>.git
    branch: <branch-name>
    subPath: <dag-path>
    sshKeySecret: airflow-ssh-secret
extraSecrets:
  airflow-ssh-secret:
    data: |
      gitSshKey: '<base64-converted-ssh-private-key>'
```
- repo : SSH 접근 경로를 입력하자. public은 https 로 가능한데, Private은 ssh 로 접근하는 경로를 입력해야 한다. 
- branch : 해당 branch를 입력한다.
- subpath : repository에 dag 파일들이 있는 위치를 입력한다. (예: airflow/dags)
- sshKeySecret: Secret 정보의 이름이다. (아래 extraSecrets에 등록된 이름이다.)
- extraSecrets: k8s의 Secret에 저장을 한다. 
- GitSshKey : base64로 변환된 문자열을 '' 안에 넣어준다. 이 정보는 k8s의 secret에서 확인할 수 있다. 


### Github knownhost 등록
한가지가 더 남아있다. SSH 접속을 위해서는 Knowhost를 등록해야 한다. 이는 Production Guide에 나와있다. 
(Production Guide - Knownhost)[https://airflow.apache.org/docs/helm-chart/stable/production-guide.html#knownhosts]

Github의 Public key를 수집한다. 
```
ssh-keyscan -t rsa github.com > github_public_key
```

Public key를 가지고 Fingerprint를 출력하고 이정보를 (Github 공식 홈페이지)[https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints]에서 확인을 한다. 우리는 rsa 방식을 선택했으니 (RSA)로 되어 있는 부분이 동일한지 확인한다.

```
ssh-keygen -lf github_public_key
```

[Github의 ssh key fingerprint 정보](/posts/assets/platform/airflow_microk8s/github_ssh_fingerprints.png){width="200"}]


(RSA)의 SHA-256의 값이 동일하면 이제 설정에 추가하면 된다. value.yaml에 추가를 해보자. 
```
...
dags:
  gitSync:
    knownHosts: |
      github.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCj7ndNxQowgcQnjshcLrqPEiiphnt+VTTvDP6mHBL9j1aNUkY4Ue1gvwnGLVlOhGeYrnZaMgRK6+PKCUXaDbC7qtbW8gIkhL7aGCsOr/C56SJMy/BCZfxd1nWzAOxSDPgVsmerOBYfNqltV9/hWCqBywINIR+5dIg6JTJ72pcEpEjcYgXkE2YEFXV1JHnsKgbLWNlhScqb2UmyRkQyytRLtL+38TGxkxCflmO+5Z8CSSNY7GidjMIZ7Q4zMjA2n1nGrlTDkzwDCsw+wqFPGQA179cnfGWOWRVruj16z6XyvxvjJwbz0wQZ75XK5tKSb7FNyeIEs4TT4jk+S4dhPeAUC5y+bDYirYgM4GC7uEnztnZyaVWQ7B381AK4Qdrwt51ZqExKbQpTUNn+EjqoTwvqNj4kqx5QUCI0ThS/YkOxJCXmPUWZbhjpCg56i+2aB6CmK2JGhn57K5mj0MNdBXA4/WnwH6XoPWJzK5Nyu2zB3nAZp+S5hpQs+p1vN1/wsjk=
...      
```

value.yaml을 저장하고 Helm 을 upgrade 하자. 
```
microk8s helm upgrade airflow apache-airflow/airflow -f value.yaml
```

# Trouble shooting
설정을 변경하거나 추가하였을 경우 helm upgrade를 이용하는데 migration job이 구동이 된다. 그러면서 worker, scheduler, triggerer를 한 개 더 구동을 하고 정상 로딩되면 기존 pod는 삭제하는  방식을 사용하고 있다. 
여기서 2가지 문제가 발생할 수 있다. 

1) Mirtagion Job에서 문제가 발생
Config가 잘못되었을 경우 이런 경우가 발생한다. 
설정을 다시 넣고 돌리면 되는 경우가 있다. 문제가 발생한 Job/pod는 kubectl delete 로 지운다. 

2) Worker, Scheduler, Trigger 에서 오류가 발생하는 경우
기본 설정은 문제가 없는데, 실제 구동할 때 오류가 발생한다. git에 접속을 한다던가 할 때이다. 

# Conclusions
설치는 의외로 간단하였으나 Dag를 구성하고 필요한 설정을 하는 부분은 시간이 걸렸다. Git의 연결 시 SSH, HTTPS와의 차이에 의해 설정 값이 달라지는 부분과 key를 넣을 때의 Space 문제로 시간이 좀 더 소요되었다.
Airflow가 다양한 환경에서 Workflow를 구성할 수 있도록 하였지만, 결국에는 Python으로 Workflow를 구현을 해야 하기 때문에 설치/설정이 실제 개발의 시작이 된다고 볼 수 있다. 

# References
(Helm Chart for Apache Airflow)[https://airflow.apache.org/docs/helm-chart/stable/index.html]
