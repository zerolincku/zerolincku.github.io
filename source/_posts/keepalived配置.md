---
title: keepalived配置
date: 2024-12-09 22:38:18
tags:
  - keepalived
---


~~~bash
cat /etc/keepalived/keepalived.conf

vrrp_script chk_mysql {
    script "/usr/local/bin/mysql_check.sh"
    interval 2
    weight 30
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp3s0
    virtual_router_id 51
    priority 90
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.76.240
    }

    track_script {
        chk_mysql
    }
}

cat /usr/local/bin/mysql_check.sh                               
#!/bin/bash
if ! mysqladmin ping -h 127.0.0.1 -P 3301 --silent; then
    exit 1
else
    exit 0
fi
~~~

