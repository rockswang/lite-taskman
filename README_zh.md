# lite-taskman 🚀

[English version](./README.md) | 中文版

`lite-taskman` 是一款极致轻量（约 100 行代码）的 Python 线程池管理利器，致力于提供令人舒适的开发体验。它专为**顺序代码的并发化改造**、**任务动态增量**及**实时进度反馈**而生。

不同于原生 `ThreadPoolExecutor` 的繁琐，它支持在任务执行中动态追加任务，并提供符合直觉的结果获取方式，助你实现从单线程到多线程的丝滑迁移。

## ✨ 核心特性

在并发编程中，复杂性带来的心智负担往往是Python新手的敌人。`lite-taskman` 将强大的功能浓缩进约 100 行源码中，为你带来：

* **丝滑迁移的体验**：无需重构核心逻辑，只需简单的 API 替换，即可将陈旧的顺序代码一键升维，享受并发带来的性能红利。
* **符合直觉的操控**：摆脱复杂的 Future 对象管理。无论是按序获取结果，还是流式消费输出，其逻辑设计高度贴合开发者的自然思维。
* **令人舒适的动态性**：像“一边吃饭一边添饭”一样自然。它支持在任务处理过程中随时注入新任务，让爬虫、递归扫描等增量场景变得前所未有的优雅。
* **极致轻量的反馈**：零依赖、毫秒级响应，配合实时进度回调，让每一个任务的状态都尽在掌握。

---

## 📦 安装

```bash
pip install lite-taskman

```

---

## 💡 常用模式

### 1. 极简模式：顺序秒改并发

只需将函数调用改为 `add()`，即可利用多核性能。

```python
# 原顺序逻辑。
# 总执行时间 = time(make_soymilk) + time(buy_buns) + time(make_dumplings)
# soymilk = make_soymilk(soy, milk)
# buns = buy_buns(2)
# dumplings = make_dumplings(8, type='pork')

# 简单加三行并稍许修改即可将顺序代码转为并发执行，极致丝滑！
from lite_taskman import TaskMan

tman = TaskMan() # 创建线程池
tman.add(make_soymilk, soy, milk) # 添加任务：简单将函数作为第一个参数
tman.add(buy_buns, 2) # 其它参数原封不动即可
tman.add(make_dumplings, 8, type='pork') # 可以用专有参数设定任务名称、任务权重、透传数据，见文档

# exec() 返回的结果列表顺序严格对应 add() 的顺序
# 总执行时间 = max(time(make_soymilk), time(buy_buns), time(make_dumplings))
soymilk, buns, dumplings = [r.result for r in tman.exec()]
```

### 2. 增量模式：“一边吃饭一边添饭”

特别适合爬虫场景：抓取第一页时发现第二页的 URL，直接 `add` 即可。

```python
with TaskMan() as tman:
    tman.add(fetch_url, "http://example.com/page1")
    for r in tman.process():
        # process() 是生成器，只要有新任务加入，循环就会继续
        new_urls = parse_links(r.result)
        for url in new_urls:
            tman.add(fetch_url, url) # 动态追加

```

### 3. 连接池复用

如果您有多个批次任务，但不想频繁销毁和创建线程池，可以使用 `all()`。

```python
with TaskMan(max_workers=4) as tman:
    # 第一批任务
    tman.add(make_soymilk, soy, milk)
    tman.add(buy_bun, money)
    soymilk, bun = (t.result for t in tman.all()) # 阻塞获取第一批结果，不关闭线程池
    
    # 第二批任务（复用上面的 4 个工作线程）
    for i in range(8): tman.add(make_dumpling, i)
    dumplings = (t.result for t in tman.all()) 

```

---

## 🛠️ API 说明

### 核心方法

| 方法 | 说明 | 适用场景 |
| --- | --- | --- |
| `add(...)` | 提交任务，支持专有参数即任务名称、条目数量及透传数据。提交就立即开始执行了。 | 所有。 |
| `process()` | **生成器**。按任务**完成顺序**实时产出结果，支持动态 `add`。 | 爬虫、递归、需要实时处理结果。 |
| `all()` | **阻塞方法**。等待所有任务完成，并按 **`add` 顺序**返回结果列表。 | 批量处理，且需要复用线程池。 |
| `exec()` | **阻塞方法**。等同于 `with` + `all()`。执行完自动关闭线程池。 | 一次性并行任务，极简调用。 |
| `shutdown()` | 立即关闭线程池并清空任务队列。 | 手动资源管理。 |

### `TaskMan.add()` 专用参数

为避免冲突，工具参数均以 `_tm_` 开头：

* `_tm_name`: 任务名称（默认为函数名），用于日志和回调。
* `_tm_batch_size`: 任务权重（即任务内条目数量）。例如一页包含 20 条记录，可设为 20。
* `_tm_extra`: 透传数据，在 `Result.extra` 中原样返回。

### 进度回调 `progress_cb`

```python
def my_cb(name, task_done, task_all, batch_done, batch_all, elapsed_sec):
    # name: 当前任务名称
    # task_done/task_all: 基于任务数量的进度
    # batch_done/batch_all: 基于业务权重(batch_size)的进度
    # elapsed_sec(float): 累计耗时，单位秒

```

---

## 📄 开源协议

本项目遵循 [MIT License](https://opensource.org/licenses/MIT) 开源协议。

**作者**: Rocks Wang ([rockswang@foxmail.com](mailto:rockswang@foxmail.com))

