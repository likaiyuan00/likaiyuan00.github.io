---
title: python threading
date: 2025-08-26 17:04:04
tags:
categories: python
---
# threading.Thread
* 适用场景：任务逻辑差异大；每个线程需要执行完全不同的逻辑
* 三个比较重要的参数
1. 锁：threading.Lock()  
两种使用方法推荐with下方演示，第二种手动加锁lock.acquire()，然后释放lock.release()
```python
#未使用的情况下
def sing():
    #with stats_lock:
        for i in range(3):
            print("正在唱歌...%d"%i)
            sleep(1)

def dance():
    #with stats_lock:
        for i in range(3):
            print("正在跳舞...%d"%i)
            sleep(1)

if __name__ == '__main__':
    print('---开始---:%s'%ctime())

    t1 = threading.Thread(target=sing)
    t2 = threading.Thread(target=dance)

    t1.start()
    t2.start()

#输出是混乱的就是因为没有给线程加锁
---开始---:Tue Aug 26 10:19:56 2025
正在唱歌...0
正在跳舞...0
正在唱歌...1正在跳舞...1

正在跳舞...2
正在唱歌...2
======================================================================
#加锁后
import threading
from time import sleep,ctime

stats_lock = threading.Lock()
def sing():
    with stats_lock:
        #stats_lock.acquire()
        for i in range(3):
            print("正在唱歌...%d"%i)
            sleep(1)
        #stats_lock.release()

def dance():
    with stats_lock:
        for i in range(3):
            print("正在跳舞...%d"%i)
            sleep(1)

if __name__ == '__main__':
    print('---开始---:%s'%ctime())

    t1 = threading.Thread(target=sing)
    t2 = threading.Thread(target=dance)

    t1.start()
    t2.start()
#输出正常
---开始---:Tue Aug 26 10:23:18 2025
正在唱歌...0
正在唱歌...1
正在唱歌...2
正在跳舞...0
正在跳舞...1
正在跳舞...2
```



2. 线程中的join等待阻塞函数
* 当有线程在统计信息时，必须等待执行完成才可以
```python
import threading
from time import sleep,ctime

stats_lock = threading.Lock()


def sing(num):
    with stats_lock:
        for i in range(num):
            print("正在唱歌...%d"%i)
            sleep(1)

def dance(num):
    with stats_lock:
        for i in range(num):
            print("正在跳舞...%d"%i)
            sleep(1)

if __name__ == '__main__':
    print('---开始---:%s'%ctime())

    t1 = threading.Thread(target=sing,args=(3,),name="唱歌线程")
    t2 = threading.Thread(target=dance,args=(3,))

    t1.start()
    t2.start()

    # t1.join()
    # t2.join()
    #sleep(5) # 屏蔽此行代码，试试看，程序是否会立马结束？
    print('---结束---:%s'%ctime())

#未使用join等待函数输出结果,主线程直接结束，不符合预期
---开始---:Tue Aug 26 10:35:09 2025
正在唱歌...0
---结束---:Tue Aug 26 10:35:09 2025
正在唱歌...1
正在唱歌...2
正在跳舞...0
正在跳舞...1
正在跳舞...2
#使用join后
---开始---:Tue Aug 26 10:36:35 2025
正在唱歌...0
正在唱歌...1
正在唱歌...2
正在跳舞...0
正在跳舞...1
正在跳舞...2
---结束---:Tue Aug 26 10:36:41 2025
=================================================================================
#注意锁的颗粒度，如果锁循环则是线程二要等线程一执行完才会运行影响效率
import threading
from time import sleep,ctime

stats_lock = threading.Lock()


def sing(num):
    #with stats_lock:
        for i in range(num):
            with stats_lock:
                print("正在唱歌...%d"%i)
            sleep(1)

def dance(num):
    #with stats_lock:
        for i in range(num):
            with stats_lock:
                print("正在跳舞...%d"%i)
            sleep(1)

if __name__ == '__main__':
    print('---开始---:%s'%ctime())

    t1 = threading.Thread(target=sing,args=(3,),name="唱歌线程")
    t2 = threading.Thread(target=dance,args=(3,))

    t1.start()
    t2.start()

    t1.join()
    t2.join()
    #sleep(5) # 屏蔽此行代码，试试看，程序是否会立马结束？
    print('---结束---:%s'%ctime())
#输出结果，同步执行，只锁输出
---开始---:Tue Aug 26 10:39:38 2025
正在唱歌...0
正在跳舞...0
正在跳舞...1
正在唱歌...1
正在跳舞...2
正在唱歌...2
---结束---:Tue Aug 26 10:39:41 2025
```

3. 队列函数 queue.Queue 来实现线程间的通信和数据交换
* 生产者（如网络请求接收）和消费者（如任务处理线程）速度不一致建议使用
* 比如爬取m3u8视频片段，就需要使用队列按照先进先出的顺序下载避免多线程片段混乱无法组合

