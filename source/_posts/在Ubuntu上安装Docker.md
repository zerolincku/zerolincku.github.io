---
title: 在Ubuntu上安装Docker
date: 2024-07-05 22:38:18
tags:
  - docker
---

## 先决条件

### 操作系统要求



### 卸载旧版本Docker

~~~shell
$ sudo apt-get remove docker docker-engine docker.io containerd runc

# 卸载 Docker Engine、CLI 和 Containerd 包
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io

# 删除所有镜像、容器和卷
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
~~~

## 使用apt-get安装在线安装

### 设置存储库

更新`apt`包索引并安装包以允许`apt`通过 HTTPS 使用存储库

~~~shell
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
~~~

添加Docker官方的GPG密钥

~~~shell
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
~~~

### 安装Docker引擎

安装最新版的Docker

~~~shell
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
~~~

如果要安装特定版本，需要先查询存储库中可用的版本列表，第二列是 `VERSION`，`stable` 表示是稳定版

~~~shell
$ apt-cache madison docker-ce


docker-ce | 5:18.09.1~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
docker-ce | 5:18.09.0~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
docker-ce | 18.06.1~ce~3-0~ubuntu       | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
docker-ce | 18.06.0~ce~3-0~ubuntu       | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
~~~

安装特定的版本，例如`5:18.09.1~3-0~ubuntu-xenial`.

~~~shell
$ sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
~~~

通过运行`docker ps`验证 Docker Engine 是否已正确安装。

~~~shell
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
~~~

此命令下载测试映像并在容器中运行它。当容器运行时，它会打印一条消息并退出。

## 离线安装Docker

### 下载deb的离线安装包

[下载地址](https://download.docker.com/linux/ubuntu/dists/ )，选择你的Ubuntu版本，然后浏览`pool/stable/`，选择`amd64`， `armhf`，`arm64`，或`s390x`，并下载`.deb`文件，建议下载 stable 稳定版

例如，下载了

```ruby
containerd.io_1.4.9-1_amd64.deb
docker-ce-cli_20.10.9~3-0~ubuntu-bionic_amd64.deb
docker-ce-rootless-extras_20.10.9~3-0~ubuntu-bionic_amd64.deb
docker-ce_20.10.9~3-0~ubuntu-bionic_amd64.deb
docker-scan-plugin_0.8.0~ubuntu-bionic_amd64.deb
```

### 离线安装

将离线安装包上传到目标服务器，进入安装包所在目录

~~~shell
$ sudo dpkg -i containerd.io_1.4.9-1_amd64.deb
$ sudo dpkg -i docker-ce-cli_20.10.9~3-0~ubuntu-bionic_amd64.deb
$ sudo dpkg -i docker-ce-rootless-extras_20.10.9~3-0~ubuntu-bionic_amd64.deb
$ sudo dpkg -i docker-ce_20.10.9~3-0~ubuntu-bionic_amd64.deb
$ sudo dpkg -i docker-scan-plugin_0.8.0~ubuntu-bionic_amd64.deb
~~~

## 卸载Docker引擎

~~~shell
# 卸载 Docker Engine、CLI 和 Containerd 包
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io

# 删除所有镜像、容器和卷
$ sudo rm -rf /var/lib/docker
$ sudo rm -rf /var/lib/containerd
~~~

