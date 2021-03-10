## DockerSwarm

官方文档:    https://docs.docker.com/engine/swarm/swarm-tutorial



#### 命令补全

- docker命令自动补全:

```
# yum -y install bash-completion
# source /usr/share/bash-completion/bash_completion
```



- 二进制安装的docker命令补全

通过yum安装相同版本的docker。将 /usr/share/bash-completion/completions/docker 文件拷贝到二进制安装的docker服务器上的 /usr/share/bash-completion/completions/ 目录下



加载生效

```shell
# source /usr/share/bash-completion/completions/docker   #bash-completion已安装
```

或

```shell
# source /usr/share/bash-completion/bash_completion
```





#### 报错

docker 启动或创建容器是报 Failed to Setup IP tables: Unable to enable SKIP DNAT rule 错误的的解决办法
1，原因：在对 linux 的防火墙进行操作之后，需要重启 docker
2，解决办法：service docker restart



#### nsenter命令

nsenter命令是一个可以在指定进程的命令空间下运行指定程序的命令。它位于util-linux包中。

参考文档:     https://www.cnblogs.com/Wshile/p/12596617.html



namespace是Linux中一些进程的属性的作用域，使用命名空间，可以隔离不同的进程。

Linux在不断的添加命名空间，目前有：

- mount：挂载命名空间，使进程有一个独立的挂载文件系统，始于Linux 2.4.19
- ipc：ipc命名空间，使进程有一个独立的ipc，包括消息队列，共享内存和信号量，始于Linux 2.6.19
- uts：uts命名空间，使进程有一个独立的hostname和domainname，始于Linux 2.6.19
- net：network命令空间，使进程有一个独立的网络栈，始于Linux 2.6.24
- pid：pid命名空间，使进程有一个独立的pid空间，始于Linux 2.6.24
- user：user命名空间，是进程有一个独立的user空间，始于Linux 2.6.23，结束于Linux 3.8
- cgroup：cgroup命名空间，使进程有一个独立的cgroup控制组，始于Linux 4.6

Linux的每个进程都具有命名空间，可以在/proc/PID/ns目录中看到命名空间的文件描述符。



通过nsenter查看容器相关命名空间:

```shell
## 1. 查看网络命名空间的信息
[root@work1 ~]# docker inspect 1c9f | grep Sandbox
            "SandboxID": "7bc9e0f961e033a22e71cfb699b5df6a05c6269677d4a1df680215820cb76f4d",
            "SandboxKey": "/var/run/docker/netns/7bc9e0f961e0",
[root@work1 ~]# nsenter --net=/var/run/docker/netns/7bc9e0f961e0 ip a
...略

## 2. 查看进程相关的命名空间信息
[root@work1 ~]# docker inspect 1c9f | grep Pid
            "Pid": 25536,
            "PidMode": "",
            "PidsLimit": 0,
[root@work1 ~]# nsenter -t 25536 --uts --ipc --net --pid 
[root@1c9f6b7182dd ~]# ip a
...略
```





#### ip netns命令

ip netns 命令是用来管理 网络命名空间 的，网络命名空间可以实现 网络隔离。每个网络命名空间都提供了一个完全独立的网络协议栈，包括网络设备接口、IPV4 和 IPV6 协议栈、IP路由表、防火墙规则、端口、sockets 等。像 docker 就是利用 Linux 的网络命名空间来实现容器网络的隔离。


当 docker 容器被创建出来后，你会发现使用 ip netns 命令无法看到容器对应的网络命名空间。这是因为 ip netns 命令是从 /var/run/netns 文件夹中读取内容的

```shell
## 1. 映射docker容器的网络命名空间
ln -s /var/run/docker/netns /var/run/netns

## 2.映射一个容器的网络命名空间
docker inspect 1c9f | grep Pid               //获取容器进程ip，在容器命名空间中进程ID为1
ln -s /proc/25536/ns/net /var/run/netns/web
```



通过ip netns进入网络命名空间查看:

```shell
[root@work1 ~]# ls /var/run/netns/   //web是一个目录软链接
1-czfcsmphml  1-stp12d6o10  6f4219f474bb  7bc9e0f961e0  9e86cce7038c  default  f20587e335c0  ingress_sbox  web
[root@work1 ~]# ip netns ls     //可以看到有两个id是一样的
web (id: 5)
7bc9e0f961e0 (id: 5)
9e86cce7038c (id: 6)
6f4219f474bb (id: 4)
1-stp12d6o10 (id: 2)
f20587e335c0 (id: 3)
1-czfcsmphml (id: 0)
ingress_sbox (id: 1)
default
[root@work1 ~]# 
```

> 分别通过```ip netns exec web bash```和```ip netns exec 7bc9e0f961e0 bash```进入查看到他们的网络命名空间是一样的





#### swarm网络

参考文档:   https://www.jianshu.com/p/249fd33bd4fe

```
$ docker network create --driver=overlay --attachable cnblogs
```

> 以前Swarm集群网络是不允许容器直接加入网络中的，因为有可能会破坏集群网络结构。
>
> 参考文档:       https://www.cnblogs.com/zoujiaojiao/p/13361683.html



**网络结构分析**

```shell
$ docker network create --driver overlay --attachable overlay_test
$ docker run -d --name busybox2 --net overlay_test busybox sleep 36000

## 在另一个node中
$ docker run -d --name busybox3 --net overlay_test busybox sleep 36000
```

