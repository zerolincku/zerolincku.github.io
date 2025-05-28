---
title: yum安装mysql客户端
date: 2025-05-28 08:19:00
tags:
  - mysql
---

```shell
sudo yum localinstall https://dev.mysql.com/get/mysql80-community-release-el8-3.noarch.rpm
sudo yum install mysql-community-client --nogpgcheck -y
```

