---
title: Coding Practices
draft: false
tags:
  - code
created: 2026-02-27 22:29
modified: 2026-02-27 22:29
---
Small coding practices

### 链表数字相加

```python
class ListNode():
  def __init__(self,digit):
    self.val = digit
    self.next = None
```

显然, 反转链表, 加法器, 反转链表就可以了



```python
def add_listnode(l1, l2):
    # 反转链表
    r1 = reverse_listnode(l1)
    r2 = reverse_listnode(l2)
    
    dummy = ListNode()
    curr = dummy
    carry = 0
    
    # 处理相加
    while r1 or r2 or carry:
        # 处理节点不等长
        v1 = r1.val if r1 else 0
        v2 = r2.val if r2 else 0
        total = v1 + v2 + carry
        carry = total // 10
        curr.next = ListNode(total % 10)
        curr = curr.next
        if r1: r1 = r1.next
        if r2: r2 = r2.next
    
    # 反转结果链表
    return reverse_listnode(dummy.next)
```

### 最大化表达式

```text
f(i,j) = arr[i] + arr[j] - |j-i|
```

实际上是需要考虑 i < j 的情况.
就变成了

```text
f(i,j) = arr[i] + arr[j] + i - j
       = arr[i] + i + arr[j] - j
```
所以是

```python
def max_f(arr):
	if len(arr) < 2: return 0
	
	max_left = arr[0] + 0
	max_val = float('-inf')
	
	for j in range(1, len(arr)):
		# 当前最大值
		max_val = max(max_val, max_left + (arr[j] -j))
		# 左值
		max_left = max(max_left, arr[j] + j)
	
	return max_val
```

### 整数列表子列乘积最大值

可以想见, 只要列表越长,绝对值就越大.
所以只有3个候选列表,\[0:n+1\], \[i+1, n+1\], \[0, j-1 \], i, j 是第一个, 最后一个负数.
特别的, 如果存在0, 0就会导致分出左右两边.
所以是个递归?

```python

def max_sub_list(lst:list):
	n = len(lst)
	if n == 0: return 0
	if n == 1: return lst[0]
	
	max_ending_here = min_ending_here = global_max = nums[0]
	
	for i in range(1, n):
		num = lst[i]
		
		candidates = (num, 
					max_ending_here * num,
					min_ending_here * num)
		max_ending_here = max(candidates)
		min_ending_hrer = min(candidates)
		
		global_max = max(global_max, max_ending_here)
	
	return global_max

```