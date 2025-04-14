# 监控Linux操作系统

### 一、环境介绍

#### 1.1 主机清单

| 职责            | ip地址           | 备注                                 |
| ------------- | -------------- | ---------------------------------- |
| Prometheus服务器 | 192.168.28.100 | docker模式的prometheus                |
| 待监控Linux      | 192.168.28.110 | 待准备组件：docker+compose+node_exporter |

#### 1.2 调用链路图

<img src="Resources\2-3.png" alt="2-3" style="zoom:80%;" />

### 二、待监控Linux机准备

#### 2.1 Centos7安装

centos2

#### 2.2 调整ip地址

```shell
cd /etc/sysconfig/network-scripts
vi ifcfg-ens33

网络地址: 192.168.28.110
子网掩码: 255.255.255.0
默认网关: 192.168.28.2
DNS:192.168.28.2
```

#### 2.3 调整host名称

```shell
#修改hosts文件
cd /etc
vi hosts

#录入以下内容，:wq保存并退出
192.168.28.110 centos2
192.168.28.100 centos1
```

#### 2.4 安装docker环境

centos2配置yum的docker源

```shell
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

centos2安装docker

```shell
yum install -y docker-ce docker-ce-cli containerd.io
```

centos2检查docker服务是否启动且开机自启动

```shell
systemctl start docker
systemctl enable docker
systemctl status docker
```

查看docker版本能运行出，版本号说明安装成功

```shell
docker -v
```

#### 2.5 centos2安装docker-compose环境

```shell
#从github中拉取下载docker-compose
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 设置文件具备执行权限
chmod +x /usr/local/bin/docker-compose

#查看已安装的版本，如果能正常输出版本号说明安装compose完毕
docker-compose -v
```

### 三、centos2安装node_exporter

192.168.28.110的centos2上，以docker-compose方式快速安装node_exporter

#### 3.1 创建node_exporter文件夹

```shell
mkdir /data/node_exporter -p
cd /data/node_exporter
```

#### 3.2 centos2创建docker-compose.yaml文件

```shell
version: '3.3'

services:
  node_exporter:
    image: prom/node-exporter:v1.8.0
    container_name: node-exporter
    restart: always
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command: 
      - '--path.procfs=/host/proc' 
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker)($$|/)'
    ports:
      - '9100:9100'
```

#### 3.3 docker-compose方式部署Node_exporter

```shell
cd /data/node_exporter
docker-compose up -d
```

检查

```shell
#查看docker的images镜像列表
docker images

#查看docker的实例容器列表
docker ps -a
```

访问node_exporter的监控url

http://192.168.28.110:9100/metrics

### 四、Prometheus的服务器上添加待监控的Linux机器

192.168.28.100的centos上，修改prometheus的配置文件

```shell
#进入docker-prometheus目录
cd /data/docker-prometheus

#修改prometheus.yml
vi prometheus/prometheus.yml
```

添加待监控的Linux服务器（centos2)

```shell
  - job_name: 'node-exporter'
    scrape_interval: 15s
    static_configs:
    #之前已经存在的Prometheus服务器
    - targets: ['node_exporter:9100']
      labels:
        instance: Prometheus服务器 
    #添加待监控的Linux服务器
    - targets: ['192.168.28.110:9100']
      labels:
        instance: centos2服务器   
```

保存配置后，让配置生效

```shell
#centos1中执行
curl -X POST http://localhost:9090/-/reload
```

刷新访问http://192.168.28.100:9090/targets?search=，确认新监控的centos2服务器是否生效
![2-4](Resources\2-4.png)

### 五、常用的Linux服务器监控指标

- CPU
- 内存
- 硬盘
- 网络流量
- 文件描述符
- 系统负载
- 系统服务

#### 5.1 cpu监控（单位为秒或者百分比

CPU的监控项名称是：node_cpu_seconds_total，使用总量

直接执行node_cpu_seconds_total查询后会出现很多监控指标，显然不是想要的

node_cpu_seconds_total执行后会出现很多监控指标，其中各种类型的比如系统态、用户态都会由mode标签来区分

我们想要查询CPU的使用率的思路是:

- 查出当前空闲的CPU百分比，最后用100减去，mode标签值idle就表示当前空闲的CPU值

```shell
node_cpu_seconds_total
```

![2-5](Resources\2-5.png)

cpu="0"代表第1核cpu

cpu="1"代表第2核cpu

- 获取空闲cpu监控数据

mode标签值为idle的为空闲（user代表mode为用户，system代表mode为系统...）

```shell
node_cpu_seconds_total{mode='idle'}
```

![2-6](Resources\2-6.png)

- 获取某台监控机器的cpu数据

instance标签值为centos2服务器的cpu数据

```shell
node_cpu_seconds_total{instance='centos2服务器'}
```

- 获取1分钟/5分钟/15分钟的cpu负载

```shell
node_load1
node_load5
node_load15
```

- 获取5分钟内的监控数据

之前虽然可以查出来结果，但是不太理想，因为CPU是不断波动的，我们可以在增加一个条件，查询5分钟内的一个CPU使用情况

```shell
node_cpu_seconds_total{mode='idle'}[5m]
```

![2-7](Resources\2-7.png)

- 获取5分钟内的cpu平均空闲状况

我们可以使用irate和avg函数结合刚才查询出5分钟内数据做一个平均情况展示

函数的使用方法：函数(指标获取方式)

```shell
avg(irate(node_cpu_seconds_total{mode='idle'}[5m])) by (instance)

