~~~python
import logging
from kubeflow.trainer import TrainerClient
from kubeflow.trainer.types.types import CustomTrainer

logging.basicConfig(level=logging.INFO)

# ================= 1. RustFS (S3) 配置 =================
# 如果你是 OrbStack: http://host.orb.internal:9000
# 如果你是 Docker Desktop: http://host.docker.internal:9000
# 如果是本机直接运行 RustFS，也可以尝试局域网 IP: http://192.168.x.x:9000
RUSTFS_ENDPOINT = "http://172.18.0.6:9000"

# ⚠️ 请替换为你 RustFS 的真实账号密码
RUSTFS_ACCESS_KEY = "minio"
RUSTFS_SECRET_KEY = "password"

# 数据配置
MODEL_BUCKET = "model"
MODEL_DIR_NAME = "Qwen3-0___6B" # Bucket 下的文件夹名

DATASET_BUCKET = "dataset"
DATASET_FILE_NAME = "test.json"

# ================= 2. 训练逻辑 (Pod 内运行) =================
def train_with_rustfs():
    import os
    import boto3
    import torch
    import json
    from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments, Trainer
    from peft import LoraConfig, get_peft_model, TaskType
    from datasets import Dataset

    # 获取环境变量
    endpoint = os.environ['RUSTFS_ENDPOINT']
    ak = os.environ['RUSTFS_ACCESS_KEY']
    sk = os.environ['RUSTFS_SECRET_KEY']

    print(f"📡 [Pod] start Connecting to RustFS at {endpoint}...")

    # 初始化 S3 客户端
    s3 = boto3.client('s3',
        endpoint_url=endpoint,
        aws_access_key_id=ak,
        aws_secret_access_key=sk
    )

    print(f"📡 [Pod] Connecting to RustFS at {endpoint}...")

    # --- 辅助函数：下载文件夹 ---
    def download_s3_folder(bucket, prefix, local_dir):
        print(f"⬇️ [Pod] Downloading folder: s3://{bucket}/{prefix} -> {local_dir}")
        paginator = s3.get_paginator('list_objects_v2')
        pages = paginator.paginate(Bucket=bucket, Prefix=prefix)

        count = 0
        for page in pages:
            if 'Contents' not in page: continue
            for obj in page['Contents']:
                key = obj['Key']
                # 处理相对路径，确保下载后结构正确
                # 例如 key="Qwen3/config.json", relative="config.json"
                if key.endswith('/'): continue # 跳过空文件夹对象

                relative_path = os.path.relpath(key, prefix)
                local_file_path = os.path.join(local_dir, relative_path)

                os.makedirs(os.path.dirname(local_file_path), exist_ok=True)
                s3.download_file(bucket, key, local_file_path)
                print(f"   - Downloaded: {relative_path}")
                count += 1
        print(f"✅ Downloaded {count} model files.")

    # --- 步骤 1: 下载模型 ---
    local_model_path = "/tmp/model"
    model_bucket = os.environ['MODEL_BUCKET']
    model_prefix = os.environ['MODEL_DIR_NAME']

    download_s3_folder(model_bucket, model_prefix, local_model_path)

    # --- 步骤 2: 下载数据集 ---
    local_data_path = "/tmp/data/test.json"
    os.makedirs(os.path.dirname(local_data_path), exist_ok=True)

    ds_bucket = os.environ['DATASET_BUCKET']
    ds_file = os.environ['DATASET_FILE_NAME']

    print(f"⬇️ [Pod] Downloading dataset: s3://{ds_bucket}/{ds_file}")
    s3.download_file(ds_bucket, ds_file, local_data_path)

    # --- 步骤 3: 加载模型 ---
    print(f"📦 [Pod] Loading Tokenizer...")
    tokenizer = AutoTokenizer.from_pretrained(local_model_path, local_files_only=True, trust_remote_code=True)
    if tokenizer.pad_token is None: tokenizer.pad_token = tokenizer.eos_token

    print(f"📦 [Pod] Loading Model (CPU Mode)...")
    model = AutoModelForCausalLM.from_pretrained(
        local_model_path,
        torch_dtype=torch.float32,
        device_map="cpu",
        local_files_only=True,
        trust_remote_code=True
    )

    print("⚙️ [Pod] Applying LoRA...")
    peft_config = LoraConfig(task_type=TaskType.CAUSAL_LM, r=8, target_modules=['q_proj', 'v_proj'])
    model = get_peft_model(model, peft_config)

    # --- 步骤 4: 加载本地 JSON 数据集 ---
    print("🔄 [Pod] Loading Dataset from JSON...")
    # 使用 HuggingFace Datasets 加载刚才下载的 JSON
    dataset = Dataset.from_json(local_data_path)

    # 简单打印一下数据看看
    print(f"   Data sample: {dataset[0]}")

    def process(x):
        # 假设 json 里有 "text" 字段，如果你的字段名不一样，请在这里修改
        # 例如: text = f"User: {x['instruction']} Assistant: {x['output']}"
        text = x.get("text", str(x))
        inputs = tokenizer(text, padding="max_length", max_length=128, truncation=True)

        # 3. ⚠️【关键修复】添加 labels
        # 如果不加这一行，Trainer 就会报 ValueError: The model did not return a loss...
        # 只有告诉模型 "labels" 是什么，它才知道怎么算 loss
        inputs["labels"] = inputs["input_ids"]

        return inputs

    tokenized_ds = dataset.map(process)

    # --- 步骤 5: 训练 ---
    print("🔥 [Pod] Starting Training...")
    args = TrainingArguments(
        output_dir="/tmp/output",
        max_steps=1000,
        use_cpu=True,
        per_device_train_batch_size=1,

        logging_steps=1,       # 关键：每走 1 步就打印一次 Loss
        disable_tqdm=False,    # 确保显示进度条
        report_to="none"       # 不尝试连接 wandb/tensorboard，纯打印
    )
    trainer = Trainer(model=model, args=args, train_dataset=tokenized_ds)
    trainer.train()
    print("✅ [Pod] Training Completed with RustFS Data!")

