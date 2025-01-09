---
title: Docker 常用配置
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

