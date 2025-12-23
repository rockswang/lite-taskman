这是一份为您精心编写的 `README_zh.md`。我采用了专业且易读的风格，重点突出了 `lite-taskman` 的核心优势：**轻量**、**增量迭代**以及**零依赖**。

---

# lite-taskman 🚀

`lite-taskman` 是一个极其轻量（仅约 100 行代码）且功能强大的 Python 线程池管理工具。它专为需要**动态增加任务**、**实时进度反馈**以及**流式处理结果**的场景而设计。

与原生 `ThreadPoolExecutor` 不同，它允许你在消费结果的同时不断往任务池里塞入新任务，非常适合爬虫、递归扫描等场景。

## ✨ 核心特性

* **动态增量执行**：支持在处理任务的过程中随时添加新任务，直到所有任务流完成。
* **极简 API**：提供 `exec()` 一键式执行和 `process()` 流式生成器两种模式。
* **进度追踪**：内置灵活的进度回调，支持“任务数量”和“业务批次权重”双维度统计。
* **线程安全**：强制任务管理在主线程完成，有效规避多线程竞态风险。
* **无第三方依赖**：纯 Python 标准库实现，极致轻量。

---

## 📦 安装

```bash
pip install lite-taskman

```

---

## 💡 快速上手

### 1. 批量任务（极简模式）

当你有一堆已知的任务需要并行处理并获取结果时，使用 `exec()` 是最快的方式。

```python
import os
from lite_taskman import TaskMan

def get_file_size(path):
    return os.stat(path).st_size

# 使用 context manager 自动管理线程池的启停
tman = TaskMan(max_workers=4)
files = ["file1.txt", "file2.txt", "file3.txt"]

for f in files:
    # _tm_extra 可以携带任意自定义元数据随结果返回
    tman.add(get_file_size, f, _tm_name=f, _tm_extra=f"path/{f}")

# exec() 会阻塞直到所有任务完成，并返回结果列表
results = tman.exec()

for r in results:
    if r.error:
        print(f"失败: {r.name}, 错误: {r.error}")
    else:
        print(f"成功: {r.name}, 大小: {r.result} bytes")

```

### 2. 增量迭代（爬虫/递归模式）

这是 `lite-taskman` 最强大的地方：支持“边跑边加”。

```python
import requests
import re
from lite_taskman import TaskMan

BASE_URL = "https://quotes.toscrape.com"

def fetch_page(url):
    return requests.get(url, timeout=5).text

tman = TaskMan(max_workers=3)
tman.add(fetch_page, BASE_URL, _tm_name="Page-1")

with tman:
    # process() 是一个生成器，只要有新任务加入，它就不会停止
    for r in tman.process():
        if r.error: continue
        
        # 解析数据
        html = r.result
        quotes = re.findall(r'<span class="text".*?>(.*?)</span>', html)
        print(f"[{r.name}] 抓取到 {len(quotes)} 条语录")

        # 发现新分页，动态加入任务池
        next_match = re.search(r'<li class="next">\s*<a href="(.*?)">', html)
        if next_match:
            next_url = BASE_URL + next_match.group(1)
            tman.add(fetch_page, next_url, _tm_name="NextPage")

```

---

## 🛠️ 参数说明

### `TaskMan.add()` 专用参数

为了避免与目标函数的参数冲突，工具专有参数均以 `_tm_` 开头：

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `_tm_name` | 任务名称 | 任务函数名
| `_tm_batch_size` | 此子任务中批次数量，比如一次抓取一页，其中有20行记录。会在进度回调中回传。 | 1 
| `_tm_extra` | 透传数据，可以是任何对象，在 `Result.extra` 中原样返回。 | None

### 进度回调 `progress_cb`

你可以自定义回调函数来打印进度或刷新UI，参数定义如下：

```python
def my_cb(name, task_done, task_all, batch_done, batch_all, elapsed_sec):
    # name: 当前完成的任务名
    # task_done/task_all: 基于任务数量的进度
    # batch_done/batch_all: 基于业务权重(batch_size)的进度
    # elapsed_sec: 累计耗时（秒）
    pass

```

---

## 📄 开源协议

本项目遵循 [MIT License](https://www.google.com/search?q=LICENSE) 开源协议。

**作者**: Rocks Wang ([rockswang@foxmail.com](mailto:rockswang@foxmail.com))

