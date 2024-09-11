---
title: Linux 防火墙
date: 2024-09-11 09:19:00
tags:
  - linux
---

## CentOS

### 设置开机启用防火墙
```bash
systemctl enable firewalld.service
```

### 设置开机禁用防火墙
```bash
systemctl disable firewalld.service
```

### 启动防火墙
```bash
systemctl start firewalld
```

### 关闭防火墙
```bash
systemctl stop firewalld
```

### 检查防火墙状态
```bash
systemctl status firewalld
```

### 开启防火墙端口
```bash
firewall-cmd --zone=public --add-port=9200/tcp --permanent

firewall-cmd --zone=public --add-port=1000-9000/tcp --permanent
```

### 重新加载配置
```bash
firewall-cmd --reload
```

### 关闭防火墙端口
```bash
firewall-cmd --zone=public --remove-port=9200/tcp --permanent
```

