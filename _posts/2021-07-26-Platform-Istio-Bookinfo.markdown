---
layout: post
title:  "Istio Sample Application Bookinfo 구동하기"
date:   2021-07-26 16:00:00 +0900
categories: Platform
---
created : 2021-07-26, updated : 2021-07-26

# Introduction
Istio는 Envoy Proxy를 이용하여 서비스간에 통신이 되도록하는 Open Source 이다. Micro service를 구성할때 필요한 기능이다. Kubernetes 에 구성되어 Service마다 Envoy Proxy를 포함하여 Traffic 관리, Security 를 관리할 수 있다. 

본 문서는 Istio의 Sample인 Bookinfo를 Kubernetes Cluster에 설치를 해보고 동작을 확인해보겠다. 거의 모든 내용을 [링크](https://istio.io/latest/docs/setup/getting-started/)를 따라해보는 것을 한글화 한것이라고 생각하면 된다.


# Prerequisites
[Kubernetes Cluster 구성](https://ahnchan.github.io/posts/Platform-kubernetes_ubuntu20/)(혹은 Docker Desktop의 Kubernetes 구성 : [링크](https://istio.io/latest/docs/setup/platform-setup/docker/)) 되어 있어야 한다.
Host는 Macos에서 VirtualBox로 구성된 Kubernetes로 Istio 를 설치하고, Bookinfo를 설치해보도록 하겠다. 


# Download Istio
Istio 를 다운로드 한다. 나는 Macos 에서 다운로드를 받아서 Virtualbox 에 설치된 Kubernetes Cluster에 설치를 진행할 것이다.
이를 위해서는 kubectl 이 kubernetes Cluster를 잘 연결되어 있어야 한다. (config  설정이 필요함)

```
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-1.10.3
```

Bookinfo 는 sample/bookinfo 디렉토리에 있음.
Istioctl 은 bin 디렉토리에 있음.

> Note. Bookinfo의 Application들의 소스는 Istio Gihub 에 있음. 위에서 받은 istio sample 에는 bookinfo의 서비스들에 대한 docker image정의와 네트워크, Gateway등만 정의(yaml)되어 있음.


# Install Istio
profile을 demo로 설정하여 설치를 한다. profile은 좀더 내용을 보고 공부를 해보자 [링크](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

```
$ istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Egress gateways installed
✔ Ingress gateways installed
✔ Installation complete
```

Application을 Deploy할때 Envoy Proxy가 자동으로 들어가도록 설정을 하자.

```
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```


# Deploy the Bookinfo Application

bookinfo를 설치해보자. productpage(python), review(Java), Detail(Ruby), Rating(Node.js) 로 구성되어 있다. 

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```


설치된 Application 을 확인해보자. 2개의 인스턴스씩 구동이 되어 있는 것을 확인할 수 있다. 설정은 bookinfo.yml[링크](https://github.com/istio/istio/blob/master/samples/bookinfo/platform/kube/bookinfo.yaml)에서 확인할 수 있다. 

```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-79f774bdb9-cjkzv       2/2     Running   2          3d3h
productpage-v1-6b746f74dc-gr5rl   2/2     Running   2          3d3h
ratings-v1-b6994bb9-xb282         2/2     Running   2          3d3h
reviews-v1-545db77b95-ddxd8       2/2     Running   2          3d3h
reviews-v2-7bf8c9648f-hd77h       2/2     Running   2          3d3h
reviews-v3-84779c7bbc-69jzz       2/2     Running   2          3d3h
```

```
$ kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.102.134.89    <none>        9080/TCP   3d3h
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    4d2h
productpage   ClusterIP   10.105.5.40      <none>        9080/TCP   3d3h
ratings       ClusterIP   10.104.113.166   <none>        9080/TCP   3d3h
reviews       ClusterIP   10.98.136.45     <none>        9080/TCP   3d3h
```


# 외부에서 접근하게 Gateway를 설치
외부(Kubernetes Cluster)에서 접근을 하기 위해서 Gateway (ingress)를 설정한다. 

```
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```

이슈가 없는지 확인을 해본다.

```
$ istioctl analyze
✔ No validation issues found when analyzing namespace: default.
```


접속할 HOST(INGRESS_HOST), PORT(INGRESS_PORT)를 가져온다 

```
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
```

검증을 위해서 설정된 값을 출력해본다. 

```
$ echo $INGRESS_PORT
30943
$ echo $SECURE_INGRESS_PORT
31908
```

```
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.100.193.188   <pending>     15021:32473/TCP,80:30943/TCP,443:31908/TCP,31400:32200/TCP,15443:32096/TCP   3d5h
```

EXTERNAL-IP가 <pending> 상태이다. Kubernetes Cluster설정할 때 무언가 하지 않아서 그런것 같다. 우리는 Kubernetes Cluster를 설정할때 master, worker1, worker2의 Static IP 를 지정했었다. 그러니 일단 알고 있는 master의 IP를 INGRESS_HOST에 설정을 하자. 

```
$ export INGRESS_HOST=192.168.0.71
```

> Note. 이 부분은 Kubernetes Cluster, Network를 좀더 공부한 후에 보강을 해보겠다. 

사용하기 편하게 하기 위해 GATEWAY_URL을 설정을 해보자.

```
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
$ echo "$GATEWAY_URL"
192.168.0.71:30943
```

위의 PORT는 설치를 어떻게 했느냐에 따라 다르게 나타납니다. Docker Desktop에 설치하였을때와 VirtualBox에 직접 구성하였을때가 다릅니다. 

위의 URL 을 브라우져에서 실행을 해보자. 

![bookinfo's productpage](/posts/assets/platform/images/istio-bookinfo-web.png)


# Kubernetes Cluster Addon 설치하기

Sample 디렉토리의 addons 에 정의된 Application 을 설치한다. 

```
$ kubectl apply -f samples/addons
$ kubectl rollout status deployment/kiali -n istio-system
Waiting for deployment "kiali" rollout to finish: 0 of 1 updated replicas are available...
deployment "kiali" successfully rolled out
```

- Prometheus : Kubernetes Cluster의 정보를 수집한다.
- Grafana : Prometheus의 수집된 정보를 웹 화면으로 표시한다.
- Kiali : 설치된 Application의 구성, 상태를 확인한다.

```
$ istioctl dashboard kiali
```

웹 브라우저가 뜨면서 kiali가 실행이 된다. 그러나 아무것도 안보일 것이다. 아마 보인다면 위에서 샘플을 많이 실행하고, 빠르게 kiali dashboard를 실행했을 것이다. (5분내)

kiali는 Application이 실시간 구동되는 정보를 수집하여 시각화해준다. 기준 시간이 5분동안으로 설정되어 있어서 처음에는 아무것도 안나오는 것 처럼 보인다. 

아래의 스크립트로 여러번 실행을 하게 해보자. 

```
$ for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

> Note. 실제 부하를 주기 위해서는 JMeter와 같은 툴을 이용하는 것이 좋다. 

Namespace를 선택하면 아래와 같이 Application이 구동된 것을 확인할 수 있다. 

![kiali dashboard](/posts/assets/platform/images/istio-kiali.png)


# Conclusions
문서를 작성하다보니 Istio 공식 문서의 Example 페이지를 번역한것 정도밖에는 안된것 같다. 약간의 경헙을 넣었지만, 도움이 되었으면 좋겠다. 
Microservice의 기본 사상을 코드로 이해하기 좋은 예제이며, 실제 Java, Ruby, Node.js의 소스들[Github Repository 링크](https://github.com/istio/istio/tree/master/samples/bookinfo/src)을 확인해서 어떻게 Application 간 정보를 교환하는지 확인해보면 좋을 것 같다. 


# References
[Docker Desktop 에서 Kubernetes, Istio](https://istio.io/latest/docs/setup/platform-setup/docker/)

[Istio Sample](https://www.docker.com/blog/getting-started-with-istio-using-docker-desktop/)

[Istio Profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)

[Istio GitHub repository](https://github.com/istio/istio)

[Configuration to Access Multiplue Clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)




