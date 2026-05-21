---
title: openEuler 安装 Docker
date: 2026-05-21 22:38:18
tags:
  - docker
  - openEuler
---

本文以 `openEuler 22.03 LTS-SP4 x86_64` 为例，记录在线安装、离线安装和 Docker 日志轮转配置。

## 1. 查看系统信息

```bash
cat /etc/os-release
lscpu | grep Architecture
```

示例输出：

```bash
NAME="openEuler"
VERSION="22.03 (LTS-SP4)"
ID="openEuler"
VERSION_ID="22.03"
PRETTY_NAME="openEuler 22.03 (LTS-SP4)"

Architecture:                       x86_64
```

openEuler 是 RPM 系发行版，包管理使用 `dnf` 或 `yum`。`ls-cpu` 不是正确命令，正确命令是 `lscpu`。

## 2. 在线安装 Docker

openEuler 官方仓库里的 Docker 包名是 `docker-engine`。

```bash
dnf makecache

# 如果已经安装过这些运行时，可能会和 docker-engine 冲突
dnf remove -y podman containerd runc

dnf install -y docker-engine

systemctl enable --now docker
docker version
docker info
```

如果提示找不到包，先检查仓库是否正常：

```bash
dnf repolist
ls /etc/yum.repos.d/
```

## 3. 配置 Docker 日志轮转

避免容器日志无限增长撑爆磁盘，可以配置 `json-file` 日志轮转。

```bash
mkdir -p /etc/docker

cp -a /etc/docker/daemon.json /etc/docker/daemon.json.bak.$(date +%F-%H%M%S) 2>/dev/null || true

cat >/etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
EOF

systemctl restart docker
docker info | grep -A5 "Logging Driver"
```

其中：

- `max-size: 50m`：单个日志文件最多 50 MB。
- `max-file: 3`：每个容器最多保留 3 个日志文件。

也就是单个容器默认最多保留约 150 MB 日志。

验证新容器的日志配置：

```bash
docker inspect <container> --format '{{json .HostConfig.LogConfig}}'
```

正常应看到类似：

```json
{"Type":"json-file","Config":{"max-file":"3","max-size":"50m"}}
```

注意：该配置主要对新创建的容器生效。已有容器通常需要重建，才能完全继承新的日志限制。

## 4. 准备离线安装包

找一台能联网、系统版本和架构一致的机器，例如同样是 `openEuler 22.03 LTS-SP4 x86_64`。

```bash
mkdir -p /root/docker-offline-rpms

dnf makecache
dnf install -y dnf-plugins-core createrepo_c

dnf download --resolve --alldeps --destdir=/root/docker-offline-rpms docker-engine

createrepo_c /root/docker-offline-rpms

cd /root
tar czf docker-offline-rpms-openeuler-22.03-sp4-x86_64.tar.gz docker-offline-rpms
```

传到离线机器：

```bash
scp /root/docker-offline-rpms-openeuler-22.03-sp4-x86_64.tar.gz root@离线机器IP:/root/
```

## 5. 离线安装 Docker

离线机器执行：

```bash
cd /root
rm -rf /root/docker-offline-rpms
tar xzf docker-offline-rpms-openeuler-22.03-sp4-x86_64.tar.gz
```

创建本地 repo：

```bash
cat >/etc/yum.repos.d/docker-offline.repo <<'EOF'
[docker-offline]
name=docker-offline
baseurl=file:///root/docker-offline-rpms
enabled=1
gpgcheck=0
EOF
```

只启用本地 repo 安装 `docker-engine`：

```bash
dnf clean all
dnf makecache --disablerepo='*' --enablerepo='docker-offline'

dnf install -y docker-engine \
  --disablerepo='*' \
  --enablerepo='docker-offline' \
  --nobest
```

启动并验证：

```bash
systemctl enable --now docker
docker version
docker info | grep -A5 "Logging Driver"
```

## 6. 离线安装注意事项

不要直接执行：

```bash
dnf install -y ./*.rpm
```

这样会让目录里所有 RPM 都参与安装或升级，可能触发基础系统包冲突。例如出现：

```bash
Problem: The operation would result in removing the following protected packages: grub2-pc
```

遇到这种情况不要加 `--allowerasing`，因为它可能会删除启动相关保护包。正确做法是把离线 RPM 目录做成本地 repo，然后只安装目标包 `docker-engine`，依赖由本地 repo 解析。

如果需要检查冲突包，先只查询：

```bash
rpm -qa | egrep '^(podman|containerd|runc|docker)'
```

确认确实存在冲突后，再单独处理。
