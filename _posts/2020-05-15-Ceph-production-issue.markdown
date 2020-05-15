---
layout: post
title:  "记录一次Ceph生产集群事故"
date:   2020-05-15 10:57:11 +0800
categories: ceph production
---
Ceph 集群版本: v15.2.1 Octopus  
Ceph 安装方式: cephadm  
Ceph 集群规模: 4  

由于需要重启的时候加载Ceph RBD，所以改动了`/etc/fstab`，结果重启之后机器无法启动。经过查询应该用[rbdmap][rbdmap]服务的方式实现开启加载Ceph RBD。注意 `/etc/fstab` 中的选项需要是 `noauto`，这样开机的时候不会被挂载，启动 rbdmap 服务的时候才会被挂载，确保安全。

好了，Ceph主节点挂了，赶快看一下Ceph集群是否有影响。Ceph Dashboard会在备用的节点启动，Dashboard页面是这样的：
![](/img/ceph-dashboard-error.png)

错误信息是：

```sh
MGR_MODULE_ERROR: Module 'rbd_support' has failed: Not found or unloadable
```

看到这条错误信息，我一度以为 Ceph RBD 不可用了。好在过了大概十几分钟这条错误消息消失了。

第二天我再次复现这个问题，用 systemctl 把 Ceph Manager 和 一台机器上的Ceph OSD 服务停掉。测试下来Ceph Manager 备用服务马上启动，而且在一台机器OSD都停掉的时候依然可以读写 Ceph RBD。

其实 Ceph Manager 挂掉了不影响 Ceph RBD 的使用，Ceph RBD取决于RADOS，而RADOS有两个服务组成，分别是：Ceph Monitor (保存了元数据，Paxos保持数据一致)和Ceph OSD。挂掉一台节点，正在使用中的Ceph RBD还是可以使用的，从上图中右下角的PG Status也可以看出，由于3备份的机制，没有红色Unfound的PG。

正是因为Ceph Manager只是协助管理Ceph集群，不影响Ceph存储服务的使用，所以官方采用了active-standby的HA模式。

#### 总结

Ceph Manager只是参与Ceph集群的管理和配置，但对 RBD 的服务没有影响。决定 RBD 服务的是 元信息管理 (Ceph Monitor, Paxos集群) 以及 一个个的 OSD。在仅挂一台服务器的情况下，服务是可以保证的。

这里还留下了一条悬而未决的错误信息，这条错误信息网上的信息很少，查出来的一条跟 Ceph 集群的升级有关，我们在下文中从 Ceph 源码层面对这个问题做进一步的追溯。

[rbdmap]:      https://docs.ceph.com/docs/master/man/8/rbdmap/