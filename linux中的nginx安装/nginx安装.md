# nginx安装

提前准备nginx编译资源

```
yum -y install openssl openssl-devel gd gd-devel pcre-devel zlib-devel
yum -y install gcc-c++
cd /opt 将ngixn-1.20.2.tar.gz包上传到此
tar -zvxf nginx-1.20.2.tar.gz
cd ngixn-1.20.2
./configure
make && make install
sudo /usr/local/nginx/sbin/nginx
ps -ef | grep nginx
```

# 最后测试

```
nginx -t 检查
```

如果您希望能够直接使用 nginx 命令而不需要输入完整路径，可以创建一个符号链接到 /usr/bin 目录：
sudo ln -s /usr/local/nginx/sbin/nginx /usr/bin/nginx

## 出现超时的问题

![8319ec8e55799237b81217ba575f5de6](.\nginx安装图片\8319ec8e55799237b81217ba575f5de6.png)

1.需要将nginx.service包（在文件夹中）上传到/etc/systemd/system

2.vim /usr/local/nginx/conf/nginx.conf新增pid /run/nginx.pid;自身带的删了

#### 1.nginx编译安装以哪个配置为主

![image-20250109152617202](.\nginx安装图片\image-20250109152617202.png)

```
在大多数基于源码编译安装 Nginx 的场景中，/usr/local/nginx/conf/nginx.conf 是主要的配置文件。
原因是通过 ./configure、make && make install 这样的常规编译安装流程，默认会将 Nginx 安装到 /usr/local/nginx 目录下，包括其配置文件、可执行文件等。这个目录下的配置文件会被运行在/usr/local/nginx/sbin/nginx 的 Nginx 进程所使用。
/opt/nginx - 1.20.2/conf/nginx.conf 的作用：
/opt/nginx - 1.20.2/conf/nginx.conf 所在目录可能是 Nginx 源码解压的目录（nginx - 1.20.2 是源码包的目录名）。这个目录下的 nginx.conf 通常是源码中提供的示例配置文件，用于参考或作为初始配置的模板。在编译安装过程中，它不会被直接作为运行时的配置文件，除非你在安装过程中进行了特殊的指定或配置。
确认方法：
你可以通过查看正在运行的 Nginx 进程的配置文件路径来进一步确认。例如，在 Linux 系统上，可以使用以下命令：

/usr/local/nginx/sbin/nginx -t

该命令会测试 Nginx 配置的语法正确性，并显示当前使用的配置文件路径。如果配置无误，输出中会显示类似 nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok 这样的信息，明确指出当前使用的配置文件。
所以，通常情况下 /usr/local/nginx/conf/nginx.conf 是运行时的主要配置文件。

```



#### 2. Nginx 开机自启

如果 Nginx 安装后已经生成了 systemd 服务单元文件（通常在`/usr/lib/systemd/system/nginx.service`），可以使用以下命令设置开机自启：

```
sudo systemctl enable nginx
```

如果没有生成服务单元文件，你可以手动创建一个。在`/etc/systemd/system/`目录下创建一个名为`nginx.service`的文件，内容如下：

```
[Unit]
Description=nginx service
After=network.target 
   
[Service] 
Type=forking 
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true 
   
[Install] 
WantedBy=multi-user.target
```

```
[Unit]:服务的说明
Description:描述服务
After:描述服务类别
[Service]服务运行参数的设置
Type=forking是后台运行的形式
ExecStart为服务的具体运行命令
ExecReload为重启命令
ExecStop为停止命令
PrivateTmp=True表示给服务分配独立的临时空间
注意：[Service]的启动、重启、停止命令全部要求使用绝对路径
[Install]运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3
 
保存退出。
```

保存文件后，执行以下命令：

```
sudo systemctl daemon-reload
sudo systemctl enable nginx
```



# nginx启动

```
nginx，systemctl start nginx
```

# nginx加载

```
nginx -s reload
```



