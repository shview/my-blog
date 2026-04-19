---
abbrlink: ''
categories: []
date: '2026-04-19T21:31:50.129569+08:00'
excerpt: 配置复现论文[2023-NIPS] Time Series as Images环境
tags:
- 教程
- 记录
title: 配置复现论文[2023-NIPS] Time Series as Images环境
updated: '2026-04-19T21:31:52.994+08:00'
---
# P100 显卡 Vision-Text 模型部署避坑指南

本文档记录了在华为云（或类似云平台）P100 显卡节点上，部署和微调 Vision-Text 双流分类模型时的核心避坑经验与环境配置标准流程。

为论文[2023-NIPS] Time Series as Images的复现

使用镜像为`pytorch\_2\_1:pytorch\_2.1.0-cuda\_12.1-py\_3.10.6-ubuntu\_22.04-x86\_64-20250305173557-cb53968`，实例规格为`1 \* Pnt1(16GB) | 8 vCPUs | 64 GB (modelarts.vm.gpu.p100)`，储存容量设定为50GB

因为原论文使用了48G显卡而华为云没有，所以TMD根本跑不动，一直报错，最后模型降维，放弃 Swin，改用 ViT才得以实现

最终结果

![](https://github.com/shview/my-blog/raw/main/source/images/2026/04-19-image.png)


以下内容是在AI(Gemini)的帮助下部署环境总结的需要修改的部分

---



## 🚀 环境依赖与包管理 (Environment Setup)

在全新或重启后的环境中，第一次安装依赖时最容易遇到版本冲突。

* **修改点 1：使用 `scikit-learn` 替代 `sklearn`**
  * **安装指令：**
    ```bash
    pip install transformers==4.30.0 datasets==2.10.0 pyarrow==11.0.0 accelerate scikit-learn torchvision torch==1.12.1+cu113 --extra-index-url [https://download.pytorch.org/whl/cu113](https://download.pytorch.org/whl/cu113)
    ```
  * **原因：** 官方已经废弃了 `sklearn` 这个包名，继续使用会导致 pip 抛出 `metadata-generation-failed` 错误。
* **注意点 2：提防 `transformers 4.30.0` 与 `pytest` 的版本冲突**
  * **现象：** 报错 `No module named '_pytest'` 或 `cannot import name 'Module' from '_pytest.doctest'`。
  * **原因：**`transformers` 内部机制在有 `pytest` 残留时会尝试加载测试工具模块，导致初始化失败。需通过**代码级精确导入**（见下文）彻底绕过。
* **注意点 3：平台特性的“失忆”**
  * **动作：** 每次**重启实例**或**新开终端**后，必须重新执行环境变量注入：
    ```bash
    export PYTHONPATH=/home/ma-user/work/ViTST/code:$PYTHONPATH
    export HF_ENDPOINT=[https://hf-mirror.com](https://hf-mirror.com)
    ```
  * **原因：** 云端实例重启后，`/home/ma-user/work` 目录下的文件会保留，但内存中的环境变量会被清空。

---

## 💻 代码逻辑修正 (Code Modifications)

为了让模型在旧版本 `transformers` 下顺利跑通，需对 `run_VisionTextCLS.py` 进行以下修改：

* **修改点 1：摒弃通配符，改为“精确导入” (约 14 行)**
  * **动作：** 将 `from transformers import *` 修改为具体的类名导入：
    ```python
    import transformers
    from transformers import (
        TrainingArguments, Trainer, AutoTokenizer, AutoFeatureExtractor, 
        EarlyStoppingCallback, AutoConfig, AutoModelForSequenceClassification, AutoModel
    )
    ```
  * **原因：** 通配符 `*` 会强行扫描所有子模块（包括存在 Bug 的 `testing_utils`）。精确导入能完美避开环境冲突地雷。
* **修改点 2：删除不支持的 `TrainingArguments` 参数 (约 264 行)**
  * **动作：** 删除 `dataloader_prefetch_factor=2` 参数。
  * **原因：** 当前安装的 `transformers` 版本尚未将该底层 PyTorch 参数暴露，保留它会引发 `TypeError`。
* **修改点 3：强制缩放图片分辨率至 224 (约 190 行)**
  * **动作：** 在 `train_transforms` 和 `val_transforms` 的 `Compose` 列表中添加 `Resize((224, 224))`。
  * **原因：** 数据预处理阶段必须进行物理图片缩放，否则会导致模型 (接受 224) 和 Dataloader (输入 384) 产生 `ValueError` 尺寸不匹配。

---

## ⚙️ 硬件与底层算子适配 (The P100 Survival Kit)

P100 显卡极易触发 `CUDNN_STATUS_NOT_INITIALIZED`，以下是终极降压策略：

* **修改点 1：模型降维，放弃 Swin，改用 ViT**
  * **动作：** 启动指令中改用 `--image_model vit`。
  * **原因：** P100 较老的 CUDA 驱动架构在初始化 Swin 复杂的“窗口平移卷积”算子时容易存在底层兼容性死锁。ViT 的纯自注意力架构对旧卡支持极佳。
* **修改点 2：强行关闭 cuDNN 卷积加速 (文件顶部)**
  * **动作：** 在 `run_VisionTextCLS.py` 顶部加入以下代码：
    ```python
    import torch
    torch.backends.cudnn.enabled = False
    torch.backends.cudnn.benchmark = False
    torch.backends.cudnn.deterministic = True
    ```
  * **原因：** 强制 PyTorch 使用不依赖专有库的底层通用算子，牺牲极少的速度换取 100% 的初始化稳定性。
* **注意点 3：显存清空与防爆**
  * **动作：** 每次报错或强制中断后，执行 `killall -9 python` 彻底清空显卡碎片。
  * **参数设定：** 224 分辨率 + ViT 架构下，P100 的 16G 显存可稳定运行 `train_batch_size=16`。

---

## 🏁 标准启动模板

综合以上所有修改，一键启动的最稳妥命令如下：

```bash
# 1. 注入环境
export PYTHONPATH=/home/ma-user/work/ViTST/code:$PYTHONPATH
export HF_ENDPOINT=[https://hf-mirror.com](https://hf-mirror.com)

# 2. 清理僵尸进程
killall -9 python

# 3. 启动最稳配置 (ViT + Roberta + BS16 + 224尺寸)
cd /home/ma-user/work/ViTST/code/Vision-Text
nohup python run_VisionTextCLS.py \
    --image_model vit \
    --text_model roberta \
    --freeze_vision_model False \
    --freeze_text_model False \
    --dataset P12 \
    --dataset_prefix order_differ_interpolation_-x1_xx2_6x6_384x384_ \
    --seed 1799 \
    --save_total_limit 1 \
    --train_batch_size 16 \
    --eval_batch_size 32 \
    --logging_steps 20 \
    --save_steps 100 \
    --epochs 4 \
    --learning_rate 2e-5 \
    --n_runs 1 \
    --n_splits 5 \
    --do_train > train_output.log 2>&1 &
```

最后记得加上

~~~
tail -f train_output.log
~~~

来查看进度

最后，希望都可以快速地配置好环境，MD这个配置了我得整整8个小时

最后还有就是逆天华为，~~华为好，华为美，华为为我增智慧~~
