---
title: Linux 磁盘挂载
date: 2024-10-15 09:19:00
tags:
  - linux
---

## 1. 查看磁盘信息

首先，使用 `lsblk` 或 `fdisk` 命令查看是否识别到了 `vdb` 磁盘

~~~bash
lsblk
~~~

输出类似如下：

~~~bash
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    252:0    0   40G  0 disk
├─vda1 252:1    0  500M  0 part /boot
└─vda2 252:2    0 39.5G  0 part /
vdb    252:16   0   20G  0 disk
~~~

## 2. 分区并格式化（如果磁盘未分区）

如果 `vdb` 是新磁盘且没有分区，可以使用 `fdisk` 或 `parted` 工具对磁盘进行分区。假设我们使用 `fdisk`：

~~~bash
sudo fdisk /dev/vdb
~~~

按提示进行分区操作：

- 输入 `n` 创建新分区。
- 选择分区类型 `p`（主分区）。
- 选择分区号（一般为 `1`）。
- 使用默认的开始和结束扇区。
- 输入 `w` 保存并退出。

完成分区后，格式化该分区（假设分区名称为 `/dev/vdb1`）：

~~~bash
sudo mkfs.ext4 /dev/vdb1
~~~

## 3. 挂载 `vdb` 到 `/home`

首先备份 `/home` 目录下现有的数据：

~~~bash
sudo rsync -av /home/ /home-backup/
~~~

挂载新磁盘到 `/home`：

~~~bash
sudo mount /dev/vdb1 /home
~~~

## 4. 设置自动挂载

为了让磁盘在系统启动时自动挂载到 `/home`，你需要编辑 `/etc/fstab` 文件：

1. 获取新磁盘的 UUID：

   ~~~bash
   sudo blkid /dev/vdb1
   ~~~

   输出类似：

   ~~~bash
   /dev/vdb1: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="ext4"
   ~~~

2. 编辑 `/etc/fstab` 文件：

   ~~~
   sudo vim /etc/fstab
   ~~~

   在文件末尾添加一行，使用 UUID 挂载分区：

   ~~~bash
   UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /home ext4 defaults 0 0		
   ~~~

## 5. 检查挂载是否成功

运行以下命令验证 `/home` 是否已正确挂载，能看到 `/dev/vdb1` 挂载到了 `/home`。

```bash
df -h
```

## 6. 还原数据（如果有备份）

如果你之前备份了 `/home` 目录的数据，执行以下命令将其还原：

~~~bash
sudo rsync -av /home-backup/ /home/
~~~

