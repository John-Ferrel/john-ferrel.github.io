---
title: Python Tips
draft: false
tags:
  - python
  - coding
created: 2026-03-07 18:48
modified: 2026-03-07 18:48
---
This is a continuously updated note collecting small Python tricks I encounter  while **building** real systems or **interviewing**.

tips framework:
```text
- Problem  
- Better way  
- Why it is better
- Takeaway
```

---
## Decorators for Cross-Cutting Concerns

### Problem

In many projects, the same logic appears repeatedly across functions:

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

This quickly pollutes business logic and becomes repetitive.

### Better way

Use a decorator to separate instrumentation from business logic.

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

Usage:

```python
@timeit  
def load_data():  
# ...
```

### Why it is better

- keeps business logic clean
- reusable across many functions
- centralizes cross-cutting logic
- easy to remove or modify

### Takeaway

Decorators are not just a language feature — they are a practical way to implement **cross-cutting concerns** without polluting core logic.

Always use `functools.wraps` when writing decorators to preserve the original function's metadata and maintain compatibility with introspection and frameworks.

---

## Progress Bars with `tqdm`

### Problem

Long-running loops provide no feedback about progress.

For example:
```python
for i in range(1000000):  
    process(i)
```


When processing large datasets or training models, it is hard to know:

- how much work is done
- how long it will take
- whether the program is stuck

Adding manual logging is noisy and uninformative.

### Better way

Use `tqdm` to add a lightweight progress bar.

from tqdm import tqdm  

```python
for i in tqdm(range(1000000)):  
    process(i)
```

`tqdm` automatically shows:

- progress percentage
- iteration speed
- estimated remaining time

Example output:
```
 45%|██████████▌       | 450000/1000000 [00:03<00:04, 120k it/s]
```

---

### Why it is better

- provides immediate feedback for long loops
- minimal code change
- very low runtime overhead
- works with iterables, lists, generators, and pandas

Example with pandas:
```python
from tqdm import tqdm  
tqdm.pandas()  
df["result"] = df["data"].progress_apply(process)
```

### Takeaway

Use `tqdm` whenever a loop may take noticeable time.  
A small progress bar greatly improves observability during data processing or experimentation.

---

## Collections Utilities (`defaultdict`, `Counter`, `deque`)

### Problem

Common data structures such as counting, grouping, or queues often require verbose boilerplate code.

For example, counting items:

```python
counts = {}  
  
for word in words:  
    if word not in counts:  
        counts[word] = 0  
    counts[word] += 1

```

Grouping values also requires repeated checks:

```python
groups = {}  
  
for k, v in pairs:  
    if k not in groups:  
        groups[k] = []  
    groups[k].append(v)
```

### Better way

Use utilities from `collections`.

Counting with `Counter`:

```python
from collections import Counter  
counts = Counter(words)
```


Grouping with `defaultdict`:
```python
from collections import defaultdict  
groups = defaultdict(list)  
for k, v in pairs:  
    groups[k].append(v)
```

Queue-like operations with `deque`:
```python
from collections import deque  
  
queue = deque()  
queue.append(1)  
queue.append(2)  
queue.popleft()
```

### Why it is better

- removes repetitive initialization logic
- clearer intent for common patterns
- optimized implementations
- reduces boilerplate in data processing code
### Takeaway

Use `collections` utilities for common data structure patterns:

- `Counter` → counting
- `defaultdict` → grouping / accumulation
- `deque` → efficient queues

They simplify code and make common operations explicit.

---

## Priority Queues with `heapq`

### Problem

Maintaining a dynamically sorted list is inefficient.

For example, repeatedly sorting:

```python
data.append(x)  
data.sort()
```

Or finding the smallest element repeatedly:

```python
min_value = min(data)
```

These operations become expensive for large datasets.
### Better way

Use `heapq`, which implements a **binary heap**.

```python
import heapq  
heap = []  
  
heapq.heappush(heap, 5)  
heapq.heappush(heap, 1)  
heapq.heappush(heap, 3)  
  
smallest = heapq.heappop(heap)
```

A heap maintains the smallest element at index `0`.
### Why it is better

- efficient insert and remove operations (`O(log n)`)
- avoids repeated sorting
- suitable for streaming data or priority queues
### Takeaway

Use `heapq` when you need to repeatedly access the smallest (or largest) element in a collection.

It provides an efficient way to maintain a **priority queue** without full sorting.

---

