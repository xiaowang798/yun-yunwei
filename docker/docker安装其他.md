## 1.更换腾讯云yum源

```shell
1.备份当前的YUM源配置文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo_bak

2.下载腾讯云的CentOS YUM源配置文件
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo

3.运行YUM源更新命令
yum clean all
yum makecache
```

## 2.安装Docker

```shell
1.添加腾讯云docker-ce仓库地址
yum install -y yum-utils
yum-config-manager --add-repo https://mirrors.cloud.tencent.com/docker-ce/linux/centos/docker-ce.repo

2.安装docker
yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

3.配置docker的国内源
vi /etc/docker/daemon.json
{
  "registry-mirrors": [
        "https://dockerpull.com",
        "https://docker.anyhub.us.kg",
        "https://dockerhub.jobcher.com",
        "https://dockerhub.icu",
        "https://docker.awsl9527.cn"
  ]
}

4.使docker的配置生效
systemctl daemon-reload
systemctl restart docker
```

## 3.安装MySQL

```she
1.创建挂载目录
mkdir -p /home/docker/mysql/{log,data,conf,mysql-files}

2.执行命令安装MySQL
docker run -p 3306:3306 --name mysql -v /home/docker/mysql/log:/var/log/mysql -v /home/docker/mysql/data:/var/lib/mysql -v /home/docker/mysql/conf:/etc/mysql -v /home/docker/mysql/mysql-files:/var/lib/mysql-files -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0.24
```

使用Navicat连接MySQL数据库，显示结果如下：





![QQ_1737711191777](D:\Desktop\bishe\docker\md图片\QQ_1737711191777.png)

## 4.安装Redis

```shell
1.创建挂载目录
mkdir -p /home/docker/redis/data

2.执行命令安装Redis
docker run -p 6379:6379 --name redis -v /home/docker/redis/data:/data -d redis redis-server --appendonly yes --requirepass redis
```

使用Another Redis Desktop Manager来连接redis，显示结果如下：



![QQ_1737711224105](D:\Desktop\bishe\docker\md图片\QQ_1737711224105.png)

## 5.安装nginx

```shell
1.创建挂载目录
mkdir -p /home/docker/nginx/{html,logs}

2.先运行一次容器（为了拷贝配置文件）：
docker run -p 80:80 --name nginx \
-v /home/docker/nginx/html:/usr/share/nginx/html \
-v /home/docker/nginx/logs:/var/log/nginx  \
-d nginx

3.将容器内的配置文件拷贝到指定目录并修改目录名称
docker container cp nginx:/etc/nginx /home/docker/nginx/ && cd /home/docker/nginx && mv nginx conf && docker rm -f nginx

4.使用如下命令正式启动nginx容器
docker run -p 80:80 --name nginx \
-v /home/docker/nginx/html:/usr/share/nginx/html \
-v /home/docker/nginx/logs:/var/log/nginx  \
-v /home/docker/nginx/conf:/etc/nginx \
-d nginx
```

然后在/home/docker/nginx/html中创建一个index.html文件，里面写hello world，可以在浏览器中通过nginx访问到

