---
title: Linux 磁盘挂载
date: 2024-10-15 09:19:00
tags:
  - linux
---

本文记录 Linux 数据盘查看、分区、格式化、挂载和开机自动挂载。示例以新数据盘 `/dev/vdb` 挂载到 `/data` 为主，也包含挂载到 `/home` 和作为 Docker 数据目录的处理方式。

## 1. 查看磁盘信息

```bash
lsblk -f
df -hT
blkid
fdisk -l
```

示例：

```bash
NAME   FSTYPE  FSVER            LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sr0    iso9660 Joliet Extension config-2 2026-05-21-17-43-40-00
vda
└─vda1 xfs                               151ba1a6-cde4-4270-9068-9674d330c10a   47.2G     5% /
vdb
```

这里：

- `vda1` 已经挂载到 `/`，是系统盘。
- `vdb` 没有分区、没有文件系统、没有挂载点，是可用的数据盘。

执行格式化前一定再次确认磁盘名，不要把系统盘 `/dev/vda` 格式化。

## 2. 分区并格式化

以下命令假设数据盘是 `/dev/vdb`。

创建 GPT 分区：

```bash
parted -s /dev/vdb mklabel gpt
parted -s /dev/vdb mkpart primary xfs 0% 100%
```

格式化分区：

```bash
mkfs.xfs -f /dev/vdb1
```

注意：`mkfs.xfs` 会清空 `/dev/vdb1` 上的数据。

如果需要使用 `ext4`，可以改成：

```bash
mkfs.ext4 /dev/vdb1
```

## 3. 挂载到 /data

```bash
mkdir -p /data
mount /dev/vdb1 /data

df -hT /data
lsblk -f
```

## 4. 配置开机自动挂载

推荐使用 UUID 写入 `/etc/fstab`，避免设备名变化导致挂载失败。

```bash
UUID=$(blkid -s UUID -o value /dev/vdb1)

cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)

cat >>/etc/fstab <<EOF
UUID=$UUID /data xfs defaults,nofail 0 0
EOF
```

验证配置：

```bash
umount /data
mount -a
df -hT /data
lsblk -f
```

如果 `mount -a` 没有报错，并且 `/data` 正常显示容量，说明配置正确。

## 5. 挂载到 /home

如果需要把新磁盘挂载到 `/home`，先备份原目录数据：

```bash
rsync -av /home/ /home-backup/
```

挂载到 `/home`：

```bash
mount /dev/vdb1 /home
```

写入 `/etc/fstab` 时，把挂载点改成 `/home`：

```bash
UUID=$(blkid -s UUID -o value /dev/vdb1)

cp -a /etc/fstab /etc/fstab.bak.$(date +%F-%H%M%S)

cat >>/etc/fstab <<EOF
UUID=$UUID /home xfs defaults,nofail 0 0
EOF
```

验证挂载：

```bash
mount -a
df -hT /home
```

如果之前备份了 `/home`，挂载完成后再还原：

```bash
rsync -av /home-backup/ /home/
```

## 6. 作为 Docker 数据目录

如果这块数据盘准备用来存放 Docker 镜像、容器和卷，建议将 Docker 根目录配置到 `/data/docker`。

停止 Docker：

```bash
systemctl stop docker
```

创建目录：

```bash
mkdir -p /data/docker
```

如果原来 `/var/lib/docker` 已有数据，需要迁移：

```bash
rsync -aHAX /var/lib/docker/ /data/docker/
```

配置 `/etc/docker/daemon.json`：

```bash
mkdir -p /etc/docker

cat >/etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "overlay2",
  "data-root": "/data/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
EOF
```

启动 Docker 并验证：

```bash
systemctl start docker
docker info | grep "Docker Root Dir"
```

看到如下输出说明 Docker 数据目录已经切到数据盘：

```bash
Docker Root Dir: /data/docker
```

确认业务容器正常后，可以再清理旧目录。清理前建议先保留一段时间，避免误删还需要的数据。
