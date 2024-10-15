---
title: Linux 创建用户
date: 2024-10-15 09:20:00
tags:
  - linux
---

## 1. 创建 `ideal` 用户

使用 `useradd` 命令创建新用户 `ideal`：

~~~bash
sudo useradd -m -s /bin/bash ideal
~~~

* `-m`：创建用户的 home 目录。

* `-s /bin/bash`：指定用户使用的 shell（例如 Bash）。

## 2. 设置密码

为 `ideal` 用户设置密码：

~~~bash
sudo passwd ideal
~~~

## 3. 确保用户可以通过 SSH 登录

* 编辑 SSH 配置文件 `/etc/ssh/sshd_config`，确保以下内容启用：

  ~~~bash
  PermitRootLogin no
  PasswordAuthentication yes
  ~~~

* 确保 `PasswordAuthentication` 是启用的，以允许通过密码进行 SSH 登录（如果你计划使用密码登录而不是公钥登录）。

* 然后重启 SSH 服务以应用配置更改：

  ~~~bash
  sudo systemctl restart sshd
  ~~~

## 4. 将用户添加到 `docker` 组（如果需要运行 docker）

将用户 `ideal` 添加到 `docker` 组，以允许该用户管理 Docker 容器：

~~~bash
sudo usermod -aG docker ideal
~~~

* `-aG`：将用户附加到 `docker` 组。

## 5. 验证用户组

要验证 `ideal` 是否成功添加到 `docker` 组，可以运行：

~~~bash
id ideal
~~~

## 6. 添加 sudo 权限（如果需要）

### 方法1：将用户添加到 `sudo` 组（适用于大多数发行版）

1. 使用 root 用户或具有 sudo 权限的用户运行以下命令：

   ~~~bash
   usermod -aG sudo username
   ~~~

2. 验证用户是否已添加到 `sudo` 组：

   ~~~bash
   groups username
   ~~~

### 方法 2：编辑 `/etc/sudoers` 文件

1. 使用 `visudo` 命令编辑 sudoers 文件，以防止语法错误：

   ~~~bash
   visudo
   ~~~

2. 在文件中找到以下行

   ~~~bash
   root    ALL=(ALL:ALL) ALL
   ~~~

3. 在下面添加你需要添加权限的用户：

   ~~~bash
   username    ALL=(ALL:ALL) ALL
   ~~~

   