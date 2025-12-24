# lite-taskman 🚀

English version | [中文版](./README_zh.md)

`lite-taskman` is an extremely lightweight (~100 lines of code), easy-to-use, and powerful thread pool management tool for Python. It is specifically designed for scenarios requiring **dynamic task addition**, **real-time progress feedback**, and **streamed result processing**.

Unlike the native `ThreadPoolExecutor`, it allows you to continuously inject new tasks into the pool while consuming results—perfect for web crawlers, recursive scanning, and more.

## ✨ Key Features

* **Minimalist API**: Provides an `exec()` one-stop concurrency mode. Transition from sequential to parallel execution with minimal code changes.
* **Dynamic Incremental Execution**: Supports the `process()` streaming generator mode, allowing you to add tasks while the pool is running—ideal for "discover-and-crawl" scenarios.
* **Progress Tracking**: Built-in flexible callbacks supporting dual-dimension statistics: "Task Count" and "Business Batch Weight."
* **Thread Safety**: Enforces task management in the main thread to eliminate common multi-threading race conditions.
* **Zero Dependencies**: Pure Python implementation using only the standard library.

---

## 📦 Installation

```bash
pip install lite-taskman

```

---

## 💡 Quick Start

### 1. Minimalist Mode: From Sequential to Parallel

Easily convert sequential function calls into concurrent execution with just a few lines.

```python
# Original sequential logic
# res1 = fun1('hello', 1, kw=3)
# res2 = fun2('world', x='x', y=[15])
# res3 = fun3(25, {'a': 1, 'b': 2})

# Modernized with lite-taskman
from lite_taskman import TaskMan
tman = TaskMan() # Defaults to CPU count
tman.add(fun1, 'hello', 1, kw=3) # Keep arguments exactly as they were
tman.add(fun2, 'world', x='x', y=[15])
tman.add(fun3, 25, {'a': 1, 'b': 2})

# exec() maintains the order of results corresponding to the 'add' calls
res1, res2, res3 = (r.result for r in tman.exec())

```

### 2. Incremental Mode: "Add while Processing"

The most powerful feature of `lite-taskman`: dynamic task expansion. Perfect for recursive web crawling.

```python
import requests as http
import re
from lite_taskman import TaskMan

BASE_URL = "[https://quotes.toscrape.com](https://quotes.toscrape.com)"

tman = TaskMan()
tman.add(http.get, BASE_URL) # Initial "seed" task

with tman: # Context manager handles the pool lifecycle
    # process() is a generator that keeps running as long as new tasks are added
    for r in tman.process():
        html = r.result.text # r.result is the Response object from requests.get
        quotes = re.findall(r'<span class="text".*?>(.*?)</span>', html)
        print(f"[{r.name}] Found {len(quotes)} quotes.")
        
        # Discover new pages and add them to the pool dynamically
        if mch := re.search(r'<li class="next">\s*<a href="(.*?)">', html):
            next_url = BASE_URL + mch.group(1)
            tman.add(http.get, next_url) # Add new task on the fly

```

---

## 🛠️ API Reference

### `TaskMan.add()` Parameters

To avoid conflicts with target function arguments, tool-specific parameters are prefixed with `_tm_`:

| Parameter | Description | Default |
| --- | --- | --- |
| `_tm_name` | Task name for identification/logging. | Function name |
| `_tm_batch_size` | The numerical weight of this task (e.g., number of items in a page). | 1 |
| `_tm_extra` | Transparent data pass-through returned in `Result.extra`. | None |

### Progress Callback `progress_cb`

Define a custom callback to log progress or update a UI:

```python
def my_cb(name, task_done, task_all, batch_done, batch_all, elapsed_sec):
    # name: Name of the task just completed
    # task_done/task_all: Progress based on task count
    # batch_done/batch_all: Progress based on business weight (batch_size)
    # elapsed_sec: Cumulative elapsed time in seconds
    pass

```

---

## 📄 License

This project is licensed under the [MIT License](https://opensource.org/licenses/MIT).

**Author**: Rocks Wang ([rockswang@foxmail.com](mailto:rockswang@foxmail.com))

```
