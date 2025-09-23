---
title: vllm
date: 2025-09-23 14:41:13
tags: llm
---

# vLLM 分布式服务部署与压测指南

## 目录
1. [Docker 部署](#docker-部署)
2. [服务启动](#服务启动)
3. [API 测试](#api-测试)
4. [压力测试](#压力测试)
5. [PD分离](#PD分离)

---
https://www.modelscope.cn/models/deepseek-ai/DeepSeek-R1-Distill-Qwen-7B<br>
https://docs.vllm.com.cn/en/latest/getting_started/quickstart.html#installation
## Docker 部署

### 启动 vLLM 服务容器
```bash
docker run -t -d \
    --name="vllm" \
    --ipc=host \
    --cap-add=SYS_PTRACE \
    --network=host \
    --gpus all \
    --privileged \
    --ulimit memlock=-1 \
    --ulimit stack=67108864 \
    -v /mnt:/mnt \
    registry.cn-hangzhou.aliyuncs.com/lky-deploy/llm:vllm-server-0.7.2
```

**关键参数说明**：
- `--gpus all`：启用所有 GPU
- `--network=host`：使用主机网络模式
- `-v /mnt:/mnt`：挂载宿主机存储卷
- `--privileged`：赋予容器特权模式

---

## 服务启动

### 1. 初始化环境
```bash
export GLOO_SOCKET_IFNAME=eth0  # 指定网络接口
export NCCL_SOCKET_IFNAME=eth0  # 指定 NCCL 网络接口

#主节点执行
ray start --head --dashboard-host 0.0.0.0
#从节点执行
ray start --address='${master_ip}:6379'
#查看状态
ray status
```

### 2. 启动 vLLM 服务
```bash
#单节点
vllm serve /mnt/7B \
--tensor-parallel-size 8 \
--served-model-name  vllm \
--trust-remote-code \
--enable-chunked-prefill \
--host 0.0.0.0
#两台节点，每个节点8张卡
vllm serve /mnt/DeepSeek-R1 \
  --tensor-parallel-size 16 \
  --trust-remote-code \
  --enable-chunked-prefill \
  --pipeline-parallel-size 2 \
  --host 0.0.0.0 \
  --max-num-batched-tokens 131072 &> vllm.log &
#--max-num-batched-tokens 131072
#--max-model-len 131072
#--enforce-eager 
#--dtype=half --quantization fp8 --no-enable-prefix-caching 
#--gpu-memory-utilization 0.95 --served-model-name  vllm
```
参数文档：https://docs.vllm.com.cn/en/latest/configuration/engine_args.html#modelconfig<br>
**核心参数**：
| 参数 | 说明 |
|-------|-------|
| `--tensor-parallel-size 16` | 张量并行度 |
| `--pipeline-parallel-size 2` | 流水线并行度 |
| `--max-num-batched-tokens 131072` | 最大批处理 token 数（约 128K） |

---

## API 测试

### 1. 聊天接口测试
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type:application/json" \
  -d '{
    "model":"/mnt/DeepSeek-V3/DeepSeek-V3/",
    "messages":[{"role": "user", "content": "写一首20字的月亮诗"}],
    "stream":false}'
```

### 2. 补全接口测试
```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/mnt/DeepSeek-R1",
    "prompt": "巴黎是法国的首都吗？",
    "max_tokens": 50}'
```

---

## 压力测试

### 1. 启动压测容器
```bash
docker run --name vllm-benchmark -it -v /mnt:/mnt -d \
  registry.cn-hangzhou.aliyuncs.com/lky-deploy/llm:vllm-benchmark-tagv0.7.2 bash
```

### 2. 准备测试数据集
```bash
modelscope download --dataset gliang1001/ShareGPT_V3_unfiltered_cleaned_split \
  ShareGPT_V3_unfiltered_cleaned_split.json --local_dir /root/
```

### 3. 执行压测脚本
```bash
#!/bin/bash

# 定义模型名称（根据实际情况替换）
NAME="/mnt/DeepSeek-R1/"

# 定义并发数和请求数的对应关系数组（格式：concurrency1:num_prompt1 concurrency2:num_prompt2 ...）
PAIRS=(
  "1:10"
  "8:80"
  "16:160"
  "24:240"
  "32:320"
)

# 固定参数（与原始命令保持一致）
RANDOM_INPUT_LEN=1
RANDOM_OUTPUT_LEN=128
RANDOM_RANGE_RATIO=1

# 遍历所有配对
for pair in "${PAIRS[@]}"; do
  # 从配对中提取并发数和请求数
  IFS=":" read -r concurrency num_prompt <<< "$pair"
  
  # 生成输出文件名（格式：inputlen_outputlen_ratio_concurrency_prompts.txt）
  output_file="${RANDOM_INPUT_LEN}_${RANDOM_OUTPUT_LEN}_${RANDOM_RANGE_RATIO}_c${concurrency}_n${num_prompt}.txt"
  
  # 执行基准测试命令
  python3 /root/vllm/benchmarks/benchmark_serving.py \
    --backend vllm \
    --model "$NAME" \
    --served-model-name "$NAME" \
    --trust-remote-code \
    --dataset-name random \
    --dataset-path /root/ShareGPT_V3_unfiltered_cleaned_split.json \
    --random-input-len $RANDOM_INPUT_LEN \
    --random-output-len $RANDOM_OUTPUT_LEN \
    --random-range-ratio $RANDOM_RANGE_RATIO \
    --num-prompts $num_prompt \
    --max-concurrency $concurrency \
    --request-rate inf \
    --host 10.112.201.25 \
    --port 8000 \
    --endpoint /v1/completions \
    &> "$output_file"
  
  echo "已完成测试：concurrency=${concurrency}, num_prompts=${num_prompt}"
  sleep 3
done

echo "所有测试任务已完成！"
```

**压测配置说明**：
- 测试模式：随机输入/输出长度
- 输入长度：1 token
- 输出长度：128 tokens
- 并发梯度：1~32 并发
- 请求量梯度：10~320 请求

---

## PD分离

[官方文档参考](https://docs.vllm.com.cn/en/latest/features/disagg_prefill.html)
### 1. 预填充节点配置（Prefill）
```bash
# 预填充节点1
CUDA_VISIBLE_DEVICES=0 vllm serve /data/7B \
  --port 8100 \
  --max-model-len 100 \
  --gpu-memory-utilization 0.9 \
  --kv-transfer-config '{
    "kv_connector": "PyNcclConnector",
    "kv_role": "kv_producer",         
    "kv_rank": 0,                     
    "kv_parallel_size": 2,            
    "kv_ip": "172.25.79.18",         
    "kv_port": 14579                  
  }'

# 预填充节点2（需修改端口和设备号）
CUDA_VISIBLE_DEVICES=1 vllm serve /data/7B \
  --port 8101 \
  ...
  --kv-transfer-config '{
    ... 
    "kv_rank": 1,
    ...
  }'
```

### 2. 解码节点配置（Decode）
```bash
# 解码节点1
CUDA_VISIBLE_DEVICES=1 vllm serve /data/7B \
  --port 8200 \
  ...
  --kv-transfer-config '{ 
    "kv_role": "kv_consumer",
    "kv_rank": 1,
    ...
  }'

