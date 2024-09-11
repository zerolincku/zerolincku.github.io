---
title: Linux 防火墙
date: 2024-09-11 09:19:00
tags:
categories:
  - linux
---

## CentOS

### <font style="color:rgb(0, 0, 0);">设置开机启用防火墙</font>
```bash
systemctl enable firewalld.service
```

### <font style="color:rgb(0, 0, 0);">设置开机禁用防火墙</font>
```bash
systemctl disable firewalld.service
```

### <font style="color:rgb(0, 0, 0);">启动防火墙</font>
```bash
systemctl start firewalld
```

### <font style="color:rgb(0, 0, 0);">关闭防火墙</font>
```bash
systemctl stop firewalld
```

### <font style="color:rgb(0, 0, 0);">检查防火墙状态</font>
```bash
systemctl status firewalld
```

### <font style="color:rgb(0, 0, 0);">开启防火墙端口</font>
```bash
firewall-cmd --zone=public --add-port=9200/tcp --permanent

firewall-cmd --zone=public --add-port=1000-9000/tcp --permanent
```

### <font style="color:rgb(0, 0, 0);">重新加载配置</font>
```bash
firewall-cmd --reload
```

### <font style="color:rgb(0, 0, 0);">关闭防火墙端口</font>
```bash
firewall-cmd --zone=public --remove-port=9200/tcp --permanent
```

