## docker popular images usage

#### 1. gitlab

启动:

```shell
$ docker pull gitlab/gitlab-ce:10.7.3-ce.0
$ mkdir /data/gitlab
$ docker run -d -p 443:443 -p 80:80 -p 222:22 --name gitlab --restart always -v /data/gitlab/config:/etc/gitlab -v /data/gitlab/logs:/var/log/gitlab -v /data/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce:10.7.3-ce.0
```



配置:

按上面的方式，gitlab容器运行没问题，但是生成的项目URL访问地址默认是容器的hostname，也就是容器的id。我们需要一个固定的URL访问地址，于是需要配置gitlab.rb

```shell
# 配置http协议所使用的访问地址,不加端口号默认为80
external_url 'http://192.168.199.231'

# 配置ssh协议所使用的访问地址和端口
gitlab_rails['gitlab_ssh_host'] = '192.168.199.231'
gitlab_rails['gitlab_shell_ssh_port'] = 222 # 此端口是run时22端口映射的222端口
```



gitlab的备份和恢复:

参考文档:       https://blog.csdn.net/foupwang/article/details/94362292



#### 2. docker-dnsmasq

https://blog.csdn.net/yanghua1012/article/details/80555487

https://blog.csdn.net/qq_34092609/article/details/87184163



#### 3. harbor

官方文档:    https://goharbor.io/docs/1.10/install-config/configure-https/

https://www.cnblogs.com/cjwnb/p/13441071.html



> unknown blob错误

https://www.rootop.org/pages/4755.html







存储驱动:    http://dockone.io/article/1513





#### 4. nexus

nexus仓库的迁移:

- 应用的目录结构

![nexus](pics\nexus.png)

conf/nexus.properties配置文件主要指定了sonatype-work目录的位置，容器中我们不需要对其进行修改，直接对sonatype-work目录进行持久化即可(storage目录默认在该目录下面)



- 备份和还原数据

将旧的nexus仓库的sonatype-work目录打包，还原到本地作为nexus容器持久化卷所在的位置 /data/nexus-data



- 容器启动

```shell
$ mkdir /data/nexus-data  && cd /data 
$ chown -R 200 nexus-data    # 容器应用用户id为200
$ docker run -d -p 8081:8081 --name nexus -v /etc/localtime:/etc/localtime:ro -v /data/nexus-data:/sonatype-work sonatype/nexus:2.14.5-02
```







https://tldp.org/LDP/abs/html/bashver2.html#EX78   间接引用