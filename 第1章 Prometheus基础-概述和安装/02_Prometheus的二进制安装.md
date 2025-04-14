# Prometheus的二进制安装

### 一、安装总体介绍

#### 1.1 需要安装的组件

- vmware虚拟机软件

​    VMware Workstation是一款功能强大的桌面虚拟计算机软件，提供用户可在单一的桌面上同时运行不同的操作系统，和进行开发、测试 、部署新的应用程序的最佳解决方案。VMware Workstation可在一部实体机器上模拟完整的网络环境，以及可便于携带的虚拟机，其更好的灵活性与先进的技术胜过了市面上其他的虚拟计算机软件。对于企业的 IT开发人员和系统管理员而言， VMware在虚拟网路，实时快照，拖曳共享文件夹，支持 PXE 等方面的特点使它成为必不可少的工具。

- centos的linux操作系统
- Prometheus软件
- Grafana软件

#### 1.2 安装步骤

- vmware虚拟机安装

- centos安装

- Prometheus的二进制安装

- 安装alertmanager

- Grafana软件的安装

- 安装node_exporter

### 二、vmware虚拟机的安装

提前进入到vmware.com的网站（要先有vmware的账号，再登录，再下载次新的vmware workstation pro的版本，此处我们下载16的版本）

![img](Resources\clipboard.png)

![img](Resources\clipboard-1622772013837.png)

![img](Resources\clipboard-1622772027735.png)

![img](Resources\clipboard-1622772033348.png)

![img](Resources\clipboard-1622772037403.png)

![img](Resources\clipboard-1622772041104.png)

#### 创建虚拟机

![image-20210604100554698](Resources\image-20210604100510773.png)

![image-20210604101237589](Resources\image-20210604101237589.png)

![image-20210604101340840](Resources\image-20210604101340840.png)

![image-20210604101515110](Resources\image-20210604101515110.png)

![image-20210604101635945](Resources\image-20210604101635945.png)

![image-20210604101732502](Resources\image-20210604101732502.png)

![image-20210604102000435](Resources\image-20210604102000435.png)

### 三、Centos的安装

#### 3.1 Vmware安装虚拟机过程中注意下面选项的设置:

- 硬盘(20G)

- 操作系统环境: CPU (2C)

- 内存(2G)

- 语言选择:中文简体

- 软件选择:基础设施服务器

- 分区选择: 自动分区

- 网络配置: 按照下面配置网路地址信息
  
  （建议提前进入vmware的界面中，编辑->虚拟网络编辑器->vmnet8网卡->左下角子网ip，搜集一下使用的网段：192.168.28.0）

![1-9](Resources\1-9.png)

```ini
cd /etc/sysconfig/network-scripts
vi ifcfg-ens33

网络地址: 192.168.28.100(主机ip按照自身vmware设定即可)
子网掩码: 255.255.255.0
默认网关: 192.168.28.2
DNS:192.168.28.2
```

```shell
#参考案例
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.28.100
NETMASK=255.255.255.0
GATEWAY=192.168.28.2
DNS1=192.168.28.2
```

保存ifcfg-ens33文件后，重启linux的网络服务即可生效

```shell
systemctl restart network
```

#### 3.2 检查操作系统版本

```shell
# 此方式下安装kubernetes集群要求Centos版本要在7.5或之上
[root@master ~]# cat /etc/redhat-release
CentOS Linux release 7.5.1804 (Core)
```

#### 3.3  使用ssh工具连接这台linux

比如winterm、finalshell、xshell工具连接这台linux

### 四、Prometheus的二进制安装

#### 4.1 获取安装包

官网：https://www.prometheus.io/download/

```shell
#切换到家目录
cd /home

#用wget命令从github.com下载指定Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.45.5/prometheus-2.45.5.linux-amd64.tar.gz

#解压
tar xf prometheus-2.45.5.linux-amd64.tar.gz 

#查看解压后的内容
ll

#创建Prometheus目录
mkdir /opt/prometheus -p

#移动解压后的文件名到/opt/并改名
mv prometheus-2.45.5.linux-amd64/ /opt/prometheus/prometheus
```

#### 4.2 创建专门用户

```shell
useradd -M -s /usr/sbin/nologin prometheus


chown prometheus:prometheus -R /opt/prometheus
```

#### 4.3 创建系统服务

```shell
cat > /etc/systemd/system/prometheus.service << "EOF"
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/prometheus/data \
  --storage.tsdb.retention.time=60d \
  --web.enable-lifecycle
[Install]
WantedBy=multi-user.target
EOF

#配置Prometheus的配置文件
#配置Prometheus的数据目录
#配置Prometheus的默认存储天数15天->60天
#配置Prometheus的热加载配置
```

启动服务

```shell
systemctl start prometheus
systemctl enable prometheus


systemctl status prometheus
```

如有启动问题，进行日志查看&故障排除

```shell
journalctl -u prometheus.service -f
ll /etc/systemd/system 查看装好的服务
```

#### 4.4 访问地址

```shell
#Prometheus的访问地址(Prometheus的服务端口：9090)
#如果9090的端口不通，一方面要检查Prometheus的service是否启动，另一方面要检查防火墙是否关闭systemctl stop firewalld，如果有iptables的话systemctl disable iptables.service 

http://192.168.111.11:9090/

#Prometheus的监控指标
http://192.168.28.100:9090/metrics
```

### 五、安装alertmanager

Alertmanager 是 Prometheus 生态系统中的一个重要组件，主要用于处理和管理由 Prometheus 发送的告警信息

#### 5.1 获取安装包

