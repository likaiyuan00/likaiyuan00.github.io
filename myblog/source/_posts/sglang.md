---
title: sglang
date: 2025-09-23 14:44:29
tags: llm
---
# SGLang 部署与使用指南

## 目录
- [SGLang 部署与使用指南](#sglang-部署与使用指南)
  - [目录](#目录)
  - [环境准备与安装](#环境准备与安装)
    - [基础环境配置](#基础环境配置)
  - [Docker容器部署](#docker容器部署)
  - [服务启动配置](#服务启动配置)
    - [单节点启动](#单节点启动)
    - [双节点分布式启动](#双节点分布式启动)
  - [功能测试](#功能测试)
  - [性能基准测试](#性能基准测试)
  - [PD分离部署](#pd分离部署)
    - [环境准备](#环境准备)
    - [服务部署](#服务部署)
  - [分布式部署 (2P1D)](#分布式部署-2p1d)
    - [Prefill节点配置](#prefill节点配置)
    - [Decode节点配置](#decode节点配置)

---

## 环境准备与安装

### 基础环境配置
```bash
# 禁用IPv6
sysctl -w net.ipv6.conf.all.disable_ipv6=1 
sysctl -w net.ipv6.conf.default.disable_ipv6=1

# 设置网络接口和NCCL参数
export GLOO_SOCKET_IFNAME=eth0
#export NCCL_DEBUG=info
export NCCL_IB_DISABLE=1
```

---

## Docker容器部署
```bash
docker run -d -t --network=host --gpus all \
    --privileged \
    --ipc=host \
    --cap-add=SYS_PTRACE \
    --name sglang \
    --ulimit memlock=-1 \
    --ulimit stack=67108864 \
    -v /mnt:/mnt registry.cn-hangzhou.aliyuncs.com/lky-deploy/llm:sglang-0.4.2.post2 bash
```

---

## 服务启动配置

https://github.com/sgl-project/sglang/blob/main/benchmark/deepseek_v3/README.md

### 单节点启动
```bash
python3 -m sglang.launch_server \
    --port 40000 \
    --model-path /mnt/DeepSeek-R1 \
    --trust-remote-code \
    --tp 32 \
    --host 0.0.0.0 \
#可选
--enable-torch-compile 
--torch-compile-max-bs
```

### 双节点分布式启动
**节点1 (Rank 0):**
```bash
python3 -m sglang.launch_server \
    --port 30000 \
    --model-path /mnt/7B \
    --mem-fraction-static 0.8 \
    --trust-remote-code \
    --tp 16 \
    --dp 2 \
    --enable-dp-attention \
    --dist-init-addr 10.0.1.105:20000 \
    --nnodes 2 \
    --node-rank 0
```

**节点2 (Rank 1):**
```bash
# 修改node-rank参数为1，其他配置保持与节点1一致
--node-rank 1
```

---

## 功能测试
```bash
curl http://localhost:30000/generate \
 -H "Content-Type: application/json" \
 -d '{
  "text": "deepseek中有几个e？",
  "sampling_params": {
  "max_new_tokens": 3000,
  "temperature": 0
 }
}'
```

---

## 性能基准测试
```bash
python -m sglang.bench_serving \
--backend sglang --host 127.0.0.1 --port 30000 \
--model /mnt/DeepSeek-V3/DeepSeek-V3/ \
--dataset-name random \
--random-input-len 20000 \
--random-output-len 4096 \
--max-concurrency 8 \
--num-prompts 80 \
--dataset-path /root/ShareGPT_V3_unfiltered_cleaned_split.json &> 20k_8_80.txt &
```

---

## PD分离部署
https://docs.sglang.ai/advanced_features/pd_disaggregation.html#deepseek-multi-node<br>
mooncake需要rdma这种InfiniBand设备支持，nixl不需要
### 环境准备
```bash
# 安装Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
~/miniconda3/bin/conda init bash
source ~/.bashrc

# 创建虚拟环境
conda create -n sglang python=3.10
conda activate sglang
pip install sglang[all]
```

### 服务部署
**Prefill服务:**
```bash
#nixl 依赖
#yum install -y ninja-build openssl3
#nixl==0.5.1 Python 3.10.12 Python 3.10.17
#pip install nixl sglang-router  
python -m sglang.launch_server \
  --model-path $path \
  --disaggregation-mode prefill \
  --port 30000 \
  --disaggregation-transfer-backend nixl
#--disaggregation-bootstrap-port 默认8998对应路由服务端口
```

**Decode服务:**
```bash
python -m sglang.launch_server \
  --model-path $path \
  --disaggregation-mode decode \
  --port 30001 \
  --base-gpu-id 1 \
  --disaggregation-transfer-backend nixl
```

**路由服务:**
https://docs.sglang.ai/advanced_features/router.html#mode-3-prefill-decode-disaggregation
```bash
python -m sglang_router.launch_router \
--pd-disaggregation \
--prefill http://${p}:30000 8998 \
--decode http://127.0.0.1:30001 \
--host 0.0.0.0 \
--port 8000
```

---

## 分布式部署 (2P1D)

### Prefill节点配置
**节点0:**
```bash
export CUDA_VISIBLE_DEVICES=0
python -m sglang.launch_server \
  --model-path $path \
  --disaggregation-transfer-backend nixl \
  --disaggregation-mode prefill \
  --host 0.0.0.0 \
  --port 30000 \
  --dist-init-addr ${prefill_master_ip}:5000 \
  --nnodes 2 \
  --node-rank 0 \
  --tp-size 2
```

**节点1:**
```bash
# 修改node-rank参数为1
--node-rank 1
```

### Decode节点配置
```bash
python -m sglang.launch_server \
  --model-path $path \
  --disaggregation-transfer-backend nixl \
  --disaggregation-mode decode \
  --port 30001 \
  --dist-init-addr ${decode_master_ip}:5000 \
  --nnodes 2 \
  --node-rank 0 \
  --tp-size 16 \
  --dp-size 8 \
  --enable-dp-attention
```

---
