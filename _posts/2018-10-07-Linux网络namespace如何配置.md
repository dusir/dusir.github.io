---
layout: post
read_time: true
show_date: true
title: "network namespace如何配置"
date: 2015-10-07
img: posts/20210420/post8-rembrandt.jpg
tags: [网络虚拟化]
category: opinion
author: Bobo Du
description: "网络虚拟化"
---
## 如何配置network namespace
我们知道一个系统要和外界通信或者系统里边的进程涉及网络通信时，我们都需要一个网络设备，而且network namespace这种隔离网络资源的新技术空间也不例外。

没错，namespace里边的（虚拟的）网络设备必不可少。从上一篇文章我们知道，一个全新的network namespace会默认带一个本地回环lo设备。除此之外，再无
其他设备，而且lo设备还是down的状态，因此当你一开始就要访问本地回环地址时，网络当然也是不通的。

我们可以做个小测试看看：
```
[root@VM-centos ~]# ip netns add test
[root@VM-centos ~]# ip netns exec test ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[root@VM-centos ~]# ip netns exec test ping 127.0.0.1
connect: Network is unreachable
[root@VM-centos ~]# 
```
在这个测试中，我们看到，你要想访问你的本地回环设备地址，首先得进入你的test 这个network namespace，其次才能操作。

接着是你想要ping通网络，你得把设备的状态设置成为UP状态,然后再尝试ping:
```
[root@VM-centos ~]# ip netns exec test ip l set lo up
[root@VM-centos ~]# ip netns exec test ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
[root@VM-centos ~]# ip netns exec test ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.027 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.027 ms
^C
--- 127.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.027/0.027/0.027/0.000 ms
[root@VM-centos ~]# 
```

但是问题来了，仅有一个本地回环设备是不足以和外界通信的。如果咱们想和外界（近的来说，主机的网卡）进行网络通信，这就需要再准备一个网卡设备，虚拟的或
者真是的物理网卡，通常云计算环境下，我们都是再network namespace里创建一对虚拟的以太网卡，也就是我们常说的veth pair。

什么是veth pair？
故顾名思义，veth pair就是成对出现并且互联互通，它就像Linux的双向通道一样（PIPE），网络报文从一头进去，然后从另外一端收到，可以理解成一根网线，
很奇妙吧。


