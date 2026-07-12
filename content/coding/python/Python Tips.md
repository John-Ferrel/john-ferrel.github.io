---
title: Python Tips
draft: false
tags:
  - python
  - coding
created: 2026-03-07 18:48
modified: 2026-07-12 00:00
---
随手记录一些做项目和准备面试时会用到的 Python 小模式。

---
## Decorators for Cross-Cutting Concerns

项目里经常有几类逻辑散落在不同函数中：

- logging
- timing
- permission checks
- retries
- validation

For example:
```python
import time  
  
def load_data():  
	start = time.time()  
	# actual work  
	# ...  
	print(f"load_data took {time.time() - start:.2f}s")
```

把它们直接写进业务函数，很快就会重复，也会遮住真正的业务逻辑。

### 用 decorator 收口

用 decorator 把这类附加逻辑从业务函数里拿出来。

```python
import time  
from functools import wraps  
  
def timeit(func):  
	# @wraps for identity and introspection
	@wraps(func)  
	def wrapper(*args, **kwargs):  
		# before code block
		start = time.time()  
		# func
		result = func(*args, **kwargs)  
		# after code block
		duration = time.time() - start  
		print(f"{func.__name__} took {duration:.2f}s")  
		return result  
	return wrapper
```

使用时：

```python
@timeit  
def load_data():  
# ...
```

### 这样做的好处

- 业务函数只保留自己的工作
- 同一套逻辑可以复用到多个函数
- 要调整或移除时有一个明确入口

Decorators are a practical way to implement **cross-cutting concerns** without polluting core logic. 写 decorator 时记得使用 `functools.wraps`，保留原函数的 metadata，避免影响 introspection 和 framework。

---

## Progress Bars with `tqdm`

耗时循环如果没有反馈，跑到一半很难判断进度。

For example:
```python
for i in range(1000000):  
    process(i)
```


处理大数据集或训练模型时，通常想知道：

- how much work is done
- how long it will take
- whether the program is stuck

手动加日志既吵，也不容易看出整体进度。

### 加上 `tqdm`

`tqdm` 可以加一条轻量进度条。

from tqdm import tqdm  

```python
for i in tqdm(range(1000000)):  
    process(i)
```

它会显示：

- 完成百分比
- 迭代速度
- 预计剩余时间

输出大致如下：
```
 45%|██████████▌       | 450000/1000000 [00:03<00:04, 120k it/s]
```

---

### 使用场景

- 改动很小
- 适用于 iterable、list、generator 和 pandas

配合 pandas：
```python
from tqdm import tqdm  
tqdm.pandas()  
df["result"] = df["data"].progress_apply(process)
```

循环会占用明显时间时就加 `tqdm`。做数据处理或实验时，一条进度条就能省掉很多等待中的猜测。

---

## Collections Utilities (`defaultdict`, `Counter`, `deque`)

计数、分组和队列这类常见操作，用普通 `dict` 往往要写不少初始化代码。

例如计数：

```python
counts = {}  
  
for word in words:  
    if word not in counts:  
        counts[word] = 0  
    counts[word] += 1

```

分组也会反复检查 key 是否存在：

```python
groups = {}  
  
for k, v in pairs:  
    if k not in groups:  
        groups[k] = []  
    groups[k].append(v)
```

### 直接用 `collections`

`collections` 已经提供了对应工具。

计数用 `Counter`：

```python
from collections import Counter  
counts = Counter(words)
```


分组用 `defaultdict`：
```python
from collections import defaultdict  
groups = defaultdict(list)  
for k, v in pairs:  
    groups[k].append(v)
```

队列操作用 `deque`：
```python
from collections import deque  
  
queue = deque()  
queue.append(1)  
queue.append(2)  
queue.popleft()
```

### 适合的场景

常见的数据结构模式可以直接对应到 `collections`：

- `Counter` → counting
- `defaultdict` → grouping / accumulation
- `deque` → efficient queues

这样写更短，也让常见操作的意图更明确。

---

## Priority Queues with `heapq`

需要不断维护排序时，反复对 list 排序会很慢。

例如每次加入元素后排序：

```python
data.append(x)  
data.sort()
```

或反复取最小值：

```python
min_value = min(data)
```

数据量大时，这些操作会越来越贵。
### 用 `heapq` 维护堆

`heapq` 实现了 **binary heap**。

```python
import heapq  
heap = []  
  
heapq.heappush(heap, 5)  
heapq.heappush(heap, 1)  
heapq.heappush(heap, 3)  
  
smallest = heapq.heappop(heap)
```

堆会把最小元素放在索引 `0`。
### 适合的场景

需要反复取集合里的最小值（或最大值）时，用 `heapq` 维护 **priority queue**，不用每次重新排序。

---
