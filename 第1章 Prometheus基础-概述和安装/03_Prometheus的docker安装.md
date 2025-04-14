# Prometheus的docker安装

### 一、Docker是什么

Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口 (类似 iPhone 的 app),更重要的是容器性能开销极低。

Docker 从 17.03 版本之后分为 CE (Community Edition:社区版) 和EE (EnterpriseEdition: 企业版)，我们用社区版就可以了

#### 应用程序部署方式的演变

在部署应用程序的方式上，主要经历了三个时代

![1-1](Resources\11-1.png)

- 传统部署：互联网早期，会直接将应用程序部署在物理机上
1. 优点: 简单，不需要其它技术的参与
2. 缺点: 不能为应用程序定义资源使用边界，很难合理地分配计算资源，而且程序之间容易产生影响
- 虚拟化部署：可以在一台物理机上运行多个虚拟机，每个虚拟机都是独立的一个环境
1. 优点: 程序环境不会相互产生影响，提供了一定程度的安全性
2. 缺点: 增加了操作系统，浪费了部分资源

![1-7](Resources\11-7.png)

![1-8](Resources\11-8.png)

- 容器化部署：与虚拟化类似，但是共享了操作系统。优点：
1. 可以保证每个容器拥有自己的文件系统、CPU、内存、进程空间等
2. 运行应用程序所需要的资源都被容器包装，并和底层基础架构解耦
3. 容器化的应用程序可以跨云服务商、跨Linux操作系统发行版进行部署
4. 部署快速。

### 二、Docker环境的准备

#### 2.1 配置yum的docker源

```shell
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 2.2 安装docker

```shell
yum install -y docker-ce docker-ce-cli containerd.io
```

检查docker服务是否启动且开机自启动

```shell
systemctl start docker
systemctl enable docker
systemctl status docker
```

查看docker版本能运行出，版本号说明安装成功

```shell
docker -v
```

##### 主要docker命令

```shell
#查看docker中的已安装镜像列表
docker images

#查看docker中已运行的实例程序列表
docker ps -a
```

#### 2.3 安装docker-compose组件

Docker Compose是Docker官方的开源项目，负责实现对Docker容器集群的快速编排。Compose 是 Docker 公司推出的一个工具软件，可以管理多个 Docker 容器组成一个应用。你需要定义一个 YAML 格式的配置文件docker-compose.yml，写好多个容器之间的调用关系。然后，只要一个命令，就能同时启动/关闭这些容器

##### 优势

- 避免了多次使用 Dockerfile、Build、Image 命令或者 DockerHub 拉取 Image（当前案例中，prometheus、grafana、node_exporter、alertmanager等需要分别拉取镜像images启动程序，使用了docker-compose则可以在一个yaml文件中一步启动）
- 需要创建多个Container，多次编写启动命令;
- Container互相依赖的如何进行管理和编排（prometheus和其他组件有启动的依赖关系、先后顺序。用docker-compose的yaml文件可以比较方便的管理这类依赖）

##### 安装

```shell
#从github中拉取下载docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 设置文件具备执行权限
chmod +x /usr/local/bin/docker-compose

#查看已安装的版本，如果能正常输出版本号说明安装compose完毕
docker-compose -v
```

##### 主要的compose命令

```shell
#启动一个compose
docker-compose up

#关闭一个compose
docker-compose down
```

### 三、Docker-compose安装Prometheus

#### 3.1 主要文件及树形结构

```shell
.
└── docker-prometheus            #docker-compose的所在目录
    ├── alertmanager
    │   └── config.yml
    ├── docker-compose.yaml        #docker-compose的核心yaml文件
    ├── grafana
    │   ├── config.monitoring
    └── prometheus
        ├── alert.yml
        └── prometheus.yml
```

#### 3.2 创建基本目录结构

```shell
#切换到root用户
mkdir /data/docker-prometheus -p
mkdir /data/docker-prometheus/{grafana,prometheus,alertmanager} -p
cd /data/docker-prometheus/
```

#### 3.3 创建alertmanager的配置文件

vi alertmanager/config.yml

```yaml
global:
  #163服务器
  smtp_smarthost: 'smtp.163.com:465'
  #发邮件的邮箱
  smtp_from: 'xxxx@163.com'
  #发邮件的邮箱用户名，也就是你的邮箱　　　　　
  smtp_auth_username: 'xxxxx@163.com'
  #发邮件的邮箱密码
  smtp_auth_password: 'your-password'
  #进行tls验证
  smtp_require_tls: false

