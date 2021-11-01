---
layout: post
title:  "Dapr Quick Start - Hello World"
date:   2021-10-30 16:00:00 +0900
categories: CloudNative
---
created : 2021-10-30, updated : 2021-10-30


# Introduction
이전 튜토리얼에서는 Dapr를 설치하고 간단하게 ((링크)[https://ahnchan.github.io/posts/CloudNative-Dapr-Installation/]) State Store에 정보를 저장, 확인하는 것을 확인했다. 
이번 튜토리얼에서는 Dapr의 공식 Quick Start문서에서 Hello-World를 실행해보도록 하겠다. 


# Pre-Installation
Dapr Installation: (https://ahnchan.github.io/posts/CloudNative-Dapr-Installation/)[https://ahnchan.github.io/posts/CloudNative-Dapr-Installation/]
Dapr Installation(Darp.io): (https://docs.dapr.io/getting-started/)[https://docs.dapr.io/getting-started/]
Docker: (https://www.docker.com/products/docker-desktop)[https://www.docker.com/products/docker-desktop] 
Node.js: (https://nodejs.org/en/download/)[https://nodejs.org/en/download/] 
Python: (https://www.python.org/downloads/)[https://www.python.org/downloads/]https://www.python.org/downloads/]
GIT: (https://gitforwindows.org/)[https://gitforwindows.org/]

> Note. Windows 환경에서 설치를 해보겠다. 각각의 설치되는 부분은 간단하게 링크로 대체하였다. 

> Note. Macos에서 해당 Tutorial이 네트워크 보안문제로 구동이 되지 않았다. 보안설정이 없는 곳에서는 잘 동작되는 것 같다. 그래서 이곳에서는 Windows 환경에서 구성 해 보았다. 


# 구성
Node로 구성된 Service를 Python으로 구성된 Service에서 호출하는 것이다. Node에서는 State정보를 Dapr에 요청하여 저장, 확인하게 되어 있다. State는 기본 설정된 Redis를 사용하였고 Dapr 컨셉문서에도 명시된 대로 다른 DB를 사용할 수도 있다. 

![Diagram](/posts/assets/cloudnative-dapr/images/dapr-hello-world-diagram.png){: width="500"} 


# Source 가져오기
Dapr의 Quick Start 의 GIT Repository를 가져온다. 공식문서에서는 버전별로 tag를 입력하도록 되어 있다. Dapr의 버전을 확인하고 Quick Start 지원 문서 ((링크))[https://github.com/dapr/quickstarts#supported-dapr-runtime-version]를 확인하여 알맞은 Source를 가져와 보자. 

```
> dapr --version
CLI version: 1.4.0
Runtime version: 1.4.3
```

Runtime Version을 확인했으니 문서를 확인해보니 v1.4.0을 가져오면 된다.

```
> git -b v1.4.0 https://github.com/dapr/quickstarts.git
…
> cd quickstart\hello-world
```

# Node.js Application 실행하기
Node.js Application 은 외부의 요청을 받아서 State Store의 정보를 저장, 읽기를 한다. State 에 저장/읽기를 할때에는 Dapr의 State Management를 이용한다.

Node 소스의 위치로 이동하고 기존 Package를 설치 한다. 
```
> cd node
> npm install
…
```

dapr를 이용하여 app.js(Node.js)를 실행을 한다. 
```
> dapr run --app-id nodeapp --app-port 3000 --dapr-http-port 3500 node app.js
Starting Dapr with id nodeapp. HTTP Port: 3500. gRPC Port: 60538
…
```

실행한 내용을 보면 nodeapp (App ID)으로  Dapr 포트는 3500으로 서비스가 실행된 것을 확인 할 수 있다. 

서비스가 잘 동작하는지 테스트를 해보자.
```
> curl -XPOST -d @sample.json -H Content-Type:application/json http://localhost:3500/v1.0/invoke/nodeapp/method/neworder
```

Node.js Application의 로고에 아래와 같이 표시된다.
```
​​== APP == Got a new order! Order ID: 42
== APP == Successfully persisted state.
```

저장된 내용을 다시 요청을 해보면 아래와 같이 결과가 나온다. 
```
> curl http://localhost:3500/v1.0/invoke/nodeapp/method/order
{"orderId":"42"}
```

잘 동작하는 것을 확인하였으니 다음 단계로 넘어가보자.
 

# Python Application 실행하기
Python Application 은 1초마다 Order ID를 1씩 증가시키면서 Node.js Service를 호출한다. Pyhthon을 실행하기 위해서 기본 패키지를 설치한다.

```
> cd python
> pip install requests
...
```

> Note. Python 설치된 버전에 따라 pip3로 요청해야할 수도 있다. 처음 설치를 한 것이면  Python으로 실행하게 되어 있다. 

app.py 코드를 보면 Darp 포트로 nodeapp Application의 neworder 를 호출하게 되어 있다. 

Python Application 을 실행해 보자.
```
> dapr run --app-id pythonapp python app.py
Starting Dapr with id pythonapp. HTTP Port: 49587. gRPC Port: 49588
…
```

pythonapp 의 App ID로 구동된 것을 확인 할 수 있다. Http, gPRC의 포트를 따로 설정하지 않았기에 임의로 포트가 자동으로 설정된 것을 확인 할 수 있다. 

nodeapp의 로그는 아래와 같이 1초마다 표시되는 것을 확인할 수 있다. 
```
== APP == Got a new order! Order ID: 1
== APP == Successfully persisted state.
== APP == Got a new order! Order ID: 2
== APP == Successfully persisted state.
== APP == Got a new order! Order ID: 3
== APP == Successfully persisted state.
== APP == Got a new order! Order ID: 4
== APP == Successfully persisted state.
== APP == Got a new order! Order ID: 5
== APP == Successfully persisted state.
== APP == Got a new order! Order ID: 6
== APP == Successfully persisted state.
…
```

Dapr로 구동된 Application 을 확인하기
CLI를 이용하여 확인을 할수도 있고 Dapr dashboard 를 활용하여 확인할 수도 있다. 

```
> dapr list
  APP ID     HTTP PORT  GRPC PORT  APP PORT  COMMAND        AGE  CREATED              PID
  nodeapp    3500       60538      3000      node app.js    14m  2021-10-29 14:33.14  14576
  pythonapp  56379      56380      0         python app.py  7s   2021-10-29 14:47.52  19280
```

dashboard를 실행해보자. 웹에서 Application의 구동 상태황과 설정값등을 확인할 수 있다.

```
> dapr dashboard -p 9999
Dapr Dashboard running on http://localhost:9999
```

9999포트로 Dapr Dashboard가 구동되었다. 브라우저에 http://localhost:9999 로 확인을 해보자.

![Dapr Dashboard](/posts/assets/cloudnative-dapr/images/dapr-quickstart-hello-world-dashboard.png){: width="500"} 



# Conclusions
Dapr는 앞으로 계속 확장될 것으로 기대가 된다. 다른 문서들을 보면 현재의 문제를 보안하기 위해 Istio 등과 같이 사용하는 방법이 제시되고 있다. Sidecar 에 Proxy 를 이용하면 좀더 정교한 컨트롤이 가능할 것으로 생각이 된다.

> Note. 이 소스를 Kubernetes에 올리는 Quick Start는 이 (링크)[https://github.com/dapr/quickstarts/tree/v1.4.0/hello-kubernetes]에 있다. 실제로 Cloud Native Application을 효과적으로 사용하기 위해서는 현재는 Kubernetes 환경이 가장 좋다. 


# References
[Dapr Quick Starts](https://docs.dapr.io/getting-started/quickstarts/)

[Dapr Quick Starts, Hello World](https://github.com/dapr/quickstarts/tree/v1.4.0/hello-world)
