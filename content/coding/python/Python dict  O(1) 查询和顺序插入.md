---
title: Python dict  O(1) 查询和顺序插入
date: 2026-05-13
tags:
  - Python
draft: false
created: 2026-05-13 19:45
modified: 2026-05-13 19:46
---
从 Python 3.6 开始，`dict` 在 CPython 实现中开始保持插入顺序。  
从 Python 3.7 开始，`dict` 保持插入顺序正式成为 Python 语言规范。

也就是说：

```python
d = {}

d["a"] = 1
d["b"] = 2
d["c"] = 3

print(d)
# {'a': 1, 'b': 2, 'c': 3}
````

这里的“有序”指的是 **插入顺序**，不是 key 的排序顺序。

```python
d = {}

d[10] = "x"
d[3] = "y"
d[7] = "z"

print(d)
# {10: 'x', 3: 'y', 7: 'z'}
```

它不会自动变成：

```python
{3: 'y', 7: 'z', 10: 'x'}
```

---

## 1. dict 为什么查询是平均 O(1)？

`dict` 的核心仍然是 **哈希表**。

当我们访问：

```python
d["name"]
```

Python 大致会做：

```text
hash("name") -> 找到哈希表中的位置 -> 取出 value
```

如果哈希分布良好，大部分 key 都能很快定位到对应位置，所以查询复杂度是：

```text
平均 O(1)
```

但严格来说，哈希表的最坏情况仍然可能退化为：

```text
最坏 O(n)
```

只是正常情况下，Python 会通过扩容、扰动探测、hash 随机化等机制，让它保持非常高效。

---

## 2. 有序和 O(1) 查询如何同时实现？

如果只用普通数组保存 key-value：

```text
[("a", 1), ("b", 2), ("c", 3)]
```

那么遍历天然有序，但是查询 `"b"` 需要从头扫描：

```text
O(n)
```

如果只用传统哈希表：

```text
slot 0: empty
slot 1: ("c", 3)
slot 2: empty
slot 3: ("a", 1)
slot 4: ("b", 2)
```

查询很快，但是遍历顺序取决于哈希槽位，不一定是插入顺序。

Python 3.6+ 的 compact dict 采用了一个很巧妙的结构：

```text
indices table + entries array
```

可以粗略理解成：

```text
indices table：稀疏哈希索引表，负责快速查找
entries array：紧凑数据数组，负责保存 key/value/hash，并保持插入顺序
```

---

## 3. compact dict 的结构

大概可以想象成这样：

```text
indices table
+------+-------+
| slot | index |
+------+-------+
|  0   |  -    |
|  1   |  2    |
|  2   |  -    |
|  3   |  0    |
|  4   |  1    |
+------+-------+

