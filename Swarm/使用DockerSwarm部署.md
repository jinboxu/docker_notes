## 使用DockerSwarm部署

**官方文档:**

Docker Swarm:   https://docs.docker.com/engine/swarm/

> Docker Stack文件:    http://c.biancheng.net/view/3211.html

Compose文件:    https://docs.docker.com/compose/compose-file/#configs

​                             https://docs.docker.com/compose/compose-file/compose-file-v3/       新



以下stack file不一定是最终的资源文件，实际使用的资源清单我已经集中放在仓库xxx



#### 1. zabbix

dockerhub:       https://hub.docker.com/r/zabbix/zabbix-server-mysql/

docker-compose:  https://cloud.tencent.com/developer/article/1612292



###### stack  file

```yaml
version: '3.7'

services:
  zabbix-server-mysql:
    image: zabbix/zabbix-server-mysql:centos-5.2.1
    networks:
      - zabbix
    environment:
      - DB_SERVER_HOST=192.168.0.158
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=123456
      - MYSQL_DATABASE=zabbix
      - ZBX_CACHESIZE=1024M
    ports:
      - 10051:10051
    volumes:
      - /etc/localtime:/etc/localtime
      - type: volume
        source: zabbix-server-data
        target: /usr/lib/zabbix
    deploy:
     resources:
      limits:
        cpus: '0.70'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M 

  zabbix-web-nginx-mysql:
    image: zabbix/zabbix-web-nginx-mysql:centos-5.2.1
    networks:
      - zabbix
    environment:
      - DB_SERVER_HOST=192.168.0.158
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=123456
      - MYSQL_DATABASE=zabbix
      - ZBX_SERVER_HOST=zabbix-server-mysql
      - PHP_TZ=Asia/Shanghai
    ports:
      - 8080:8080
    volumes:
      - /etc/localtime:/etc/localtime
    deploy:
      replicas: 2

volumes:
  zabbix-server-data:
    driver_opts:
      type: "nfs"
      o: "addr=192.168.0.230,rw"
      device: ":/data/nfs-data/zabbix-server-data" 


networks:
  zabbix:
    driver: overlay
    attachable: true
```

> Linux 上默认的 Bridge 网络是不支持通过 Docker DNS 服务进行域名解析的。自定义桥接网络可以！



```
$ docker volume create --driver local --opt type=nfs --opt o=addr=192.168.0.230,rw --opt device=:/data/zabbix-server-data  zabbix-server-data
```



###### 问题须知

- 变量传入不带引号

```yaml
    environment:
      - DB_SERVER_HOST=192.168.0.158
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=123456
      - MYSQL_DATABASE=zabbix
      - ZBX_SERVER_HOST=zabbix-server-mysql
      - PHP_TZ=Asia/Shanghai
```

> PHP_TZ="Asia/Shanghai" 不生效



- 字符集报错

```mysql
## 查看库字符集
mysql> select * from information_schema.SCHEMATA;

## 解决办法,重新建库并定义字符集（不使用默认字符集）
mysql> drop database zabbix; 
mysql> create database zabbix charset utf8 collate utf8_bin; 
```

> 参考文档:  https://blog.csdn.net/qq_40722582/article/details/109567170





**需要注意的地方：**

- volume是无法更新的
- config配置是不可变的，所以无法更改现有服务的文件，可以创建一个新的 Config 来使用不同的文件，或者删除服务和配置(docker  service rm  {serviceName}  &&  docker config  rm  {dockerConfig})再更新



#### 2. 工单系统

