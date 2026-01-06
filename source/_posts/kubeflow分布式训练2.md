---
title: kubeflow训练2
date: 2024-11-11 17:14:18
tags:
  - kubeflow
---

## 安装

~~~bash
https://github.com/kubeflow/trainer.git

git fetch --tags
git checkout v2.1.0
kubectl apply --server-side -k manifests/overlays/manager

## 确认 pod 启动
kubectl get pods -n kubeflow-system
NAME                                                  READY   STATUS    RESTARTS        AGE
jobset-controller-manager-7b54585c9c-j7zbx            1/1     Running   8 (3m8s ago)    38m
kubeflow-trainer-controller-manager-86667676b-bshbf   1/1     Running   2 (6m32s ago)   57m

##!! 如果镜像下载不下来 kubectl edit deployment jobset-controller-manager -n kubeflow-system, 修改 deployment 中的镜像配置

## 安装 runtimes
kubectl apply --server-side -k manifests/overlays/runtimes

## 查看 runtimes
kubectl get clustertrainingruntimes
NAME                     AGE
deepspeed-distributed    38m
mlx-distributed          38m
torch-distributed        38m
torchtune-llama3.2-1b    38m
torchtune-llama3.2-3b    38m
torchtune-qwen2.5-1.5b   38m

## 修改 runtime 中的镜像配置
kubectl edit clustertrainingruntimes torch-distributed
~~~

~~~yaml
# volcano 配置
apiVersion: trainer.kubeflow.org/v1alpha1
kind: ClusterTrainingRuntime
metadata:
  creationTimestamp: "2026-01-05T10:07:51Z"
  finalizers:
  - trainer.kubeflow.org/resource-in-use
  generation: 8
  labels:
    trainer.kubeflow.org/framework: torch
  name: torch-distributed
  resourceVersion: "183940"
  uid: 61da91e8-8f96-4d7a-920c-66b314ebf5d1
spec:
  mlPolicy:
    numNodes: 1
    torch:
      numProcPerNode: auto
  podGroupPolicy:
    volcano:
      networkTopology:
        mode: hard
        highestTierAllowed: 1
  template:
    metadata: {}
    spec:
      replicatedJobs:
      - groupName: default
        name: node
        replicas: 1
        template:
          metadata:
            labels:
              trainer.kubeflow.org/trainjob-ancestor-step: trainer
          spec:
            template:
              metadata: {}
              spec:
                containers:
                - image: hub.dev.iot5gx.com:2443/pytorch/pytorch:2.7.1-cuda12.8-cudnn9-runtime
                  name: node
                  resources: {}
                schedulerName: volcano
~~~

