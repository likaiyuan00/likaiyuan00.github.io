---
title: Embedding-Reranker
date: 2026-01-06 17:31:47
tags: llm
---
# Embedding 和 Reranker 模型指南

---

## 1. 理论

### 1.1 Embedding 模型：文字的「数字身份证」
**作用**：将文字转换为高维向量，建立语义空间中的坐标定位  
**使用场景**：
- 🔍 搜索（"猫" → 匹配"猫咪""橘猫"）
- 🎥 推荐系统（科幻片→科幻片）
- 📊 聚类分析（自动分类用户评论）  
**比喻**：图书馆管理员快速搬来所有「狗」相关书籍，但顺序杂乱

---

### 1.2 Reranker 模型：结果的「智能排序员」
**作用**：对初步结果重新排序，提升精准度  
**使用场景**：
- ❓ 问答系统（从100条答案选最优解）
- 🔍 搜索引擎（"苹果"优先显示水果）
- 🎬 推荐系统（按评分/热度排序）  
**比喻**：管理员二次整理书籍，按评分/出版时间排序

---

### 1.3 两者协作关系
|          | Embedding              | Reranker               |
|----------|------------------------|------------------------|
| **阶段** | 粗筛（召回）           | 精排（排序）           |
| **速度** | 快                     | 较慢                   |
| **目标** | 解决"有没有"           | 解决"哪个更好"         |

**经典组合案例**：
1. 电商搜索：Embedding找"运动鞋" → Reranker按价格/销量排序
2. 智能客服：Embedding匹配问题 → Reranker选最准回答

---

## 2. 部署实践

### 2.1 Ollama
- 官方模型库：[Embedding Models](https://ollama.com/search?c=embedding)
- 启动命令：
  ```bash
  ollama run <model-name>
  ```

---

### 2.2 vLLM
#### Embedding 部署
```bash
vllm serve /mnt/Qwen3-Embedding-8B/ \
  --tensor-parallel-size 2 \
  --trust-remote-code \
  --host 0.0.0.0
```

#### Reranker 部署
```bash
vllm serve /mnt/Qwen3-Reranker-0.6B/ \
  --hf_overrides '{"architectures": ["Qwen3ForSequenceClassification"]}'
```

#### Benchmark 测试
```bash
#https://docs.vllm.ai/en/latest/benchmarking/cli/?h=reranker#text-embeddings
vllm bench serve \
  --model /mnt/Qwen3-Embedding-8B/ \
  --dataset-name sharegpt \
  --num_prompts 1000 \
  --port 8000
```
> 文档参考：[vLLM Models](https://docs.vllm.ai/en/latest/models/supported_models/?h=reranker)

---

### 2.3 SGLang
#### Embedding 部署
```bash
python3 -m sglang.launch_server \
  --model-path Qwen/Qwen3-Embedding-4B \
  --is-embedding \
  --port 30000
```
> 支持模型：[Embedding Models](https://docs.sglang.io/supported_models/embedding_models.html)

#### Reranker 部署
```bash
python3 -m sglang.launch_server \
  --model-path BAAI/bge-reranker-v2-m3 \
  --disable-radix-cache \
  --attention-backend triton \
  --port 30000
```
> 支持模型：[Reranker Models](https://docs.sglang.io/supported_models/rerank_models.html)

---

## 关键要点
1. **RAG 流程**：Embedding 提升召回 → Reranker 优化排序
2. **硬件建议**：
   - Embedding：需要更大显存（8B模型约需24G）
   - Reranker：计算密集型，建议使用GPU加速
3. **版本注意**：
   - vLLM 需 ≥0.11.0
   - SGLang 需配置 Triton 后端
