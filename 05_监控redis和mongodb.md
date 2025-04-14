# 监控redis和mongodb

### 一、环境介绍

#### 主机清单

| 职责            | ip地址           | 备注                            |
| ------------- | -------------- | ----------------------------- |
| Prometheus服务器 | 192.168.28.100 | docker模式的prometheus           |
| 待监控Linux      | 192.168.28.110 | 待准备组件：redis6版本、mongodb4.2.5版本 |

#### redis概述

Redis是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的Key-Value数据库，并提供多种语言的API

redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string、list(链表)、set(集合)和zset(有序集合)。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序，在部 分场合可以对关系数据库起到很好的补充作用

#### mongodb概述

MongoDB是一个基于分布式文件存储的数据库。由C++语言编写。旨在为WEB应用提供可扩展的高性能数据存储解决方案。

MongoDB是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似json的bson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是它支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引

#### 二、prometheus监控Redis   （在监控linux中有）

docker安装（略）

docker-compose安装(略)

#### 2.1 192.168.28.110待监控Linux安装Redis

创建docker-compose目录

```shell
mkdir /data/docker-compose -p
cd /data/docker-compose
```

创建docker-compose.yaml

```yaml
version: '3.3'

services:
  redis:
    image: redis:6
    container_name: redis
    restart: always
    volumes:
      - /data/redis/data:/data
    command: redis-server --maxmemory 512mb
    ports:
      - 6379:6379
```

运行docker-compose

```shell
docker-compose up -d
当你curl 127.0.0.1:6379
出现Empty reply from server 就说明redis已经启动起来了
染过出现failed connect，就说明redis没有启动起来
```

查看docker 的信息

```shell
docker images
docker ps -a
```

#### 2.2 192.168.28.110待监控Linux安装redis_exporter

二进制安装或者docker安装(或者docker-compose安装)，此处选用docker-compose安装

修改docker-compose.yaml文件（修改上面的文件，新增以下内容）

```shell
cd /data/docker-compose
vi docker-compose.yaml
```

yaml文件增加redis_exporter组件

```yaml
version: '3.3'

services:
  redis_exporter:
    image: oliver006/redis_exporter
    container_name: redis_exporter
    restart: always
    environment: 
      REDIS_ADDR: "192.168.28.110:6379"
      REDIS_PASSWORD: 
    ports:
      - 9121:9121
```

 ports:
      - 9121:9121是指在采集的数据在这个端口上
      - 

![1734506338667](C:\Users\xiaowang798abc\Desktop\Prometheus笔记\第2章 Prometheus基础-系统监控\Resources\1734506338667.png)

启动docker-compose.yaml

```shell
docker-compose up -d
```

访问redis_exporter

http://192.168.28.110:9121/metrics

#### 2.3 prometheus服务器添加redis_exporter的地址

192.168.28.100的centos1上，修改prometheus的配置文件

```shell
#进入docker-prometheus目录
cd /data/docker-prometheus

#修改prometheus.yml
vi prometheus/prometheus.yml
```

添加待监控的redis_exporter

```shell
  - job_name: 'redis-exporter'
    static_configs:
    - targets: ['192.168.28.110:9121']
      labels:
        instance: redis服务器 
```

保存配置后，让配置生效

```shell
#centos1中执行
curl -X POST http://localhost:9090/-/reload
```

刷新访问http://192.168.28.100:9090/targets?search=，确认新监控的redis服务器是否生效

![2-21](Resources\2-21.png)

#### 2.4 redis服务器指标查询

```shell
redis_up                                                服务器是否在线  1代表启动
redis_uptime_in_seconds                                    运行时长,单位s
rate(redis_cpu_sys_seconds_total[1m])+rate(redis_cpu_user_seconds_total[1m])        占用CPU核数
redis_memory_used_bytes/1024/1024                                    占用内存量  单位为M了
redis_memory_max_bytes                                    限制的最大内存,如果没限制则为0
delta(redis_net_input_bytes_total[1m])                     网络接受的bytes
delta(redis_net_output_bytes_total[1m])                     网络发送的bytes


redis_connected_clients                                    客户端连接数
redis_connected_clients / redis_config_maxclients        连接数使用率
redis_rejected_connections_total                        拒绝的客户端连接数
redis_connected_slaves                                    slave连接数
```

