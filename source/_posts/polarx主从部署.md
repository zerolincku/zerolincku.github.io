---
title: polarx 主从部署
date: 2025-01-23 09:20:00
tags:
  - database
---

## 系统配置

1. 集群内的所有机器都能访问互联网。
2. 在集群内的所有机器上安装 Docker。
3. 集群机器建配置免密登录。选择任意一台机器作为部署机，配置部署机到集群所有机器的免密登录。

~~~bash
# 生成密钥对
ssh-keygen -t rsa

# 复制免登公钥到目标机器，修改user和ip
# 部署机也需要能免密登录自己
ssh-copy-id {user}@{ip}


# 例如集群中有如下三台机器：
#192.168.1.100 # PXD 部署机
#192.168.1.101
#192.168.1.102

# 则需要在192.168.1.100 上执行如下命令完成免密配置：
ssh-keygen -t rsa

ssh-copy-id {user}@192.168.1.100
ssh-copy-id {user}@192.168.1.101
ssh-copy-id {user}@192.168.1.102
~~~

## 在部署机上安装 PXD

选择任意一台机器作为部署机，在这台机器上安装 PXD 即可。PXD 会通过部署机在集群内创建 PolarDB-X 数据库。

#### 准备工作

安装 Python3，如果你的机器上已经安装了 python3，可以跳过。建议使用 python3.7，高版本 python 可能会出现 PyYAML 不兼容，需要修改代码处理。

1. 创建一个 Python3 的 virtual environment 环境并激活

~~~bash
python3 -m venv venv
source venv/bin/activate
~~~

#### 按照 PXD

~~~~bash
# 安装前建议先执行如下命令升级pip
pip install --upgrade -i https://mirrors.aliyun.com/pypi/simple/ pip
pip install -i https://mirrors.aliyun.com/pypi/simple/ pxd
~~~~

#### 注意事项

如果使用的是高版本 python 可能会出现 PyYAML 不兼容，无法下载 PyYAML 5.4.1，可以使用如下方案

1. 手动安装 pxd 所需依赖

   ~~~bash
   pip install -i https://mirrors.aliyun.com/pypi/simple/ certifi==2021.5.30
   pip install -i https://mirrors.aliyun.com/pypi/simple/ charset-normalizer==2.0.4
   pip install -i https://mirrors.aliyun.com/pypi/simple/ click==8.0.1
   pip install -i https://mirrors.aliyun.com/pypi/simple/ docker==5.0.0
   pip install -i https://mirrors.aliyun.com/pypi/simple/ idna==3.2
   pip install -i https://mirrors.aliyun.com/pypi/simple/ colorama==0.4.4
   pip install -i https://mirrors.aliyun.com/pypi/simple/ pycryptodomex==3.10.1
   pip install -i https://mirrors.aliyun.com/pypi/simple/ PyMySQL==1.0.2
   pip install -i https://mirrors.aliyun.com/pypi/simple/ PyYAML
   pip install -i https://mirrors.aliyun.com/pypi/simple/ requests==2.26.0
   pip install -i https://mirrors.aliyun.com/pypi/simple/ retrying==1.3.3
   pip install -i https://mirrors.aliyun.com/pypi/simple/ six==1.16.0
   pip install -i https://mirrors.aliyun.com/pypi/simple/ urllib3==1.26.6
   pip install -i https://mirrors.aliyun.com/pypi/simple/ websocket-client==1.2.1
   pip install -i https://mirrors.aliyun.com/pypi/simple/ spurplus==2.3.4
   pip install -i https://mirrors.aliyun.com/pypi/simple/ humanfriendly==10.0
   pip install -i https://mirrors.aliyun.com/pypi/simple/ packaging>=21.0
   ~~~

2. 忽略依赖安装 pxd

   ~~~bash
   pip install -i https://mirrors.aliyun.com/pypi/simple/ pxd --no-deps
   ~~~

