---
layout: post
title:  "Kubernetes Cluster 종료하기/시작하기"
date:   2021-07-22 16:00:00 +0900
categories: Platform
---
created : 2021-07-22, updated : 2021-07-22

# Introduction
이미 만들어진 Kubernetes Cluster(링크)를 종료하고 시작하는 방법을 기술해본다. 처음에 재시작하고 문제가 발생하여 해결하기 위해 문서를 작성하기 시작하였는데, Tip에 설명한 문제였다 몇번의 재설치(Ansible 만세)로 보강을 할 수 있었다.

# Prerequisites
[Ansible을 이용하여 Ubuntu 20.04에 Kubernetes 구성하기](https://ahnchan.github.io/posts/Platform-kubernetes_ubuntu20/)

# Stop Kubernetes Cluster
Worker node들을 종료를 한다.
Master node를 종료한다.

# Start Kubernetes Cluster
Master node를 시작한다.
Worker node들을 시작한다.
VirtualBox 환경에서는 순서대로 실행만 하면 된다.

> Note. Reference 문서를 보면 NFS, Docker Registry 등 외부 시스템연계시 순서를 지켜야하는 것 같다. 

# Tip
Swap이 Disable되어 있지 않으면 시스템이 부팅할때 kubelet 오류가 나온다. 

```
$ kubectl get nodes
The connection to the server 192.168.0.71:6443 was refused - did you specify the right host or port?
```

```
$ more /var/syslog
…
Jul 22 03:17:37 ubuntu kubelet[6874]: E0722 03:17:37.139357    6874 server.go:204] "Failed to load kubelet config file" err="failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file \"/var/lib/kubelet/config.yaml\", error: open /var/lib/kubelet/config.yaml: no such file or directory" path="/var/lib/kubelet/config.yaml"
…
```

위의 오류는 아래의 명령으로 kubelet을 다시 실행할 수 있다. 

```
$ sudo swapoff -a
$ sudo systemctl start kubelet
$ ps -ef | grep kubelet
root         672       1  2 05:05 ?        00:00:19 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrapkubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1
root        2352    2271  4 05:05 ?        00:00:39 kube-apiserver --advertise-address=192.168.0.40 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt ...
```

프로세스가 동작하는 것을 확인할 수 있다. 

```
$ sudo docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS     NAMES
a27eeafa09c2   296a6d5035e2             "/coredns -conf /etc…"   13 minutes ago   Up 13 minutes             k8s_coredns_coredns-558bd4d5db-tkzgn_kube-system_1dfff9a5-3469-4387-b560-31829340109d_3
fc7425ee2df2   296a6d5035e2             "/coredns -conf /etc…"   13 minutes ago   Up 13 minutes             k8s_coredns_coredns-558bd4d5db-zhgg8_kube-system_36d7074b-ea89-44b6-a5cd-b1b47e8fb7a0_3
f17287acaaeb   k8s.gcr.io/pause:3.4.1   "/pause"                 13 minutes ago   Up 13 minutes             k8s_POD_coredns-558bd4d5db-zhgg8_kube-system_36d7074b-ea89-44b6-a5cd-b1b47e8fb7a0_77
9082067fd537   k8s.gcr.io/pause:3.4.1   "/pause"                 13 minutes ago   Up 13 minutes             
...
```

docker에 컨테이너들이 동작하는 것을 확인할 수 있다.
 
# References
[Stopping and starting a Kubernetes cluster and pods](https://www.ibm.com/docs/en/fci/1.0.2?topic=SSCKRH_1.0.2/platform/t_start_stop_kube_cluster.html)
[Kubernetes restart error](https://stackoverflow.com/questions/43603545/kubernetes-does-not-start-after-restart-system-ubuntu)


