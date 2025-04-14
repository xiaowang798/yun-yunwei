# Prometheus概述

### 一、是什么

Prometheus是一个开源系统监控和警报工具包，受启发于Google的Brogmon监控系统(相似的Kubernetes是从Google的Brog系统演变而来)，从2012年开始由前Google工程师在Soundcoud以开源软件的形式进行研发，并目于2015年早期对外发布早期版本，2016年5月继Kubernetes之后成为第二个正式加入CNCE基金会的项目，同年6月正式发布1.0版本。2017年底发布了基于全新存储层的2.0版本能更好地与容器平台、云平台配合。

![1-1](Resources\1-1.png)

#### 主要特点

支持多维数据模型由指标名称和键值对标识的时间序列数据

内置时间序列库TSDB(Time Serices Database)

支持PromQL(Promethues Query Language)对数据的查询和分析、图形展示和监控告警。

不依赖分布式存储;单个服务器节点是自治的

支持HTTP 的拉取(pull)方式收集时间序列数据

通过中间网关Pushgateway推送时间序列

通过服务发现或静态配置2种方式发现目标

支持多种可视化和仪表盘，如:grafana

### 二、组成

#### 2.1 核心组件

Prometheus Server，主要用于抓取数据和存储时序数据，另外还提供査询和 Alert Rule 配置管理

client libraries，用于检测应用程序代码的客户端库。

pushgateway，用于批量，短期的监控数据的汇总节点，主要用于业务数据汇报等。

exporters 收集监控样本数据，并以标准格式向 Prometheus 提供。例如:收集服务器系统数据的 node exporter,收集 MSOL 监控样本数据的是 MySQL exporter 等等

用于告警通知管理的 alertmanager 。

#### 2.2 基础架构

![1-2](Resources\1-2.png)

从这个架构图，也可以看出 Prometheus的主要模块包含， Server,Exporters,Pushgateway,PromQL,Alertmanager, WebUl等
它大致使用逻辑是这样:

- Prometheus server 定期从静态配置的 targets 或者服务发现的 targets 拉取数据(Targets是Prometheus采集Agent需要抓取的采
  集目标)
- 当新拉取的数据大于配置内存缓存区的时候，Prometheus 会将数据持久化到磁盘(如果使用remote storage 将持久化到云端)
- Prometheus 可以配置 rules，然后定时査询数据，当条件触发的时候，会将 alerts 推送到配置的 Alertmanager。
- Aertmanager 收到警告的时候，可以根据配置(163，钉钉等)，聚合，去重，降噪，最后发送警告。
- 可以使用 API Prometheus Console 或者 Grafana 查询和聚合数据

### 三、其他的监控软件

在监控系统中，常见的开源的监控系统有Zabbix、Prometheus、Nagios、Cacti、Graphite、Ganglia等，大部分企业都在基于这些开源监控系统的基础上进行开发。虽然每种监控系统都有各自的特点和功能，面向的用户对象与侧重点也有所不同，但是都吸纳了采集数据、告警、展示等基本功能。

- Cacti

最初发布于2001年， Cacti 是一款开源的基于Web的网络监控和专为数据记录而设计的图形化工具。它可以用于实时显示网络数据，如CPU负载或带宽利用率。

Cacti是RRDtool的前端应用程序，RRDtool是一种用于存储实时变化数据的开源数据库工具，其使用SNMP作为其默认收集算法，但如果你喜欢本地Perl的PHP脚本，那么你也可以使用它们。

其最新版本0.8.8h于2016年5月发布，主要功能包括无限图形项目、图形自动填充支持、图形数据处理、自定义数据采集脚本、内置SNMP支持、图形模板、数据源模板、主机模板和基于用户的管理

- Zabbix

Zabbix 作为企业级的网络监控工具，通过从服务器，虚拟机和网络设备收集的数据提供实时监控，自动发现，映射和可扩展等功能。

Zabbix的企业级监控软件为用户提供内置的**Java应用服务器**监控，硬件监控和CPU，内存，网络，磁盘空间性能监控。

该企业级网络监控工具能够**每分钟进行 3,000,000 次检查**，具有更高的安全性和数据中心监控功能

- Prometheus

Prometheus（普罗米修斯）是一套开源的监控&报警&时间序列数据库的组合，由 SoundCloud 公司开发。

Prometheus 基本原理是通过 HTTP 协议周期性抓取被监控组件的状态，这样做的好处是任意组件只要提供 HTTP 接口就可以接入监控系统，不需要任何 SDK 或者其他的集成过程。这样做非常适合**虚拟化环境比如 VM 或者 Docker** 。

Prometheus 应该是为数不多的适合 Docker、Mesos、Kubernetes 环境的监控系统之一

- Grafana

Grafana:是一款采用go语言编写的开源应用，主要用于大规模指标数据的**可视化展现**，是网络架构和应用分析中最流行的时序数据展示工具。目前已经支持绝大部分常用的时序数据库。

#### 与zabbix的对比

| Zabbix                               | Prometheus                                                         |
| ------------------------------------ | ------------------------------------------------------------------ |
| 后端用C开发，界面用PHP开发，定制化难度很高。             | 后端用golang开发，前端是Grafana，JSON编辑即可解决定制化难度较低                           |
| 6.0支持单个Zabbix实例监控超过10万个业务服务          | 支持更大的集群规模，速度也更快                                                    |
| 更适合监控物理机环境(物理主机，交换机，网络等监控)           | 更适合云环境的监控，对OpenStack，Kubernetes有更好的集成                              |
| 监控数据存储在关系型数据库内，如 MySQL很难从现有数据中扩展维度   | 监控数据存储在基于时间序列的数据库内，便于对已有数据进行新的聚合。十万级监控数据，Prometheus数据查询速率比Zabbix更快 |
| 安装简单，zabbix-server 一个软件包中包括了所有的服务端功能 | 安装相对复杂，监控、告警和界面都分属于不同的组件                                           |
| 图形化界面比较成熟，界面上基本上能完成全部的配置操作           | 界面相对较弱，很多配置需要修改配置文件                                                |
| 发展时间更长，对于很多监控场景，都有现成的解决方案            | 2015 年后开始快速发展，发展时间短，但现在也非常的成熟                                      |

#### 总结

监控系统没有绝对的谁好谁不好，最重要的是适合自己的公司团队，能够合理利用最小的成本解决问题。prometheus，zabbix 都只是工具，监控思想才是最重要的。

- 物理机、硬件设备的监控推荐使用Zabbix
- docker容器，Kubernetes监控推荐用Prometheus
- 云服务器厂商自带有监控系统，有的监控不全面，也可以搭配zabbix和Prometheus来一起使用。
