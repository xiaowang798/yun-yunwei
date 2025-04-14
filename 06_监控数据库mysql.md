# 监控数据库mysql

### 一、软件环境介绍

#### 1.1 mysql介绍

- 是什么
  
  MySQL是一个关系型数据库管理系统，目前属于 Oracle 旗下产品
  
  它是最流行的关系型数据库管理系统之一，存放表的数据

- 与其他关系型数据库管理系统相比，市占率
  
  ![img](Resources\mysql1.png) 

- 优势

|       | Mysql | Oracle | sqlserver |
| ----- | ----- | ------ | --------- |
| 开源    | 是     | 是      | 否-闭源      |
| 社区活跃度 | 最活跃   | 活跃     | 活跃        |
| 使用门槛  | 低     | 高      | 中         |
| 学习门槛  | 低     | 高      | 中         |
| 性能    | 较快    | 最快     | 较快        |

#### 1.2 主机清单

| 职责            | ip地址           | 备注                  |
| ------------- | -------------- | ------------------- |
| Prometheus服务器 | 192.168.28.100 | docker模式的prometheus |
| 待监控Linux      | 192.168.28.110 | 待准备组件：mysql8版本      |

### 二、prometheus监控mysql

docker安装（略）

docker-compose安装(略)

#### 2.1 待监控Linux安装Mysql8

centos2中，创建docker-compose目录

```shell
mkdir /data/docker-compose -p
cd /data/docker-compose
```

创建docker-compose.yaml

```yaml
version: '3.3'

services:
  db:
    image: mysql:8.0
    restart: always
    container_name: mysql
    environment:
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: 123456
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --lower_case_table_names=1
      --performance_schema=1
      --sql-mode=""
      --skip-log-bin
    volumes:
      - /data/mysql/data:/var/lib/mysql
    ports:
      - 3306:3306
```

运行docker-compose

```shell
docker-compose up -d
```

查看docker 的信息

```shell
docker images
docker ps -a
```

#### 2.2 待监控Linux安装mysqld_exporter

##### 2.2.1 创建exporter账户

centos2机器

```shell
#进入mysql容器
docker exec -it mysql mysql -uroot -p123456

#创建exporter用户
CREATE USER 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS,REPLICATION CLIENT,SELECT ON *.* TO 'exporter'@'%';
#刷新权限
flush privileges;
#退出mysql容器
exit;

#测试登录
docker exec -it mysql mysql -uexporter -ppassword
```

##### 2.2.2 安装mysqld_exporter

二进制安装或者docker安装(或者docker-compose安装)，此处选用docker-compose安装

centos2机器上，修改docker-compose.yaml文件

```shell
cd /data/docker-compose
vi docker-compose.yaml
```

yaml文件增加mysqld_exporter组件

```yaml
version: '3.3'

services:
  mysqld-exporter:
    image: prom/mysqld-exporter:v0.12.1
    container_name: mysqld-exporter
    restart: always
    command:
      - '--collect.info_schema.processlist'
      - '--collect.info_schema.innodb_metrics'
      - '--collect.info_schema.tablestats'
      - '--collect.info_schema.tables'
      - '--collect.info_schema.userstats'
      - '--collect.engine_innodb_status'
    environment:
      - DATA_SOURCE_NAME=exporter:password@(192.168.28.110:3306)/
    ports:
      - 9104:9104
```

启动docker-compose.yaml

```shell
docker-compose up -d
```

访问mysqld_exporter

http://192.168.28.110:9104/metrics

#### 2.3 prometheus服务器添加mysqld_exporter的地址

192.168.28.100的centos1上，修改prometheus的配置文件

```shell
#进入docker-prometheus目录
cd /data/docker-prometheus

#修改prometheus.yml
vi prometheus/prometheus.yml
```

添加待监控的mysqld_exporter

```shell
  - job_name: 'mysqld-exporter'
    static_configs:
    - targets: ['192.168.28.110:9104']
      labels:
        instance: mysqld服务器 
```

保存配置后，让配置生效

```shell
#centos1中执行
curl -X POST http://localhost:9090/-/reload
```

刷新访问http://192.168.28.100:9090/targets?search=，确认新监控的redis服务器是否生效

<img src="Resources\2-24.png" alt="221" style="zoom:67%;" />

#### 2.4 mysql服务器指标查询

```shell
mysql_up                                              # 服务器是否在线
mysql_global_status_uptime                            # 运行时长，单位 s
delta(mysql_global_status_bytes_received[1m])         # 网络接收的 bytes
delta(mysql_global_status_bytes_sent[1m])             # 网络发送的 bytes


mysql_global_status_threads_connected                 # 当前的客户端连接数
mysql_global_variables_max_connections                # 允许的最大连接数
mysql_global_status_threads_running                   # 正在执行命令的客户端连接数，即非 sleep 状态
delta(mysql_global_status_aborted_connects[1m])       # 客户端建立连接失败的连接数，比如登录失败
delta(mysql_global_status_aborted_clients[1m])        # 客户端连接之后，未正常关闭的连接数


delta(mysql_global_status_commands_total{command="xx"}[1m]) > 0     # 每分钟各种命令的次数
delta(mysql_global_status_handlers_total{handler="xx"}[1m]) > 0     # 每分钟各种操作的次数
delta(mysql_global_status_handlers_total{handler="commit"}[1m]) > 0 # 每分钟 commit 的次数
delta(mysql_global_status_table_locks_immediate[1m])  # 请求获取锁，且立即获得的请求数
delta(mysql_global_status_table_locks_waited[1m])     # 请求获取锁，但需要等待的请求数。该值越少越好


delta(mysql_global_status_queries[1m])                # 每分钟的查询数
delta(mysql_global_status_slow_queries[1m])           # 慢查询数。如果未启用慢查询日志，则为 0
mysql_global_status_innodb_page_size                  # innodb 数据页的大小，单位 bytes
mysql_global_variables_innodb_buffer_pool_size        # innodb_buffer_pool 的限制体积
mysql_global_status_buffer_pool_pages{state="data"}   # 包含数据的数据页数，包括洁页、脏页
mysql_global_status_buffer_pool_dirty_pages           # 脏页数


锁指标
mysql_global_status_innodb_row_lock_current_waits #当前正在等待的 InnoDB 行锁数量。
mysql_global_status_innodb_row_lock_time #从服务器启动以来的总 InnoDB 行锁等待时间（以毫秒为单位）。
mysql_global_status_innodb_row_lock_time_avg #每次等待 InnoDB 行锁的平均时间（以毫秒为单位）
mysql_global_status_innodb_row_lock_time_max #单次等待 InnoDB 行锁的最长时间（以毫秒为单位）。
mysql_global_status_innodb_row_lock_waits  #从服务器启动以来的总 InnoDB 行锁等待次数。
```

#### 2.5 grafana中对mysql进行监控

copy id to clipboard->grafana的dashboards中Import dashboard

https://grafana.com/grafana/dashboards/20016-mysql-8-0/

![220](Resources\2-25.png)
