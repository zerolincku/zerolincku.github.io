---
title: RabbitMQ Federation 插件使用
date: 2024-09-26 08:19:00
tags:
  - RabbitMQ
---

### 启用 Federation 插件

~~~shell
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management
~~~

### 联邦交换机

#### 1. 下游配置上游地址

![image-20240926101635871](assets/image-20240926101635871.png)

#### 2. 下游配置策略

![image-20240926101906798](assets/image-20240926101906798.png)

#### 3. 查看联邦状态

![image-20240926102035608](assets/image-20240926102035608.png)

