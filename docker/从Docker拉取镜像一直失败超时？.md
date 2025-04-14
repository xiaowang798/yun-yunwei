## 解决目前无法访问，超时连接方法

**解决方案1：配置加速地址**

配置加速地址：适用于Ubuntu 16.04+、Debian 8+、CentOS 7+

# 方式一：使用以下命令设置registry mirror：但是需要重启docker服务

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://do.nark.eu.org",
        "https://dc.j8.work",
        "https://docker.m.daocloud.io",
        "https://dockerproxy.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://docker.nju.edu.cn"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

```

**检查加速是否生效：**
查看docker系统信息 docker info，如果从输出结果中看到了 registry mirror 刚配置的内容地址，说明配置成功。

# 方式二：

第三方镜像：

AtomHub 可信镜像中心 - 大部分需要的镜像都是有的。
可信镜像中心官网：https://atomhub.openatom.cn/
通过搜索需要的镜像名称，进行pull拉取，用法示例：

docker pull atomhub.openatom.cn/amd64/redis:7.0.13
1
注意：docker compose 中要执行部署时，可以把版本与 atomhub 提供的版本匹配上，之后通过【拉取命令】进行单独拉取后，在执行 docker compose 就可以了。

# 方式三：

```
方式一：直接获取 Docker Hub 镜像
docker pull docker.rainbond.cc/library/node:20
docker pull docker.rainbond.cc/rainbond/rainbond:v5.17.2-release-allinone

方式二：配置镜像加速器
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://docker.rainbond.cc"]
}
EOF
systemctl daemon-reload
systemctl restart docker

技术栈参考LINK
https://www.rainbond.com/docs/quick-start/quick-install

```

