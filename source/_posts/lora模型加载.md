---
title: lora模型加载
date: 2024-11-11 17:14:18
tags:
  - kubeflow
---

针对 M1 Mac 用户，如果你不想写 Python 代码，而是想用类似 Linux 命令行工具（CLI）的方式来聊天，我有两个最佳推荐：

1. **Apple MLX (最推荐 M1/M2/M3)**：Apple 官方出的 AI 框架，专门为 Mac 芯片优化，速度最快，而且有现成的命令行工具。
2. **Ollama (体验最好)**：目前最流行的命令行 AI 工具，但需要先进行“模型合并”的操作。

鉴于你是 DevOps 背景，我强烈建议走 **“合并模型 -> Ollama”** 这条路。这是最标准的生产化流程。

------

### 方案一：使用 Ollama (最标准的 DevOps 方式)

Ollama 不能直接读取分离的 LoRA 文件，所以我们需要先运行一个脚本，把 **底座模型 (Base)** 和 **LoRA 补丁** 永久“焊接”在一起，生成一个新的模型文件。

#### 第一步：合并模型 (Merge)

在你的项目目录下，创建一个 `merge.py` 脚本：

Python

```
import torch
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer
import os

# 1. 配置路径 (请确保这些路径和你之前的一致)
base_path = os.path.expanduser("~/ai-models/qwen/Qwen1.5-0.5B-Chat")
lora_path = os.path.expanduser("~/Downloads/qwen-lora-result")
save_path = os.path.expanduser("~/ai-models/qwen-merged") # 合并后的保存位置

print("🚀 正在加载底座模型...")
base_model = AutoModelForCausalLM.from_pretrained(
    base_path, device_map="cpu", trust_remote_code=True, torch_dtype=torch.float16
)

print("🛠 正在加载并合并 LoRA...")
model = PeftModel.from_pretrained(base_model, lora_path)
model = model.merge_and_unload() # 关键步骤：焊接！

print(f"💾 正在保存合并后的模型到: {save_path}")
model.save_pretrained(save_path)
# 顺便把 tokenizer 也拷过去，Ollama 需要
tokenizer = AutoTokenizer.from_pretrained(base_path, trust_remote_code=True)
tokenizer.save_pretrained(save_path)

print("✅ 合并完成！")
```

**运行合并：**

Bash

```
./venv/bin/python merge.py
```

*这会在 `~/ai-models/qwen-merged` 生成一个完整的模型。*

#### 第二步：安装 Ollama

如果你还没安装 Ollama：

1. 去 [ollama.com](https://ollama.com/) 下载 Mac 版并安装。
2. 安装后在终端验证：`ollama --version`

#### 第三步：创建 Modelfile

Ollama 需要一个配置文件来知道怎么运行你的模型。在终端创建一个名为 `Modelfile` 的文件（无后缀）：

Dockerfile

```
# 告诉 Ollama 模型文件在哪 (请替换为你的真实路径)
FROM /Users/linchangkun/ai-models/qwen-merged

# 设置参数 (千问推荐参数)
PARAMETER temperature 0.7
PARAMETER top_p 0.8
PARAMETER stop "<|endoftext|>"
PARAMETER stop "<|im_start|>"
PARAMETER stop "<|im_end|>"

# 设置对话模板 (千问标准模板)
TEMPLATE """<|im_start|>system
{{ .System }}<|im_end|>
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""

# 设置系统提示词
SYSTEM """你是一个乐于助人的 AI 助手。"""
```

*注意：`FROM` 后面必须是绝对路径。*

#### 第四步：构建并运行

在终端运行以下命令：

Bash

```
# 1. 创建模型 (名字叫 qwen-lora)
ollama create qwen-lora -f Modelfile

# 2. 运行对话 (进入 CLI 聊天界面)
ollama run qwen-lora
```

**现在，你就可以在漂亮的命令行界面里和你的模型对话了！** 还可以随时输入 `/bye` 退出。

