---
title: LlamaIndex
date: 2026-01-16 16:20:08
tags: llm
---
# LlamaIndex 使用指南


## 简介
LlamaIndex 是一个强大的开源工具，帮助开发者构建基于大型语言模型 (LLM) 的应用程序。提供工具和 API 连接 LLM 与外部数据源，功能类似 LangChain。

---

## 快速入门

### 环境配置
```bash
# 创建虚拟环境
python -m venv LlamaIndex
source LlamaIndex/bin/activate

# 安装核心库
pip install llama-index
```

### 基础依赖
```bash
pip install \
llama-index-core \
llama-index-llms-openai \
llama-index-embeddings-openai \
llama-index-readers-file
```

### 5行入门代码
```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader("data").load_data()  # 加载文档
index = VectorStoreIndex.from_documents(documents)     # 构建索引
query_engine = index.as_query_engine()                 # 创建查询引擎
response = query_engine.query("你的问题")               # 执行查询
print(response)
```

> **注意**：默认使用 OpenAI 模型，需配置 `OPENAI_API_KEY` 环境变量

---

## 模型配置

### 1. LLM 配置
#### 本地部署
```python
import os
from llama_index.core import Settings
from llama_index.llms.openai_like import OpenAILike
 
# LlamaIndex默认使用的大模型被替换
Settings.llm = OpenAILike(
    model="DeepSeek-R1-Distill-Qwen-1.5B",
    api_base="http://localhost:8000/v1",#如vllm，sglang，ollama
    is_chat_model=True
)
response = llm.complete("Hello World!")
print(response)
)
```

#### 云平台（示例：阿里百炼）
```python
import os
from llama_index.core import Settings
from llama_index.llms.openai_like import OpenAILike
 
# LlamaIndex默认使用的大模型被替换为百炼
Settings.llm = OpenAILike(
    model="qwen-plus",
    api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    is_chat_model=True
)
response = llm.complete("Hello World!")
print(response)
)
```

### 2. Embedding 配置
#### 本地部署
```python
from llama_index.embeddings.textembed import TextEmbedEmbedding 
# 初始化 TextEmbedEmbedding 类 
embed = TextEmbedEmbedding( 
    model_name="Qwen3-Embedding-8B", 
    base_url="http://0.0.0.0:8000/v1", 
    auth_token="TextEmbed", ) 
# 获取一批文本的 embeddings 
embeddings = embed.get_text_embedding_batch( [ "这里下着倾盆大雨！", "印度拥有多元化的文化遗产。", ] ) 
print(embeddings)
```

#### 云平台（示例：DashScope）
```python
from llama_index.embeddings.dashscope import DashScopeEmbedding
 
# 初始化 Embedding 模型
embedder = DashScopeEmbedding(
    model_name="text-embedding-v2"
)
text_to_embedding = ["风急天高猿啸哀", "渚清沙白鸟飞回", "无边落木萧萧下", "不尽长江滚滚来"]
# 调用 Embedding 模型
result_embeddings = embedder.get_text_embedding_batch(text_to_embedding)
# 显示 Embedding 后结果
for index, embedding in enumerate(result_embeddings):
    print("Dimension of embeddings: %s" % len(embedding))
    print(
        "Input: %s, embedding is: %s"
        % (text_to_embedding[index], embedding[:5])
    )
```

---

## RAG 检索问答