![img](https://mmbiz.qpic.cn/mmbiz_png/aiaDibGD31uibh2UypAUp6NhNFcg45Zr97RU8s73sbyoKr0PvpOy7Ne7xyJMMePtm6hgW4HyhicXK50ia4SkweqHMZg/640?wx_fmt=png&from=appmsg)

## 6.安装RabbitMQ

```shell
1.执行命令安装RabbitMQ
docker run -p 5672:5672 -p 15672:15672 --name rabbitmq -d rabbitmq

2.进入容器并开启管理功能
docker exec -it rabbitmq /bin/bash
rabbitmq-plugins enable rabbitmq_management
```

访问15672端口输入账号密码并登录：guest guest





![QQ_1737711241805](D:\Desktop\bishe\docker\md图片\QQ_1737711241805.png)

## 7.安装Elasticsearch

```shell
1.创建挂载目录并且增加权限
mkdir -p /home/docker/elasticsearch/{plugins,data,config} && chmod 777 /home/docker/elasticsearch/*

2.在/home/docker/elasticsearch/config目录下创建elasticsearch.yml，并写入以下内容
http.host: 0.0.0.0

3.执行命令安装Elasticsearch
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms256m -Xmx512m" -v /home/docker/elasticsearch/plugins:/usr/share/elasticsearch/plugins -v /home/docker/elasticsearch/data:/usr/share/elasticsearch/data -v /home/docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -d elasticsearch:7.17.5

4.进入容器内部并安装中文分词器（也可以采用离线安装的方式）
docker exec -it elasticsearch /bin/bash
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.5/elasticsearch-analysis-ik-7.17.5.zip

5.重启容器
docker restart elasticsearch
```

然后在浏览器中访问9200端口，显示结果如下![img](D:\Desktop\bishe\docker\md图片\640)

## 8.安装Logstash

```shell
1.创建挂载目录
mkdir -p /home/docker/logstash/{pipeline,config}

2.在/home/docker/logstash/pipeline目录下创建logstash.conf，并写入以下内容
#日志采集入口，项目的logback会和这个input交互
input {
    tcp {
       #模式为 server
      mode => "server"
       #ip为logstash的地址
      host => "0.0.0.0"
       #监听的端口，以此端口获得日志数据
      port => 9413
       #数据格式为json
      codec => json_lines
    }
}
#日志存储目标：es
output {
    elasticsearch {
      hosts => "192.168.75.128:9200"
      index => "springboot-logs-%{+YYYY.MM.dd}"
      codec => "json"
    }
}

3.在/home/docker/logstash/config目录下创建logstash.yml,并写入以下内容
api.http.host: 0.0.0.0
xpack.monitoring.elasticsearch.hosts: ["http://192.168.75.128:9200"]

4.执行命令安装Logstash
docker run --name logstash -m 1000M -p 9600:9600 -p 9413:9413 --privileged=true -e ES_JAVA_OPTS="-Duser.timezone=Asia/Shanghai" -v /home/docker/logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf -v /home/docker/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml -d logstash:7.17.5

5.进入容器内部
docker exec -it logstash /bin/bash

6.安装json_lines插件
logstash-plugin install --no-verify logstash-codec-json_lines
```

然后在浏览器中访问9600端口，显示结果如下



![QQ_1737711253362](D:\Desktop\bishe\docker\md图片\QQ_1737711253362.png)



## 9.安装Kibana

```shell
1.创建挂载目录
mkdir -p /home/docker/kibana/config

2.在/home/docker/kibana/config目录下创建kibana.yml文件，并写入以下内容
server.host: "0.0.0.0"
server.shutdownTimeout: "5s"
elasticsearch.hosts: [ "http://192.168.75.128:9200" ]
monitoring.ui.container.elasticsearch.enabled: true

2.执行命令安装Kibana
docker run --name kibana -p 5601:5601 -v /home/docker/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml -d kibana:7.17.5
```

然后在浏览器中访问5601端口，显示结果如下





![QQ_1737711265946](D:\Desktop\bishe\docker\md图片\QQ_1737711265946.png)



## 10.安装MongoDB

```shell
1.创建挂载目录
mkdir -p /home/docker/mongo/db

2.执行命令安装MongoDB
docker run -p 27017:27017 --name mongo -v /home/docker/mongo/db:/data/db -d mongo --auth

3.进入容器内部
docker exec -it mongo mongosh admin

4.创建用户
db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'},"readWriteAnyDatabase"]});

5.验证用户
db.auth('admin','123456')
```

使用Navicat连接MongoDB数据库，显示结果如下：



![QQ_1737711275061](D:\Desktop\bishe\docker\md图片\QQ_1737711275061.png)



## 11.安装Nacos

```shell
1.在mysql中创建nacos数据库,并在nacos数据库中执行下面的sql脚本
```

```sql
/*
* Copyright 1999-2018 Alibaba Group Holding Ltd.
*
* Licensed under the Apache License, Version 2.0 (the "License");
* you may not use this file except in compliance with the License.
* You may obtain a copy of the License at
*
*     http://www.apache.org/licenses/LICENSE-2.0
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an "AS IS" BASIS,
* WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific language governing permissions and
* limitations under the License.
*/

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info   */
/******************************************/
CREATE TABLE `config_info` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
 `data_id` varchar(255) NOT NULL COMMENT 'data_id',
 `group_id` varchar(255) DEFAULT NULL,
 `content` longtext NOT NULL COMMENT 'content',
 `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
 `src_user` text COMMENT 'source user',
 `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
 `app_name` varchar(128) DEFAULT NULL,
 `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
 `c_desc` varchar(256) DEFAULT NULL,
 `c_use` varchar(64) DEFAULT NULL,
 `effect` varchar(64) DEFAULT NULL,
 `type` varchar(64) DEFAULT NULL,
 `c_schema` text,
 `encrypted_data_key` text NOT NULL COMMENT '秘钥',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_aggr   */
/******************************************/
CREATE TABLE `config_info_aggr` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
 `data_id` varchar(255) NOT NULL COMMENT 'data_id',
 `group_id` varchar(255) NOT NULL COMMENT 'group_id',
 `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
 `content` longtext NOT NULL COMMENT '内容',
 `gmt_modified` datetime NOT NULL COMMENT '修改时间',
 `app_name` varchar(128) DEFAULT NULL,
 `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_beta   */
/******************************************/
CREATE TABLE `config_info_beta` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
 `data_id` varchar(255) NOT NULL COMMENT 'data_id',
 `group_id` varchar(128) NOT NULL COMMENT 'group_id',
 `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
 `content` longtext NOT NULL COMMENT 'content',
 `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
 `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
 `src_user` text COMMENT 'source user',
 `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
 `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
 `encrypted_data_key` text NOT NULL COMMENT '秘钥',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_tag   */
/******************************************/
CREATE TABLE `config_info_tag` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
 `data_id` varchar(255) NOT NULL COMMENT 'data_id',
 `group_id` varchar(128) NOT NULL COMMENT 'group_id',
 `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
 `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
 `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
 `content` longtext NOT NULL COMMENT 'content',
 `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
 `src_user` text COMMENT 'source user',
 `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_tags_relation   */
/******************************************/
CREATE TABLE `config_tags_relation` (
 `id` bigint(20) NOT NULL COMMENT 'id',
 `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
 `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
 `data_id` varchar(255) NOT NULL COMMENT 'data_id',
 `group_id` varchar(128) NOT NULL COMMENT 'group_id',
 `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
 `nid` bigint(20) NOT NULL AUTO_INCREMENT,
 PRIMARY KEY (`nid`),
 UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
 KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = group_capacity   */
/******************************************/
CREATE TABLE `group_capacity` (
 `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
 `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
 `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
 `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
 `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
 `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
 `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
 `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = his_config_info   */
/******************************************/
CREATE TABLE `his_config_info` (
 `id` bigint(20) unsigned NOT NULL,
 `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
 `data_id` varchar(255) NOT NULL,
 `group_id` varchar(128) NOT NULL,
 `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
 `content` longtext NOT NULL,
 `md5` varchar(32) DEFAULT NULL,
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
 `src_user` text,
 `src_ip` varchar(50) DEFAULT NULL,
 `op_type` char(10) DEFAULT NULL,
 `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
 `encrypted_data_key` text NOT NULL COMMENT '秘钥',
 PRIMARY KEY (`nid`),
 KEY `idx_gmt_create` (`gmt_create`),
 KEY `idx_gmt_modified` (`gmt_modified`),
 KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = tenant_capacity   */
/******************************************/
CREATE TABLE `tenant_capacity` (
 `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
 `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
 `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
 `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
 `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
 `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
 `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
 `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
 `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
 `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
 `kp` varchar(128) NOT NULL COMMENT 'kp',
 `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
 `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
 `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
 `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
 `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
 `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
 PRIMARY KEY (`id`),
 UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
 KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE `users` (
`username` varchar(50) NOT NULL PRIMARY KEY,
`password` varchar(500) NOT NULL,
`enabled` boolean NOT NULL
);

CREATE TABLE `roles` (
`username` varchar(50) NOT NULL,
`role` varchar(50) NOT NULL,
UNIQUE INDEX `idx_user_role` (`username` ASC, `role` ASC) USING BTREE
);

CREATE TABLE `permissions` (
   `role` varchar(50) NOT NULL,
   `resource` varchar(255) NOT NULL,
   `action` varchar(8) NOT NULL,
   UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
);

INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```

```shell
2.创建挂载目录
mkdir -p /home/docker/nacos/conf

3.在/home/docker/nacos/conf目录下创建application.properties,并写入
```

```properties
server.servlet.contextPath=/nacos
server.error.include-message=ON_PARAM
server.port=8848

### If use MySQL as datasource:
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://192.168.75.128:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=Asia/Shanghai
db.user.0=root
db.password.0=root

### Connection pool configuration: hikariCP
db.pool.config.connectionTimeout=30000
db.pool.config.validationTimeout=10000
db.pool.config.maximumPoolSize=20
db.pool.config.minimumIdle=2


nacos.naming.empty-service.auto-clean=true
nacos.naming.empty-service.clean.initial-delay-ms=50000
nacos.naming.empty-service.clean.period-time-ms=30000


management.metrics.export.elastic.enabled=false

management.metrics.export.influx.enabled=false

server.tomcat.accesslog.enabled=true

server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i

server.tomcat.basedir=file:.

nacos.security.ignore.urls=/,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-ui/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**

nacos.core.auth.system.type=nacos

nacos.core.auth.enabled=false

nacos.core.auth.caching.enabled=true

nacos.core.auth.enable.userAgentAuthWhite=false

nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=security

nacos.core.auth.plugin.nacos.token.expire.seconds=18000

nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789

nacos.istio.mcp.server.enabled=false
```

```shell
4.执行命令安装nacos
docker run -d -p 8848:8848 -p 9848:9848 -p 9849:9849 -e MODE=standalone -e JVM_XMS=256m -e JVM_XMX=256m -v /home/docker/nacos/conf/application.properties:/home/nacos/conf/application.properties --name nacos nacos/nacos-server:v2.1.1
```

然后在浏览器中访问192.168.75.128:8848/nacos，显示结果如下



![QQ_1737711295571](D:\Desktop\bishe\docker\md图片\QQ_1737711295571.png)





## 12.安装sentinel

```shell
1.执行命令安装sentinel
docker run --name sentinel -p 8858:8858 -d bladex/sentinel-dashboard
```

然后在浏览器中访问192.168.75.128:8858，显示结果如下



![QQ_1737711304217](D:\Desktop\bishe\docker\md图片\QQ_1737711304217.png)



## 13.安装Seata

```shell
1.在mysql中创建seata数据库，并在seata数据库中执行如下脚本
```

```sql
--
-- Licensed to the Apache Software Foundation (ASF) under one or more
-- contributor license agreements. See the NOTICE file distributed with
-- this work for additional information regarding copyright ownership.
-- The ASF licenses this file to You under the Apache License, Version 2.0
-- (the "License"); you may not use this file except in compliance with
-- the License. You may obtain a copy of the License at
--
--     http://www.apache.org/licenses/LICENSE-2.0
--
-- Unless required by applicable law or agreed to in writing, software
-- distributed under the License is distributed on an "AS IS" BASIS,
-- WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
-- See the License for the specific language governing permissions and
-- limitations under the License.
--

-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
   `xid`                       VARCHAR(128) NOT NULL,
   `transaction_id`            BIGINT,
   `status`                    TINYINT      NOT NULL,
   `application_id`            VARCHAR(32),
   `transaction_service_group` VARCHAR(32),
   `transaction_name`          VARCHAR(128),
   `timeout`                   INT,
   `begin_time`                BIGINT,
   `application_data`          VARCHAR(2000),
   `gmt_create`                DATETIME,
   `gmt_modified`              DATETIME,
   PRIMARY KEY (`xid`),
   KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
   KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
 DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
   `branch_id`         BIGINT       NOT NULL,
   `xid`               VARCHAR(128) NOT NULL,
   `transaction_id`    BIGINT,
   `resource_group_id` VARCHAR(32),
   `resource_id`       VARCHAR(256),
   `branch_type`       VARCHAR(8),
   `status`            TINYINT,
   `client_id`         VARCHAR(64),
   `application_data`  VARCHAR(2000),
   `gmt_create`        DATETIME(6),
   `gmt_modified`      DATETIME(6),
   PRIMARY KEY (`branch_id`),
   KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
 DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
   `row_key`        VARCHAR(128) NOT NULL,
   `xid`            VARCHAR(128),
   `transaction_id` BIGINT,
   `branch_id`      BIGINT       NOT NULL,
   `resource_id`    VARCHAR(256),
   `table_name`     VARCHAR(32),
   `pk`             VARCHAR(36),
   `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
   `gmt_create`     DATETIME,
   `gmt_modified`   DATETIME,
   PRIMARY KEY (`row_key`),
   KEY `idx_status` (`status`),
   KEY `idx_branch_id` (`branch_id`),
   KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
 DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
   `lock_key`       CHAR(20) NOT NULL,
   `lock_value`     VARCHAR(20) NOT NULL,
   `expire`         BIGINT,
   primary key (`lock_key`)
) ENGINE = InnoDB
 DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```

```shell
2.创建挂载目录
mkdir -p /home/docker/seata/resources

3.在/home/docker/seata/resources目录下创建application.yml，并写入
```

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata
  extend:
    logstash-appender:
      destination: 192.168.75.128:4560
    kafka-appender:
      bootstrap-servers: 192.168.75.128:9092
      topic: logback_to_logstash

console:
  user:
    username: seata
    password: seata

seata:
  config:
    type: nacos
    nacos:
      server-addr: 192.168.75.128:8848
      namespace:
      group: SEATA_GROUP
      username: nacos
      password: nacos
      data-id: seataServer.properties
  registry:
    type: nacos
    preferred-networks: 30.240.*
    nacos:
      application: seata-server
      server-addr: 192.168.75.128:8848
      group: SEATA_GROUP
      namespace:
      cluster: default
      username: nacos
      password: nacos
  store:
    mode: db
    db:
      datasource: druid
      db-type: mysql
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://192.168.75.128:3306/seata?rewriteBatchedStatements=true&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=Asia/Shanghai
      user: root
      password: root
      min-conn: 5
      max-conn: 100
      global-table: global_table
      branch-table: branch_table
      lock-table: lock_table
      distributed-lock-table: distributed_lock
      query-limit: 100
      max-wait: 5000
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
```

```shell
4.执行命令安装seata
docker run -d --name seata-server -p 8091:8091 -p 7091:7091 -v /home/docker/seata/resources/application.yml:/seata-server/resources/application.yml seataio/seata-server:1.5.2
```

然后在浏览器中访问192.168.75.128:7091，显示结果如下



![QQ_1737711406933](D:\Desktop\bishe\docker\md图片\QQ_1737711406933.png)



## 14.安装portainer

```shell
1.创建挂载目录
mkdir -p /home/docker/portainer/data

2.执行命令安装portainer
docker run -d -p 9000:9000 --name portainer -v /var/run/docker/sock:/var/run/docker.sock -v /home/docker/portainer/data:/data portainer/portainer
```

然后在浏览器中访问192.168.75.128:9000，显示结果如下



![QQ_1737711322425](D:\Desktop\bishe\docker\md图片\QQ_1737711322425.png)

## 15.安装docker-compose

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