3. 修改 `venv/lib/python3.11/site-packages/deployer/pxc/polardbx_manager.py`

   ~~~python
   data = yaml.load(stream)
   # 修改为
   data = yaml.load(stream, Loader=yaml.FullLoader)
   ~~~

## 创建 PolarDB-X 集群

#### 准备 PolarDB-X 标准版拓扑文件

> PolarDB-X 标准版采用一主一备一日志的三节点架构，性价比高，通过多副本同步复制，确保数据的强一致性。面向具备超高并发、复杂查询及轻量分析的在线业务场景。

首先执行如下命令获取 PolarDB-X 各个组件的最新镜像版本（需要填入YAML文件）：

~~~bash
curl -s "https://polardbx-opensource.oss-cn-hangzhou.aliyuncs.com/scripts/get-version.sh" | sh
~~~

输出内容如下所示

~~~bash
CN polardbx-opensource-registry.cn-beijing.cr.aliyuncs.com/polardbx/polardbx-sql:v2.4.1_5.4.19
DN polardbx-opensource-registry.cn-beijing.cr.aliyuncs.com/polardbx/polardbx-engine:v2.4.1_8.4.19
CDC polardbx-opensource-registry.cn-beijing.cr.aliyuncs.com/polardbx/polardbx-cdc:v2.4.1_5.4.19
COLUMNAR polardbx-opensource-registry.cn-beijing.cr.aliyuncs.com/polardbx/polardbx-columnar:v2.4.1_5.4.19
~~~

PolarDB-X 标准版无CN、GMS与CDC，仅具有一个DN节点。编写如下的 YAML 文件，指定 PolarDB-X 标准版的名称与三节点的拓扑信息

~~~bash
# polarx.yaml
version: v1
type: polardbx
cluster:
  name: pxc_test
  dn:
    image: polardbx-opensource-registry.cn-beijing.cr.aliyuncs.com/polardbx/polardbx-engine:v2.4.1_8.4.19
    replica: 1
    nodes:
      - host_group: [127.0.0.1,127.0.0.1,127.0.0.1]
    resources:
      mem_limit: 2G
~~~

通过以上拓扑文件创建的 PolarDB-X 标准版集群。拓扑文件包括如下属性:

- version: 拓扑文件版本，无需修改
- type: polardbx， 无需修改
- cluster.name：PolarDB-X 集群名称
- cluster.dn
  - image: 数据节点镜像名称，建议填上述命令的获取到的 DN 镜像，如不填，默认为最新镜像
  - replica: 数据节点数目，标准版中默认设置为 1
  - nodes: 存储节点的host_group列表，一个 host_group 表示一个dn节点多副本的部署机器，比如Paxos三副本的话需要填入三个ip，例如：[172.16.1.11,172.16.1.12,172.16.1.13]。三副本集群的Leader节点将从前两个ip的节点上随机选出
  - resources: 存储节点使用的资源
    - mem_limit: 内存上限，默认 2G

## 创建 PolarDB-X 集群

~~~bash
pxd create -file polarx.yaml

