---
title: Docker 快速安装中间件服务
date: 2024-07-05 22:38:18
tags:
  - docker
---
## 配置宿主机地址
#### docker run
~~~bash
host.docker.internal 表示宿主机 ip
~~~

#### docker-compose

~~~yaml
# eg: 需要 extra_hosts 中配置了 host-gateway
services:
  passiflora-gateway:
    build:
      context: .
      dockerfile: Dockerfile-gateway
    mem_limit: 2048m
    environment:
      TZ: Asia/Shanghai
    volumes:
      - /etc/localtime:/etc/localtime:ro
    container_name: passiflora-gateway
    restart: unless-stopped
    image: passiflora-gateway
    extra_hosts:
      - "passiflora-nacos:host-gateway"
      - "passiflora-redis:host-gateway"
    expose:
      - 51000
    ports:
      - "51000:51000"
    healthcheck:
      test: [ "CMD", "curl", "http://localhost:51000" ]
      start_period: 90s
      interval: 60s
      timeout: 10s
      retries: 5
~~~

## 限制资源

限制 cpu --cpus=''1"
限制内存 -m 300m
限制日志 

~~~
# compose
services:
  myapp:
    image: myimage
    logging:
      driver: json-file
      options:
        max-size: "50m"  # 最大日志文件大小为100MB
        max-file: "2"     # 最多保留5个日志文件
# docker run
--log-opt max-size=50m --log-opt max-file=2 

## /etc/docker/daemon.json 配置文件
{
    "log-driver" : "json-file",
    "log-opts" : {
      "max-size" : "50m",
      "max-file" : "2"
    }
}
~~~

## RabbitMQ
```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  -v /etc/localtime:/etc/localtime:ro \
	-e TZ="Asia/Shanghai" \
  rabbitmq:3.7.7-management

# 延迟交换机插件安装
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.8.0/rabbitmq_delayed_message_exchange-3.8.0.ez
docker cp rabbitmq_delayed_message_exchange-3.8.0.ez rabbitmq:/plugins/
docker exec -it rabbitmq bash
cd plugins
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

## Redis
```bash
docker run -d \
	--restart=always \
	--name myredis \
	-p 6379:6379 \
	-v /etc/localtime:/etc/localtime:ro \
  -e TZ="Asia/Shanghai" \
	redis:7.4.1-alpine --requirepass "123456"
```

## Nginx
```bash
# 生成相关初始化文件
docker run -d --restart=on-failure:10 --name=nginx  nginx:1.21

# 拷贝Ngixn容器中相关初始化文件到宿主机中，并删除容器
mkdir -p /opt/docker/nginx
docker cp nginx:/var/log/nginx /opt/docker/nginx/logs
docker cp nginx:/etc/nginx /opt/docker/nginx/conf
docker cp nginx:/usr/share/nginx /opt/docker/nginx/webapps

docker rm -f nginx

# 新部署Nginx容器
docker run -d \
	-p 80:80 \
	--restart=always \
	--privileged=true \
	-v /opt/docker/nginx/logs:/var/log/nginx \
	-v /opt/docker/nginx/conf:/etc/nginx/ \
	-v /opt/docker/nginx/webapps:/usr/share/nginx \
	-v /etc/localtime:/etc/localtime:ro \
  -e TZ="Asia/Shanghai" \
	--name=nginx  nginx:1.27.2
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
-v /etc/localtime:/etc/localtime:ro \
-e TZ="Asia/Shanghai" \
-e MYSQL_ROOT_PASSWORD=123456 \
-d --privileged=true --restart=unless-stop mysql:8.4.2 --lower-case-table-names=1

```

## Zookeeper
```shell
docker run -d --restart=on-failure:10 \
-v /etc/localtime:/etc/localtime:ro \
-e TZ="Asia/Shanghai" \
-p 2181:2181 \
--name zookeeper --restart always zookeeper:3.7
```

## Kafka
```shell
docker network create kafka-bridge \
  --subnet 192.169.0.0/16 \
  --gateway 192.169.0.1 \
  --driver bridge

docker run -d \
  --name zookeeper \
  --net kafka-bridge \
  --ip 192.169.0.2 \
  -v /etc/localtime:/etc/localtime:ro \
	-e TZ="Asia/Shanghai" \
  -e ALLOW_ANONYMOUS_LOGIN=yes \
  -p 2181:2181 bitnami/zookeeper:3.9.2


docker run -d --name=kafka \
 -p 9092:9092 \
 --net kafka-bridge \
 --ip 192.169.0.3 \
 -v /etc/localtime:/etc/localtime:ro \
 -e TZ="Asia/Shanghai" \
 -e ALLOW_PLAINTEXT_LISTENER=yes \
 -e KAFKA_CFG_ZOOKEEPER_CONNECT=192.169.0.2:2181 \
 -e KAFKA_BROKER_ID=1 \
 -e KAFKA_HEAP_OPTS="-Xmx180m -Xms180m" \
 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092 \
 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092  \
 bitnami/kafka:3.3.2
```

## XXL-JOB
```shell
docker run -e --restart=on-failure:10 \
PARAMS="--spring.datasource.url=jdbc:mysql://10.3.0.61:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai --spring.datasource.username=root --spring.datasource.password=123456" \
-p 8081:8080 \
--name xxl-job \
-v /tmp:/data/applogs \
-v /etc/localtime:/etc/localtime:ro \
-e TZ="Asia/Shanghai" \
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
    -v /etc/localtime:/etc/localtime:ro \
		-e TZ="Asia/Shanghai" \
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
-v /etc/localtime:/etc/localtime:ro \
-e TZ="Asia/Shanghai" \
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
    -v /etc/localtime:/etc/localtime:ro \
		-e TZ="Asia/Shanghai" \
    -d postgres:17.0-alpine
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
			-v /etc/localtime:/etc/localtime:ro \
			-e TZ="Asia/Shanghai" \
			-d neo4j:4.4-community
```

## MongDB
```shell
docker run --name mongodb -p 27017:27017 -d mongodb/mongodb-community-server:6.0.5-ubi8

docker exec -it mongodb mongo admin

db.createUser({ user: 'admin', pwd: 'admin', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });

db.auth("admin","admin");
```

## Nacos
```shell
docker run -d --name nacos \
  --restart=on-failure:10 \
  -p 8848:8848 \
  -p 9848:9848 \
  -p 9849:9849 \
  -e MODE=standalone \
  -e EMBEDDED_STORAGE \
  -v /etc/localtime:/etc/localtime:ro \
	-e TZ="Asia/Shanghai" \
  nacos/nacos-server:v2.4.3-slim
```

## Minio
```shell
docker run -d --name passiflora-minio \
    --restart=always \
    --publish 9000:9000 \
    --publish 9001:9001 \
    --env MINIO_ROOT_USER="minio" \
    --env MINIO_ROOT_PASSWORD="minio" \
    -v /etc/localtime:/etc/localtime:ro \
	  -e TZ="Asia/Shanghai" \
    bitnami/minio:2024.10.13
```

## Prometheus
```shell
docker run \
    -d --name prometheus \
    -p 9090:9090 \
    -v /etc/localtime:/etc/localtime:ro \
		-e TZ="Asia/Shanghai" \
    prom/prometheus
```

## PolarDB-X

~~~shell
docker run -d \
	--name polardb-x \
	-m 1024m \
	-p 8527:8527 \
	-v /etc/localtime:/etc/localtime:ro \
	-e TZ="Asia/Shanghai" \
	polardbx/polardb-x:2.3.1
	
# 出现不支持外键提示的话，需要 set global enable_foreign_key = true
~~~

