---
title: Docker 常用配置和命令
date: 2025-01-09 22:38:18
tags:
  - docker
---

## Docker 常用配置

Docker 的配置文件通常位于 `/etc/docker/daemon.json`。可以在这个文件中进行一些关键配置。

#### 日志驱动

配置日志驱动以管理日志文件的大小和数量，避免磁盘空间耗尽。

~~~json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
~~~

#### **数据存储位置**

默认情况下，Docker 存储数据在 `/var/lib/docker` 目录下。如果需要更改数据存储位置，可以配置 `data-root`。

~~~json
{
  "data-root": "/mnt/docker-data"
}
~~~

#### 配置镜像仓库

~~~json
{
    "insecure-registries": ["192.168.80.101:5000"],
    "registry-mirrors": ["https://9yhxvwku.mirror.aliyuncs.com"]
}

insecure-registries    私有仓库地址
registry-mirrors   镜像加速地址，也就是第三方仓库，也可以改成自己的仓库地址http://192.168.10.7:666，这样docker pull的时候就不用加上私有仓库的地址和端口了。

~~~

## Docker 常用操作

#### 检查容器日志大小

```bash
ls -lh /var/lib/docker/containers/<container-id>/<container-id>-json.log
```

#### 列出所有 Docker 容器日志的大小

```bash
find /var/lib/docker/containers/ -name "*json.log" | xargs du -h | sort -hr
```