#输出
Processing [##################------------------] 50% create dn
Processing [#################################---] 92% wait PolarDB-X ready
Processing [####################################] 100%

PolarDB-X cluster create successfully, you can try it out now.
Connect PolarDB-X using the following command:
mysql -h127.0.0.1 -P17825 -uadmin -prafpsDOQ
~~~

#### 注意事项

自动创建的 admin 账户无 GRANT 授权权限，需要使用 admin 账户修改 root 账户密码，再使用 root 账户操作

~~~SQL
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
RENAME USER 'root'@'localhost' TO 'root'@'%';
~~~

## ProxySQL

ProxySQL作为一款成熟的MySQL中间件，能够无缝对接MySQL协议支持PolarDB-X，并且支持故障切换，动态路由等高可用保障

#### 配置PolarDB-X标准版

1. 创建依赖视图，目的让ProxySQL识别PolarDB-X标准版的元数据（Leader、Follower）

   ~~~SQL
   CREATE VIEW sys.gr_member_routing_candidate_status AS 
   SELECT IF(ROLE='Leader' OR ROLE='Follower', 'YES', 'NO' ) as viable_candidate,
          IF(ROLE <>'Leader', 'YES', 'NO' ) as read_only,
          IF (ROLE = 'Leader', 0, LAST_LOG_INDEX - LAST_APPLY_INDEX) as transactions_behind,
          0 as 'transactions_to_cert' 
   FROM information_schema.ALISQL_CLUSTER_LOCAL;
   # 创建proxysql的监控账户，ProxySQL运行通用依赖
   create user 'proxysql_monitor'@'%' identified with mysql_native_password by '123456';
   GRANT SELECT on sys.* to 'proxysql_monitor'@'%';
   ~~~

2. 创建代理连接账户

   ~~~SQL
   create user 'admin2'@'%' identified with mysql_native_password by '123456';
   GRANT all privileges on *.* to 'admin2'@'%';
   ~~~

3. 检查配置是否生效

   ~~~sql
   # 本文PolarDB-X标准版的三节点均部署本地，监听端口分别为3301/3302/3303
   
   mysql> select @@port;
   +--------+
   | @@port |
   +--------+
   |   3301 |
   +--------+
   1 row in set (0.00 sec)
   
   mysql> select * from information_schema.ALISQL_CLUSTER_GLOBAL;
   +-----------+-----------------+-------------+------------+----------+-----------+------------+-----------------+----------------+---------------+------------+--------------+
   | SERVER_ID | IP_PORT         | MATCH_INDEX | NEXT_INDEX | ROLE     | HAS_VOTED | FORCE_SYNC | ELECTION_WEIGHT | LEARNER_SOURCE | APPLIED_INDEX | PIPELINING | SEND_APPLIED |
   +-----------+-----------------+-------------+------------+----------+-----------+------------+-----------------+----------------+---------------+------------+--------------+
   |         1 | 127.0.0.1:16800 |           1 |          0 | Leader   | Yes       | No         |               5 |              0 |             0 | No         | No           |
   |         2 | 127.0.0.1:16801 |           1 |          2 | Follower | Yes       | No         |               5 |              0 |             1 | Yes        | No           |
   |         3 | 127.0.0.1:16802 |           1 |          2 | Follower | Yes       | No         |               5 |              0 |             1 | Yes        | No           |
   +-----------+-----------------+-------------+------------+----------+-----------+------------+-----------------+----------------+---------------+------------+--------------+
   3 rows in set (0.00 sec)
   
   mysql> select * from sys.gr_member_routing_candidate_status ;
   +------------------+-----------+---------------------+----------------------+
   | viable_candidate | read_only | transactions_behind | transactions_to_cert |
   +------------------+-----------+---------------------+----------------------+
   | YES              | NO        |                   0 |                    0 |
   +------------------+-----------+---------------------+----------------------+
   1 row in set (0.00 sec)
   
   mysql> select User,Host from mysql.user where User in ('admin2', 'proxysql_monitor');
   +------------------+------+
   | User             | Host |
   +------------------+------+
   | admin2           | %    |
   | proxysql_monitor | %    |
   +------------------+------+
   2 rows in set (0.01 sec)
   ~~~

#### 安装ProxySQL

1. 准备配置文件 `/home/polarx/proxysql.cnf`

   ~~~
   datadir="/var/lib/proxysql"
   
   admin_variables=
   {
       admin_credentials="admin:admin;radmin:radmin"
       mysql_ifaces="0.0.0.0:6032"
   }
   
   mysql_variables=
   {
       threads=4
       max_connections=2048
       default_query_delay=0
       default_query_timeout=36000000
       have_compress=true
       poll_timeout=2000
       interfaces="0.0.0.0:6033"
       default_schema="information_schema"
       stacksize=1048576
       server_version="5.5.30"
       connect_timeout_server=3000
       monitor_username="proxysql_monitor"
       monitor_password="123456"
       monitor_history=600000
       monitor_connect_interval=60000
       monitor_ping_interval=10000
       monitor_read_only_interval=1500
       monitor_read_only_timeout=500
       ping_interval_server_msec=120000
       ping_timeout_server=500
       commands_stats=true
       sessions_sort=true
       connect_retries_on_failure=10
   }
   ~~~

2. 运行 proxysql

   ~~~bash
   docker run --name proxy -p 16032:6032 -p 16033:6033 -p 16070:6070 -d -v /home/polarx/proxysql.cnf:/etc/proxysql.cnf proxysql/proxysql:2.4.8
   ~~~

#### 配置 ProxySQL

1. 登录账号

   ~~~bash
   mysql -h127.0.0.1 -P16032 -uradmin -pradmin --prompt "ProxySQL Admin>"
   ~~~

2. 检查监控账号

   ~~~sql
   mysql> select * from global_variables where variable_name in ('mysql-monitor_username', 'mysql-monitor_password');
   +------------------------+------------------+
   | variable_name          | variable_value   |
   +------------------------+------------------+
   | mysql-monitor_password | 123456           |
   | mysql-monitor_username | proxysql_monitor |
   +------------------------+------------------+
   
   ## 如需修改
   UPDATE global_variables SET variable_value='proxysql_monitor' WHERE variable_name='mysql-monitor_username';
   UPDATE global_variables SET variable_value='123456' WHERE variable_name='mysql-monitor_password';
   ~~~

3. 添加代理链接账号

   ~~~sql
   INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('admin2','123456',10);
   
   #检查
   mysql> select * from mysql_users;
   +----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+
   | username | password | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections | attributes | comment |
   +----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+
   | admin2   | 123456   | 1      | 0       | 10                | NULL           | 0             | 1                      | 0            | 1       | 1        | 10000           |            |         |
   +----------+----------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+------------+---------+
   1 row in set (0.00 sec)
   ~~~

4. 设置读写组，写组10，备写组20，读组30，离线组40，主节点可作为读节点

   ~~~sql
   INSERT INTO mysql_group_replication_hostgroups (writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,writer_is_also_reader) VALUES(10,20,30,40,1,1);
   
   # 检查
   mysql> select * from mysql_group_replication_hostgroups;
   +------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
   | writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
   +------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
   | 10               | 20                      | 30               | 40                | 1      | 1           | 1                     | 0                       | NULL    |
   +------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
   1 row in set (0.00 sec)
   ~~~

5. 添加后端mysql_servers，leader节点定义为写组10, follower节点定义为备写库20。 注意这里port为节点的PolarDB-X标准版的监听端口port。

   ~~~sql
   INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (10,'127.0.0.1',3301);
   INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (20,'127.0.0.1',3302);
   INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (20,'127.0.0.1',3303);
   
   #检查
   mysql> select * from mysql_servers;
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   | hostgroup_id | hostname  | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   | 10           | 127.0.0.1 | 3301 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   | 20           | 127.0.0.1 | 3302 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   | 20           | 127.0.0.1 | 3303 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   3 rows in set (0.00 sec)
   ~~~

6. 配置读写规则。

   ~~~sql
   # 读写分离
   INSERT INTO mysql_query_rules(active,match_pattern,destination_hostgroup,apply) VALUES(1,'^select.*for update$',10,1);
   INSERT INTO mysql_query_rules(active,match_pattern,destination_hostgroup,apply) VALUES(1,'^select',30,1);
   
   # 读写都在主库
   INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup, apply) VALUES (1, '^SELECT.*', 10, 1);
   INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup, apply) VALUES (2, '^(INSERT|UPDATE|DELETE).*', 10, 1);
   INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup, apply) VALUES (3, '^(SET|COMMIT|ROLLBACK|BEGIN).*', 10, 1);
   
   #检查
   mysql> select * from mysql_query_rules;
   +---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+------------+---------+
   | rule_id | active | username | schemaname | flagIN | client_addr | proxy_addr | proxy_port | digest | match_digest | match_pattern        | negate_match_pattern | re_modifiers | flagOUT | replace_pattern | destination_hostgroup | cache_ttl | cache_empty_result | cache_timeout | reconnect | timeout | retries | delay | next_query_flagIN | mirror_flagOUT | mirror_hostgroup | error_msg | OK_msg | sticky_conn | multiplex | gtid_from_hostgroup | log | apply | attributes | comment |
   +---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+------------+---------+
   | 1       | 1      | NULL     | NULL       | 0      | NULL        | NULL       | NULL       | NULL   | NULL         | ^select.*for update$ | 0                    | CASELESS     | NULL    | NULL            | 10                    | NULL      | NULL               | NULL          | NULL      | NULL    | NULL    | NULL  | NULL              | NULL           | NULL             | NULL      | NULL   | NULL        | NULL      | NULL                | NULL | 1     |            | NULL    |
   | 2       | 1      | NULL     | NULL       | 0      | NULL        | NULL       | NULL       | NULL   | NULL         | ^select              | 0                    | CASELESS     | NULL    | NULL            | 30                    | NULL      | NULL               | NULL          | NULL      | NULL    | NULL    | NULL  | NULL              | NULL           | NULL             | NULL      | NULL   | NULL        | NULL      | NULL                | NULL | 1     |            | NULL    |
   +---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+------------+---------+
   2 rows in set (0.00 sec)
   ~~~

