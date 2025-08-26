---
title: python as_completed
date: 2025-08-26 17:12:29
tags:
categories: python
---
# `as_completed` 方法详解

## 🎯 核心区别：处理顺序
| 方法                   | 执行顺序       | 结果获取顺序     | 适用场景                     |
|------------------------|----------------|------------------|----------------------------|
| `submit` + 顺序处理    | 按提交顺序执行 | 按提交顺序获取   | 需要严格保持结果顺序的场景    |
| `as_completed`         | 并行执行       | 按完成顺序获取   | 需要及时处理已完成任务的场景  |

---

## 📌 使用场景分析

### 1. `submit` + 顺序处理
**典型代码**：
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor() as executor:
    futures = [executor.submit(task, param) for param in params_list]
    for future in futures:  # 按提交顺序处理
        print(future.result())
```

**特点**：
- 必须等待前一个任务完成才能处理下一个
- 严格保持任务提交顺序
- 适合场景：
  - 日志处理需要按时间顺序记录
  - 结果需要顺序写入文件/数据库

---

### 2. `as_completed`
**典型代码**：
```python
from concurrent.futures import as_completed

with ThreadPoolExecutor() as executor:
    futures = [executor.submit(task, param) for param in params_list]
    for future in as_completed(futures):  # 按完成顺序处理
        print(future.result())
```

**特点**：
- 优先处理最快完成的任务
- 不保证结果顺序
- 适合场景：
  - 需要实时显示进度条
  - 处理时间差异大的任务（如不同大小的文件下载）
  - 快速获取部分可用结果（如爬虫先抓取先分析）

---

## 🧪 性能对比实验
### 模拟耗时任务
```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def task(n):
    time.sleep(n)  # 模拟耗时操作
    return f"任务 {n}s 完成"

if __name__ == "__main__":
    times = [3, 1, 2]  # 三个任务分别需要3秒、1秒、2秒
    
    with ThreadPoolExecutor() as executor:
        futures = [executor.submit(task, t) for t in times]
        
        print("--- 使用 as_completed ---")
        start = time.time()
        for f in as_completed(futures):
            print(f"{time.time()-start:.1f}s 收到: {f.result()}")
        
        print("\n--- 按提交顺序处理 ---")
        start = time.time()
        for f in futures:
            print(f"{time.time()-start:.1f}s 收到: {f.result()}")
```

### 实验结果
```text
--- 使用 as_completed ---
1.0s 收到: 任务 1s 完成
2.0s 收到: 任务 2s 完成
3.0s 收到: 任务 3s 完成

--- 按提交顺序处理 ---
3.0s 收到: 任务 3s 完成
3.0s 收到: 任务 1s 完成
3.0s 收到: 任务 2s 完成
```

---

## 💡 最佳实践建议

### 1. 优先使用 `as_completed` 当：
✅ 需要实时显示进度状态  
✅ 任务执行时间差异较大（如同时处理图片缩略图和4K视频）  
✅ 不需要保持结果顺序（如独立的数据抓取任务）

### 2. 使用顺序处理当：
⛔ 必须保持结果顺序（如时间序列数据分析）  
⛔ 后续任务依赖前序结果（如分步骤数据处理流水线）  
⛔ 需要严格控制资源使用（如顺序写入数据库）

### 3. 混合使用技巧
```python
# 同时获取结果和原始任务索引
for future in as_completed(futures):
    original_index = futures.index(future)  # 获取提交时的顺序索引
    result = future.result()
    print(f"第 {original_index} 个提交的任务完成：{result}")
```

---

## 📚 扩展知识
### `concurrent.futures` 模块对比
| 方法             | 特点                         | 适用场景              |
|------------------|-----------------------------|---------------------|
| ThreadPoolExecutor | 使用线程池，适合IO密集型任务  | 网络请求/文件操作等   |
| ProcessPoolExecutor | 使用进程池，适合CPU密集型任务 | 数学计算/图像处理等   |
```python
# 进程池用法（接口与线程池一致）
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as executor:
    ...