|  作用   | 说明  |
|  ----  | ----  |
| 线程安全的数据传递  | 自动处理多线程并发操作，无需手动加锁/释放锁。 |
| 任务解耦  | 生产任务的线程（如主线程）和消费任务的线程（工作线程）完全解耦。 |
|   流量控制   |   通过队列大小限制（maxsize）防止内存爆炸。       |
|   任务状态跟踪   |  支持 task_done() 和 join() 机制，方便等待所有任务完成。   |

```python
#未使用
tasks = []  # 全局任务列表
lock = threading.Lock()

# 生产者线程
def producer():
    global tasks
    for i in range(100):
        with lock:
            tasks.append(i)

# 消费者线程
def consumer():
    while True:
        with lock:
            if not tasks:
                break
            item = tasks.pop(0)
        process(item)

#使用
import threading
from queue import Queue

task_queue = Queue(maxsize=10)  # 队列容量限制

def producer():
    for i in range(100):
        task_queue.put(i)  # 自动阻塞队列满时

def consumer():
    while True:
        item = task_queue.get()  # 自动阻塞队列空时
        process(item)
        task_queue.task_done()

# 启动线程
producer_thread = threading.Thread(target=producer)
consumer_thread = threading.Thread(target=consumer)

producer_thread.start()
consumer_thread.start()

task_queue.join()  # 等待所有任务完成
producer_thread.join()
consumer_thread.join()


#顺序先入先出
import threading
from queue import Queue
import time
import random

# 共享队列和结果容器
task_queue = Queue()
results = {}
lock = threading.Lock()  # 保证结果字典的线程安全

def worker():
    """工作线程：处理无序任务，保存结果到字典"""
    while True:
        # 获取任务（包含序号和参数）
        index, url = task_queue.get()
        print(f"开始处理第 {index} 章: {url}")
        time.sleep(random.uniform(0.5, 2))  # 模拟耗时操作
        content = f"第 {index} 章内容"
        
        # 保存结果（加锁保证线程安全）
        with lock:
            results[index] = content
        
        task_queue.task_done()

def download_ordered_chapters(chapters):
    """主线程：提交任务、启动工作线程、等待并排序结果"""
    # 提交任务到队列
    for index, url in chapters:
        task_queue.put((index, url))
    
    # 启动工作线程（3个线程）
    threads = []
    for _ in range(3):
        t = threading.Thread(target=worker, daemon=True)
        t.start()
        threads.append(t)
    
    # 等待所有任务完成
    task_queue.join()
     # 等待所有线程结束（非守护线程必须调用 join()）
    # for t in threads:
    #     t.join()  # 确保线程完全结束
    
    # 按序号排序结果
    sorted_indices = sorted(results.keys())
    ordered_results = [results[i] for i in sorted_indices]
    
    return ordered_results

# 示例调用
if __name__ == "__main__":
    chapters = [
        (1, "http://example.com/ch1"),
        (2, "http://example.com/ch2"),
        (3, "http://example.com/ch3")
    ]
    
    ordered_contents = download_ordered_chapters(chapters)
    for content in ordered_contents:
        print(f"保存: {content}")

```




