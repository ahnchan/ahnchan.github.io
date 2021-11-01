---
layout: post
title:  "Dapr Installation"
date:   2021-10-29 16:00:00 +0900
categories: CloudNative
---
created : 2021-10-29, updated : 2021-10-29

# Introduction
Cloud Native Application을 구현하기 위해 서비스간의 통신, 분산된 서비스의 관리는 중요하다. 이를 위해 프로그램을 하려면 고민을 많이 해야한다. Dapr는 이런 고민은 Sidecar Pattern을 이용하여 손쉽게 개발을 할 수 있게 하였다.
본 튜토리얼에서는 Dapr를 위한 환경을 구성하고, 설치 및 간단한 테스트를 해보겠다. 

> Note. Dapr의 공식문서를 가지고 직접 진행하면서 정리한 문서이다.


# Pre-Installation
Docker: (https://www.docker.com/products/docker-desktop)[https://www.docker.com/products/docker-desktop] 

> Note. Windows 환경에서 설치를 해보겠다. 각각의 설치되는 부분은 간단하게 링크로 대체하였다. 이번 튜토리얼은 Dapr를 설치하고 Dapr만을 이용하여 State management 기능을 사용해 볼 것이다. 


# Sidecar Pattern
서비스A에서 다른 서비스B를 호출할때 직접 호출하는 것이 아니라 Sidecar 를 통해서 다른 서비스를 호출을 한다. 이런 Architecture는 SIdecar부분에서 여러가지 추가를 할수 있어 확장을 하거나 정책을 추가할때 손쉽게 할 수 있다. 

![Diagram](/posts/assets/cloudnative-dapr/images/dapr-concept-diagram.png){: width="500"} 

Dapr는 이런 Sidecar 기능에 State, Pub/Sub 을 연결을 기본 제공하여 따로 서비스를 구성하지 않고 Sidecar에서 State 저장/읽기, Pub/Sub을 연결할 수 있다. 

자세한 내용은 Dapr Concept을 보면 알수 있을 것이다. (Dapr Concept)[https://docs.dapr.io/concepts/overview/]


# Dapr Installation 
Command Prompt 에서 아래 명령을 이용하여 설치를 할 수 있다. 권한 문제가 있을 경우는 Command Prompt를 관리자 권한으로 실행을 해야한다.

```
> powershell -Command "iwr -useb https://raw.githubusercontent.com/dapr/cli/master/install/install.ps1 | iex"
```

설치가 왼료되면 c:\dapr이 생성이 되었고, Path에도 추가되었을 것이다. 다른 Powershell이나 Command Prompt를 열어서 설치를 확인할 수 있을 것이다. 

```
> dapr 

         __
    ____/ /___ _____  _____
   / __  / __ '/ __ \/ ___/
  / /_/ / /_/ / /_/ / /
  \__,_/\__,_/ .___/_/
              /_/

===============================
Distributed Application Runtime

Usage:
  dapr [command]

Available Commands:
  completion     Generates shell completion scripts
  components     List all Dapr components. Supported platforms: Kubernetes
  configurations List all Dapr configurations. Supported platforms: Kubernetes
  dashboard      Start Dapr dashboard. Supported platforms: Kubernetes and self-hosted
  help           Help about any command
  init           Install Dapr on supported hosting platforms. Supported platforms: Kubernetes and self-hosted
  invoke         Invoke a method on a given Dapr application. Supported platforms: Self-hosted
  list           List all Dapr instances. Supported platforms: Kubernetes and self-hosted
  logs           Get Dapr sidecar logs for an application. Supported platforms: Kubernetes
  mtls           Check if mTLS is enabled. Supported platforms: Kubernetes
  publish        Publish a pub-sub event. Supported platforms: Self-hosted
  run            Run Dapr and (optionally) your application side by side. Supported platforms: Self-hosted
  status         Show the health status of Dapr services. Supported platforms: Kubernetes
  stop           Stop Dapr instances and their associated apps. . Supported platforms: Self-hosted
  uninstall      Uninstall Dapr runtime. Supported platforms: Kubernetes and self-hosted
  upgrade        Upgrades a Dapr control plane installation in a cluster. Supported platforms: Kubernetes

Flags:
  -h, --help      help for dapr
  -v, --version   version for dapr

Use "dapr [command] --help" for more information about a command.
```

# Dapr 환경을 Docker에 구성하기
Self-hosted 환경으로 Local의 Docker에 설치를 해보겠다. 
```
> dapr init
```

설치된것을 확인하려면 docker의 container를 확인할 수 있다. 
```
> docker ps
CONTAINER ID   IMAGE               COMMAND                  CREATED             STATUS                       PORTS                              NAMES
0c47b28de401   daprio/dapr:1.4.3   "./placement"            About an hour ago   Up About an hour             0.0.0.0:6050->50005/tcp            dapr_placement
b73b0a70f8c6   redis               "docker-entrypoint.s…"   About an hour ago   Up About an hour             0.0.0.0:6379->6379/tcp             dapr_redis
0d4ca45df33d   openzipkin/zipkin   "start-zipkin"           About an hour ago   Up About an hour (healthy)   9410/tcp, 0.0.0.0:9411->9411/tcp   dapr_zipkin
```

Docker Desktop을 설치 하였으면 아래와 Docker Desktop에서도 확인을 할 수 있다. Redis은 state를 dapr를 통해 사용하는 것을 확인 하기 위해 미리 설치된 것이다. zipkin은 서비스간의 추적을 할 수 있는 기능으로 기본 설정이 되어 있다. 

![Docker Desktop](/posts/assets/cloudnative-dapr/images/dapr-docker-desktop.png){: width="500"} 


# Sidecar 실행
Dapr CLI를 이용하여 Sidecar만 실행해 보겠다. 서비스는 없이 Sidecar만 실행하여 State에 저장/읽기를 진행할 것이다. Command Prompt에서 실행을 해보자. (PowerShell에서는 명령문이 다르니 이 부분은 원문을 확인하기 바란다. (링크)[https://docs.dapr.io/getting-started/get-started-api/])

```
> dapr run --app-id myapp --dapr-http-port 3500
[37;1mWARNING: no application command found.[0m
Starting Dapr with id myapp. HTTP Port: 3500. gRPC Port: 49955
Checking if Dapr sidecar is listening on HTTP port 3500
time="2021-10-29T10:52:02.8504097+09:00" level=info msg="starting Dapr Runtime -- version 1.4.3 -- commit a8ee30180e1183e2a2e4d00c283448af6d73d0d0" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8520011+09:00" level=info msg="log level set to: info" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8525303+09:00" level=info msg="metrics server started on :49956/" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.metrics type=log ver=1.4.3
time="2021-10-29T10:52:02.8572716+09:00" level=info msg="standalone mode configured" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8572716+09:00" level=info msg="app id: myapp" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8583124+09:00" level=info msg="mTLS is disabled. Skipping certificate request and tls validation" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8663346+09:00" level=info msg="local service entry announced: myapp -> 192.168.1.30:49960" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.contrib type=log ver=1.4.3
time="2021-10-29T10:52:02.8663346+09:00" level=info msg="Initialized name resolution to mdns" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8673208+09:00" level=info msg="loading components" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8773228+09:00" level=info msg="component loaded. name: pubsub, type: pubsub.redis/v1" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8782643+09:00" level=info msg="waiting for all outstanding components to be processed" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8846169+09:00" level=info msg="component loaded. name: statestore, type: state.redis/v1" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8855046+09:00" level=info msg="all outstanding components processed" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8866165+09:00" level=info msg="enabled gRPC tracing middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.grpc.api type=log ver=1.4.3
time="2021-10-29T10:52:02.8866165+09:00" level=info msg="enabled gRPC metrics middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.grpc.api type=log ver=1.4.3
time="2021-10-29T10:52:02.8871525+09:00" level=info msg="API gRPC server is running on port 49955" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8882499+09:00" level=info msg="enabled metrics http middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.http type=log ver=1.4.3
time="2021-10-29T10:52:02.8882499+09:00" level=info msg="enabled tracing http middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.http type=log ver=1.4.3
time="2021-10-29T10:52:02.8893867+09:00" level=info msg="http server is running on port 3500" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8910776+09:00" level=info msg="The request body size parameter is: 4" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.891605+09:00" level=info msg="enabled gRPC tracing middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.grpc.internal type=log ver=1.4.3
time="2021-10-29T10:52:02.8921493+09:00" level=info msg="enabled gRPC metrics middleware" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.grpc.internal type=log ver=1.4.3
time="2021-10-29T10:52:02.8921493+09:00" level=info msg="internal gRPC server is running on port 49960" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8932554+09:00" level=info msg="actor runtime started. actor idle timeout: 1h0m0s. actor scan interval: 30s" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.actor type=log ver=1.4.3
time="2021-10-29T10:52:02.8938103+09:00" level=warning msg="app channel not initialized, make sure -app-port is specified if pubsub subscription is required" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8943618+09:00" level=warning msg="failed to read from bindings: app channel not initialized " app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.8948881+09:00" level=info msg="dapr initialized. Status: Running. Init Elapsed 37.616499999999995ms" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime type=log ver=1.4.3
time="2021-10-29T10:52:02.9136662+09:00" level=info msg="placement tables updated, version: 0" app_id=myapp instance=AHNCHAN-WIN-MINI scope=dapr.runtime.actor.internal.placement type=log ver=1.4.3
Checking if Dapr sidecar is listening on GRPC port 49955
Dapr sidecar is up and running.
You're up and running! Dapr logs will appear here.
```

app의 id는 myapp으로 하였고 dapr port는 3500으로 sidecar만인 Service가 실행이 되었다. gRPC는 선언을 하지 않아 임의로 포트를 설정한 것을 볼 수 있다. 

다른 창을 새로 실행하여 아래를 입력하면 dapr가 실행되어 있는 Application들을 확인할 수 있다. 

```
> dapr list
  APP ID  HTTP PORT  GRPC PORT  APP PORT  COMMAND  AGE  CREATED              PID
  myapp   3500       49955      0                  1m   2021-10-29 10:52.02  6508
```

# State를 저장/읽기
Dapr의 State management를 이용하여 Sate를 저장해보자. Redis를 이용하였지만, Azure CosmosDB, Azure SQL Server, PostgreSQL, AWS DynamoDB 를 사용할 수 있다.

```
> curl -X POST -H "Content-Type: application/json" -d "[{ \"key\": \"name\", \"value\": \"Bruce Wayne\"}]" http://localhost:3500/v1.0/state/statestore
```

이제는 저장된 값을 읽어보자.
```
> curl http://localhost:3500/v1.0/state/statestore/name
"Bruce Wayne"
```

특별한 프로그램없이 Dapr의 State Management 기능을 이용하면 Sate를 저장/관리할 수 있다. 
State에 대한 기본 설정은 dapr가 설치된 .\dapr\components\statestore.yaml 에 설정되어 있다. 

# Conclusions
Sidecar Pattern을 이용하면 Cloud Native Application 을 개발하면서 중복되는 부분을 줄여주고, 로직에 집중할 수 있도록 도와 준다. 기존의 HTTP, gRPC를 이용한 시스템은 PATH만 수정하는 것으로 Dapr를 사용할 수 있기 떄문에 장점이 될 수 있을 것 같다. 

다음은 Dapr의 Quick Starts의 예제를 실행해가면서 설명을 해보도록 하겠다. 


# References
[Dapr Concept](https://docs.dapr.io/concepts/overview/)

[Dapr Installation](https://docs.dapr.io/getting-started/install-dapr-cli/)