#### 2.5 grafana中对redis进行监控

copy id to clipboard->grafana的dashboards中Import dashboard

https://grafana.com/grafana/dashboards/11835-redis-dashboard-for-prometheus-redis-exporter-helm-stable-redis-ha/

![2-20](Resources\2-20.png)



![2-40](C:\Users\xiaowang798abc\Desktop\Prometheus笔记\第2章 Prometheus基础-系统监控\Resources\2-40.jpg)



![2-41](C:\Users\xiaowang798abc\Desktop\Prometheus笔记\第2章 Prometheus基础-系统监控\Resources\2-41.jpg)

![2-42](C:\Users\xiaowang798abc\Desktop\Prometheus笔记\第2章 Prometheus基础-系统监控\Resources\2-42.jpg)

#### 三、prometheus监控mongodb

docker安装（略）

docker-compose安装(略)

#### 3.1 待监控Linux安装mongodb

centos2，创建docker-compose目录

```shell
mkdir /data/docker-compose -p
cd /data/docker-compose
```

修改docker-compose.yaml，增加mongodb配置节

```yaml
version: '3.3'

services:
  #mongodb配置节
  mongo:
    image: mongo:4.2.5
    container_name: mongo
    restart: always
    volumes:
      - /data/mongo/db:/data/db
    command: [--auth]
    ports:
      - 27017:27017
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: 123456
```

运行docker-compose up -d的命令，观察mongodb的程序是否成功运行

#### 3.2 待监控Linux安装monogodb_exporter

##### 3.2.1 创建监控用户

centos2机器上，登录MongoDB创建监控用户,权限为"readAnyDatabase",如果是cluster环境,需要有"clusterMonitor"

登录MongoDB

```shell
docker exec -it mongo mongo admin
```

创建监控用户（输出1的话就说明输入成功）

```shell
> db.auth('root','123456')
1
>db.createUser({ user: 'exporter',pwd : 'password',roles:[{role: 'readAnyDatabase',db : 'admin'},{role: 'clusterMonitor',db : 'admin'}]});
#提示创建成功
#测试 使用上面创建的用户信息进行连接
    > db.auth('exporter','password')
1
#表示成功
> exit
```

##### 3.2.2 安装monogodb_exporter

centos2机器上，编辑/data/docker-compose/docker-compose.yaml文件，新增以下内容

```yaml
  mongodb_exporter:
    image: ccr.ccs.tencentyun.com/rig-agent/mongodb-exporter:0.10.0
    container_name: mongodb_exporter
    restart: always
    environment:
      MONGODB_URI: "mongodb://exporter:password@192.168.28.110:27017/admin?ssl=false"
    ports:
      - "9216:9216"
```

```
docker-compose up -d 重新拉取镜像
```

宿主机上访问mongodb_exporter http://192.168.28.110:9216

宿主机上访问mongodb_exporter的metrics

http://192.168.28.110:9216/metrics

#### 3.3 prometheus服务器添加monogodb_exporter的地址

192.168.28.100的centos上，修改prometheus的配置文件

```shell
#进入docker-prometheus目录
cd /data/docker-prometheus

#修改prometheus.yml
vi prometheus/prometheus.yml
```

添加待监控的monogodb_exporter

```shell
  - job_name: 'mongodb-exporter'
    static_configs:
    - targets: ['192.168.28.110:9216']
      labels:
        instance: mongodb服务器 
```

保存配置后，让配置生效

```shell
#centos1中执行
curl -X POST http://localhost:9090/-/reload
```

刷新访问http://192.168.28.100:9090/targets?search=，确认新监控的mongodb服务器是否生效

![2-23](Resources\2-23.png)

#### 3.4 mongodb服务器指标查询

```shell
mongodb_connections{state="available"} 可用的连接数
mongodb_connections{state="current"}   当前连接数 


#关于server status
mongodb_up    服务器是否在线
mongodb_instance_uptime_seconds  服务器的运行时长,单位为秒
```

#### 3.5 grafana中对mongodb进行监控

copy id to clipboard->grafana的dashboards中Import dashboard

https://grafana.com/grafana/dashboards/12079-mongodb/

一定要复制id，直接输入数字好像识别不了



