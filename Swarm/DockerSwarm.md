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
> **在这个namespace中，有一个vxlan出口。docker overlsy就是通过overlay隧道与其它容器通信的。**
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