```yaml
version: '3.7'

services:
  eureka-0:
    image: docker-hub.qhgctech.com/work-order-dev/eureka-server:b6a6ea8
    networks:
      - workorder
    hostname: "eureka-0"
    environment:
      - EUREKA_POD_NAME=eureka-0
      #- EUREKA_IN_SERVICE_NAME=eureka-server
      #- EUREKA_POD_NAMESPACE=workorder
    env_file:
      - ./eureka.env
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=2.5"]
    deploy:
     resources:
      limits:
        cpus: '2.5'
        memory: 2G
      reservations:
        cpus: '1'
        memory: 512M  
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8100"]
      start_period: 20s
      interval: 10s
      timeout: 2s
      retries: 2 

  eureka-1:
    image: docker-hub.qhgctech.com/work-order-dev/eureka-server:b6a6ea8
    networks:
      - workorder
    hostname: "eureka-1"
    environment:
      - EUREKA_POD_NAME=eureka-1
    env_file:
      - ./eureka.env
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=2.5"]
    deploy:
     resources:
      limits:
        cpus: '2.5'
        memory: 2G
      reservations:
        cpus: '1'
        memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8100"]
      start_period: 20s
      interval: 10s
      timeout: 2s
      retries: 2 

  eureka-2:
    image: docker-hub.qhgctech.com/work-order-dev/eureka-server:b6a6ea8
    networks:
      - workorder
    hostname: "eureka-2"
    environment:
      - EUREKA_POD_NAME=eureka-2
    env_file:
      - ./eureka.env
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=2.5"]
    deploy:
     resources:
      limits:
        cpus: '2.5'
        memory: 2G
      reservations:
        cpus: '1'
        memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8100"]
      start_period: 20s
      interval: 10s
      timeout: 2s
      retries: 2 

  gateway-server:
    image: docker-hub.qhgctech.com/work-order-dev/gateway-server:b6a6ea8
    networks:
      - workorder
    environment:
      - TZ=Asia/Shanghai
      - LOG_PATH=/log
    env_file:
      - ./app.env
    extra_hosts:
      - "mysql:192.168.10.4"
      - "sentinel1:192.168.10.3"
      - "sentinel2:192.168.1.4"
      - "sentinel3:192.168.1.12"
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=1"]
    deploy:
     replicas: 2
     resources:
      limits:
        cpus: '1'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M

  word-server:
    image: docker-hub.qhgctech.com/work-order-dev/word-server:afa1f38
    networks:
      - workorder
    environment:
      - TZ=Asia/Shanghai
      - LOG_PATH=/log
    env_file:
      - ./app.env
    extra_hosts:
      - "mysql:192.168.10.4"
      - "sentinel1:192.168.10.3"
      - "sentinel2:192.168.1.4"
      - "sentinel3:192.168.1.12"
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=1"]
    volumes:
      - type: volume
        source: volume-images
        target: /image/workserver/upload
      - type: volume
        source: volume-userinfo
        target: /sharedir/HZFILE
    deploy:
     replicas: 2
     placement:
       constraints:
         - node.labels.external-network != true
     resources:
      limits:
        cpus: '1'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M

  user-server:
    image: docker-hub.qhgctech.com/work-order-dev/user-server:b6a6ea8
    networks:
      - workorder
    environment:
      - TZ=Asia/Shanghai
      - LOG_PATH=/log
    env_file:
      - ./app.env
    extra_hosts:
      - "mysql:192.168.10.4"
      - "sentinel1:192.168.10.3"
      - "sentinel2:192.168.1.4"
      - "sentinel3:192.168.1.12"
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=1"]
    deploy:
     replicas: 2
     placement:
       constraints:
         - node.labels.external-network == true
     resources:
      limits:
        cpus: '1'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M

  queuing-server:
    image: docker-hub.qhgctech.com/work-order-dev/queuing-server:b6a6ea8
    networks:
      - workorder
    environment:
      - TZ=Asia/Shanghai
      - LOG_PATH=/log
    env_file:
      - ./app.env
    extra_hosts:
      - "mysql:192.168.10.4"
      - "sentinel1:192.168.10.3"
      - "sentinel2:192.168.1.4"
      - "sentinel3:192.168.1.12"
    command: ["-XX:+UnlockExperimentalVMOptions","-XX:+UseCGroupMemoryLimitForHeap","-XX:ActiveProcessorCount=1"]
    deploy:
     replicas: 2
     resources:
      limits:
        cpus: '1'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M     

  word-admin:
    image: docker-hub.qhgctech.com/work-order-dev/word-admin:e35621b
    networks:
      - workorder
    configs:
      - source: admin_conf
        target: /etc/nginx/conf.d/default.conf
    ports:
      - 8080:80
    volumes:
      - type: volume
        source: volume-images
        target: /image/workserver/upload
    deploy:
     replicas: 2
     placement:
       constraints:
         - node.labels.external-network != true
     resources:
      limits:
        cpus: '2'
        memory: 2G
      reservations:
        cpus: '0.5'
        memory: 512M

volumes:
  volume-images:
    driver_opts:
      type: "nfs"
      o: "addr=192.168.10.3,rw"
      device: ":/nfs_data/workorder-uat-images"
  volume-userinfo:
    driver_opts:
      type: "nfs"
      o: "addr=192.168.10.3,rw"
      device: ":/nfs_data/workorder-uat-userinfo"

configs:
  admin_conf:
    file: ./admin.conf

networks:
  workorder:
    driver: overlay
    attachable: true
```

