---
layout: post
title:  "Kubernetes Service and iptables"
date:   2018-09-27 13:00:00 +0800
categories: kubernetes network service
---
最近研究 Kubernetes 的网络，终于有空看下 Service 的转发规则。

Service 的内部实现是 kube-proxy。kube-proxy有三种模式，userspace/iptables/ipvs. 关于这三种模式可以参考[官网](https://kubernetes.io/docs/concepts/services-networking/service/)。userspace是最初始的时候用的，iptables使用的最广泛，也是默认选项，ipvs是v1.11.2 GA的功能，主要为了解决 iptables 规则多了以后的性能下降。接下去我们讨论的是 iptables 模式。

首先，起一个echoserver Pod：

```sh
kubectl run echoserver --image=gcr.io/google_containers/echoserver --port=8080
kubectl get pods
NAME                                                   READY   STATUS      RESTARTS   AGE
echoserver                                             1/1     Running     0          3h4m
```

我们为这个 Pod 开启3个 Service，分别是 ClusterIP, NodePort, LoadBalancer:

```sh
kubectl expose pod echoserver --name=echoserver-clusterip --type=ClusterIP
kubectl expose pod echoserver --name=echoserver-clusterip --type=NodePort
kubectl expose pod echoserver --name=echoserver-clusterip --type=LoadBalancer
kubectl get svc
NAME                      TYPE           CLUSTER-IP        EXTERNAL-IP   PORT(S)          AGE
echoserver-nodeport       NodePort       169.169.211.218   <none>        8080:30411/TCP   115m
echoserver-clusterip      ClusterIP      169.169.52.128    <none>        8080/TCP         89m
echoserver-loadbalancer   LoadBalancer   169.169.117.159   <pending>     8080:30460/TCP   28m
```

访问 nodeport:

```sh
curl localhost:30411
CLIENT VALUES:
client_address=192.168.178.64
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://localhost:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=localhost:30411
user-agent=curl/7.61.1
BODY:
-no body in request-
```

其实在主机上是看不到 service iptables rules。仔细研究了发现 kube-proxy 是以 host network 跑在每个集群节点上。kube-proxy 会监听 nodeport:

```sh
lsof -i:30411
COMMAND    PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
kube-prox 3623 root   19u  IPv4 56100432      0t0  TCP *:30411 (LISTEN)
lsof -i:30460
COMMAND    PID USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
kube-prox 3623 root   22u  IPv4 56408891      0t0  TCP *:30460 (LISTEN)
```

这里有了第一个结论，nodeport请求到达主机以后，会通过 kube-proxy 的监听端口先进入 kube-proxy。

我们接着往下，进入 kube-proxy 看 iptables：

```sh
kubectl exec kube-proxy-ts54q -n kube-system -- iptables -t nat -S | grep 30411
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/echoserver-nodeport:" -m tcp --dport 30411 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/echoserver-nodeport:" -m tcp --dport 30411 -j KUBE-SVC-ZFAVNEBGNVMBQPKH
```

请求通过 KUBE-NODEPORTS chain 进入了 KUBE-SVC-ZFAVNEBGNVMBQPKH。

```sh
kubectl exec kube-proxy-ts54q -n kube-system -- iptables -t nat -S KUBE-SVC-ZFAVNEBGNVMBQPKH
-N KUBE-SVC-ZFAVNEBGNVMBQPKH
-A KUBE-SVC-ZFAVNEBGNVMBQPKH -m comment --comment "default/echoserver-nodeport:" -j KUBE-SEP-YURMZH5MPUXHWC2S
```

又进入了 KUBE-SEP-YURMZH5MPUXHWC2S chain。SEP 应该是 Service Endpoint

```sh
kubectl exec kube-proxy-ts54q -n kube-system -- iptables -t nat -S KUBE-SEP-YURMZH5MPUXHWC2S
-N KUBE-SEP-YURMZH5MPUXHWC2S
-A KUBE-SEP-YURMZH5MPUXHWC2S -s 192.168.192.117/32 -m comment --comment "default/echoserver-nodeport:" -j KUBE-MARK-MASQ
-A KUBE-SEP-YURMZH5MPUXHWC2S -p tcp -m comment --comment "default/echoserver-nodeport:" -m tcp -j DNAT --to-destination 192.168.192.117:8080
```

终于进入到 Pod 了。

我们注意到 LoadBalancer Service 也提供了 NodePort 端口，用以上的方法发现 iptables 转发途径和 NodePort 是一摸一样的。

再来查下转入该 Pod 的 iptables rules:

```sh
kubectl exec kube-proxy-ts54q -n kube-system -- iptables -t nat -S | grep 192.168.192.117:8080
-A KUBE-SEP-2YAJQQQEQS4RATHW -p tcp -m comment --comment "default/echoserver-loadbalancer:" -m tcp -j DNAT --to-destination 192.168.192.117:8080
-A KUBE-SEP-SSA7YNBBX76QBJQ5 -p tcp -m comment --comment "default/echoserver-clusterip:" -m tcp -j DNAT --to-destination 192.168.192.117:8080
-A KUBE-SEP-YURMZH5MPUXHWC2S -p tcp -m comment --comment "default/echoserver-nodeport:" -m tcp -j DNAT --to-destination 192.168.192.117:8080
```

分别对应了3种不同类型的 Service。

### 总结

在 kube-proxy iptables 模式下，Service 的本质就是 kube-proxy 容器中的一段引向 Pod IP 的 iptable rules。  ClusterIP Service 的 iptables 路径是 Service ClusterIP => Pod IP  
NodePort Service 的 iptables 路径是 host:NodePort => kube-proxy:NodePort => Service ClusterIP => Pod IP



### 相关资料

[service](https://kubernetes.io/docs/concepts/services-networking/service/)  

[kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)  

[kube-proxy-iptables src](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/iptables/proxier.go)