7. 保存配置并载入内存生效

   ~~~sql
   save mysql users to disk;
   save mysql servers to disk;
   save mysql query rules to disk;
   save mysql variables to disk;
   save admin variables to disk;
   load mysql users to runtime;
   load mysql servers to runtime;
   load mysql query rules to runtime;
   load mysql variables to runtime;
   load admin variables to runtime;
   
   #检查
   mysql> select * from runtime_mysql_servers;
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   | hostgroup_id | hostname  | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   | 10           | 127.0.0.1 | 3301 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   | 30           | 127.0.0.1 | 3301 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   | 30           | 127.0.0.1 | 3302 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   | 30           | 127.0.0.1 | 3303 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
   +--------------+-----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
   4 rows in set (0.00 sec)
   ~~~

8. 检查路由规则

   ~~~sql
   mysql> select hostgroup,digest_text from stats_mysql_query_digest;
   +-----------+------------------------------------------+
   | hostgroup | digest_text                              |
   +-----------+------------------------------------------+
   | 10        | select * from d1.t1 for update           |
   | 30        | select * from d1.t1                      |
   | 10        | insert into d1.t1 values(?,?)            |
   | 10        | commit                                   |
   | 10        | create table d1.t1(c1 int,c2 varchar(?)) |
   | 10        | create database d1                       |
   | 10        | begin                                    |
   | 10        | select @@version_comment limit ?         |
   +-----------+------------------------------------------+
   ~~~

## 注意事项

1. 如果应用程序连接时出现错误 ``，是因为 ProxySQL 配置 mysql_version 版本为 5.5，但是实际代理的 mysql 版本为 8，需要修改配置

   ~~~sql
   update global_variables set variable_value="8.0.4 (ProxySQL)" where variable_name='mysql-server_version';
   load mysql variables to run;save mysql variables to disk;
   ~~~

## 参考文章

- [通过 PXD 部署集群](https://doc.polardbx.com/zh/quickstart/topics/quickstart-pxd-cluster.html)
- [使用开源ProxySQL构建PolarDB-X标准版高可用路由服务](https://zhuanlan.zhihu.com/p/697117089)