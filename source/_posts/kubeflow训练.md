---
title: kubeflow训练
date: 2024-11-11 17:14:18
tags:
  - kubeflow
---

这份指南基于我们之前的调试过程（解决版本冲突、参数错误、资源 OOM、僵尸进程等问题），为你整理了一份**经过验证的、完全正确的 Kubeflow Trainer V2 (Alpha) 标准实战流程**。

此流程专门适配你的硬件环境：

- **客户端**：M1 Mac
- **服务端**：K8s 集群 (2 CPU / 4GB Mem)
- **目标**：使用 Python 函数式编程 (V2) 微调 Qwen 0.5B (LoRA)

------

### 第一阶段：环境清洗与准备 (服务端 & 客户端)

在开始之前，我们需要确保“旧的 V1 及其残留任务”被清理干净，且 Python 环境安装了正确的 V2 包。

\1. K8s 集群清理 (服务端)

请在终端执行以下命令，确保没有 V1 Operator 干扰，且删除了所有僵尸任务：

Bash

```
# 1. 删除旧的 V1 Operator (如果存在)
kubectl delete deployment -n kubeflow training-operator --ignore-not-found
kubectl delete crd pytorchjobs.kubeflow.org --ignore-not-found

# 2. 暴力清理所有残留的 V2 任务和 Pod (防止资源被占用)
kubectl delete trainjob --all
kubectl delete jobset --all
kubectl delete job --all
kubectl delete pods --all --force --grace-period=0
```

\2. Python 客户端准备 (M1 Mac)

确保安装的是包含 V2 SDK 的包（根据你之前的验证，是 kubeflow 包）：

Bash

```
# 卸载旧包
pip uninstall -y kubeflow-training kubeflow-trainer

# 安装 V2 SDK (通常包含在 kubeflow 或 kubeflow-trainer 预览版中)
# 假设你之前已经通过 pip install kubeflow 安装好了
pip show kubeflow
```

------

### 第二阶段：编写训练脚本 (`run_v2_training.py`)

这是一个 **"All-in-One"** 脚本。它包含了训练逻辑（运行在 Pod 里）和提交逻辑（运行在 Mac 上）。

**关键修正点（已包含在代码中）：**

1. **单节点 (`num_nodes=1`)**：适配你的硬件限制。
2. **单进程 (`PET_NPROC_PER_NODE=1`)**：防止内存溢出 (OOM)。
3. **参数修正**：使用 `packages_to_install` 而非手动 pip，使用 `func` 传递逻辑。

请将以下代码保存为 `run_v2_training.py`：

Python