#by(instance)表示以instance标签进行分组
```

![2-8](Resources\2-8.png)

- 获取cpu5分钟内的使用率

最后我们可以*100得出一个百分比的空闲率，再由100-即可得到CPU的使用率

```shell
100 - (avg(irate(node_cpu_seconds_total{mode='idle'}[5m])) by (instance) *100)
```

![2-9](Resources\2-9.png)

#### 5.2 内存监控（单位为字节（bytes）

由于内存的监控项没有像CPU一样区分了很多标签，因此内存监控相较于CPU则需要结合很多个监控项

node_memory_MemFree_bytes //空闲内存

node_memory_MemTotal_bytes //总内存

node_memory_Cached_bytes //缓存

node_memory_Buffers_bytes //缓冲区内存

监控内存使用的思路：

1. 空闲内存+缓存+缓冲区内存得出空闲总内存

2. 得出的空闲总内存再除总内存大小再乘100，得出空闲率

3. 再用100-空闲率就得出使用率
- 首先是获取空闲内存

```shell
(node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes)
```

![2-10](Resources\2-10.png)

- 获取空闲内存率

```shell
(node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100
```

![2-11](Resources\2-11.png)

- 获取内存使用率

```shell
100 - ((node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100)
```

![2-12](Resources\2-12.png)

#### 5.3 磁盘使用率（单位为字节（bytes）

关于磁盘使用率，这里我们用到的主要有：

 node_filesystem_free_bytes //剩余磁盘空间

 node_filesystem_size_bytes //磁盘空间总大小

 node_disk相关

这两个监控项中都有相同的标签可以关联，我们这里用到的标签有fstype，fstype标签值是关于磁盘的文件系统类型，对于磁盘监控，我们主要对xfs、ext4等文件系统的磁盘进行监控，像tmpfs这种的不必要监控，另一个主要的标签是mountpoint，这个标签值主要用来储存磁盘的挂载点，我们可以通过标签来选择要对那个挂载点的磁盘进行监控

磁盘使用率实现思路：

 1.由磁盘空闲容量除磁盘总容量乘100即可得到磁盘空闲率

 2.用100减磁盘空闲率即可得到磁盘使用率

- 首先是获取磁盘空闲率

```shell
node_filesystem_free_bytes{fstype=~"ext4|xfs",mountpoint="/"} / node_filesystem_size_bytes{fstype=~"ext4|xfs",mountpoint="/"} *100
```

![2-13](Resources\2-13.png)

```shell
#linux中获取磁盘空闲率命令
df -hT
```

- 获取磁盘使用率

```shell
100 - (node_filesystem_free_bytes{fstype=~"ext4|xfs",mountpoint="/"} / node_filesystem_size_bytes{fstype=~"ext4|xfs",mountpoint="/"} *100)
```

![2-14](Resources\2-14.png)

#### 5.4 网络采集（单位为字节（bytes）或者包

node_network_ 相关都属于网络采集数据

```shell
#网络流出流量
node_network_transmit_bytes_total
#网络流入流量  
node_network_receive_bytes_total
```

#### 5.5 系统服务状态

监控服务的状态，例如nginx、docker这种服务器的启动状态

node_exporter是根据systemd去监控的，因此只有能用systemctl启动的服务器才能被监控到

配置非常简单，只需要在启动时开启system监控，并指定监控什么服务即可

配置system监控的参数：

--collector.systemd

--collector.systemd.unit-whitelist=".+"

##### 5.5.1 配置待监控Linux的node_exporter启动参数

Prometheus服务器(centos1)，以及待监控的Linux服务器(centos2)都要操作

```shell
vi /usr/lib/systemd/system/node_exporter.service 
ExecStart=/data/node_exporter/node_exporter --collector.systemd --collector.systemd.unit-whitelist=(docker|sshd|node_exporter).service
```

重启服务

```shell
systemctl daemon-reload 
systemctl restart node_exporter.service 
```

##### 5.5.2 查看服务的监控状态

以docker为例，我们查询docker存活状态

node_systemd_unit_state使用这个监控项查看，里面也有很多标签，name=“docker.service”，标签name表示服务的名称， state=“active”，state表示服务的状态，active表示活动的，对应的监控值也是1，如果为1则表示正常，不为1表示异常

```shell
node_systemd_unit_state{name="docker.service", state="active"}
```

![2-15](Resources\2-15.png)
