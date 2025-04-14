# 1.**什么是 DPanel ？**

> `DPanel` 是一款 `Docker` 可视化管理面板，旨在简化 `Docker` 容器、镜像和文件的管理。它提供了一系列功能，使用户能够更轻松地管理和部署 `Docker` 环境。

## 2.docker-compose安装

```

sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 3.用docker-compose 安装

所以，我们也可以用 `docker-compose` 一起安装，这样话，就算是第一次打开文件管理也是很快的

将下面的内容保存为 `docker-compose.yml` 文件

```
version: '3'

services:
  dpanel:
    image: registry.cn-hangzhou.aliyuncs.com/dpanel/dpanel:latest
    # image: dpanel/dpanel:latest
    container_name: dpanel
    restart: unless-stopped
    ports:
      - 8807:8080 
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/dpanel
    environment:
      - APP_NAME=dpanel
      - INSTALL_USERNAME=admin
      - INSTALL_PASSWORD=admin
      
  dpanel-plugin-explorer:
    image: alpine:latest    
    container_name: dpanel-plugin-explorer
    restart: unless-stopped
    privileged: true
    pid: host
    command: ["sh", "-c", "tail -f /dev/null"]

```

- `APP_NAME` ：请保持与 `container_name` 一致；
- `INSTALL_USERNAME`：用于指定用户名；
- `INSTALL_PASSWORD`：用于指定密码；

然后执行下面的命令

```
# 新建文件夹 dpanel 和 子目录
mkdir -p /volume1/docker/dpanel/data

# 进入 dpanel 目录
cd /volume1/docker/dpanel

# 将 docker-compose.yml 放入当前目录

# 一键启动
docker-compose up -d
```

节点用香港的

## 运行

在浏览器中输入 `http://群晖IP:8807` 就能看到登录界面

![QQ_1739845247734](.\md图片\QQ_1739845247734.png)