### 完整示例
```python
import os
import dashscope
from llama_index.core import SimpleDirectoryReader, Settings
from llama_index.core import VectorStoreIndex
from llama_index.core import VectorStoreIndex, get_response_synthesizer
from llama_index.core.node_parser import SentenceSplitter, TokenTextSplitter, SemanticSplitterNodeParser
from llama_index.core.retrievers import VectorIndexRetriever
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.postprocessor import SimilarityPostprocessor
from llama_index.embeddings.dashscope import DashScopeEmbedding
from llama_index.llms.openai_like import OpenAILike
os.environ['DASHSCOPE_API_KEY']=''
 
dashscope.api_key=api_key=os.getenv("DASHSCOPE_API_KEY")
# LlamaIndex默认使用的Embedding模型被替换为百炼的Embedding模型
Settings.embed_model = DashScopeEmbedding(
    model_name="text-embedding-v2"
)
# 替换掉默认的openai的llm
Settings.llm = OpenAILike(
    model="qwen-plus",
    api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    is_chat_model=True
)
#使用最常用的加载器加载文档
#https://docs.llamaindex.org.cn/en/stable/module_guides/loading/simpledirectoryreader/
documents = SimpleDirectoryReader("./data",
                                  exclude=['text.txt'], #过滤某些文件
                                  exclude_hidden=True, #过滤隐藏文件
                                  recursive=True, #递归
                                  required_exts=['.jsonl'] #只加载
                                  ).load_data()
print(f"加载了 {len(documents)} 个文档")
#不同的切片规则
# token_splitter = TokenTextSplitter(
#     chunk_size=1024,
#     chunk_overlap=20
# )
# semantic_splitter = SemanticSplitterNodeParser(
#     buffer_size=1,
#     breakpoint_percentile_threshold=95,
#     embed_model=Settings.embed_model
# )
#chunk_size‌：定义单块文本的最大字符/Token 数量。例如 chunk_size=1024 表示每块最多包含 1024 个字符或 Token‌。
#chunk_overlap‌：指定相邻文本块之间的重叠字符数，用于保持语义连贯性（如 chunk_overlap=200）‌。
#切片规则
splitter = SentenceSplitter(chunk_size=1024, chunk_overlap=200)
#嵌入建立索引
index = VectorStoreIndex.from_documents(documents,transformations=[splitter],show_progress=True)
#查询
query_engine = index.as_query_engine(streaming=True, similarity_top_k=5) # 一次检索出 5 个文档切片，默认为 2
response = query_engine.query(
    "基本竞争战略是什么"
)
print(response)
 
#data文件夹的数据内容
#{"text":"基本竞争战略是由美国哈佛商学院著名的战略管理学家迈克尔·波特提出的,分别为:成本领先战略,差异化战略,集中化战略.企业必须从这三种战略中选择一种,作为其主导战略.要么把成本控制到比竞争者更低的程度;要么在企业产品和服务中形成与众不同的特色,让顾客感觉到你提供了比其他竞争者更多的价值;要么企业致力于服务于某一特定的市场细分,某一特定的产品种类或某一特定的地理范围."}
#{"text":"交通运行监测调度中心,简称TOCC(Transportation Operations Coordination Center)TOCC围绕综合交通运输协调体系的构建,实施交通运行的监测,预测和预警,面向公众提供交通信息服务,开展多种运输方式的调度协调,提供交通行政管理和应急处置的信息保障.\nTOCC是综合交通运行监测协调体系的核心组成部分,实现了涵盖城市道路,高速公路,国省干线三大路网,轨道交通,地面公交,出租汽车三大市内交通方式,公路客运,铁路客运,民航客运三大城际交通方式的综合运行监测和协调联动,在综合交通的政府决策,行业监管,企业运营,百姓出行方面发挥了突出的作用."}
#{"text":"美国职业摄影师协会(简称PPA)创立于1880年,是一个几乎与摄影术诞生历史一样悠久的享誉世界的非赢利性国际摄影组织,是由世界上54个国家的25000余名职业摄影师个人会员和近二百个附属组织和分支机构共同组成的,是世界上最大的专业摄影师协会.本世纪初PPA创立了美国视觉艺术家联盟及其所隶属的美国国际商业摄影师协会,美国新闻及体育摄影师协会,美国学生摄影联合会等组织.PPA在艺术,商业,纪实,体育等摄影领域一直引领世界潮流,走在世界摄影艺术与技术应用及商业规划管理的最前沿."}
```

---

## 文档切片规则

| 切片类型       | 特点                          | 适用场景                     | 示例配置                     |
|----------------|-------------------------------|------------------------------|------------------------------|
| **Token切片**  | 按Token数量切分               | 小上下文模型                 | `TokenTextSplitter(chunk_size=1024)` |
| **句子切片**   | 保持句子完整性（默认）        | 通用场景                     | `SentenceSplitter(chunk_size=512)` |
| **句子窗口**   | 包含上下文窗口                | 需要上下文关联的任务         | `SentenceWindowNodeParser(window_size=3)` |
| **语义切片**   | 按语义相关性切分              | 复杂语义分析                 | `SemanticSplitterNodeParser()` |

---

## 最佳实践建议
1. 根据模型上下文长度选择切片策略
2. 检索时设置 `similarity_top_k=3-5` 平衡精度与效率
3. 使用语义切片时配合后处理器提升效果
4. 生产环境建议使用云平台部署保证稳定性

> 更多细节参考 [LlamaIndex 官方文档](https://docs.llamaindex.org.cn)
