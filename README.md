# SDC-Net Docker镜像结构说明
SDC-Net Docker镜像，可提供用于训练与测试SDC-Net模型的完整环境。

- 项目主要路径组织如下：
app/
├── 📂 configs/
│   ├── 📂 _base_/
│   │   ├── 📂 models/
│   │   │   └── 📄 isdnet_r50-d8.py        # 模型基座：继承自 ISDNet
│   │   ├── 📂 datasets/
│   │   │   └── 📄 IDRiD_512x512.py        # 数据集配置
│   │   └── 📂 schedules/
│   │       └── 📄 schedule_40k.py         # 训练时长配置 (40k iterations)
│   ├── 📂 sdcnet/
│   │   └── 📄 sdcnetplus_r18...IDRiD.py   # ⭐ SDC-Net 核心配置文件
│   └── 📄 default_runtime.py              # 运行日志配置
├── 📂 mmseg/
│   ├── 📂 models/                         # 模型核心代码存放
│   │   └── 📂 decode_heads/               # 项目模块定义 (ASPP、DSWM等)
│   ├── 📂 apis/                           # mmseg基础功能 (部分继承自train.py)
│   └── 📂 backbones/                      # ResNet18 的标准实现
├── 📂 requirements/                       # 依赖安装目录
└── 📂 tools/
    ├── 📄 train.py                        # 训练脚本
    ├── 📄 test.py                         # 测试脚本
    └── 📄 inference.py                    # 推理脚本

# SDC-Net Docker镜像使用说明
下面教程将基于已获得的SDC-Net docker镜像，说明如何规范化地使用SDC-Net模型及相关环境，包括镜像加载、container启动、VS Code连接配置，以及常见问题的解决。

## 📋 前置要求

在开始之前，请做好如下准备工作：
* 获得SDC-Net docker镜像，并传入服务器。
* 服务器中已安装 **Docker Engine**。
* 如需通过SSH CLI使用Docker中的模型，提前安装MobaXterm等终端软件。
* 如需通过VS Code使用Docker中的模型，本地或远程 VS Code 已安装 **Dev Containers** (ms-vscode-remote.remote-containers) 扩展插件。
* 如需测试模型的训练过程，请提前准备好IDRID、FGADR、DDR数据集，并放置在服务器的特定位置。

---

## 🚀 快速开始 (Quick Start)

### 1. 加载镜像
如果你使用的是离线镜像包 (以`sdcnet.tar`为例)，请先执行加载：

```bash
# 加载镜像包
docker load -i sdcnet.tar
# 查看当前所有镜像，验证镜像是否加载成功
docker images

```

### 2. 启动容器命令说明 (标准指令)

**通用的**、**完整的**启动容器的命令如下所示。

```bash
docker run -itd \      
    --gpus all \      
    --shm-size 8g \
    -v /home_nfs/dataset \
    -v $(pwd)/work_dirs:/app/work_dirs \
    sdc-net:latest /bin/bash

```

该指令已包含显卡调用、内存优化和数据挂载配置。由于采用了数据卷，可以确保训练稳定性和数据安全。

**关键参数说明：**

* `-itd`: **后台静默运行**。即使关闭 VS Code 或断开 SSH，容器内的训练任务也不会中断。正常启动请改为`-it`。
* `--gpus all`: 允许容器调用服务器所有显卡。
* `--shm-size 8g`: **重要**。将共享内存扩大至 8GB，防止 PyTorch DataLoader 报错 `Bus error`。
* `-v (Volume)`: **数据挂载**。格式为 `宿主机路径:容器内路径`，一般来说，docker封装禁止内部封装预权重、数据集等内容，因此预权重和数据集一般是在docker外部的宿主机上通过挂载供docker使用。
* **建议**：将宿主机的 `work_dirs` 挂载到容器内，确保训练权重 (`.pth`) 和日志文件持久化保存，防止删除容器后数据丢失。


### 3. 路径映射规划及容器启动命令

SDC-Net docker内路径和宿主机路径对应关系可规划为下表所示。

| 序号 | Docker路径 | Host路径 | 说明 |
|:---:|:---|:---|:---|
| 1 | `**RESNET18_MODEL**` | `/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net/model` | Resnet18预训练模型存放地址【训练模型时使用】 |
| 2 | `/SDC-Net_use/work_dirs` | `/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net/work_dirs` | 训练模型时训练好的模型存放路径 |
| 3 | `/home_nfs/jiaoli.liu/ISDNet/data/IDRID` | `/home_nfs/group.img.op/dataset/data-jiaoli/IDRID/` | IDRID数据集存放路径 |
| 4 | `/home_nfs/jiaoli.liu/ISDNet/data/FGADR` | `/home_nfs/group.img.op/dataset/data-jiaoli/FGADR/` | FGADR数据集存放路径 |
| 5 | `/home_nfs/jiaoli.liu/ISDNet/data/DDR` | `/home_nfs/group.img.op/dataset/data-jiaoli/DDR/` | DDR数据集存放路径 |
| 6 | `/SDC-Net_use/model/latest.pth` | `/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net/model` | SDC-Net预训练模型存放路径 |
| 7 | `/SDC-Net_use/input_img/test.jpg` |`/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net/input_img` | 推理图片输入路径 |
| 8 | `/SDC-Net_use/output_img` | `/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net/output_img` | 推理图片输出路径 |

*注：`**RESNET18_MODEL**` 表示 `/SDC-Net_use/model/resnet18_8xb32_in1k_20210831-fbbb1da6.pth`*

- 基于以上目录映射关系，完整的启动container的样例命令如下：

```bash
docker run -it \      
    --gpus all \      
    --shm-size 8g \
    -v /home_nfs/group.img.op/dataset/data-jiaoli:/home_nfs/jiaoli.liu/ISDNet/data
    -v /home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net:/SDC-Net_use \
    sdc-net:latest /bin/bash
```

- 从上面的项目代码中指定的各路径，我们在外部挂载时，只需要指定两个路径【数据集+其他】，:前的路径就是你本地挂载的路径，`/home_nfs/group.img.op/dataset/data-jiaoli/IDRID/` 存放数据集，`/home_nfs/xiaopeng.sun/project/medical_seg/SDC-Net`应该像项目文件指定一样，组织几个文件夹：`model`、`work_dirs`、`input_img`、`output_img`

---

## 💻 开发环境配置 (VS Code)

推荐使用 VS Code 的 **Dev Containers** 插件直连容器进行开发，体验与本地一致。

1. **安装插件**：在 VS Code 扩展商店搜索并安装 `Dev Containers`。
2. **连接容器**：
* 按 `F1` 或 `Ctrl+Shift+P` 打开命令面板。
* 输入并选择：`Dev Containers: Attach to Running Container`。
* 在列表中选择刚才启动的容器 `nnunet`。


3. **打开项目**：
* 连接成功后，点击左侧资源管理器 -> **打开文件夹 (Open Folder)**。
* 输入项目路径（默认为 `/app`），点击确定。



现在，你可以在 VS Code 中直接编辑代码、使用终端和调试 Python 脚本。

---

