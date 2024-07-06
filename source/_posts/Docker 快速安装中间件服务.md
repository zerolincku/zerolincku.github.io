---
title: Docker 快速安装中间件服务
date: 2024-07-05 22:38:18
tags:
categories:
  - docker
---

## RabbitMq
```bash
docker run -d --restart=always --name rabbitmq -p 5672:5672 -p 15672:15672 -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin rabbitmq:3.7.7-management

# 延迟交换机插件安装
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.8.0/rabbitmq_delayed_message_exchange-3.8.0.ez
docker cp rabbitmq_delayed_message_exchange-3.8.0.ez rabbitmq:/plugins/
docker exec -it rabbitmq bash
cd plugins
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```
## Redis
```bash
docker run -d --restart=always --name myredis -p 6379:6379 redis:6.2.7 --requirepass "123456"
```
## Nginx
```bash
# 生成相关初始化文件
docker run -d --restart=on-failure:10 --name=nginx  nginx:1.21

# 拷贝Ngixn容器中相关初始化文件到宿主机中，并删除容器
mkdir -p /opt/docker/nginx
docker cp  nginx:/var/log/nginx /opt/docker/nginx/logs
docker cp  nginx:/etc/nginx /opt/docker/nginx/conf
docker cp  nginx:/usr/share/nginx /opt/docker/nginx/webapps

docker rm -f nginx

# 新部署Nginx容器
docker run -d -p 80:80 --restart=always --user=root --privileged=true -v /opt/docker/nginx/logs:/var/log/nginx -v /opt/docker/nginx/conf:/etc/nginx/ -v /opt/docker/nginx/webapps:/usr/share/nginx --name=nginx  nginx:1.21cd
```
## MySQL
```bash
#创建数据存储目录
mkdir -p /opt/docker/mysql/conf
mkdir -p /opt/docker/mysql/data
mkdir -p /opt/docker/mysql/mysql-files
#设置忽略大小写
docker run -p 3306:3306 --name=mysql \
-v /opt/docker/mysql/conf/:/etc/mysql/ \
-v /opt/docker/mysql/data:/var/lib/mysql \
-v /opt/docker/mysql/mysql-files/:/var/lib/mysql-files \
-e MYSQL_ROOT_PASSWORD=123456 \
-d --privileged=true --restart=on-failure:10 mysql:8.0.16 --lower-case-table-names=1

```
## Zookeeper
```shell
docker run -d --restart=on-failure:10 \
-e TZ="Asia/Shanghai" \
-p 2181:2181 \
--name zookeeper --restart always zookeeper:3.7
```
## XXL-JOB
```shell
docker run -e --restart=on-failure:10 \
PARAMS="--spring.datasource.url=jdbc:mysql://10.3.0.61:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai --spring.datasource.username=root --spring.datasource.password=123456" \
-p 8081:8080 \
--name xxl-job \
-v /tmp:/data/applogs \
-d xuxueli/xxl-job-admin:2.2.0
```
## Elasticsearch
```shell
docker run -d \
	--name es \
    -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "discovery.type=single-node" \
    -v es-data:/usr/share/elasticsearch/data \
    -v es-plugins:/usr/share/elasticsearch/plugins \
    --privileged \
    -p 9200:9200 \
    -p 9300:9300 \
elasticsearch:7.17.1

# 安装ik分词器
docker exec -it es /bin/bash
./bin/elasticsearch-plugin  install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.1/elasticsearch-analysis-ik-7.17.1.zip
docker restart es
```
## Kibana
```shell
docker run -d \
--name kibana \
--link=es:es \
-e ELASTICSEARCH_HOSTS=http://es:9200 \
-p 5601:5601  \
kibana:7.17.1
```
## PostgresQL
```shell
mldir -p /opt/docker/postgresql/data

docker run --name postgres \
    --restart=unless-stopped \
    -e POSTGRES_PASSWORD=postgres \
    -p 5432:5432 \
    -v /opt/docker/postgresql/data:/var/lib/postgresql/data \
    -d postgres:13.5
```
## Neo4j
```shell
mkdir -p /opt/docker/neo4j/data
mkdir -p /opt/docker/neo4j/logs
mkdir -p /opt/docker/neo4j/conf
mkdir -p /opt/docker/neo4j/import

docker run --name neo4j \
			-p 7474:7474 \
			-p 7687:7687 \
			-e "NEO4J_AUTH=neo4j/123456" \
			-v /opt/docker/neo4j/data:/mydata/data \
			-v /opt/docker/neo4j/logs:/mydata/logs \
			-v /opt/docker/neo4j/conf:/var/lib/neo4j/conf \
			-v /opt/docker/neo4j/import:/var/lib/neo4j/import \
			-d neo4j:3.5.22-community
```
## MongDB
```shell
docker run --name mongodb -p 27017:27017 -d mongo:4.4.3

docker exec -it mongodb mongo admin

db.createUser({ user: 'admin', pwd: '123456', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });

db.auth("admin","123456");
```
## Nacos
```shell
docker run -d --name nacos --restart=on-failure:10 -p 8848:8848 -p 9848:9848 -p 9849:9849 -e MODE=standalone -e EMBEDDED_STORAGE nacos/nacos-server:v2.2.3-slim
```
限制 cpu --cpus=''1"
限制内存 -m 300m
