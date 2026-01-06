---
r57114c5deb2-node-0-0-xqpjhr57114c5deb2-node-0-0-xqpjhtitle: kubeflow分布式训练
date: 2024-11-11 17:14:18
tags:
  - kubeflow
---

~~~python
import datetime
import logging

from kubeflow.trainer import TrainerClient
from kubeflow.trainer.types.types import CustomTrainer

logging.basicConfig(level=logging.INFO)

# ================= 1. RustFS (S3) 配置 =================
RUSTFS_ENDPOINT = "http://172.17.0.1:9000"
RUSTFS_ACCESS_KEY = "minio"
RUSTFS_SECRET_KEY = "password"

# 输入配置
MODEL_BUCKET = "model"
MODEL_DIR_NAME = "Qwen1___5-0___5B-Chat"
DATASET_BUCKET = "dataset"
DATASET_FILE_NAME = "test.json"

# 【修改点 1】输出 Bucket 固定为 ai_train_output
# 文件夹名 (OUTPUT_PREFIX) 将在 submit_job 中动态生成
OUTPUT_BUCKET = "ai-train-output"

# ================= 2. 训练逻辑 (Pod 内运行) =================
def train_with_rustfs():
    import os

    import boto3
    import torch
    from datasets import Dataset
    from peft import LoraConfig, TaskType, get_peft_model
    from transformers import (
        AutoModelForCausalLM,
        AutoTokenizer,
        Trainer,
        TrainingArguments,
    )

    # --- 获取环境变量 ---
    endpoint = os.environ['RUSTFS_ENDPOINT']
    ak = os.environ['RUSTFS_ACCESS_KEY']
    sk = os.environ['RUSTFS_SECRET_KEY']

    # 获取输出配置
    output_bucket = os.environ['OUTPUT_BUCKET']
    output_prefix = os.environ['OUTPUT_PREFIX'] # 这里就是 Job ID

    print(f"📡 [Pod] Connecting to RustFS at {endpoint}...")
    print(f"🎯 [Pod] Output Target: s3://{output_bucket}/{output_prefix}")

    # 初始化 S3 客户端
    s3 = boto3.client('s3',
        endpoint_url=endpoint,
        aws_access_key_id=ak,
        aws_secret_access_key=sk
    )

    # --- 辅助函数：下载文件夹 ---
    def download_s3_folder(bucket, prefix, local_dir):
        print(f"⬇️ [Pod] Downloading folder: s3://{bucket}/{prefix} -> {local_dir}")
        paginator = s3.get_paginator('list_objects_v2')
        pages = paginator.paginate(Bucket=bucket, Prefix=prefix)
        for page in pages:
            if 'Contents' not in page: continue
            for obj in page['Contents']:
                key = obj['Key']
                if key.endswith('/'): continue
                relative_path = os.path.relpath(key, prefix)
                local_file_path = os.path.join(local_dir, relative_path)
                os.makedirs(os.path.dirname(local_file_path), exist_ok=True)
                s3.download_file(bucket, key, local_file_path)

    # --- 辅助函数：上传文件夹 ---
    def upload_folder_to_s3(local_dir, bucket, s3_prefix):
        print(f"⬆️ [Pod] Uploading results: {local_dir} -> s3://{bucket}/{s3_prefix}")
        files_count = 0
        for root, dirs, files in os.walk(local_dir):
            for file in files:
                local_path = os.path.join(root, file)
                relative_path = os.path.relpath(local_path, local_dir)
                # 最终路径: job-id/relative_path
                s3_key = os.path.join(s3_prefix, relative_path)

                print(f"   - Uploading: {s3_key}")
                try:
                    s3.upload_file(local_path, bucket, s3_key)
                    files_count += 1
                except Exception as e:
                    print(f"   ❌ Failed to upload {file}: {e}")
        print(f"✅ Uploaded {files_count} files.")

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
    s3.download_file(ds_bucket, ds_file, local_data_path)

    # --- 步骤 3: 加载模型 & LoRA ---
    print(f"📦 [Pod] Loading Model...")
    tokenizer = AutoTokenizer.from_pretrained(local_model_path, local_files_only=True, trust_remote_code=True)
    if tokenizer.pad_token is None: tokenizer.pad_token = tokenizer.eos_token

    model = AutoModelForCausalLM.from_pretrained(
        local_model_path, torch_dtype=torch.float32, device_map="cpu", local_files_only=True, trust_remote_code=True
    )

    peft_config = LoraConfig(task_type=TaskType.CAUSAL_LM, r=8, target_modules=['q_proj', 'v_proj'])
    model = get_peft_model(model, peft_config)

    # --- 步骤 4: 处理数据 ---
    dataset = Dataset.from_json(local_data_path)
    def process(x):
        text = x.get("text", str(x))
        inputs = tokenizer(text, padding="max_length", max_length=128, truncation=True)
        inputs["labels"] = inputs["input_ids"]
        return inputs
    tokenized_ds = dataset.map(process)

    # --- 步骤 5: 训练 ---
    print("🔥 [Pod] Starting Training...")
    args = TrainingArguments(
        output_dir="/tmp/output",
        max_steps=20,
        use_cpu=True,
        per_device_train_batch_size=1,
        logging_steps=1,
        save_strategy="no",
        report_to="none"
    )
    trainer = Trainer(model=model, args=args, train_dataset=tokenized_ds)
    trainer.train()

    # --- 步骤 6: 保存并上传结果 ---
    print("💾 [Pod] Saving final model locally...")
    final_save_path = "/tmp/final_model"
    trainer.save_model(final_save_path)
    tokenizer.save_pretrained(final_save_path)

    # 上传到 s3://ai_train_output/{job_id}/...
    upload_folder_to_s3(final_save_path, output_bucket, output_prefix)

    print("✅ [Pod] All tasks finished!")

