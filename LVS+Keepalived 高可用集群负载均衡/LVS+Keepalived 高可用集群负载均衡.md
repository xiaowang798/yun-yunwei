# 1.实验练习

实验环境：

```
主keepalived服务器：192.168.47.110
备keepalived服务器：192.168.47.111
web1服务器：192.168.47.112
web2服务器：192.168.47.113
```

#### **1.配置负载调度器（主keepalived服务器：192.168.47.110）**

 1.关闭防火墙

```
systemctl stop firewalld.service
setenforce 0
```

2.安装服务

```
yum install ipvsadm keepalived -y
```

3.修改配置文件keeplived.conf

```
cd /etc/keepalived/
cp keepalived.conf keepalived.conf.bak
vim keepalived.conf
```

```
! Configuration File for keepalived

global_defs {
   router_id lvs_node1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.175.19
    }
}

virtual_server 192.168.111.19 80{
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 5
    protocol TCP

    real_server 192.168.111.11 80 {
        weight 1
    TCP_CHECK {
    	connect_port 80
    	connect_timeout 2
    	nb_get_retry 2
    	delay_before_retry 3
    }
    }
    real_server 192.168.111.12 80 {
        weight 1
    TCP_CHECK {
    	connect_port 80
    	connect_timeout 2
    	nb_get_retry 2
    	delay_before_retry 3
    }
    }
}

```

4.启动服务、查看虚拟网卡vip

```
systemctl start keepalived
ip addr show dev ens33
```

5.调整proc响应参数，关闭Linux内核的重定向参数响应

```
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.ens33.send_redirects = 0
```

6.刷新一下

```
sysctl -p
```

############################# 配置ipvsadm #################################

7.配置负载分配策略，并启动服务

```
ipvsadm-save >/etc/sysconfig/ipvsadm
systemctl start ipvsadm.service
```

8.清空ipvsadm，并做策略

```
ipvsadm -C
ipvsadm -A -t 192.168.111.19:80 -s rr
ipvsadm -a -t 192.168.111.19:80 -r 192.168.111.13:80 -g
ipvsadm -a -t 192.168.111.19:80 -r 192.168.111.14:80 -g
```

9.保存设置

```
ipvsadm
ipvsadm -ln
ipvsadm-save >/etc/sysconfig/ipvsadm
```

#### 2.配置负载调度器（备keepalived服务器：192.168.47.111）

1.关闭防火墙

```
systemctl stop firewalld.service
setenforce 0
```

2.安装服务

```
yum install ipvsadm keepalived -y
```

3.修改配置文件keeplived.conf

```
cd /etc/keepalived/
cp keepalived.conf keepalived.conf.bak
vim keepalived.conf
```

```
! Configuration File for keepalived

global_defs {
   router_id lvs_node1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.175.19
    }
}

virtual_server 192.168.175.19 80{
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 5
    protocol TCP

    real_server 192.168.175.111 80 {
        weight 1
    TCP_CHECK {
    	connect_port 80
    	connect_timeout 2
    	nb_get_retry 2
    	delay_before_retry 3
    }
    }
    real_server 192.168.175.112 80 {
        weight 1
    TCP_CHECK {
    	connect_port 80
    	connect_timeout 2
    	nb_get_retry 2
    	delay_before_retry 3
    }
    }
}
```

4.启动服务、查看虚拟网卡vip

```
systemctl start keepalived
ip addr show dev ens33
```

5.调整proc响应参数，关闭Linux内核的重定向参数响应

```
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.ens33.send_redirects = 0
```

6.刷新一下

```
sysctl -p
```

############################# 配置ipvsadm #################################

7.配置负载分配策略，并启动服务

```
ipvsadm-save >/etc/sysconfig/ipvsadm
systemctl start ipvsadm.service
```

8.清空ipvsadm，并做策略

```
ipvsadm -C
ipvsadm -A -t 192.168.111.19:80 -s rr
ipvsadm -a -t 192.168.111.19:80 -r 192.168.111.13:80 -g
ipvsadm -a -t 192.168.111.19:80 -r 192.168.111.14:80 -g
```

9.保存设置

```
ipvsadm
ipvsadm -ln
ipvsadm-save >/etc/sysconfig/ipvsadm
```

#### **3.配置节点服务器（web1服务器：192.168.47.112）**

1. ##### 关闭防火墙

  ```
  systemctl stop firewalld
  setenforce 0
  ```

2. 配置虚拟vip

  ```
  vim /etc/sysconfig/network-scripts/ifcfg-lo:0
  DEVICE=lo:0
  ONBOOT=yes
  IPADDR=192.168.111.19
  NETMASK=255.255.255.255
  ```

3. 重启网络服务，开启虚拟网卡

  ```
  systemctl restart network
  ifup lo:0
  ifconfig lo:0
  ```

4. 设置路由

  ```
  route add -host 192.168.111.19 dev lo:0
  route -n
  ```

5. 调整 proc 响应参数
    #添加系统只响应目的IP为本地IP的ARP请求
    #系统不使用原地址来设置ARP请求的源地址，而是物理mac地址上的IP

  ```
  vim /etc/sysctl.conf
  ```

  ```
  net.ipv4.conf.all.arp_ignore = 1
  net.ipv4.conf.all.arp_announce = 2
  net.ipv4.conf.default.arp_ignore = 1
  net.ipv4.conf.default.arp_announce = 2
  net.ipv4.conf.lo.arp_ignore = 1
  net.ipv4.conf.lo.arp_announce = 2
  ```

6. 刷新proc参数

  ```
  sysctl -p
  ```

#### **配置节点服务器（web2服务器：192.168.47.113）**

1. 关闭防火墙

  ```
  systemctl stop firewalld
  setenforce 0
  ```

2. #配置虚拟vip

  ```
  vim /etc/sysconfig/network-scripts/ifcfg-lo:0
  ```

  ```
  DEVICE=lo:0
  ONBOOT=yes
  IPADDR=192.168.111.19
  NETMASK=255.255.255.255
  ```

3. #重启网络服务，开启虚拟网卡

  ```
  systemctl restart network
  ifup lo:0
  ifconfig lo:0
  ```

4. 设置路由

  ```
  route add -host 192.168.111.19 dev lo:0
  route -n
  ```

  调整 proc 响应参数
  添加系统只响应目的IP为本地IP的ARP请求
  系统不使用原地址来设置ARP请求的源地址，而是物理mac地址上的IP

  ```
  vim /etc/sysctl.conf
  ```

  ```
  net.ipv4.conf.all.arp_ignore = 1
  net.ipv4.conf.all.arp_announce = 2
  net.ipv4.conf.default.arp_ignore = 1
  net.ipv4.conf.default.arp_announce = 2
  net.ipv4.conf.lo.arp_ignore = 1
  net.ipv4.conf.lo.arp_announce = 2
  ```

5. 刷新proc参数

  ```
  sysctl -p
  ```

#####   常用命令

```
常用命令
查看LVS配置 ipvsadm -L -n
添加虚拟服务器 ipvsadm -A -t : -s
删除虚拟服务器 ipvsadm -D -t :
添加后端服务器 ipvsadm -a -t : -r : -
删除后端服务器 ipvsadm -d -t : -r :
清空LVS配置 ipvsadm -C 保存LVS配置 ipvsadm-save > /etc/sysconfig/ipvsadm
恢复LVS配置 ipvsadm-restore < /etc/sysconfig/ipvsadm
```







