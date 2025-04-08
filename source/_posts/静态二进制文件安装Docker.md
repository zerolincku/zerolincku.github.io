---
title: 静态二进制文件安装Docker
date: 2025-04-08 22:38:18
tags:
  - docker
---

> 什么是静态二进制文件？
>
> 静态二进制文件通常是静态链接的，意味着它们不依赖于系统中的动态库（.so 文件）。所有必要的代码都已经被打包进这些二进制文件中，因此无需额外的安装或依赖管理。

### 步骤

1. 下载 Docker 的静态二进制包，https://download.docker.com/linux/static/stable/

2. 复制文件

   ```shell
   tar -xzf docker-*.tgz
   sudo cp docker/* /usr/bin/
   ```

3. 编辑 docker 配置

   ```shell
   sudo mkdir /etc/docker
   sudo vim /etc/docker/daemon.json
   ```

4. 加入一下内容

   ```json
   {
      "data-root": "/data/docker"
   }
   ```

5. 编辑systemd配置文件

   ```shell
   sudo vim /usr/lib/systemd/system/docker.service
   ```

6. 加入以下内容

   ```ini
   [Unit]
   Description=Docker Application Container Engine
   Documentation=https://docs.docker.com
   After=network-online.target firewalld.service 
   Wants=network-online.target
   
   [Service]
   Type=notify
   ExecStart=/usr/bin/dockerd
   ExecReload=/bin/kill -s HUP $MAINPID
   LimitNOFILE=infinity
   LimitNPROC=infinity
   TimeoutStartSec=0
   Delegate=yes
   KillMode=process
   Restart=on-failure
   StartLimitBurst=3
   StartLimitInterval=60s
   
   [Install]
   WantedBy=multi-user.target
   ```

7. 设置 docker 自启动

   ```shell
   sudo systemctl daemon-reload
   
   sudo systemctl enable docker
   sudo systemctl start docker
   ```
   
8. 测试docker命令

   ```shell
   sudo docker ps
   ```


### 为普通用户分配docker权限

1. 检查是否存在`docker`组

   ```shell
   sudo getent group docker
   ```

2. 如果没有则手动创建

   ```shell
   # 创建
   sudo groupadd docker
   # 再次验证
   sudo getent group docker
   ```

3. 将用户添加到docker组

   ```shell
   sudo usermod -aG docker cloudcare
   ```

4. 验证用户是否已加入

   ```shell
   groups cloudcare
   ```

5. 重启docker服务

   ```shell
   sudo systemctl restart docker
   ```

6. 刷新组成员身份

   ```shell
   sudo newgrp docker
   ```

7. 测试docker权限

   ```shell
   docker ps
   ```

### docker compose

1. 下载 docker compose，https://github.com/docker/compose/releases

   ```shell
   curl -L "https://github.com/docker/compose/releases/download/v2.34.0/docker-compose-linux-x86_64" -o docker-compose
   # 将 v2.34.0 替换为所需的版本号
   ```

2. 添加权限

   ```shell
   chmod +x docker-compose
   ```

3. 移动到可执行路径（可选）

   ```shell
   sudo mv docker-compose /usr/local/bin/
   ```

4. 验证安装

   ```shell
   docker-compose version
   ```

#### 注意事项

* **版本选择**：截至 2025 年 4 月，Docker Compose V2 是主流版本（例如 v2.34.0 是 2024 年末的最新版本，可能会有更新）。V1 版本已于 2023 年停止支持，不推荐使用。
* **静态编译特性**：从 GitHub Releases 下载的二进制文件通常是静态编译的，适用于大多数 Linux 发行版，无需额外依赖。但某些特殊架构（如 RISC-V）可能需要自行编译。
* **插件模式 vs 独立二进制**：
  - Docker Compose V2 默认作为 Docker CLI 的插件运行（安装在 ~/.docker/cli-plugins/ 或 /usr/lib/docker/cli-plugins/）。
  - 如果你需要独立二进制文件，下载的静态文件可以直接使用，不依赖 Docker CLI。