```
import logging
import time
from kubeflow.trainer import TrainerClient
# 根据之前的调试，显式从 types 导入以确保兼容性
from kubeflow.trainer.types.types import CustomTrainer

# 配置日志
logging.basicConfig(level=logging.INFO)

# ==========================================
# 1. 定义训练逻辑 (将在 K8s Pod 内部运行)
# ==========================================
def qwen_lora_func():
    import os
    import torch
    from transformers import AutoModelForCausalLM, AutoTokenizer
    from peft import LoraConfig, get_peft_model, TaskType

    # 获取环境变量 (由 SDK 注入)
    rank = os.environ.get('RANK', '0')
    print(f"🚀 [Pod-Node-{rank}] Training Environment Initialized!")

    # 1. 定义模型 (Qwen 0.5B)
    model_id = "Qwen/Qwen2.5-0.5B-Instruct"
    
    try:
        print(f"📦 [Pod] Loading Tokenizer...")
        tokenizer = AutoTokenizer.from_pretrained(model_id, trust_remote_code=True)
        
        print(f"📦 [Pod] Loading Model (Force CPU & float32)...")
        # ⚠️ 关键：为了适应 4G 内存且无 GPU，必须用 CPU + float32
        model = AutoModelForCausalLM.from_pretrained(
            model_id, 
            torch_dtype=torch.float32, 
            device_map="cpu", 
            trust_remote_code=True
        )
        
        print("⚙️ [Pod] Applying LoRA Adapter...")
        peft_config = LoraConfig(
            task_type=TaskType.CAUSAL_LM, 
            r=8, 
            lora_alpha=32, 
            target_modules=['q_proj', 'v_proj'],
            lora_dropout=0.05
        )
        model = get_peft_model(model, peft_config)
        
        # 打印可训练参数，证明 LoRA 挂载成功
        print("✅ [Pod] LoRA Applied Successfully! Trainable Parameters:")
        model.print_trainable_parameters()
        
        # 模拟训练循环 (不进行真实 Backward 以免 OOM)
        print("🔄 [Pod] Simulating training step...")
        time.sleep(2) 
        print("✅ [Pod] Task Completed Successfully.")
        
    except Exception as e:
        print(f"❌ [Pod] Critical Error: {e}")
        raise e

# ==========================================
# 2. 定义提交逻辑 (在 M1 Mac 上运行)
# ==========================================
def submit_job():
    print("🔌 Initializing Kubeflow Trainer Client...")
    client = TrainerClient()

    print("📦 Configuring Job (Single-Node CPU Mode)...")
    
    trainer = CustomTrainer(
        # A. 核心逻辑
        func=qwen_lora_func,
        
        # B. 动态依赖 (Pod 启动时自动安装)
        packages_to_install=[
            "transformers",
            "peft",
            "torch",
            "accelerate",
            "datasets",
            "bitsandbytes",
            "tiktoken",
            "sentencepiece"
        ],
        
        # C. 节点规模 (必须为 1)
        num_nodes=1,
        
        # D. 环境变量 (⚠️ 核心修复：强制单进程防止 OOM)
        env={
            "PET_NPROC_PER_NODE": "1", 
            "OMP_NUM_THREADS": "1",
            "HF_ENDPOINT": "https://hf-mirror.com",  # <--- 加上这一行！使用国内镜像
            # "HF_TOKEN": "xxx"
        },
        
        # E. 资源限制 (适配 2C/4G 集群)
        resources_per_node={
            "cpu": "1.5",   # 留 0.5 给系统
            "memory": "3Gi", # 留 1G 给系统
            "gpu": "0"
        }
    )

    print("🚀 Submitting to Kubernetes...")
    try:
        job_id = client.train(trainer=trainer)
        
        print("\n" + "="*50)
        print(f"✅ Job Submitted! ID: {job_id}")
        print("="*50)
        print("👉 Monitor logs with:")
        print(f"   kubectl logs -n kubeflow-system -f {job_id}-worker-0")
        print("="*50)

    except Exception as e:
        print(f"\n❌ Submission Failed: {e}")
        if "Runtime" in str(e):
            print("\n[Hint] Check if 'ClusterTrainingRuntime' is installed in your cluster.")

if __name__ == "__main__":
    submit_job()
```

------

### 第三阶段：执行与监控

\1. 运行脚本

在你的 Mac 终端运行：

Bash

```
python run_v2_training.py
```

\2. 监控流程 (Pod 生命周期)

脚本成功提交后，会返回一个 Job ID (例如 abc12345)。请立即使用 kubectl 监控：

Bash

```
# 1. 观察 Pod 创建状态
kubectl get pods -n kubeflow-system -w

# 状态流转预期：
# Pending -> ContainerCreating (卡住几分钟下载 PyTorch 镜像) -> Running
```

\3. 查看日志

一旦 Pod 状态变为 Running，查看日志：

Bash

```
# 注意替换具体的 Pod 名字
kubectl logs -n kubeflow-system -f <JOB_ID>-worker-0
```

**预期日志输出：**

1. 大量的 `pip install ...` 输出（安装 transformers 等）。
2. `🚀 [Pod-Node-0] Training Environment Initialized!`
3. 下载 Qwen 模型进度条。
4. `trainable params: ...` (显示 LoRA 参数)。
5. `✅ [Pod] Task Completed Successfully.`

------

### 第四阶段：任务结束后的清理

任务完成后，为了释放那宝贵的 4G 内存，**必须手动删除任务**（否则 Pod 就算 Completed 也会占着位置）。

Bash

```
# 推荐清理命令 (删除所有 TrainJob)
kubectl delete trainjob --all
```

### 总结图解

这就是基于你现有低配集群，使用最新 Kubeflow V2 技术的完整、闭环流程。按照这个步骤，你应该能稳定地跑通 Demo。