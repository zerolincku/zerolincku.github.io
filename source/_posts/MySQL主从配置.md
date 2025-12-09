---
title: MySQL 主从部署
date: 2025-09-15 20:20:00
tags:
  - database
---

### 主库配置

#### 创建用户

```sql
CREATE USER 'replication'@'%' IDENTIFIED BY '123456';
```

#### 授权

```sql
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
```

#### 刷新权限

```sql
FLUSH PRIVILEGES;
```

