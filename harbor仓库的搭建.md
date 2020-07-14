## 搭建一个容器化的Harbor仓库

**参考文档： https://www.cnblogs.com/already/p/11678360.html**



#### 1. 创建harbor数据存储挂载点

```shell
[root@VM_0_12_centos ~]# lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0   50G  0 disk 
└─vda1 253:1    0   50G  0 part /
vdb    253:16   0  200G  0 disk 
[root@VM_0_12_centos ~]# pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
[root@VM_0_12_centos ~]# vgcreate docker_vg /dev/vdb
  Volume group "docker_vg" successfully created
[root@VM_0_12_centos ~]# 
[root@VM_0_12_centos ~]# lvcreate -L 100G -n dockerlv docker_vg 
  Logical volume "dockerlv" created.
[root@VM_0_12_centos ~]# mkfs.ext4 /dev/docker_vg/dockerlv 
```



#### 2. 部署harbor

- 下载harnor安装包v1.9.1

```shell
# wget https://storage.googleapis.com/harbor-releases/release-1.9.0/harbor-offline-installer-v1.9.1.tgz
```

> 官方网址下载：https://goharbor.io/



- Python pip 安装和docker-compose安装

```
# curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py   # 下载安装脚本
# python get-pip.py    # 运行安装脚本
# pip --version
pip 19.3.1 from /usr/lib/python2.7/site-packages/pip (python 2.7)
# pip --default-timeout=100 install docker-compose==1.18.0  \\先检查harbor需要的版本
docker-compose version 1.18.0, build 8dd22a9
```

> 1. 用哪个版本的 Python 运行安装脚本，pip 就被关联到哪个版本 
> 2. 将harbor安装包解压后，进去查看install.sh ,得到需要安装的docker-compose版本



- 部署harbor

**注意：当前的部署版本为v1.9.1，其他版本的配置文件可能有些许差异**

**修改harbor.cfg配置文件:**

**<harbor目录下harbor.yml参数描述： https://blog.csdn.net/zhaosongbin/article/details/90486034\>**

> - **hostname: abc.qhgctech.com**
> - **harbor_admin_password: 123456**    \\\The initial password of Harbor admin
> - **database:**
>
> ​             **password:  root1234**
>
> - **data_volume: /data/tools/harbor**    \\\The default data volume
> - 等其他可配置项

**/data/tools/harbor是单独定义的harbor数据存放目录**



```shell
[root@continers1 harbor]# ./prepare
[root@continers1 harbor]# ls /data/tools/harbor/
ca_download  database  job_logs  lost+found  psc  redis  registry  secret
[root@continers1 harbor]# ./install.sh    \\根据docker-compose.yml创建service
```

> 在v1.9.1版本下的docker-compose.yml是根据harbor.cfg生成的，所以一般不用做更改

```shell
[root@continers1 harbor]# docker-compose ps
```



#### 3. 遇到的问题

- docker push "server gave HTTP response to HTTPS client"问题

> 在/etc/docker下，修改daemon.json文件，写入： 
>
> ```{"insecure-registries":["172.20.120.211:32002"]}```
>
> 重载或重启docker
>
> ```systemctl reload docker.service``` 
>
> 或者 ```systemctl restart docker.service```



