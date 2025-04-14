# 监控Windows操作系统

### 一、环境介绍

#### 主机清单

| 职责            | ip地址           | 备注                  |
| ------------- | -------------- | ------------------- |
| Prometheus服务器 | 192.168.28.100 | docker模式的prometheus |
| 待监控Windows    | 192.168.28.109 | 待准备组件：node_exporter |

### 二、待监控Windows机准备

#### Windows Server 2019概述

Windows Server 2019是微软公司研发的**服务器操作系统**，于2018年10月2日发布，于2018年10月25日正式商用。

Windows Server 2019基于Long-Term Servicing Channel 1809内核开发 [3]，相较于之前的Windows Server版本主要围绕混合云、安全性、应用程序平台、超融合基础设施（HCI）四个关键主题实现了很多创新

#### 2.1 点击VMware Workstation软件左上角【文件】-【新建虚拟机】

![image-20210604100554698](Resources/image-20210604100510773.png)

#### 2.2 选择典型安装虚拟机

![image-20210604101237589](Resources\image-20210604101237589.png)

选择【稍后安装操作系统】

![1-26](Resources/1-26.png)

选择客户机操作系统，版本选择Windows Server 2019

![1-27](Resources/1-27.png)

配置虚拟机名称和存储位置

![1-28](Resources/1-28.png)

配置虚拟机处理器，根据自己电脑硬件配置即可，此处调用的是物理机处理器逻辑线程，一般2就够，建议给自己物理机预留4个线程

![1-29](Resources/1-29.png)

配置内存，推荐2G

![1-30](Resources/1-30.png)

配置网络类型

![1-31](Resources/1-31.png)

指定磁盘容量

![1-32](Resources/1-32.png)

确认虚拟机参数后点击自定义硬件

![1-33](Resources/1-33.png)

点击【新CD/DVD】，右侧选择【使用ISO映像文件】

![1-34](Resources/1-34.png)

浏览找到准备好的windowsServer2019安装映像，点击关闭，开机虚拟机

#### 2.3 Win2019的安装

点击开启虚拟机

![1-35](Resources/1-35.png)

等待映像自动加载。到本界面后点击下一步

![1-36](Resources/1-36.png)

点击现在安装

![1-37](Resources/1-37.png)

等待安装程序启动完成，跳转到激活界面，点击【我没有产品密钥】

![1-38](Resources/1-38.png)

勾选含有“桌面体验"的操作系统，不然安装后为纯命令行页面，点击下一步

![1-39](Resources/1-39.png)

接受许可条款后点击下一步，选择自定义安装Windows

![1-40](Resources/1-40.png)

选中主分区，点击下一步，等待安装程序进行安装

![1-41](Resources/1-41.png)

安装完成后会自动进行重启

#### 2.4 Win2019的系统配置

上一步重启完成后就会自动进入系统配置界面

配置管理员密码后点击完成，两遍密码必须完全一样，且须保持密码复杂度要求

![1-42](Resources/1-42.png)

直接即可进入系统桌面

![1-43](Resources/1-43.png)

#### 2.5 优化虚拟机

##### 关闭系统自动更新、关闭防火墙

开始菜单——》控制面板——》系统和安全——》windows 防火墙——》打开或关闭windows防火墙

开始菜单——》控制面板——》系统和安全——》windows Update——》更改设置——》下拉菜单选择从不检查更新

##### 安装VMtools

点击安装VMtools，一直下一步即可

![1-25](Resources/1-25.png)

#### 2.6 修改win2019的ip地址

win2019的虚拟机->设置->网络适配器，确保处于NAT网络模式

控制面板->网络和Internet->查看网络状态和任务->Ethernet0->属性->TCP/IPv4属性

![2-16](Resources\2-16.png)

### 三、win上安装windows_exporter

#### 3.1 虚拟机中下载

```shell
https://github.com/prometheus-community/windows_exporter/releases/download/v0.20.0/windows_exporter-0.20.0-amd64.exe
```

放到window2019下的c:\盘目录下

#### 3.2 注册windows_exporter为win服务

win+s搜索cmd，使用管理员身份打开命令提示符
执行下面命令将其注册为系统服务

```shell
sc create windows_exporter binpath=c:\windows_exporter-0.20.0-amd64.exe type= own start= auto displayname= windows_exporter
```

windows_exporter服务注册成功Windows服务列表查看windows_exporter服务

![2-16](Resources\2-17.png)

选中windows_exporter服务，右键菜单中点击属性，在属性对话框输入启动参数；并启动服务

```shell
 --telemetry.addr=0.0.0.0:9182
```

从宿主机中访问http://192.168.28.109:9182/metrics，可以采集到数据

![2-18](Resources\2-18.png)

### 四、prometheus服务器中添加windows_exporter监控

192.168.28.100的centos上，修改prometheus的配置文件

```shell
#进入docker-prometheus目录
cd /data/docker-prometheus

#修改prometheus.yml
vi prometheus/prometheus.yml
```

添加待监控的win服务器

```shell
    #添加待监控的win服务器
    - targets: ['192.168.28.109:9182']
      labels:
        instance: win服务器
```

保存配置后，让配置生效

```shell
#centos1中执行
curl -X POST http://localhost:9090/-/reload
```

刷新访问http://192.168.28.100:9090/targets?search=，确认新监控的win服务器是否生效

![2-19](Resources\2-19.png)

### 五、win服务器的指标查询

- cpu利用率

```shell
100 - (avg by (instance,job) (irate(windows_cpu_time_total{mode="idle"}[2m])) * 100)
```

- 剩余内存

```shell
windows_os_physical_memory_free_bytes /1024/1024/1024
```

- 内存利用率

```shell
100 - 100 * windows_os_physical_memory_free_bytes / windows_cs_physical_memory_bytes
```

- 硬盘使用率

```shell
100- 100 * (windows_logical_disk_free_bytes / windows_logical_disk_size_bytes)
```

- 预测硬盘使用天数

```shell
100 * (windows_logical_disk_free_bytes / windows_logical_disk_size_bytes) < 15 and predict_linear(windows_logical_disk_free_bytes[6h], 4 * 24 * 3600)
```

- 网卡发送速率

```shell
((sum(rate (windows_net_bytes_sent_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance,job)) / 100)
```

- 网卡接收速率

```shell
((sum(rate (windows_net_bytes_received_total{device!~'tap.*|veth.*|br.*|docker.*|virbr*|lo*'}[5m])) by (instance,job)) / 100)
```