entries array
+-------+------+-------+
| index | key  | value |
+-------+------+-------+
|   0   | "a"  |   1   |
|   1   | "b"  |   2   |
|   2   | "c"  |   3   |
+-------+------+-------+
```

查询时：

```text
hash(key) -> indices table -> entries array -> value
```

遍历时：

```text
直接从 entries array 头到尾遍历
```

因为新的 key 会按插入顺序追加到 `entries array`，所以遍历自然就是插入顺序。

这就是 Python dict 能同时做到：

```text
查询：平均 O(1)
插入：平均 O(1)
遍历：O(n)，且按插入顺序
```

的原因。

---

## 4. 这是用双倍空间换时间吗？

不完全是。

更准确地说，它确实维护了两类结构：

```text
1. indices table
2. entries array
```

但这不是保存了两份完整数据。

真正的 key、value、hash 存在 `entries array` 里。  
`indices table` 只保存一个较小的整数索引，用来告诉 Python：

```text
这个哈希槽位对应 entries array 里的第几个 entry
```

所以它不是：

```text
一份 dict 变成两份 dict
```

而是：

```text
稀疏索引 + 紧凑数据
```

这种结构不仅让 dict 保持了插入顺序，很多时候还比老版本 dict 更省内存。

因为老版本哈希表的空槽位也要预留完整 entry 空间；  
而新结构里，空槽位主要只存在于较轻量的 `indices table` 中。

---

## 5. 哈希冲突如何解决？

不同 key 可能映射到同一个哈希槽位，这叫 **哈希冲突**。

Python dict 主要使用：

```text
开放寻址 open addressing + 扰动探测 probing
```

它不是链表法。

也就是说，不是这样：

```text
slot 3: ("a", 1) -> ("b", 2) -> ("c", 3)
```

而是这样：

```text
slot 3: "a"
slot 5: "b"
slot 1: "c"
```

当某个位置已经被占用时，Python 会根据一套探测规则继续寻找下一个候选位置。

简化理解：

```text
hash(key) 得到初始位置
如果位置为空，直接放入
如果位置被占用，继续探测下一个位置
直到找到空位
```

查找时也使用同样的探测路径：

```text
hash(key) 得到初始位置
如果 key 相同，返回 value
如果 key 不同，继续按照同样的探测规则找
直到找到 key 或遇到真正的 empty 槽位
```

---

## 6. 删除为什么需要 dummy 标记？

开放寻址有一个问题：删除时不能简单地把槽位清空。

假设某个 key 的查找路径是：

```text
slot 3 -> slot 5
```

如果 `slot 3` 被删除后直接变成 empty，那么后续查找可能在 `slot 3` 就停止，导致找不到其实存在于 `slot 5` 的元素。

所以删除时，Python 会留下类似 `dummy` / `tombstone` 的标记。

查找时：

```text
遇到 dummy：继续探测
遇到 empty：说明真的不存在
```

插入时：

```text
dummy 槽位可以被复用
```

如果删除很多，dict 后续可能会在扩容或重建时清理这些 dummy。

---

## 7. 更新和重新插入的顺序规则

更新已有 key 不改变顺序：

```python
d = {}

d["a"] = 1
d["b"] = 2
d["a"] = 100

print(d)
# {'a': 100, 'b': 2}
```

`"a"` 的 value 被更新了，但位置没有变。

但是删除后再插入，会移动到最后：

```python
d = {
    "a": 1,
    "b": 2,
    "c": 3,
}

del d["b"]
d["b"] = 200

print(d)
# {'a': 1, 'c': 3, 'b': 200}
```

因为这次 `"b"` 是重新插入，所以它变成了新的最后一个元素。

---

## 8. 和 OrderedDict 的区别

现在普通 `dict` 已经保持插入顺序，所以很多场景不再需要 `OrderedDict`。

但 `OrderedDict` 仍然有一些特殊能力，比如：

```python
from collections import OrderedDict

od = OrderedDict()
od["a"] = 1
od["b"] = 2

od.move_to_end("a")

print(od)
# OrderedDict([('b', 2), ('a', 1)])
```

如果需要频繁调整元素顺序，比如实现 LRU Cache，`OrderedDict` 仍然有意义。

普通 `dict` 更适合大多数日常 key-value 存储。

---

## 9. 总结

Python 3.6+ 的 dict 可以理解为：

```text
一个哈希表外壳 + 一个按插入顺序排列的紧凑 entries 数组
```

它的关键设计是：

```text
indices table 负责快速查找
entries array 负责真实数据存储和有序遍历
```

所以它能同时做到：

```text
d[key] 查询：平均 O(1)
d[key] 插入：平均 O(1)
d[key] 删除：平均 O(1)
遍历 dict：O(n)，且按插入顺序
```

但需要注意：

```text
dict 的有序 = 插入顺序
不是 key 的排序顺序
```

一句话总结：

> Python dict 不是靠排序来保持有序，而是通过 compact hash table，把“查找用的哈希索引”和“按插入顺序保存的数据数组”拆开，从而同时获得了平均 O(1) 查询和稳定的插入顺序遍历。