> 一下以busybox2为例



查看容器的网络信息:

```shell
[root@manager1 ~]# docker exec  busybox2 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
46: eth0@if47: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue 
    link/ether 02:42:0a:00:02:07 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.7/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
48: eth1@if49: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
[root@manager1 ~]# 
## 得到这个容器的网络命名空间ID，/var/run/docker/netns/下其中之一
[root@manager1 ~]# docker inspect -f "{{.NetworkSettings.SandboxKey}}" busybox2
/var/run/docker/netns/6692168374d3
[root@manager1 ~]# nsenter --net=/var/run/docker/netns/6692168374d3 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
46: eth0@if47: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:0a:00:02:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.2.7/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
48: eth1@if49: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
[root@manager1 ~]# 
```

> 网络命名空间查看的ip信息与进入容器查看的结果是一样的
>
> **查看容器的网络配置。10.0.2.7所在接口为46: eth0@if47，即本接口ifindex为46，连接到ifindex为47的接口上**



查看容器接口对应overlay网络的接口:

```shell
[root@manager1 ~]# docker network inspect overlay_test -f "{{.Id}}"
6ttr438f3yv6nesktirdujdw6
[root@manager1 ~]# ls /var/run/docker/netns/    # 注意overlay网络命名空间名称与容器的不同
1-6ttr438f3y  1-kk4l2wm29r  6692168374d3  d956fbf74dbb  ingress_sbox
[root@manager1 ~]# 
[root@manager1 ~]# 
[root@manager1 ~]# nsenter --net=/var/run/docker/netns/1-6ttr438f3y ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 3a:f6:ba:bb:eb:ff brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.1/24 brd 10.0.2.255 scope global br0
       valid_lft forever preferred_lft forever
43: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN 
    link/ether b2:36:a5:f6:88:d1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
45: veth0@if44: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP 
    link/ether 7a:21:cf:ee:91:ab brd ff:ff:ff:ff:ff:ff link-netnsid 1
47: veth1@if46: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP 
    link/ether 3a:f6:ba:bb:eb:ff brd ff:ff:ff:ff:ff:ff link-netnsid 2
[root@manager1 ~]# 
```

> 可见47接口处于1-6ttr438f3y这个namespace中(同样，另外一个容器busybox3在其所在的节点上也像这样查看)
>
> **在这个namespace中，有一个vxlan出口。docker overlay就是通过overlay隧道与其它容器通信的。**
>
> **两个容器虽然是通过vxlan隧道通信，但容器内部却不感知。它们只能看到两个容器处于同一个二层网络中。由vxlan接口将二层报文封装在UDP报文的payload中，发到对端，再由对端的vxlan接口解封装**



![overlay](pics\overlay.webp)





#### Docker Swarm中的LB

参考文档:    https://www.jianshu.com/p/c83a9173459f

![SwarmLB](pics\SwarmLB.webp)

> Swarm mode下，docker会创建一个默认的overlay网络—ingress network。Docker也会为每个worker节点创建一个特殊的net namespace（sandbox）-- ingress_sbox。
>
> ingress_sbox有两个endpoint，一个用于连接ingress network，另一个用于连接local bridge network docker_gwbridge。Ingress network的IP空间为10.255.0.0/16，所有router mesh的service都共用此空间。



Ingress  Load  Balancing实现方式:

![LB](pics\LB.jpg)

1. 宿主机网络通过worker节点IP和service published port来访问服务。

![LB1](pics\LB1.jpg)

> 每个节点iptables中NAT表定义规则，对于匹配published的宿主机端口的数据，将其dst IP转换成ingress_sbox中的ip：172.18.0.2。使数据包转发到ingress_sbox的ns中交给该ns处理做下一步的转发。



2. Ingress_sbox是swarm为每个节点默认创建的net namespace，用于连接ingress overlay network。此处会设置mangle表，将dst port（比如8080）的数据做标记(fwmark)。同时做DNAT转换成vip地址使数据包能正常转发到ingress的ns中，该vip由ingress_sbox的ipvs做负载转发。

![LB2](pics\LB2.jpg)

> 标签值为1551



3. Ingress_sbox会设置kernel中的LVS模块，将标记fwmark的包LB到各个实际IP中，默认round-robin算法，forware为VS/NAT方式。容器底层间通过overlay网络互连通信。

![LB3](pics\LB3.jpg)

> 宿主机上安装ipvsadm



4. 通过overlay网络，最终访问到真实的服务.

![LB4-1](pics\LB4-1.png)



![LB4-2](pics\LB4-2.png)





总结:

从上面可以看到一个请求到主机端口8080之后，数据包的流量如下所示=>

主机端口8080 => Ingress-sbox-VIP:8080 => 容器Ingress-sbox => IPVS分发到containers。

> 大家可以看到访问主机之后数据包流到了一个特殊的Sandbox容器里， 这个容器和我们的容器共享一个Ingress网络，通过Iptables和IPVS等重定向到了最终容器之上。 达到了服务在任何一台主机的8090端口都可达的目的。



#### 补充

上面通过部署一个service使用**默认的ingress  overlay网络**做的演示，如果这个服务有一个自己的overlay网络呢？



![SwarmLB2](pics\SwarmLB2.png)

> ingress流量和应用之间的流量分开



查看容器的网络命名空间:

![容器网络](pics\容器网络.jpg)