# 解码节点2（需修改端口和设备号）
CUDA_VISIBLE_DEVICES=2 vllm serve /data/7B \
  --port 8201 \
  ...
  --kv-transfer-config '{
    ...
    "kv_rank": 0,
    ...
  }'
```

### 3. 路由服务配置
https://github.com/vllm-project/vllm/blob/main/examples/online_serving/disaggregated_serving/disagg_proxy_demo.py
```bash
python3 examples/online_serving/disaggregated_serving/disagg_proxy_demo.py \
  --model your_model_name \
  --prefill localhost:8100 localhost:8101 \
  --decode localhost:8200 localhost:8201 \
  --port 8000
```

---

## 参数说明

### 公共参数
| 参数 | 说明 |
|-------|-------|
| `--max-model-len` | 最大序列长度（需与内存匹配） |
| `--gpu-memory-utilization` | GPU 显存利用率阈值（0.0~1.0） |
| `--kv-parallel-size` | KV 传输并行度（需与节点数一致） |

### KV 传输参数
```json
{
  "kv_connector": "PyNcclConnector",  // 传输实现方式
  "kv_role": ["producer"/"consumer"], // 角色类型
  "kv_rank": 0,                      // 节点编号
  "kv_ip": "master_ip",              // 主节点IP
  "kv_port": 14579                   // 通讯端口
}
```

---