# ================= 3. 提交任务 =================
def submit_job():
    client = TrainerClient()

    # 【修改点】使用 yyyyMMddHHmmss 作为 S3 文件夹名
    # 例如: 20260103194527
    s3_run_id = datetime.datetime.now().strftime("%Y%m%d%H%M%S")

    print(f"🆔 S3 Storage ID: {s3_run_id}")
    print(f"📂 Output S3 Path: s3://{OUTPUT_BUCKET}/{s3_run_id}/")

    trainer = CustomTrainer(
        func=train_with_rustfs,
        packages_to_install=[
            "boto3", "transformers", "peft", "torch", "accelerate", "datasets", "tiktoken"
        ],
        num_nodes=1,
        env={
            "PET_NPROC_PER_NODE": "1",
            "OMP_NUM_THREADS": "1",
            "RUSTFS_ENDPOINT": RUSTFS_ENDPOINT,
            "RUSTFS_ACCESS_KEY": RUSTFS_ACCESS_KEY,
            "RUSTFS_SECRET_KEY": RUSTFS_SECRET_KEY,
            "MODEL_BUCKET": MODEL_BUCKET,
            "MODEL_DIR_NAME": MODEL_DIR_NAME,
            "DATASET_BUCKET": DATASET_BUCKET,
            "DATASET_FILE_NAME": DATASET_FILE_NAME,
            # 将时间戳注入环境变量
            "OUTPUT_BUCKET": OUTPUT_BUCKET,
            "OUTPUT_PREFIX": s3_run_id,
        },
        resources_per_node={"cpu": "2", "memory": "6Gi", "gpu": "0"}
    )

    print("🚀 Submitting Job...")

    # 清理旧资源
    import subprocess
    subprocess.run("kubectl delete trainjob --all", shell=True)
    subprocess.run("kubectl delete pods --all --force --grace-period=0", shell=True)

    # SDK 自动生成 K8s 任务 ID
    k8s_job_id = client.train(trainer=trainer)

    print("-" * 40)
    print(f"✅ Job Submitted Successfully!")
    print(f"🏷️  K8s Job Name: {k8s_job_id} (用于查看日志)")
    print(f"📦 S3 Output Dir: {s3_run_id} (用于下载模型)")
    print("-" * 40)

    print(f"🔍 Watch logs: python print_log.py --name {k8s_job_id}")

if __name__ == "__main__":
    submit_job()

~~~

~~~python
## print_log.py
import argparse
import logging
import time

from kubeflow.trainer import TrainerClient

# 设置日志级别
logging.basicConfig(level=logging.INFO)

def main():
    parser = argparse.ArgumentParser(description="循环增量获取 Kubeflow Trainer Job 日志")
    parser.add_argument(
        "--name",
        type=str,
        required=True,
        help="Job 的名称 (例如: kd7fa8a6c9b8)"
    )
    args = parser.parse_args()
    job_name = args.name

    client = TrainerClient()
    print(f"开始监听 Job: {job_name} 的日志 (增量模式, 按 Ctrl+C 停止)...")

    # 记录上次打印到的行数下标
    last_line_index = 0

    while True:
        try:
            # === 修改点在这里 ===
            # get_job_logs 返回的是 generator，必须转成 list 才能获取长度和切片
            log_generator = client.get_job_logs(name=job_name)
            all_logs = list(log_generator)

            # 计算当前总行数
            current_total_lines = len(all_logs)

            # 判断是否有新日志
            if current_total_lines > last_line_index:
                # 获取从 last_line_index 开始的所有新行
                new_lines = all_logs[last_line_index:]

                # 打印新行
                if new_lines:
                    print("\n".join(new_lines))

                # 更新游标
                last_line_index = current_total_lines

            # 如果日志变少了（比如 Pod 重启），重置游标
            elif current_total_lines < last_line_index:
                # 只有当 current_total_lines > 0 时才认为是重启，防止网络波动获取到空列表误判
                if current_total_lines > 0:
                    logging.warning("日志行数减少，可能 Pod 已重启，重新输出...")
                    last_line_index = 0
                # 如果获取到 0 行，可能是暂时没取到，保持 last_line_index 不变或根据实际情况处理

            time.sleep(2)

        except KeyboardInterrupt:
            print("\n停止监听。")
            break
        except Exception as e:
            # 打印具体错误类型，方便调试
            logging.error(f"获取日志时发生错误: {type(e).__name__}: {e}")
            time.sleep(5)

if __name__ == "__main__":
    main()

~~~

