---
title: Docker使用宿主代理
date: 2020-04-28 15:40:24
tags:
- Docker
categories: Linux
---

![20200428102310](https://img.yjll.art/img/20200428102310.png)


在docker中安装软件，软件的官网在外国，下载速度奇慢，这个时候想通过宿主的代理访问。

<!-- more -->

Docker启动容器，默认使用bridge模式分配网络，会在宿主机安装一个虚拟网卡docker0。我们通过`ifconfig`命令查看一下

```
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:caff:fefe:2f7e  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ca:fe:2f:7e  txqueuelen 0  (Ethernet)
        RX packets 7941959  bytes 525663055 (525.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8654991  bytes 11073643780 (11.0 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

对于容器来说，宿主的IP地址为`172.17.0.1`，我们进入容器进行代理配置

```
export http_proxy=http:172.17.0.1:1081
export https_proxy=http:172.17.0.1:1081
```

好了，可以愉快的乱搞了。




