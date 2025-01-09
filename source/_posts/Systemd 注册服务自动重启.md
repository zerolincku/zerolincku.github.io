---
title: systemd注册服务自动重启
date: 2024-09-11 09:18:00
tags:
  - linux
---


使用 systemd 注册服务，可以配置使得进程在崩溃宕机的时候自动重启

在大多数 Linux 发行版中，systemd 服务单元文件通常存储在 /etc/systemd/system 目录中。需要将 .service 后缀的服务单元文件放在该目录下，并确保其权限为 644.

一旦将服务单元文件放在正确的位置，就可以使用 systemctl 命令启用和管理服务。例如，要启用名称为 "management.service" 的服务，请运行以下命令：

```plain
sudo systemctl enable management.service
```

然后，可以使用以下命令来检查服务状态：

```plain
systemctl status management.service
```

要启动或停止服务，请使用以下命令：

```plain
sudo systemctl start management.service
sudo systemctl stop management.service
```

注意，如果对 systemd 进行任何更改（如创建、编辑或删除服务），都需要运行以下命令重新加载 systemd 配置：

```plain
sudo systemctl daemon-reload
```

这会告诉 systemd 重新加载配置文件并更新内部配置数据库。

下面是一个 service 的示例文件

```bash
[Unit]
Description=management
After=network.target

[Service]
Type=simple
ExecStart=/opt/jdk-11/bin/java -jar /opt/management/management-1.0.1.jar
ExecReload=/bin/kill -HUP ${MAINPID}
Restart=on-failure
RestartSec=10s
ExecStop=/bin/kill -9 ${MAINPID}

[Install]
WantedBy=multi-user.target
```