route:
  group_by: ['alertname']
  # 当收到告警的时候，等待group_wait配置的时间，看是否还有告警，如果有就一起发出去
  group_wait: 10s
  #  如果上次告警信息发送成功，此时又来了一个新的告警数据，则需要等待group_interval配置的时间才可以发送出去
  group_interval: 10s
  # 如果上次告警信息发送成功，且问题没有解决，则等待 repeat_interval配置的时间再次发送告警数据
  repeat_interval: 10m
  # 全局报警组，这个参数是必选的
  receiver: email

receivers:
- name: 'email'
  #收邮件的邮箱
  email_configs:
  - to: 'xxxx@163.com'
inhibit_rules:
 - source_match:
     severity: 'critical'
   target_match:
     severity: 'warning'
   equal: ['alertname', 'dev', 'instance']
```

#### 3.4 创建grafana的配置文件

vi grafana/config.monitoring

```shell
# admin登录密码为password
GF_SECURITY_ADMIN_PASSWORD=password
GF_USERS_ALLOW_SIGN_UP=false
```

#### 3.5 创建prometheus的配置文件

vi prometheus/prometheus.yml

```yaml
# 全局配置
global:
  scrape_interval:     15s # 将搜刮间隔设置为每15秒一次。默认是每1分钟一次。
  evaluation_interval: 15s # 每15秒评估一次规则。默认是每1分钟一次。

# Alertmanager 配置
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager:9093']

# 报警(触发器)配置
rule_files:
  - "alert.yml"

# 搜刮配置(一共创建五个Prometheus监控项)
scrape_configs:
  - job_name: 'prometheus'
    # 覆盖全局默认值，每15秒从该作业中刮取一次目标
    scrape_interval: 15s
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'alertmanager'
    scrape_interval: 15s
    static_configs:
    - targets: ['alertmanager:9093']
  - job_name: 'cadvisor'
    scrape_interval: 15s
    static_configs:
    - targets: ['cadvisor:8080']
      labels:
        instance: Prometheus服务器 

  - job_name: 'node-exporter'
    scrape_interval: 15s
    static_configs:
    - targets: ['node_exporter:9100']
      labels:
        instance: Prometheus服务器 
```

#### 3.6 创建prometheus的告警文件

vi prometheus/alert.yml

```yaml
groups:
- name: Prometheus alert
  rules:
  # 对任何实例超过30秒无法联系的情况发出警报
  - alert: 服务告警
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "服务异常,实例:{{ $labels.instance }}"
      description: "{{ $labels.job }} 服务已关闭"
```

#### 3.7 创建docker-compose.yaml文件

yaml中指定的镜像版本与前述二进制安装中的各组件版本保持一致。cadvisor表示普罗米修斯对docker平台的监控组件，此处采用latest最新版

| 镜像名           | 版本号     |
| ------------- | ------- |
| prometheus    | v2.45.5 |
| alertmanager  | v0.27.0 |
| node-exporter | v1.8.0  |
| grafana       | 9.3.16  |
| cadvisor      | latest  |

vi docker-compose.yaml

```yaml
version: '3.3'

# 存储卷
volumes:
  prometheus_data: {}
  grafana_data: {}

networks:
  monitoring:
    driver: bridge

services:
  prometheus:
    image: prom/prometheus:v2.45.5
    container_name: prometheus
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro  # 本地时区挂载在镜像中
      - ./prometheus/:/etc/prometheus/
      - prometheus_data:/prometheus # 数据存储位置
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries' # 控制台库
      - '--web.console.templates=/usr/share/prometheus/consoles' # 控制台模板
      #热加载配置
      - '--web.enable-lifecycle'
      #api配置
      #- '--web.enable-admin-api'
      #历史数据最大保留时间，默认15天
      - '--storage.tsdb.retention.time=30d'  
    networks:
      - monitoring
    links:
      - alertmanager
      - cadvisor
      - node_exporter
    expose:
      - '9090'
    ports:
      - 9090:9090
    depends_on:
      - cadvisor # 等待cadvisor启动完成后prometheus再启动

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./alertmanager/:/etc/alertmanager/
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring
    expose:
      - '9093'
    ports:
      - 9093:9093

