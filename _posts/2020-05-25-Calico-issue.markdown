---
layout: post
title:  "Readiness probe failed: caliconode is not ready: BIRD is not ready: BGP not established with"
date:   2020-05-25 13:00:00 +0800
categories: kubernetes calico issue
---
今天发现 k8s 集群有个节点的 calico-node 挂了，显示 unready。

根据 pod readiness probe 到容器中发现以下错误：

```sh
# calico-node -bird-ready
health.go 156: Number of Node(s) with BGP peering established = 0
caliconode is not ready: BIRD is not ready: BGP not established with ...
```

原因是 calico-node BGP peering 的时候无法和其他节点通信和同步数据。

根据[这篇博客](https://blog.csdn.net/u011327801/article/details/100579803)，calico-node启动的时候会自动发现网卡，默认配置下很容易导致异常的ip作为nodeIP被注册。去节点上查看果然发现有两个以前docker留下来的network bridge。把 bridges 删除后重启 calico-node，服务恢复。