> eureka服务是有状态的
>
> **eureka服务在stackfile中定义了3个：**
>
> eureka服务是有状态的服务，而swarm不支持有状态的多副本，所以对这个服务的脚本做了一些处理，让它既能运行在k8s又能运行在swarm下面了。
>
> **eureka变量文件定义了ENV_MODE变量:**
>
> ```shell
> $ cat eureka.env
> ENV_MODE=swarm      #这个变量必须有，不然eureka服务起不来
> EUREKA_REPLICAS=3
> EUREKA_APPLICATION_NAME=eureka
> BOOL_REGISTER=true
> BOOL_FETCH=true
> SELF_PRESERVATION=true
> EVICTION_INTERVAL_TIMER=30000
> TZ=Asia/Shanghai
> LOG_PATH=/log
> ```
>
> > 脚本通过判断ENV_MODE环境变量是否存在，进而影响eureka服务直接的相互注册





#### 3. elasticsearch和kibana

参考手册:

```
https://www.elastic.co/guide/en/elasticsearch/reference/index.html
https://www.elastic.co/guide/en/kibana/current/index.html
```



###### 3.1 准备工作

1. The `vm.max_map_count` kernel setting needs to be set to at least `262144` for production use. 

```
$ grep vm.max_map_count /etc/sysctl.conf
vm.max_map_count=262144
```

2. 修改各个节点的ulimits(注意，swarm集群方式下不能直接对ulimit进行修改)
3. 创建elasticsearch运行节点的目录并给予读写权限

```shell
$ cd /data
$ mkdir esdatadir
$ cd esdatadir && mkdir data logs
$ chmod 777 data logs   # 容器数据和日志目录
```



###### 3.2 elasticsearch、kibana以及nginx代理

```yaml
# stack文件清单:
elasticsearch-kibana
- ek-stack.yaml
- kibana_proxy.conf
- password_file
- readme.md
```



ek-stack.yaml :

```yaml
version: '3.7'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.13
    hostname: es01
    environment:
      - node.name=es01
      - cluster.name=es-swarm-cluster
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      #- bootstrap.memory_lock=true
      - bootstrap.system_call_filter=false
      - network.host=192.168.10.3
      - discovery.zen.minimum_master_nodes=2
      - discovery.zen.ping.unicast.hosts=es01,es02,es03
    extra_hosts:
      - "es01:192.168.10.3"
      - "es02:192.168.1.4"
      - "es03:192.168.1.12"
    volumes:
      - /data/esdatadir/data:/usr/share/elasticsearch/data
      - /data/esdatadir/logs:/usr/share/elasticsearch/logs
    networks:
      hostnet: {}
    deploy:
      placement:
        constraints:
          - "node.hostname==manager1"
      resources:
       limits:
         cpus: '4'
         memory: 6G
       reservations:
         cpus: '2'
         memory: 4G

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.13
    hostname: es02
    environment:
      - node.name=es02
      - cluster.name=es-swarm-cluster
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      #- bootstrap.memory_lock=true
      - bootstrap.system_call_filter=false
      - network.host=192.168.1.4
      - discovery.zen.minimum_master_nodes=2
      - discovery.zen.ping.unicast.hosts=es01,es02,es03
    extra_hosts:
      - "es01:192.168.10.3"
      - "es02:192.168.1.4"
      - "es03:192.168.1.12"
    volumes:
      - /data/esdatadir/data:/usr/share/elasticsearch/data
      - /data/esdatadir/logs:/usr/share/elasticsearch/logs
    networks:
      hostnet: {}
    deploy:
      placement:
        constraints:
          - "node.hostname==work1"
      resources:
       limits:
         cpus: '4'
         memory: 6G
       reservations:
         cpus: '2'
         memory: 4G

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.13
    hostname: es03
    environment:
      - node.name=es03
      - cluster.name=es-swarm-cluster
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      #- bootstrap.memory_lock=true
      - bootstrap.system_call_filter=false
      - network.host=192.168.1.12
      - discovery.zen.minimum_master_nodes=2
      - discovery.zen.ping.unicast.hosts=es01,es02,es03
    extra_hosts:
      - "es01:192.168.10.3"
      - "es02:192.168.1.4"
      - "es03:192.168.1.12"
    volumes:
      - /data/esdatadir/data:/usr/share/elasticsearch/data
      - /data/esdatadir/logs:/usr/share/elasticsearch/logs
    networks:
      hostnet: {}
    deploy:
      placement:
        constraints:
          - "node.hostname==work2"
      resources:
       limits:
         cpus: '4'
         memory: 6G
       reservations:
         cpus: '2'
         memory: 4G

  kibana:
    image: docker.elastic.co/kibana/kibana:6.8.13
    networks:
      - kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://192.168.1.4:9200
    #ports:
    #  - 5601:5601
    deploy:
      resources:
       limits:
         cpus: '1'
         memory: 1G
       reservations:
         cpus: '0.2'
         memory: 200m


  nginx:
    image: nginx:1.18.0
    networks:
      - kibana
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
      - source: nginx_passwd
        target: /usr/local/src/nginx/passwd
    ports:
      - 8081:80

configs:
  nginx_conf:
    file: ./kibana_proxy.conf
  nginx_passwd:
    file: ./password_file

networks:
  hostnet:
    external: true
    name: host
  kibana:
    driver: overlay
    attachable: true
```

