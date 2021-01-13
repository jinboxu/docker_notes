## docker安装

#### 1. 安装

**参考文档： https://www.runoob.com/docker/centos-docker-install.html**

**设置仓库**

```shell
# 安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2
[root@containers ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
# 使用以下命令来设置稳定的仓库
[root@containers ~]# yum-config-manager --add-repo \ https://download.docker.com/linux/centos/docker-ce.repo
```



**要安装特定版本的 Docker Engine-Community，请在存储库中列出可用版本，然后选择并安装：**

```shell
# 列出并排序您存储库中可用的版本。此示例按版本号（从高到低）对结果进行排序。
[root@containers ~]#  yum list docker-ce --showduplicates | sort -r   \\使用--showduplicates参数列出所有版本

[root@containers ~]# yum install docker-ce-18.06.3.ce-3.el7 
```

> 通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1 
>
> ```yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io```



**Docker修改默认存储路径**

**参考文档： https://www.cnblogs.com/yangww/p/11334895.html**

```shell
# 我当前的系统版本为centos7
# systemctl status docker  \\找到systemd管理下的docker.service
...
ExecStart=/usr/bin/dockerd --graph /data/tools/docker   
# systemctl daemon-reload
# systemctl  restart docker.service  \\请先关闭容器
```

> /data/tools/docker 是单独为docker存储创建的挂载点 



**docker启动文件系统无法创建**

**参考文档： https://www.cnblogs.com/loopsun/p/9650301.html**

- 查看日志:  "Error while creating filesystem xfs on device docker-253:4-17133027-base: exit status 1  storage-driver=devicemapper"
- 解决办法: ```yum install xfsprogs```



**普通用户使用docker**

将普通用户添加到docker组并重启docker服务:

```shell
# groupadd docker
# usermod -aG docker {USER}
# systemctl restart docker
```

> 也可不必重启docker服务，手动附加权限即可。重启docker服务将重新生成的/var/run/docker.sock的所属组改为了docker



#### 2. 二进制安装

```shell
# tar zxf docker-18.06.3-ce.tag         //将安装包传到服务器上并解压
# cd docker  &&  cp docker*  /usr/bin   //将解压的包复制到程序启动默认的环境变量路径下

## 设置docker参数
# mkdir /etc/docker
# cat daemon.json   //创建daemon.json文件，内容如下
{
 "log-driver":"json-file",
 "log-opts": {
  "max-size":"20m",
  "max-file": "3"
 },
 "storage-driver": "overlay2"
}

## 开启数据包转发
# cat /etc/sysctl.d/docker.conf  //新建docker.conf文件，内容如下（具体原因网上查阅）
## for docker 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
# sysctl -p /etc/sysctl.d/docker.conf   //加载
//Linux系统默认是禁止数据包转发的。所谓转发即当主机拥有多于一块的网卡时，其中一块收到数据包，根据数据包的目的ip地址将数据包发往本机另一块网卡，该网卡根据路由表继续发送数据包。这通常是路由器所要实现的功能.//

## 使用system管理docker
## 启动docker之前，我们先单独为docker数据创建一个存储卷。docker默认的存储路径为/var/lib/docker目录下面，由于我们系统盘过小，所以单独将docker数据另外存储。
# 略，创建数据卷
# cat /usr/lib/systemd/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd --graph /data/docker    //将/data/docker目录作为docker的数据目录
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
# systemctl daemon-reload
# systemctl restart docker
# systemctl enable  docker
```

> docker.service文件可参考yum安装生成的



