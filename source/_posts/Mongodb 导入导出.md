---
title: Mongodb 导入导出
date: 2024-11-11 17:14:18
tags:
  - mongodb
---

~~~shell
# 导出需要认证的数据库
mongodump -h localhost:27017 \
          -d database_name \
          -u admin_user \
          --authenticationDatabase admin \
          -o /backup/path
          
mongodump --uri "mongodb://username:password@localhost:27017/database_name?authSource=admin" -o /backup/path

# 导入需要认证的数据库
mongorestore -h localhost:27017 \
            -d database_name \
            -u admin_user \
            --authenticationDatabase admin \
            /backup/path/database_name
            
mongorestore --uri "mongodb://username:password@localhost:27017/database_name?authSource=admin" /backup/path/database_name
~~~

### 导入时参数

- --drop：会在导入前删除目标数据库中的对应集合，只删除要导入的集合，不会影响其他集合
- --dropDatabase：会在导入前删除整个目标数据库，删除所有集合和数据
- --upsertFields "_id"：根据指定字段（如 _id）进行更新或插入，如果找到匹配的文档就更新，否则插入新文档