```shell
cd /home
#下载alertmanager二进制压缩包
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

#解压
tar xf alertmanager-0.27.0.linux-amd64.tar.gz

#查看解压后的文件名
ll

#移动解压后的文件名到/opt/，并改名为alertmanager
mv alertmanager-0.27.0.linux-amd64 /opt/prometheus/alertmanager
```

#### 5.2 更改owner权限

```shell
chown prometheus:prometheus -R /opt/prometheus/alertmanager
```

#### 5.3  创建系统服务

```shell
cat >/etc/systemd/system/alertmanager.service << "EOF"
[Unit]
Description=Alert Manager
Wants=network-online.target
After=network-online.target
[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/opt/prometheus/alertmanager/alertmanager \
--config.file=/opt/prometheus/alertmanager/alertmanager.yml \
--storage.path=/opt/prometheus/alertmanager/data
Restart=always
[Install]
WantedBy=multi-user.target
EOF
```

启动alertmanager

```shell
systemctl daemon-reload
systemctl start alertmanager.service
systemctl enable alertmanager.service

systemctl status alertmanager.service
```

#### 5.4 修改prometheus配置

加入alertmanager

```shell
#vi /opt/prometheus/prometheus/prometheus.yml
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      #根据实际填写alertmanager的地址
      - localhost:9093

rule_files:
  #根据实际名修改文件名
  - "alert.yml"
```

增加触发器配置文件

```shell
cat > /opt/prometheus/prometheus/alert.yml <<"EOF"
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
      instance: "服务异常,实例:{{ $labels.instance }}"
      description: "{{ $labels.job }} 服务已关闭"
EOF
```

检查配置

```shell
cd /opt/prometheus/prometheus/
./promtool check config prometheus.yml  用普罗米修斯来检查它的配置，它会提示两个success，第一个是检查prometheus，第二个是检查alertmanager
```

重启prometheus或重新加载配置文件

```shell
#重启
systemctl restart prometheus
#或重载配置文件，需要--web.enable-lifecycle配置（热加载）
systemctl restart prometheus
curl -X POST http://localhost:9090/-/reload
```

#### 5.5 访问地址

```shell
http://192.168.111.11:9093/
```

### 六、Grafana软件的安装

本次课程选择离线安装包方式，grafana版本9.3.16-1

#### 6.1 上传离线包（这个包在官网下载很慢的）

grafana-9.3.16-1.x86_64.rpm

```shell
#切换到/home目录
cd /home

#上传grafana-9.3.16-1.x86_64.rpm
ll
```

#### 6.2 离线包安装，并开机自启动

- 离线包安装

```shell
yum localinstall  grafana-9.3.16-1.x86_64.rpm -y
```

- 开机自启动

```shell
systemctl start grafana-server.service
systemctl enable grafana-server.service

#确认3000端口是否被grafana程序占据
ss -ntulp | grep 3000
```

#### 6.3 访问图形界面

http://192.168.28.100:3000/

初始密码：admin/admin

<img src="Resources\9-1.png" alt="9-1" style="zoom:67%;" /> 

### 七、安装node_exporter

#### 7.1 获取安装包

```shell
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz

#解压
tar xvf node_exporter-1.8.0.linux-amd64.tar.gz

#查看内容
ll

#移动到指定目录
mv node_exporter-1.8.0.linux-amd64 /opt/prometheus/node_exporter
```

#### 7.2 更改owner权限

```shell
chown prometheus:prometheus -R /opt/prometheus/node_exporter
```

#### 7.3 创建系统服务

```shell
cat > /etc/systemd/system/node_exporter.service <<"EOF"
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target
[Service]
User=prometheus
Group=prometheus
ExecStart=/opt/prometheus/node_exporter/node_exporter
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

启动服务

```shell
systemctl daemon-reload
systemctl start node_exporter.service
systemctl enable node_exporter.service

#查看服务状态
systemctl status node_exporter.service
```

#### 7.4 访问地址

```shell
http://192.168.175.120:9100/metrics
```

#### 7.5 配置Prometheus

```shell
#vi /opt/prometheus/prometheus/prometheus.yml
  #node-exporter配置（新增。格式要一致）
  - job_name: "node-exporter"
    scrape_interval: 15s
    static_configs:
    - targets: ["localhost:9100"]
      labels:
        instance: Prometheus服务器
```

重新加载Prometheus配置

```shell
curl -X POST http://localhost:9090/-/reload
```

prometheus的web检查

```shell
http://192.168.28.100:9090/
```

检查status

![1-3](Resources\1-3.png)

检查alert

![1-4](Resources\1-4.png)

### 八、配置Grafana

#### 8.1 配置Prometheus数据源

- 访问grafana

http://192.168.28.100:3000/

- 选择配置->Data sources

![1-5](Resources\1-5.png)

- 填写Prometheus地址

保存并测试（save & test)

![1-6](Resources\1-6.png)

#### 8.2 添加node_exporter

- 访问grafana官网

https://grafana.com/grafana/dashboards/

找到node exporter full的插件

![1-7](Resources\1-7.png)

- 复制node exporter的id

![1-8](Resources\1-8.png)

- grafana的数据源->import

![1-10](Resources\1-10.png)

- 填写复制的node exporter插件的id

选择load，填写服务器监控名称，选择Prometheus的监控

![1-11](Resources\1-11.png)



- 点击load之后就可以修改Name，和prometheus的数据源
- ![image-20241217163545017](C:\Users\xiaowang798abc\AppData\Roaming\Typora\typora-user-images\image-20241217163545017.png)
- 
- 
- 
- 
- 
- 
- 即可在grafana中查看监控数据

![1-12](Resources\1-12.png)
