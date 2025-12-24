# lite-taskman 🚀

[English version](./README.md) | 中文版

`lite-taskman` 是一个极其轻量（仅约 100 行代码）易于使用且功能强大的 Python 线程池管理工具。它专为需要**动态增加任务**、**实时进度反馈**以及**流式处理结果**的场景而设计。

与原生 `ThreadPoolExecutor` 不同，它允许你在消费结果的同时不断往任务池里塞入新任务，非常适合爬虫、递归扫描等场景。

## ✨ 核心特性

* **极简 API**：提供 `exec()` 一键式并发模式。仅需一点点修改即可将顺序执行的代码改成并发执行。
* **动态增量执行**：支持 `process()` 流式生成器模式，可以边跑边追加任务，特别适合爬虫场景。
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

### 1. 极简模式：顺序秒改并发

只需一点点修改即可将原来顺序执行的调用轻易转换成并发执行。

```python
# 原逻辑
# res1 = fun1('hello', 1, kw=3)
# res2 = fun2('world', x='x', y=[15])
# res3 = fun3(25, {'a': 1, 'b': 2})

# 使用 lite-taskman 快速改造
from lite_taskman import TaskMan
tman = TaskMan() # 创建线程池，默认和系统线程数一致
tman.add(fun1, 'hello', 1, kw=3) # 把函数名作为第一个参数，后面参数一点不用变
tman.add(fun2, 'world', x='x', y=[15])
tman.add(fun3, 25, {'a': 1, 'b': 2})
res1, res2, res2 = (r.result for r in tman.exec()) # 结果排序和add调用顺序一致
```

### 2. 增量模式：“一边吃饭一边添饭”

这是 `lite-taskman` 最强大的地方：支持“边跑边加”。特别适合爬虫类场景。

```python
import requests as http
import re
from lite_taskman import TaskMan

BASE_URL = "https://quotes.toscrape.com"

tman = TaskMan()
tman.add(http.get, BASE_URL) # 作为“种子”的初始任务
with tman: # 使用上下文管理器，自动管理线程池生命周期
    for r in tman.process(): # process() 是一个生成器，只要有新任务加入，它就不会停止
        html = r.result.text # r.result 是 requests.get方法返回的 Response 对象
        quotes = re.findall(r'<span class="text".*?>(.*?)</span>', html)
        print(f"[{r.name}] 抓取到 {len(quotes)} 条语录")
        # 查找新分页，动态加入任务池
        if mch := re.search(r'<li class="next">\s*<a href="(.*?)">', html):
            next_url = BASE_URL + mch.group(1) # 获取下一页的 URL
            tman.add(fetch_page, next_url) # 动态添加新任务

```

---

## 🛠️ 参数说明

### `TaskMan.add()` 专用参数

为了避免与目标函数的参数冲突，工具专有参数均以 `_tm_` 开头：

| 参数 | 说明 | 默认值 |
| --- | --- | --- |
| `_tm_name` | 任务名称 | 任务函数名，仅用于打印回显
| `_tm_batch_size` | 此子任务中批次数量，比如一次抓取一页作为一个子任务，其中有20行记录。会在进度回调中回传。 | 1 
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