> dockerfiles:   https://github.com/elastic/dockerfiles/tree/v6.8.13



kibana_proxy.conf :

```
server {
    listen 80;
    server_name kibana-swarm.qhgctech.com;

    auth_basic "Please input password";
    auth_basic_user_file /usr/local/src/nginx/passwd;

    location / {
        proxy_pass http://kibana:5601;
    }

}
```



通过openssl创建nginx的用户密码文件:

````shell
$ echo -n 'kibana-swarm:' > password_file
$ openssl passwd 'abc123456' >> password_file
````



创建或更新服务:

```shell
$ docker stack deploy -c ek-stack.yaml elasticsearch-kibana
```



使用注意:

- 通过环境变量的设置并没有更新容器的配置文件
- elasticsearch的JVM heap内存在容器里面看到的总是1G，但实际是生效的





#### 4. prometheus in swarm

```yaml
version: '3.7'

services:
  cadvisor:
    image: google/cadvisor
    #labels:
    #  prometheus-job: 'cadvisor'
    networks:
      - net
    command: -logtostderr -docker_only
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /:/rootfs:ro
      - /var/run:/var/run
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    deploy:
      mode: global
      labels:
        prometheus-job: 'cadvisor'
        prometheus.io/port: '8080'
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  grafana:
    #image: stefanprodan/swarmprom-grafana:5.3.4
    image: docker-hub.qhgctech.com/basic/grafana:27a2e8121b
    networks:
      - net
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
      #- GF_SERVER_ROOT_URL=${GF_SERVER_ROOT_URL:-localhost}
      #- GF_SMTP_ENABLED=${GF_SMTP_ENABLED:-false}
      #- GF_SMTP_FROM_ADDRESS=${GF_SMTP_FROM_ADDRESS:-grafana@test.com}
      #- GF_SMTP_FROM_NAME=${GF_SMTP_FROM_NAME:-Grafana}
      #- GF_SMTP_HOST=${GF_SMTP_HOST:-smtp:25}
      #- GF_SMTP_USER=${GF_SMTP_USER}
      #- GF_SMTP_PASSWORD=${GF_SMTP_PASSWORD}
    volumes:
      - grafana:/var/lib/grafana
    deploy:
      mode: replicated
      replicas: 1
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  alertmanager:
    image: prom/alertmanager:v0.21.0
    #image: prom/alertmanager:v0.16.2
    networks:
      - net
    #environment:
    #  - SLACK_URL=${SLACK_URL:-https://hooks.slack.com/services/TOKEN}
    #  - SLACK_CHANNEL=${SLACK_CHANNEL:-general}
    #  - SLACK_USER=${SLACK_USER:-alertmanager}
    configs:
      - source: alertmanager_config
        #target: /tmp/alertmanager.yml
        target: /etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    volumes:
      - alertmanager:/alertmanager
    deploy:
      mode: replicated
      replicas: 1
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  #unsee:
  #  image: cloudflare/unsee:v0.9.2
  #  networks:
  #    - net
  #  environment:
  #    ALERTMANAGER_URIS: 'default:http://alertmanager:9093'
  #    #- "ALERTMANAGER_URIS=default:http://alertmanager:9093"
  #  deploy:
  #    mode: replicated
  #    replicas: 1

  svc-alertmanager-wechatbot-webhook:
    image: registry.qhgctech.com/basic/alertmanager-wechatbot-webhook
    networks:
      - net
    configs:
      - source: wechatbot-webhook-template
        target: /conf/wechat.tmpl
    environment:
      -  template_path=/conf/wechat.tmpl
    deploy:
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 256M


  # 使用新版的prometheus以支持swarm服务发现,同时将需要经常改变的配置文件(prometheus.yml和*.rule)定义到持久卷
  prometheus:
    #image: stefanprodan/swarmprom-prometheus:v2.5.0
    image: prom/prometheus:v2.25.0
    networks:
      - net
    command:
      #- '--config.file=/etc/prometheus/prometheus.yml'
      - '--config.file=/prometheus/conf/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention=${PROMETHEUS_RETENTION:-24h}'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    volumes:
      #- prometheus:/prometheus
      - /data/promdatadir/promtheusdata-and-conf:/prometheus
      - /var/run/docker.sock:/var/run/docker.sock
    #configs:
    #  - source: node_rules
    #    target: /etc/prometheus/swarm_node.rules.yml
    #  - source: task_rules
    #    target: /etc/prometheus/swarm_task.rules.yml
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 2048M
        reservations:
          memory: 128M


  nginx:
    image: nginx:1.18.0
    networks:
      - net
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
      - source: nginx_passwd
        target: /usr/local/src/nginx/passwd
    ports:
      - 8083:80


networks:
  net:
    driver: overlay
    attachable: true

volumes:
    grafana:
      driver_opts:
        type: "nfs"
        o: "addr=192.168.10.3,rw"
        device: ":/nfs_data/monitoring-grafana"

    alertmanager:
      driver_opts:
        type: "nfs"
        o: "addr=192.168.10.3,rw"
        device: ":/nfs_data/monitoring-alertmanager"


configs:
  alertmanager_config:
    file: ./alertmanager/alertmanager.yml
  wechatbot-webhook-template:
    file: ./webhook/wechat.tmpl
  nginx_conf:
    file: ./nginx/proxy.conf
  nginx_passwd:
    file: ./nginx/password_file
```