* 综合示例代码
```python
import queue
import threading
import time
import requests
from queue import Queue

# 压测配置
CONFIG = {
    "target_url": "http://localhost/",  # 目标地址
    "thread_num": 5000,  # 并发线程数
    "total_requests": 100000,  # 总请求量 (设置为0表示无限持续)
    "timeout": 5,  # 单请求超时时间（秒）
    "headers": {  # 请求头（按需修改）
        "User-Agent": "Stress Test/1.0"
    }
}

# 全局统计
stats = {
    'total': 0,
    'success': 0,
    'fail': 0,
    'total_time': 0.0,
    'max_time': 0.0,
    'min_time': float('inf')
}
stats_lock = threading.Lock()  # 线程安全锁

# 请求任务队列
task_queue = Queue()


def worker():
    """工作线程函数"""
    while True:
        try:
            # 从队列获取任务（阻塞模式）
            task_id = task_queue.get(timeout=2)

            start_time = time.time()
            try:
                # 发送请求（可修改为POST等其他方法）
                response = requests.get(
                    CONFIG["target_url"],
                    headers=CONFIG["headers"],
                    timeout=CONFIG["timeout"]
                )
                elapsed = time.time() - start_time

                # 更新统计（200~399状态码视为成功）
                with stats_lock:
                    stats['total'] += 1
                    if 200 <= response.status_code < 400:
                        stats['success'] += 1
                    else:
                        stats['fail'] += 1

                    stats['total_time'] += elapsed
                    stats['max_time'] = max(stats['max_time'], elapsed)
                    stats['min_time'] = min(stats['min_time'], elapsed)

            except Exception as e:
                with stats_lock:
                    stats['fail'] += 1
                    stats['total'] += 1

            # 标记任务完成
            task_queue.task_done()

        except queue.Empty:
            break


def print_stats():
    """实时打印统计信息"""
    start_time = time.time()
    while True:
        time.sleep(1)  # 每秒更新
        with stats_lock:
            if stats['total'] == 0:
                continue

            duration = time.time() - start_time
            qps = stats['total'] / duration
            avg_time = stats['total_time'] / stats['total']

            print(f"\r[STAT] "
                  f"Requests: {stats['total']} | "
                  f"Success: {stats['success']} | "
                  f"Fail: {stats['fail']} | "
                  f"QPS: {qps:.1f} | "
                  f"Avg: {avg_time:.3f}s | "
                  f"Min/Max: {stats['min_time']:.3f}s/{stats['max_time']:.3f}s",
                  end='', flush=True)

        # 检查是否完成所有任务
        if CONFIG["total_requests"] > 0 and stats['total'] >= CONFIG["total_requests"]:
            break


if __name__ == "__main__":
    # 初始化任务队列
    if CONFIG["total_requests"] > 0:
        for i in range(CONFIG["total_requests"]):
            task_queue.put(i)
    else:  # 持续模式填充队列
        while True:
            task_queue.put(1)

    # 创建工作者线程
    threads = []
    for _ in range(CONFIG["thread_num"]):
        t = threading.Thread(target=worker)
        t.daemon = True
        t.start()
        threads.append(t)

    # 启动统计线程
    stat_thread = threading.Thread(target=print_stats)
    stat_thread.start()

    # 等待任务完成
    task_queue.join()
    stat_thread.join()

    print("\n压力测试完成")

#使用了join等待压测结束统计信息
#使用了queue.put填充队列为请求数量，线程为并发数量
#使用了with stats_lock加锁避免线程冲突


#推荐使用这种方式，因为是大量重复任务
#使用ThreadPoolExecutor线程池优化，因为是大量重复任务，thread每次都要新创建线程消耗较大
import threading
import time
import requests
from concurrent.futures import ThreadPoolExecutor

CONFIG = {
    "target_url": "http://localhost/",
    "thread_num": 300,
    "total_requests": 10000,
    "timeout": 5,
    "headers": {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/139.0.0.0 Safari/537.36"
    },
    "show_progress": True
}

stats = {
    'total': 0, 'success': 0, 'fail': 0,
    'total_time': 0.0, 'max_time': 0.0, 'min_time': float('inf')
}
stats_lock = threading.Lock()

def worker(_):
    start_time = time.time()
    try:
        response = requests.get(
            CONFIG["target_url"],
            headers=CONFIG["headers"],
            timeout=CONFIG["timeout"]
        )
        elapsed = time.time() - start_time
        with stats_lock:
            stats['total'] += 1
            if 200 <= response.status_code < 400:
                stats['success'] += 1
            else:
                stats['fail'] += 1
            stats['total_time'] += elapsed
            stats['max_time'] = max(stats['max_time'], elapsed)
            stats['min_time'] = min(stats['min_time'], elapsed)
    except Exception as e:
        with stats_lock:
            stats['fail'] += 1
            stats['total'] += 1

def print_stats():
    start_time = time.time()
    while True:
        time.sleep(1)
        with stats_lock:
            current_total = stats['total']
            duration = time.time() - start_time
            qps = current_total / duration if duration > 0 else 0
            avg_time = stats['total_time'] / current_total if current_total > 0 else 0

            progress_percent = (current_total / CONFIG["total_requests"]) * 100 if CONFIG["total_requests"] > 0 else 0
            progress_bar = ""
            if CONFIG["show_progress"] and CONFIG["total_requests"] > 0:
                bar_length = 20
                filled = int(bar_length * current_total // CONFIG["total_requests"])
                progress_bar = "[" + "=" * filled + " " * (bar_length - filled) + "] "

            output = (
                f"\r[STAT] "
                f"进度: {progress_bar}{progress_percent:.1f}% | "
                f"请求: {current_total}/{CONFIG['total_requests'] if CONFIG['total_requests'] > 0 else '∞'} | "
                f"成功: {stats['success']} | "
                f"失败: {stats['fail']} | "
                f"QPS: {qps:.1f} | "
                f"平均: {avg_time:.3f}s | "
                f"最慢: {stats['max_time']:.3f}s"
            )
            print(output, end='', flush=True)

            if CONFIG["total_requests"] > 0 and current_total >= CONFIG["total_requests"]:
                break

if __name__ == "__main__":
    stat_thread = threading.Thread(target=print_stats)
    stat_thread.start()

    with ThreadPoolExecutor(max_workers=CONFIG["thread_num"]) as executor:
        if CONFIG["total_requests"] > 0:
            executor.map(worker, range(CONFIG["total_requests"]))
        else:
            while True:
                executor.submit(worker, None)

    stat_thread.join()
    print("\n压力测试完成")


```

# 额外
* ThreadPoolExecutor 适用场景：短期、批量、同质化任务

* multiprocessing 适用场景：大量cpu密集型计算，多进程绕过GIL；io密集型建议使用异步编程























