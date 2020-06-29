---
layout: post
title:  "Ceph RBD performance test"
date:   2020-05-29 13:00:00 +0800
categories: ceph rbd performance
---
Ceph 集群版本: v15.2.1 Octopus
Ceph 安装方式: cephadm
Ceph 集群规模: 
	hosts: 4
	OSDs: 12 * SSD-cached HDD
测试工具：rados bench, rbd bench

首先在 Ceph 集群中新建一个 pool：

```sh
ceph osd pool create testbench
```

清除文件系统 cache：

```sh
echo 3 | tee /proc/sys/vm/drop_caches && sync
```

测试写、顺序读、随机读：

```sh
# rados bench -p testbench 10 write --no-cleanup
hints = 1
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 10 seconds or 0 objects
Object prefix: benchmark_data_shoyb18plpcdp161.mcd.com.cn_39383
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      16        55        39   155.968       156    0.379957    0.354705
    2      16       100        84   167.971       180    0.420306    0.337481
    3      16       163       147   195.967       252    0.449747     0.30892
    4      16       220       204   203.964       228    0.393217    0.301159
    5      16       291       275    219.96       284    0.185473     0.28066
    6      16       340       324   215.962       196    0.417506    0.287384
    7      16       385       369   210.819       180   0.0772004    0.289772
    8      16       43       415   207.462       184    0.124724    0.298966
    9      16       483       467   207.517       208   0.0871156    0.291997
   10      15       536       521   208.361       216    0.345622    0.299994
   11       1       536       535   194.509        56     0.16792    0.298941
   12       1       536       535     178.3         0           -    0.298941
Total time run:         12.9555
Total writes made:      536
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     165.49
Stddev Bandwidth:       79.186
Max bandwidth (MB/sec): 284
Min bandwidth (MB/sec): 0
Average IOPS:           41
Stddev IOPS:            19.7965
Max IOPS:               71
Min IOPS:               0
Average Latency(s):     0.304479
Stddev Latency(s):      0.228196
Max latency(s):         3.26692
Min latency(s):         0.060205
```

```sh
# rados bench -p testbench 10 seq
hints = 1
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      15       159       144   575.709       576    0.471626   0.0966268
    2      16       312       296    591.82       608     0.14828    0.101285
    3      16       474       458   610.519       648   0.0499225    0.100228
Total time run:       3.75091
Total reads made:     536
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   571.594
Average IOPS:         142
Stddev IOPS:          9.0185
Max IOPS:             162
Min IOPS:             144
Average Latency(s):   0.108694
Max latency(s):       0.640254
Min latency(s):       0.00529039
```

```sh
# rados bench -p testbench 10 rand
hints = 1
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      16       260       244   975.837       976   0.0282401   0.0601993
    2      16       510       494   987.854      1000    0.204872   0.0626225
    3      15       778       763   1017.15      1076   0.0218655   0.0597019
    4      16      1121      1105   1104.82      1368   0.0157524   0.0567073
    5      16      1436      1420   1135.82      1260  0.00875568     0.05476
    6      16      1756      1740   1159.81      1280   0.0138018   0.0541582
    7      15      2098      2083   1190.02      1372   0.0126553   0.0526946
    8      15      2469      2454   1226.64      1484   0.0204296   0.0513035
    9      15      2882      2867   1273.85      1652   0.0278166   0.0493902
   10      15      3319      3304   1321.21      1748   0.0701715   0.0476415
Total time run:       10.0395
Total reads made:     3319
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   1322.38
Average IOPS:         330
Stddev IOPS:          65.1259
Max IOPS:             437
Min IOPS:             244
Average Latency(s):   0.0475984
Max latency(s):       0.507579
Min latency(s):       0.00336697
```

清除 testbench pool 的内容：

```sh
rados -p testbench cleanup
```



Ceph RBD 测试

新建一个 RBD:

```sh
rbd create image01 -s 1G -p testbench
```

挂载 RBD：

```sh
rbd map -p testbench image01
```

刷 ext4：

```sh
mkfs -t ext4 /dev/rbd/testbench/image01
```

挂载：

```sh
mkdir /mnt/ceph-test-block
mount /dev/rbd1 /mnt/ceph-test-block
```

测试顺序读：

```sh
# rbd bench --io-type write image01 --pool=testbench
bench  type write io_size 4096 io_threads 16 bytes 1073741824 pattern sequential
  SEC       OPS   OPS/SEC   BYTES/SEC
    1     18480   18312.7    72 MiB/s
    2     41120   20445.1    80 MiB/s
    3     69664   23226.4    91 MiB/s
    4     99600     24910    97 MiB/s
    5    130928     26053   102 MiB/s
    6    154368   27220.9   106 MiB/s
    7    182944   28438.4   111 MiB/s
    8    213984   28748.7   112 MiB/s
    9    242352   28550.1   112 MiB/s
elapsed: 9   ops: 262144   ops/sec: 27312.1   bytes/sec: 107 MiB/s
```



参考：

https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/4/html/administration_guide/ceph-performance-benchmarking