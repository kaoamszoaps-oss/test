[README.md](https://github.com/user-attachments/files/24875716/README.md)
# Docker 开发环境使用指南

本项目提供了封装好的 Docker 镜像，用于模型的训练与测试。本文档详细说明了环境的加载、启动、VS Code 连接配置以及常见问题的解决方案。

## 📋 前置要求

在开始之前，请确保服务器满足以下条件：
* 已安装 **Docker Engine**。
* 已安装 **NVIDIA Container Toolkit** (支持 `--gpus all`)。
* 本地或远程 VS Code 已安装 **Dev Containers** (ms-vscode-remote.remote-containers) 扩展插件。

---

## 🚀 快速开始 (Quick Start)

### 1. 加载镜像
如果你使用的是离线镜像包 (`.tar`)，请先执行加载：

```bash
docker load -i sdc-net_latest.tar
# 验证镜像是否加载成功
docker images

```

### 2. 启动容器 (标准指令)

为了确保训练稳定性和数据安全，请使用以下**完整指令**启动容器。该指令已包含显卡调用、内存优化和数据挂载配置。

```bash
docker run -itd \
    --gpus all \
    --shm-size 8g \
    --name sdcnet_dev \
    -v /home_nfs/jiaoli.liu/ISDNet/data/IDRID:/home_nfs/jiaoli.liu/ISDNet/data/IDRID \
    -v $(pwd)/work_dirs:/app/work_dirs \
    sdc-net:latest /bin/bash

```

**关键参数说明：**

* `-itd`: **后台静默运行**。即使关闭 VS Code 或断开 SSH，容器内的训练任务也不会中断。
* `--gpus all`: 允许容器调用服务器所有显卡。
* `--shm-size 8g`: **重要**。将共享内存扩大至 8GB，防止 PyTorch DataLoader 报错 `Bus error`。
* `-v (Volume)`: **数据挂载**。格式为 `宿主机路径:容器内路径`。
* **建议**：将宿主机的 `work_dirs` 挂载到容器内，确保训练权重 (`.pth`) 和日志文件持久化保存，防止删除容器后数据丢失。



---

## 💻 开发环境配置 (VS Code)

推荐使用 VS Code 的 **Dev Containers** 插件直连容器进行开发，体验与本地一致。

1. **安装插件**：在 VS Code 扩展商店搜索并安装 `Dev Containers`。
2. **连接容器**：
* 按 `F1` 或 `Ctrl+Shift+P` 打开命令面板。
* 输入并选择：`Dev Containers: Attach to Running Container`。
* 在列表中选择刚才启动的容器 `sdcnet_dev`。


3. **打开项目**：
* 连接成功后，点击左侧资源管理器 -> **打开文件夹 (Open Folder)**。
* 输入项目路径（默认为 `/app`），点击确定。



现在，你可以在 VS Code 中直接编辑代码、使用终端和调试 Python 脚本。

---

## 🔧 常用操作与问题排查

### 1. Conda 环境激活报错

**现象**：执行 `conda activate base_env` 时提示 `CondaError: Run 'conda init' before 'conda activate'`。
**解决**：容器首次启动时 Shell 可能未初始化，请执行一次以下命令：

```bash
conda init
source ~/.bashrc
conda activate base_env

```

### 2. DataLoader "Bus error" / Shared Memory Insufficient

**现象**：训练时出现 `RuntimeError: DataLoader worker... is killed by signal: Bus error`。
**原因**：Docker 默认 `/dev/shm` 只有 64MB，不足以支撑多进程数据加载。
**解决**：

* 必须在 `docker run` 时添加 `--shm-size 8g` 参数。
* 如果容器已运行，需删除并重新启动容器。

### 3. 关闭 VS Code 后训练中断

**现象**：本地 VS Code 关闭后，服务器上的训练进程被杀掉。
**原因**：未使用后台模式 (`-d`) 启动容器。
**解决**：

* 请确保使用 `docker run -itd ...` 启动。
* 在容器终端内运行长时间任务时，建议使用 `nohup` 命令：
```bash
nohup bash tools/dist_train.sh ... > train.log 2>&1 &

```



### 4. 磁盘空间清理

若提示磁盘空间不足，可使用以下命令清理无用的数据：

```bash
# 查看磁盘占用
docker system df

# 删除所有已停止的容器（慎用，确保重要数据已备份）
docker container prune

# 删除悬空镜像（标签为 <none> 的镜像）
docker image prune

```

---

## ⚠️ 数据安全最佳实践

**"容器是临时的，数据是永恒的"**

1. **模型权重保存**：切勿将训练好的模型直接保存在容器的系统目录下。**务必**通过 `-v` 挂载，将 `work_dirs` 指向物理机磁盘。否则一旦执行 `docker rm`，所有训练成果将永久丢失。
2. **环境变更**：如果在容器内通过 `pip/conda install` 安装了新包，请同步更新 `requirements.txt`，否则重建容器后环境将重置。

---

**维护者**: SXP
**最后更新时间**: 2026-01-27

```

```