> - grafana可以单独放在一个stack下，通过连接各个overlay网络来使用
> - prometheus比较新的版本才支持swarm的服务发现`dockerswarm_sd_config`，同时将其配置文件放在映射的宿主机卷下
> - (通过服务标签采集我在另一份文档中解释)
> - unsee不支持v2版本的prometheus接口，所以没有使用



```cat nginx/proxy.conf```:

```
server {
    listen 80;
    server_name 0.0.0.0;

    auth_basic "Please input password";
    auth_basic_user_file /usr/local/src/nginx/passwd;

    location / {
        resolver 127.0.0.11 valid=60s;
        set $prometheus http://prometheus:9090;
        set $alertmanager http://alertmanager:9093;

        if ( $host = 'swarm-monitoring-prometheus.qhgctech.com' ) {
            proxy_pass $prometheus;
        }

        if ( $host = 'swarm-monitoring-alertmanager.qhgctech.com' ) {
            proxy_pass $alertmanager;
        }


    }
}


server {
    listen 80;
    server_name swarm-monitoring-grafana.qhgctech.com;

    location / {
        resolver 127.0.0.11 valid=60s;
        set $grafana http://grafana:3000;

        proxy_pass $grafana;
    }
}
```

- 通过不同的host路由，因为grafana自带有认证机制，所以将其单独列出到一个service块
- nginx容器做反向代理的问题已经在 '问题记录' 文档描述