# ================= 3. 提交任务 =================
def submit_job():
    client = TrainerClient()

    trainer = CustomTrainer(
        func=train_with_rustfs,
        # ⚠️ 关键：必须安装 boto3 用于连接 RustFS
        packages_to_install=[
            "boto3", "transformers", "peft", "torch", "accelerate", "datasets", "tiktoken"
        ],
        num_nodes=2,
        env={
            "PET_NPROC_PER_NODE": "1",
            "OMP_NUM_THREADS": "1",
            # 注入配置
            "RUSTFS_ENDPOINT": RUSTFS_ENDPOINT,
            "RUSTFS_ACCESS_KEY": RUSTFS_ACCESS_KEY,
            "RUSTFS_SECRET_KEY": RUSTFS_SECRET_KEY,
            "MODEL_BUCKET": MODEL_BUCKET,
            "MODEL_DIR_NAME": MODEL_DIR_NAME,
            "DATASET_BUCKET": DATASET_BUCKET,
            "DATASET_FILE_NAME": DATASET_FILE_NAME
        },
        resources_per_node={"cpu": "4", "memory": "8Gi", "gpu": "0"}
    )

    print("🚀 Submitting Job (RustFS Mode)...")

    # 清理旧资源
    import subprocess
    subprocess.run("kubectl delete trainjob --all", shell=True)
    subprocess.run("kubectl delete pods --all --force --grace-period=0", shell=True)

    job_id = client.train(trainer=trainer)
    print(f"✅ Job Submitted! ID: {job_id}")
    print(f"🔍 Watch logs: kubectl logs -n kubeflow-system -f {job_id}-worker-0")

if __name__ == "__main__":
    submit_job()
~~~
