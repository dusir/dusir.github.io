---
layout: post
read_time: true
show_date: true
title: "Linux网络虚拟化"
date: 2015-10-07
img: posts/20210420/post8-rembrandt.jpg
tags: [网络虚拟化]
category: opinion
author: Bobo Du
description: "网络虚拟化"
---
## 网络虚拟化的基石
### Namespace
而且准确的说是Network Namespace。
因为它真的很重要，毫不夸张的说，它是整个云计算领域网络虚拟化技术的基石。顾名思义，Linux的namespace直接翻译过来就是"名字空间"，它的根本作用就是
"隔离资源"。

到底隔离啥资源呢？
隔离的就是我们经常看到的，用到的以下资源：
文件系统挂载点、主机名、POSIX进程间通信、进程PID资源、网络（IP地址，路由，协议栈，网络设备）、用户等全局资源。namespace技术把这些资源分割到了
一个个抽象的独立空间里。而隔离他们用的具体隔离技术就是：Mount namespace、UTS namespace、IPC namespace、PID namesapce、network 
namespace和user namespace。

那么，你要用到哪个namespace里的资源，就要进入相应的ns（namespace）里去。比如对进程来说，要想使用namespace里的资源，首先就要
"进入"（后文介绍操作）到整个namespace去进行访问。

这样一个PID namespace给Linux的进程造成了几个错觉：

    (1)我是系统里唯一的进程。
    
    (2)我独享系统的所有资源。


默认情况下，Linux系统里的进程处于和宿主机相同的系统默认的namespace当中，也就是初始的根namespace里，整个ns默认享有系统全局所有资源。

这种技术什么时候引入的？
我们可以看到，Linux内核从2.4.19版本开始接入第一个namespace的支持：Mount namespace（用于隔离文件系统挂载点）起，到了Linux 内核3.8版本引入
的user namespace（用于隔离用户资源的），总共实现了上文提到的6种不同类型的namespace。

这么好的东西，又是如何进入大众视野的呢？
虽然Linux的namespace技术引入的也比较早了，但是一直鲜为人知。直到前些年一个新星产品的出现，才引领了整个新的技术时代：它就是Docker，它直接引领了
容器技术的革命爆发，是的普罗大众开始进入ns的世界，Docker用的资源隔离技术就是Linux内核提供的namespace技术。

我们这里要说的是网络 namespace，它是从Linux内核2.6版本才引入的，作用很明显，隔离网络设备，以及IP地址、端口，路由表，防火墙等网络资源。也就说，
在整个namespace里，有自己的网络设备（比如IP地址、端口范围、路由表信息，/proc/net资源等）。从容器网络的角度来讲，每个容器都有自己的网络设备，
并且容器里的进程可以安全的绑定在端口上而不用担心资源冲突，这就可以使一个主机上同时监听多个80端口的服务成为可能。

![ns demo](./assets/img/posts/network/ns示意图.jpg)

揭开面纱的时刻
和其他namespace一样，network namespace 可以通过系统调用或者命令行来创建，我们可以使用Linux的clone（）（fork系统调用就是它实现的）API创建
一个通用的namespace，然后给接口传入CLONE_NEWNET参数，即可创建一个network namespace。其他ns同样也可以使用我们熟悉的c语言代码调用系统API创
建对应的namespace，同时更加友好的是，Linux系统已经network namespace的增删改查功能集成到了Linux的ip工具里，就是ip命令的netns子命令中，大大
节省了我们的学习体验门槛。

接下来，我们创建一个名字为test的network namespace，使用如下命令：

```
    ip netns add test
```

当我们用ip命令创建了一个test 的network namespace时，系统就会在/var/run/netns路径下生成对应的挂载点。如下：

```
    [root@test_10_1_36_206 ~]# ls /var/run/netns/
    test
    [root@test_10_1_36_206 ~]# 
    
    [root@test_10_1_36_206 ~]# mount|grep netns
    tmpfs on /run/netns type tmpfs (rw,nosuid,nodev,mode=755)
    proc on /run/netns/test type proc (rw,relatime)
    proc on /run/netns/test type proc (rw,relatime)
```

这是干啥呢？
linux系统生成这个挂载点的作用一方面是方便对namespace管理，另一方面是使ns即使没有进程运行也能继续存在。

一个network namespace被创建出来了，怎么进去查看呢？

我们使用ip netns exec命令进入，做一些简单的网络查询或者配置工作。

```
    [root@test_10_1_36_206 ~]# ip netns exec test ip a
    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
如上所示，我们使用命令进入了这个network namespace查询网络信息。因为目前我们没有任何配置，所以只有一块系统默认的本地回环设备lo，而且还是down
状态，这个需要手动UP起来。

如果你想看系统中有那些network namespace，可以使用以下命令：

```
    [root@test_10_11_36_206 ~]# ip netns
    test
```

如果你想删除network namespace，可以这样做：

```
    [root@test_10_1_36_206 ~]# ip netns delete test
    [root@test_10_1_36_206 ~]# 
```

这里要注意的是，上边的命令实际上并没有删除test这个network namespace，它这只是移除了这个network namespace的挂载点，如下：

```
    [root@test_10_1_36_206 ~]# mount|grep netns
    tmpfs on /run/netns type tmpfs (rw,nosuid,nodev,mode=755)
    看不到test的挂载点了。
```

只要network namespace里还有进程运行着，network namespace便会一直存在。

