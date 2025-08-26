---
title: python ThreadPoolExecutor
date: 2025-08-26 17:08:36
tags:
categories: python
---
# 并发执行方法对比（submit/as_completed/map）

## 核心特性对比表

| 特征                | `executor.map()`           | `submit`+顺序处理          | `as_completed`             |
|---------------------|---------------------------|---------------------------|---------------------------|
| **执行顺序**         | 并发执行                  | 并发执行                  | 并发执行                  |
| **结果顺序**         | 保持输入顺序              | 保持提交顺序              | 按完成顺序                |
| **异常处理**         | 遇到第一个异常立即抛出     | 可逐个处理异常            | 可单独处理每个任务异常     |
| **代码复杂度**       | ★☆☆ 最简单               | ★★☆ 中等                 | ★★★ 最灵活               |
| **内存消耗**         | 低（惰性迭代）            | 高（需存储所有future）    | 中（动态处理）            |
| **进度反馈**         | 无法实时获取              | 需手动实现                | 自动实时反馈              |
| **适用场景**         | 简单转换/批量处理         | 需要严格顺序的结果        | 需要及时处理完成的场景     |

---

## 方法详解

### 1. `executor.map()`
**典型用法**：
```python
from concurrent.futures import ThreadPoolExecutor

def process_data(x):
    return x * 2

with ThreadPoolExecutor() as executor:
    results = executor.map(process_data, [1, 2, 3])  # 保持输入顺序
    for res in results:
        print(res)  # 按1→2→3的顺序输出2,4,6


#示例下载有顺序的小说
def download_chapter(index, url):
    # 模拟下载（并行，执行顺序不确定）
    return (index, f"第 {index} 章内容")


# 通过bs4获取列表，输入是有序的章节列表
chapters = [
    (1, "http://example.com/ch1"),
    (2, "http://example.com/ch2"),
    (3, "http://example.com/ch3")
]

with ThreadPoolExecutor(max_workers=3) as executor:
    # 提交任务（按顺序）
    results = executor.map(
        lambda args: download_chapter(*args),
        chapters
    )

    # 拆解元组参数为独立的参数
    lambda args: download_chapter(*args)
    # 等效于：
    # def unpack_and_call(args):
    #     return download_chapter(args[0], args[1])  # 解包元组


    # 按输入顺序写入文件
    with open("book.txt", "w", encoding="utf-8") as f:
        for index, content in results:
            f.write(f"{content}\n")
```

**特点**：
- 自动将可迭代参数映射到函数
- 结果顺序与输入参数严格一致
- 遇到第一个异常时会立即停止迭代

**最佳场景**：
- 数据并行转换（如批量图片压缩）
- 需要保持输入输出顺序对应的任务

---

### 2. `submit` + 顺序处理
**典型用法**：参数必须是可迭代对象，如 i for i in range(1, 4)；submit不需要因为是手动提交
```python
def multiply(x, y):
    return x * y

with ThreadPoolExecutor() as executor:
    # 生成器作为可迭代对象
    nums1 = (i for i in range(1, 4))
    nums2 = (i for i in range(10, 13))
    results = executor.map(multiply, nums1, nums2)
    print(list(results))  # 输出 [10, 22, 36]

# 提交单个任务，参数直接传递，可以不用是列表
future = executor.submit(multiply, 2, 3)
print(future.result())  # 输出 8



#示例代码，不同任务
 定义三类不同参数的任务函数
def download(url, timeout):
    """模拟下载任务（参数：URL + 超时时间）"""
    print(f"开始下载: {url}, 超时设置: {timeout}秒")
    time.sleep(random.uniform(0.5, 2))
    if random.random() < 0.2:
        raise URLError(f"无法访问 {url}")
    return f"下载完成: {url}"


def calculate(a, b, operator):
    """模拟计算任务（参数：两个数 + 运算符）"""
    print(f"计算: {a} {operator} {b}")
    time.sleep(0.5)
    if operator == "+":
        return a + b
    elif operator == "*":
        return a * b
    else:
        raise ValueError(f"不支持的运算符: {operator}")


def log_message(message, priority="INFO"):
    """模拟日志任务（参数：消息 + 优先级）"""
    print(f"[{priority}] 记录日志: {message}")
    time.sleep(0.1)
    return f"日志已保存: {message}"


if __name__ == "__main__":
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = []

        # 动态提交不同类型的任务
        # 任务1：下载任务（参数：url, timeout）
        futures.append(executor.submit(download, "https://example.com", timeout=3))

        # 任务2：计算任务（参数：5, 3, "+"）
        futures.append(executor.submit(calculate, 5, 3, "+"))

        # 任务3：日志任务（参数：message="系统启动", priority="HIGH"）
        futures.append(executor.submit(log_message, "系统启动", "HIGH"))

        # 动态追加任务（根据条件）
        if random.choice([True, False]):
            # 任务4：随机添加一个计算任务（参数：8, 4, "*"）
            futures.append(executor.submit(calculate, 8, 4, "*"))

        # 处理结果（按完成顺序，独立捕获异常）
        for future in futures:
            try:
                result = future.result()
                print(f"结果: {result}")
            except URLError as e:
                print(f"下载失败: {e.reason}")
            except ValueError as e:
                print(f"计算错误: {e}")
            except Exception as e:
                print(f"未知错误: {e}")
```

**特点**：
- 手动控制每个任务的提交
- 结果处理顺序固定
- 可以单独处理每个任务的异常

**最佳场景**：
- 需要跟踪任务来源的场景
- 需要逐步处理结果的日志系统

---

### 3. `as_completed`
**典型用法**：
```python
from concurrent.futures import as_completed

futures = [executor.submit(task, param) for param in params]

for future in as_completed(futures):
    res = future.result()
    print(f"收到结果: {res}")  # 按完成顺序输出
```

**特点**：
- 实时获取已完成任务结果
- 可以优先处理耗时短的任务
- 需要额外维护future列表

**最佳场景**：
- 文件下载（小文件优先完成）
- 实时仪表盘更新
- 需要快速获取部分结果的场景

---

## 性能特征对比

### 模拟耗时任务（3个任务分别需要3s/1s/2s）

```python
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

def task(sec):
    time.sleep(sec)
    return f"{sec}秒任务"

params = [3, 1, 2]
```

| 方法                | 输出顺序          | 总耗时 | 首个结果时间 |
|--------------------|------------------|-------|------------|
| `map`             | 3→1→2            | 3s    | 3s后        |
| `submit`顺序处理   | 3→1→2            | 3s    | 3s后        |
| `as_completed`     | 1→2→3            | 3s    | 1s后        |

---


## 总结对比

| 维度               | map                         | submit+顺序               | as_completed             |
|--------------------|----------------------------|--------------------------|--------------------------|
| **代码简洁度**     | 最简洁（自动管理）          | 中等（需手动维护）        | 最复杂（需动态处理）      |
| **结果顺序**       | 输入顺序                    | 提交顺序                  | 完成顺序                  |
| **实时反馈**       | 无                         | 无                       | 有                        |
| **异常容忍度**     | 低（遇到错误立即停止）      | 高（可单独处理）          | 高（可单独处理）          |
| **内存效率**       | 高（惰性迭代）              | 低（全存储）              | 中（动态释放）            |

