---
title: Python
draft: "false"
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
