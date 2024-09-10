---
title: 在CentOS上安装Docker
date: 2024-07-05 22:38:18
tags:
categories:
  - docker
---

## 先决条件

### 操作系统要求

需要 CentOS 7 或 8 的版本

### 卸载旧版本Docker

~~~shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
 # 卸载 Docker Engine、CLI 和 Containerd 包
$ sudo yum remove docker-ce docker-ce-cli containerd.io

# 删除所有镜像、容器和卷
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
~~~

## 使用yum安装在线安装

### 设置存储库

安装 `yum-utils`包和设置存储库

~~~shell
# 安装 `yum-utils`包
$ sudo yum install -y yum-utils
# 设置存储库
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
~~~

### 安装Docker引擎

安装最新版的Docker

~~~shell
$ sudo yum install docker-ce docker-ce-cli containerd.io
~~~

如果要安装特定版本，需要先查询存储库中可用的版本列表，第二列是 `VERSION`，第三列的 `stable` 表示是稳定版

~~~shell
$ yum list docker-ce --showduplicates | sort -r
Loading mirror speeds from cached hostfile
Loaded plugins: fastestmirror
docker-ce.x86_64            3:20.10.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.8-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.7-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.6-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.5-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.4-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.3-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.2-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.1-3.el7                     docker-ce-stable
docker-ce.x86_64            3:20.10.0-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.9-3.el7                     docker-ce-stable
docker-ce.x86_64            3:19.03.8-3.el7                     docker-ce-stable
~~~

安装特定的版本

~~~shell
$ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
~~~

启动Docker

~~~shell
$ sudo systemctl start docker
~~~

通过运行`docker ps`验证 Docker Engine 是否已正确安装。

~~~shell
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
~~~

此命令下载测试映像并在容器中运行它。当容器运行时，它会打印一条消息并退出。

## 离线安装Docker

### 下载rpm的离线安装包

[下载地址](https://download.docker.com/linux/centos/ )，选择CentOS对应的版本，建议下载 stable 稳定版

> 使用 `rpm -q centos-release`可以查看CentOS版本

例如，下载了
containerd.io-1.4.9-3.1.el7.x86_64.rpm
docker-ce-20.10.9-3.el7.x86_64.rpm
docker-ce-cli-20.10.9-3.el7.x86_64.rpm
docker-ce-rootless-extras-20.10.9-3.el7.x86_64.rpm
docker-ce-selinux-17.03.3.ce-1.el7.noarch.rpm
docker-scan-plugin-0.8.0-3.el7.x86_64.rpm

### 离线安装

将离线安装包上传到目标服务器，进入安装包所在目录

~~~shell
$ sudo rpm -Uvh *.rpm --nodeps --force
~~~

## 卸载Docker引擎

~~~shell
# 卸载 Docker Engine、CLI 和 Containerd 包
$ sudo yum remove docker-ce docker-ce-cli containerd.io

# 删除所有镜像、容器和卷
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
~~~