# 监控容器
  cadvisor:
    image: google/cadvisor:latest
    container_name: cadvisor
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring
    expose:
      - '8080'

  node_exporter:
    image: prom/node-exporter:v1.8.0
    container_name: node-exporter
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command: 
      - '--path.procfs=/host/proc' 
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker)($$|/)'
    networks:
      - monitoring
    ports:
      - '9100:9100'

  grafana:
    image: grafana/grafana:9.3.16
    container_name: grafana
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    env_file:
      - ./grafana/config.monitoring
    networks:
      - monitoring
    links:
      - prometheus
    ports:
      - 3000:3000
    depends_on:
      - prometheus
```

#### 3.8 运行docker-compose

```shell
[root@localhost docker-prometheus]# cd /data/docker-prometheus
[root@localhost docker-prometheus]# docker-compose up
Creating network "docker-prometheus_monitoring" with driver "bridge"
Creating node-exporter ... done
Creating cadvisor      ... done
Creating alertmanager  ... done
Creating prometheus    ... done
Creating grafana       ... done
...
```

运行时间较慢，需要拉取若干docker镜像文件，运行完毕不报错说明成功

#### 3.9 检查compose的运行状态

检查镜像，发现5个镜像都已正常拉取

```shell
docker images
```

检查容器，发现五个容器都处于up运行状态

```shell
[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE                       COMMAND                   CREATED              STATUS              PORTS                                       NAMES
0abde93cde05   grafana/grafana:9.3.16      "/run.sh"                 About a minute ago   Up About a minute   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   grafana
f4b829722315   prom/prometheus:v2.45.5     "/bin/prometheus --c…"   About a minute ago   Up About a minute   0.0.0.0:9090->9090/tcp, :::9090->9090/tcp   prometheus
b20652e73594   prom/alertmanager:v0.27.0   "/bin/alertmanager -…"   About a minute ago   Up About a minute   0.0.0.0:9093->9093/tcp, :::9093->9093/tcp   alertmanager
e0060c8a8fa7   google/cadvisor:latest      "/usr/bin/cadvisor -…"   About a minute ago   Up About a minute   8080/tcp                                    cadvisor
8ca577e88389   prom/node-exporter:v1.8.0   "/bin/node_exporter …"   About a minute ago   Up About a minute   0.0.0.0:9100->9100/tcp, :::9100->9100/tcp   node-exporter
```

检查端口

```shell
[root@localhost ~]# ss -lntp|egrep "3000|9090|9100|9093"
LISTEN     0      128          *:3000                     *:*                   users:(("docker-proxy",pid=69903,fd=4))
LISTEN     0      128          *:9090                     *:*                   users:(("docker-proxy",pid=69807,fd=4))
LISTEN     0      128          *:9093                     *:*                   users:(("docker-proxy",pid=69568,fd=4))
LISTEN     0      128          *:9100                     *:*                   users:(("docker-proxy",pid=69505,fd=4))
LISTEN     0      128       [::]:3000                  [::]:*                   users:(("docker-proxy",pid=69908,fd=4))
LISTEN     0      128       [::]:9090                  [::]:*                   users:(("docker-proxy",pid=69812,fd=4))
LISTEN     0      128       [::]:9093                  [::]:*                   users:(("docker-proxy",pid=69590,fd=4))
LISTEN     0      128       [::]:9100                  [::]:*                   users:(("docker-proxy",pid=69513,fd=4))
```

web访问地址

| 应用            | 访问地址                         | 账号密码           |
| ------------- | ---------------------------- | -------------- |
| prometheus    | http://centosip:9090/        | 无              |
| grafana       | http://centosip:3000/        | admin/password |
| alertmanager  | http://centosip:9093/        | 无              |
| node_exporter | http://centosip:9100/metrics | 无              |

##### 配置grafana的数据源

- 访问grafana

http://192.168.28.100:3000/

- 选择配置->Data sources

![1-5](Resources/1-5.png)

- 填写Prometheus地址

保存并测试（save & test)

![1-6](Resources/1-13.png)

添加node_exporter

- 访问grafana官网

https://grafana.com/grafana/dashboards/

找到node exporter full的插件

![1-7](Resources/1-7.png)

- 复制node exporter的id

![1-8](Resources/1-8.png)

- grafana的数据源->import

![1-10](Resources/1-10.png)

- 填写复制的node exporter插件的id

选择load，填写服务器监控名称，选择Prometheus的监控

![1-11](Resources/1-11.png)

- 即可在grafana中查看监控数据

![1-12](Resources/1-12.png)
