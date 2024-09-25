---
title: rssocks 配置与使用
date: 2024-09-25 14:19:00
tags:
  - linux
---

### 安装

~~~shell
apt-get install redsocks
~~~

### 配置

~~~shell
vim /etc/redsocks.conf

base {
        log_debug = off;
        log_info = on;
        log = "syslog:daemon";
        daemon = on;
        redirector = iptables;
}

redsocks {
				# 本地端口
        local_ip = 127.0.0.1;
        local_port = 2001;
        # 代理服务器
        ip = 192.168.1.1;
        port = 3333;
        type = socks5;
}
~~~

### iptables 配置

~~~shell
# 10.1.17.111 为要走代理的 ip
iptables -t nat -A OUTPUT -p tcp -d 10.1.17.111/32 -j REDIRECT --to-port 2001
~~~

### 启动与查看日志

~~~shell
systemctl start redsocks

journalctl -u redsocks [-f]
~~~

