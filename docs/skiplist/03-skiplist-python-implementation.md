# 跳表的Python实现和多语言对比

## 概述

Python作为一门高级语言，虽然不像C++或Rust那样直接操作内存，但其简洁的语法和丰富的库支持使得实现跳表变得更加直观。本文将深入探讨Python跳表的实现，并与Rust版本进行详细对比。

## Python实现跳表的优势

### 1. 语法简洁，易于理解
```python
# Python: 简洁直观
class SkipListNode:
    def __init__(self, key, value, level):
        self.key = key
        self.value = value
        self.forward = [None] * level

# Rust: 需要处理内存和类型安全
pub struct SkipListNode<K, V> {
    key: K,
    value: V,
    forward: Vec<AtomicPtr<SkipListNode<K, V>>>,
}
```

### 2. 动态类型，灵活使用
```python
# Python: 同一个跳表可以存储不同类型
skiplist = SkipList()
skiplist.insert("name", "Alice")
skiplist.insert("age", 25)
skiplist.insert("score", 95.5)
```

### 3. 内置支持，开发效率高
- 无需手动内存管理
- 丰富的标准库支持
- 更容易调试和测试

## 基础实现

### 1. 节点定义

```python
import random
from typing import Any, Optional, List, Generic, TypeVar, Callable
from dataclasses import dataclass
from functools import total_ordering

K = TypeVar('K')
V = TypeVar('V')

@total_ordering
@dataclass
class SkipListNode(Generic[K, V]):
    """跳表节点"""
    key: K
    value: V
    forward: List[Optional['SkipListNode[K, V]']]

    def __init__(self, key: K, value: V, level: int):
        self.key = key
        self.value = value
        self.forward = [None] * level

    def __lt__(self, other):
        if not isinstance(other, SkipListNode):
            return NotImplemented
        return self.key < other.key

    def __eq__(self, other):
        if not isinstance(other, SkipListNode):
            return NotImplemented
        return self.key == other.key

    def __hash__(self):
        return hash(self.key)

    def __repr__(self):
        return f"SkipListNode(key={self.key!r}, value={self.value!r})"
```

### 2. 基础跳表实现

```python
class SkipList(Generic[K, V]):
    """基础跳表实现"""

    def __init__(self, max_level: int = 16, p: float = 0.5):
        """
        初始化跳表

        Args:
            max_level: 最大层数
            p: 节点升级的概率
        """
        assert max_level > 0, "max_level must be positive"
        assert 0 < p <= 1, "probability must be between 0 and 1"

        self.max_level = max_level
        self.p = p
        self.level = 1  # 当前最大层数
        self.length = 0

        # 创建头节点
        self.head = SkipListNode(None, None, max_level)

        # 可选：自定义比较函数
        self._key_cmp: Optional[Callable[[K, K], int]] = None

    def set_key_comparator(self, cmp_func: Callable[[K, K], int]):
        """设置自定义比较函数"""
        self._key_cmp = cmp_func

    def _compare_keys(self, key1: K, key2: K) -> int:
        """比较两个键"""
        if self._key_cmp is not None:
            return self._key_cmp(key1, key2)

        # 默认比较
        if key1 < key2:
            return -1
        elif key1 > key2:
            return 1
        else:
            return 0

    def random_level(self) -> int:
        """随机生成节点高度"""
        level = 1
        while random.random() < self.p and level < self.max_level:
            level += 1
        return level

    def __len__(self) -> int:
        """返回跳表长度"""
        return self.length

    def __bool__(self) -> bool:
        """判断跳表是否为空"""
        return self.length > 0

    def __contains__(self, key: K) -> bool:
        """检查键是否存在"""
        return self.get(key) is not None

    def __repr__(self) -> str:
        """字符串表示"""
        items = []
        for key, value in self:
            items.append(f"{key!r}: {value!r}")
        return f"SkipList({', '.join(items)})"

    def __str__(self) -> str:
        """格式化输出"""
        lines = []
        lines.append(f"SkipList (len={self.length}, level={self.level}/{self.max_level})")

        for level in range(self.level - 1, -1, -1):
            nodes = []
            current = self.head.forward[level]
            while current:
                nodes.append(str(current.key))
                current = current.forward[level]
            lines.append(f"Level {level + 1}: {' -> '.join(nodes)}")

        return '\n'.join(lines)
```

### 3. 核心操作实现

```python
class SkipList(Generic[K, V]):
    # ... 前面的代码 ...

    def insert(self, key: K, value: V) -> Optional[V]:
        """
        插入键值对

        Args:
            key: 键
            value: 值

        Returns:
            如果键已存在，返回旧值；否则返回None
        """
        # 创建update数组，记录各层的前驱节点
        update = [None] * self.max_level
        current = self.head

        # 从最高层开始查找插入位置
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and self._compare_keys(current.forward[i].key, key) < 0:
                current = current.forward[i]
            update[i] = current

        # 检查是否已存在
        next_node = current.forward[0]
        if next_node and self._compare_keys(next_node.key, key) == 0:
            # 键已存在，更新值
            old_value = next_node.value
            next_node.value = value
            return old_value

        # 随机生成新节点的高度
        new_level = self.random_level()

        # 如果新高度超过当前最大高度，更新头节点指针
        if new_level > self.level:
            for i in range(self.level, new_level):
                update[i] = self.head
            self.level = new_level

        # 创建新节点
        new_node = SkipListNode(key, value, new_level)

        # 更新各层的指针
        for i in range(new_level):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node

        self.length += 1
        return None

    def get(self, key: K) -> Optional[V]:
        """
        获取键对应的值

        Args:
            key: 键

        Returns:
            值，如果键不存在则返回None
        """
        current = self.head

        # 从最高层开始查找
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and self._compare_keys(current.forward[i].key, key) < 0:
                current = current.forward[i]

        # 在最底层检查是否找到
        next_node = current.forward[0]
        if next_node and self._compare_keys(next_node.key, key) == 0:
            return next_node.value

        return None

    def remove(self, key: K) -> Optional[V]:
        """
        删除键值对

        Args:
            key: 键

        Returns:
            被删除的值，如果键不存在则返回None
        """
        update = [None] * self.max_level
        current = self.head

        # 从最高层开始查找
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and self._compare_keys(current.forward[i].key, key) < 0:
                current = current.forward[i]
            update[i] = current

        # 检查是否存在
        target = current.forward[0]
        if not target or self._compare_keys(target.key, key) != 0:
            return None

        old_value = target.value

        # 更新各层指针
        for i in range(self.level):
            if update[i].forward[i] != target:
                break
            update[i].forward[i] = target.forward[i]

        # 如果删除的节点是最高层的节点，降低层数
        while self.level > 1 and self.head.forward[self.level - 1] is None:
            self.level -= 1

        self.length -= 1
        return old_value
```

### 4. 批量操作

```python
class SkipList(Generic[K, V]):
    # ... 前面的代码 ...

    def update(self, key: K, value: V) -> None:
        """
        更新或插入键值对

        Args:
            key: 键
            value: 值
        """
        self.insert(key, value)

    def setdefault(self, key: K, default_value: V) -> V:
        """
        如果键不存在则设置默认值

        Args:
            key: 键
            default_value: 默认值

        Returns:
            键对应的值或默认值
        """
        value = self.get(key)
        if value is None:
            self.insert(key, default_value)
            return default_value
        return value

    def pop(self, key: K, default: Any = None) -> V:
        """
        删除并返回键对应的值

        Args:
            key: 键
            default: 如果键不存在，返回的默认值

        Returns:
            键对应的值或默认值

        Raises:
            KeyError: 如果键不存在且未提供默认值
        """
        value = self.remove(key)
        if value is None:
            if default is not None:
                return default
            raise KeyError(key)
        return value

    def clear(self) -> None:
        """清空跳表"""
        self.level = 1
        self.length = 0
        self.head = SkipListNode(None, None, self.max_level)
        for i in range(self.max_level):
            self.head.forward[i] = None

    def batch_insert(self, items: List[tuple[K, V]]) -> None:
        """
        批量插入多个键值对

        Args:
            items: 键值对列表
        """
        for key, value in items:
            self.insert(key, value)

    def batch_remove(self, keys: List[K]) -> List[Optional[V]]:
        """
        批量删除多个键

        Args:
            keys: 键列表

        Returns:
            被删除的值列表
        """
        return [self.remove(key) for key in keys]
```

### 5. 查询操作

```python
class SkipList(Generic[K, V]):
    # ... 前面的代码 ...

    def first(self) -> Optional[tuple[K, V]]:
        """获取第一个键值对"""
        first_node = self.head.forward[0]
        if first_node:
            return (first_node.key, first_node.value)
        return None

    def last(self) -> Optional[tuple[K, V]]:
        """获取最后一个键值对"""
        current = self.head

        # 从最高层开始，快速到达末尾
        for i in range(self.level - 1, -1, -1):
            while current.forward[i]:
                current = current.forward[i]

        if current != self.head:
            return (current.key, current.value)
        return None

    def ceiling(self, key: K) -> Optional[K]:
        """
        返回大于等于给定键的最小键

        Args:
            key: 键

        Returns:
            大于等于给定键的最小键
        """
        current = self.head

        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and self._compare_keys(current.forward[i].key, key) < 0:
                current = current.forward[i]

        next_node = current.forward[0]
        if next_node:
            return next_node.key
        return None

    def floor(self, key: K) -> Optional[K]:
        """
        返回小于等于给定键的最大键

        Args:
            key: 键

        Returns:
            小于等于给定键的最大键
        """
        current = self.head
        result = None

        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and self._compare_keys(current.forward[i].key, key) <= 0:
                current = current.forward[i]
                if self._compare_keys(current.key, key) <= 0:
                    result = current.key

        return result

    def range_query(self, start: Optional[K], end: Optional[K]) -> List[tuple[K, V]]:
        """
        范围查询

        Args:
            start: 起始键（包含）
            end: 结束键（包含）

        Returns:
            范围内的键值对列表
        """
        result = []
        current = self.head

        # 定位到起始位置
        for i in range(self.level - 1, -1, -1):
            while current.forward[i]:
                cmp_result = self._compare_keys(current.forward[i].key, start) if start is not None else -1
                if cmp_result < 0:
                    current = current.forward[i]
                else:
                    break

        # 遍历到结束位置
        while current.forward[0]:
            current = current.forward[0]

            # 检查是否超过结束位置
            if end is not None and self._compare_keys(current.key, end) > 0:
                break

            result.append((current.key, current.value))

        return result
```

### 6. 迭代器实现

```python
class SkipListIterator(Generic[K, V]):
    """跳表迭代器"""

    def __init__(self, node: Optional[SkipListNode[K, V]]):
        self.current = node

    def __iter__(self):
        return self

    def __next__(self) -> tuple[K, V]:
        if self.current is None:
            raise StopIteration
        result = (self.current.key, self.current.value)
        self.current = self.current.forward[0]
        return result

class SkipList(Generic[K, V]):
    # ... 前面的代码 ...

    def __iter__(self) -> SkipListIterator[K, V]:
        """返回迭代器"""
        return SkipListIterator(self.head.forward[0])

    def keys(self) -> List[K]:
        """返回所有键"""
        return [key for key, _ in self]

    def values(self) -> List[V]:
        """返回所有值"""
        return [value for _, value in self]

    def items(self) -> List[tuple[K, V]]:
        """返回所有键值对"""
        return list(self)

    def reverse(self) -> List[tuple[K, V]]:
        """返回反转的键值对列表"""
        return list(self)[::-1]

    def count(self, key: K) -> int:
        """统计键的出现次数"""
        count = 0
        current = self.head.forward[0]

        while current:
            if self._compare_keys(current.key, key) == 0:
                count += 1
            elif self._compare_keys(current.key, key) > 0:
                break
            current = current.forward[0]

        return count
```

## 高级特性

### 1. 有序字典接口

```python
class SkipListDict(SkipList[K, V]):
    """类似字典的跳表接口"""

    def __init__(self, max_level: int = 16, p: float = 0.5):
        super().__init__(max_level, p)

    def __getitem__(self, key: K) -> V:
        value = self.get(key)
        if value is None:
            raise KeyError(key)
        return value

    def __setitem__(self, key: K, value: V) -> None:
        self.insert(key, value)

    def __delitem__(self, key: K) -> None:
        if not self.remove(key):
            raise KeyError(key)

    def get_keys_between(self, start: K, end: K) -> List[K]:
        """获取两个键之间的所有键"""
        result = []
        for key, _ in self.range_query(start, end):
            result.append(key)
        return result

    def get_values_between(self, start: K, end: K) -> List[V]:
        """获取两个键之间的所有值"""
        result = []
        for _, value in self.range_query(start, end):
            result.append(value)
        return result
```

### 2. 统计和监控

```python
class SkipListWithStats(SkipList[K, V]):
    """带统计信息的跳表"""

    def __init__(self, max_level: int = 16, p: float = 0.5):
        super().__init__(max_level, p)
        self._insert_count = 0
        self._search_count = 0
        self._delete_count = 0
        self._level_distribution = [0] * max_level

    def insert(self, key: K, value: V) -> Optional[V]:
        self._insert_count += 1
        result = super().insert(key, value)

        # 统计新节点的层级分布
        if result is None:  # 新插入的节点
            # 找到刚插入的节点
            current = self.head
            for i in range(self.level - 1, -1, -1):
                while current.forward[i] and self._compare_keys(current.forward[i].key, key) < 0:
                    current = current.forward[i]
            if current.forward[0] and self._compare_keys(current.forward[0].key, key) == 0:
                node_level = len(current.forward[0].forward)
                self._level_distribution[node_level - 1] += 1

        return result

    def get(self, key: K) -> Optional[V]:
        self._search_count += 1
        return super().get(key)

    def remove(self, key: K) -> Optional[V]:
        self._delete_count += 1
        return super().remove(key)

    def get_stats(self) -> dict:
        """获取统计信息"""
        total_nodes = sum(self._level_distribution)
        return {
            'insert_count': self._insert_count,
            'search_count': self._search_count,
            'delete_count': self._delete_count,
            'current_length': self.length,
            'current_level': self.level,
            'max_level': self.max_level,
            'level_distribution': self._level_distribution.copy(),
            'theoretical_distribution': [
                round(self.length * (self.p ** i) * (1 - self.p), 2)
                for i in range(self.max_level)
            ],
            'avg_search_depth': self._calculate_avg_search_depth(),
            'memory_efficiency': self._calculate_memory_efficiency(),
        }

    def _calculate_avg_search_depth(self) -> float:
        """计算平均搜索深度"""
        if self.length == 0:
            return 0.0

        # 采样计算平均搜索深度
        sample_size = min(100, self.length)
        total_depth = 0

        # 采样一些键进行搜索深度计算
        sampled_keys = []
        current = self.head.forward[0]
        step = max(1, self.length // sample_size)
        count = 0

        while current and count < sample_size:
            sampled_keys.append(current.key)
            current = current.forward[0]
            count += 1

        for key in sampled_keys:
            depth = 0
            search_current = self.head
            for i in range(self.level - 1, -1, -1):
                depth += 1
                while search_current.forward[i] and self._compare_keys(search_current.forward[i].key, key) < 0:
                    search_current = search_current.forward[i]
            total_depth += depth

        return total_depth / len(sampled_keys) if sampled_keys else 0.0

    def _calculate_memory_efficiency(self) -> float:
        """计算内存效率"""
        # 理论节点数
        theoretical_nodes = self.length * (1 / (1 - self.p))
        # 实际节点数（近似）
        actual_nodes = sum((i + 1) * count for i, count in enumerate(self._level_distribution))

        if theoretical_nodes == 0:
            return 1.0

        return theoretical_nodes / actual_nodes

    def print_stats(self):
        """打印统计信息"""
        stats = self.get_stats()
        print("SkipList Statistics:")
        print(f"  Operations: Insert={stats['insert_count']}, Search={stats['search_count']}, Delete={stats['delete_count']}")
        print(f"  Size: {stats['current_length']} nodes")
        print(f"  Levels: {stats['current_level']}/{stats['max_level']}")
        print(f"  Avg Search Depth: {stats['avg_search_depth']:.2f}")
        print(f"  Memory Efficiency: {stats['memory_efficiency']:.2f}")
        print("  Level Distribution:")
        for i, (actual, theoretical) in enumerate(zip(stats['level_distribution'], stats['theoretical_distribution'])):
            if actual > 0:
                print(f"    Level {i+1}: {actual} nodes (theoretical: {theoretical:.1f})")
```

### 3. 持久化支持

```python
import pickle
import json
import csv
from pathlib import Path

class PersistentSkipList(SkipList[K, V]):
    """支持持久化的跳表"""

    def save_to_pickle(self, filepath: str) -> None:
        """保存到pickle文件"""
        data = {
            'max_level': self.max_level,
            'p': self.p,
            'items': list(self.items())
        }

        with open(filepath, 'wb') as f:
            pickle.dump(data, f)

    def load_from_pickle(self, filepath: str) -> None:
        """从pickle文件加载"""
        with open(filepath, 'rb') as f:
            data = pickle.load(f)

        self.__init__(data['max_level'], data['p'])
        for key, value in data['items']:
            self.insert(key, value)

    def save_to_json(self, filepath: str, key_serializer: Callable = str, value_serializer: Callable = str) -> None:
        """保存到JSON文件"""
        data = {
            'max_level': self.max_level,
            'p': self.p,
            'items': [
                {'key': key_serializer(key), 'value': value_serializer(value)}
                for key, value in self.items()
            ]
        }

        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def load_from_json(self, filepath: str, key_deserializer: Callable = lambda x: x, value_deserializer: Callable = lambda x: x) -> None:
        """从JSON文件加载"""
        with open(filepath, 'r', encoding='utf-8') as f:
            data = json.load(f)

        self.__init__(data['max_level'], data['p'])
        for item in data['items']:
            key = key_deserializer(item['key'])
            value = value_deserializer(item['value'])
            self.insert(key, value)

    def save_to_csv(self, filepath: str) -> None:
        """保存到CSV文件"""
        with open(filepath, 'w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(['key', 'value'])
            for key, value in self.items():
                writer.writerow([key, value])

    def load_from_csv(self, filepath: str) -> None:
        """从CSV文件加载"""
        with open(filepath, 'r', encoding='utf-8') as f:
            reader = csv.reader(f)
            next(reader)  # 跳过标题行
            for row in reader:
                if len(row) >= 2:
                    self.insert(row[0], row[1])
```

## Python vs Rust 对比

### 1. 实现复杂度对比

| 方面 | Python | Rust |
|------|--------|------|
| **代码行数** | ~200行 | ~400行 |
| **内存管理** | 自动垃圾回收 | 手动管理，但内存安全 |
| **类型安全** | 动态类型，运行时检查 | 静态类型，编译时检查 |
| **并发安全** | GIL保护，需要显式锁 | 编译时保证并发安全 |
| **性能** | 较慢，但开发效率高 | 极快，但开发较复杂 |

### 2. 内存使用对比

```python
# Python版本 - 内存使用分析
def analyze_python_memory():
    import sys
    sl = SkipList()

    # 测量空跳表
    empty_size = sys.getsizeof(sl)

    # 插入10000个整数键值对
    for i in range(10000):
        sl.insert(i, f"value_{i}")

    with_data_size = sys.getsizeof(sl) + sum(
        sys.getsizeof(node) for node in [sl.head.forward[0]] if sl.head.forward[0]
    )

    print(f"Python SkipList:")
    print(f"  Empty: {empty_size} bytes")
    print(f"  With 10K items: ~{with_data_size} bytes")
    print(f"  Per item overhead: ~{(with_data_size - empty_size) / 10000} bytes")
```

### 3. 性能测试对比

```python
import time
import random

def performance_comparison():
    """Python跳表性能测试"""
    sizes = [1000, 10000, 100000]

    for size in sizes:
        print(f"\nTesting with {size} items:")

        # 测试Python跳表
        py_sl = SkipList()

        # 插入测试
        start = time.time()
        for i in range(size):
            py_sl.insert(i, f"value_{i}")
        py_insert_time = time.time() - start

        # 查找测试
        start = time.time()
        for i in range(size):
            py_sl.get(i)
        py_search_time = time.time() - start

        # 删除测试
        start = time.time()
        for i in range(size):
            py_sl.remove(i)
        py_delete_time = time.time() - start

        print(f"  Python - Insert: {py_insert_time:.4f}s")
        print(f"  Python - Search: {py_search_time:.4f}s")
        print(f"  Python - Delete: {py_delete_time:.4f}s")

        # 对比Python内置字典
        py_dict = {}

        start = time.time()
        for i in range(size):
            py_dict[i] = f"value_{i}"
        dict_insert_time = time.time() - start

        start = time.time()
        for i in range(size):
            _ = py_dict.get(i)
        dict_search_time = time.time() - start

        start = time.time()
        for i in range(size):
            py_dict.pop(i, None)
        dict_delete_time = time.time() - start

        print(f"  Python Dict - Insert: {dict_insert_time:.4f}s")
        print(f"  Python Dict - Search: {dict_search_time:.4f}s")
        print(f"  Python Dict - Delete: {dict_delete_time:.4f}s")

        # 性能比率
        print(f"  Performance Ratios (SkipList / Dict):")
        print(f"    Insert: {py_insert_time / dict_insert_time:.2f}x")
        print(f"    Search: {py_search_time / dict_search_time:.2f}x")
        print(f"    Delete: {py_delete_time / dict_delete_time:.2f}x")
```

### 4. 使用场景对比

#### Python跳表适合的场景：
- **教学和学习**：代码简洁易懂
- **原型开发**：快速验证算法
- **中小规模数据**：性能足够
- **需要有序操作**：范围查询、统计等
- **开发效率优先**：快速迭代

#### Rust跳表适合的场景：
- **高性能要求**：每秒百万级操作
- **内存敏感**：嵌入式或资源受限环境
- **并发访问**：多线程高并发
- **生产环境**：稳定性和性能要求高
- **系统集成**：作为其他Rust项目的组件

## 完整测试示例

```python
def test_skiplist_comprehensive():
    """综合测试Python跳表"""
    print("=== Python SkipList Comprehensive Test ===")

    # 1. 基础操作测试
    print("\n1. Basic Operations Test:")
    sl = SkipList()

    # 插入测试
    test_items = [(i, f"value_{i}") for i in range(100)]
    for key, value in test_items:
        sl.insert(key, value)

    assert len(sl) == 100
    print(f"  ✓ Inserted {len(sl)} items")

    # 查找测试
    for key, value in test_items:
        assert sl.get(key) == value
    print("  ✓ All searches successful")

    # 包含性测试
    assert 50 in sl
    assert 999 not in sl
    print("  ✓ Contains test passed")

    # 2. 范围查询测试
    print("\n2. Range Query Test:")
    range_result = sl.range_query(10, 20)
    assert len(range_result) == 11  # 包含边界
    assert range_result[0][0] == 10
    assert range_result[-1][0] == 20
    print(f"  ✓ Range query returned {len(range_result)} items")

    # 3. 统计功能测试
    print("\n3. Statistics Test:")
    stats_sl = SkipListWithStats()
    for i in range(1000):
        stats_sl.insert(i, f"value_{i}")

    stats = stats_sl.get_stats()
    print(f"  ✓ Insert count: {stats['insert_count']}")
    print(f"  ✓ Current level: {stats['current_level']}")
    print(f"  ✓ Memory efficiency: {stats['memory_efficiency']:.2f}")

    # 4. 持久化测试
    print("\n4. Persistence Test:")
    persist_sl = PersistentSkipList()
    for i in range(50):
        persist_sl.insert(f"key_{i}", f"value_{i}")

    # 保存和加载
    persist_sl.save_to_json("test_skiplist.json")
    loaded_sl = PersistentSkipList()
    loaded_sl.load_from_json("test_skiplist.json")

    assert len(loaded_sl) == 50
    assert loaded_sl.get("key_25") == "value_25"
    print("  ✓ Persistence test passed")

    # 5. 性能测试
    print("\n5. Performance Test:")
    performance_comparison()

    print("\n=== All Tests Passed! ===")

if __name__ == "__main__":
    test_skiplist_comprehensive()
```

## 实际应用示例

### 1. 游戏排行榜

```python
class GameLeaderboard:
    """游戏排行榜实现"""

    def __init__(self):
        self.scores = SkipListDict()
        self.players = {}  # player_id -> name mapping

    def add_score(self, player_id: str, player_name: str, score: int):
        """添加玩家分数"""
        self.players[player_id] = player_name
        self.scores[score] = player_id

    def get_top_players(self, n: int = 10) -> List[tuple[str, int]]:
        """获取前N名玩家"""
        top_scores = self.scores.range_query(None, None)[-n:]
        return [(self.players[player_id], score) for score, player_id in reversed(top_scores)]

    def get_player_rank(self, player_id: str) -> int:
        """获取玩家排名"""
        player_score = None
        for score, pid in self.scores.items():
            if pid == player_id:
                player_score = score
                break

        if player_score is None:
            return 0

        # 计算排名
        rank = 1
        for score, _ in self.scores.items():
            if score > player_score:
                rank += 1

        return rank

# 使用示例
leaderboard = GameLeaderboard()
leaderboard.add_score("player1", "Alice", 8500)
leaderboard.add_score("player2", "Bob", 9200)
leaderboard.add_score("player3", "Charlie", 7800)

print("Top 3 players:")
for name, score in leaderboard.get_top_players(3):
    print(f"  {name}: {score}")

print(f"Alice's rank: {leaderboard.get_player_rank('player1')}")
```

### 2. 时间序列数据存储

```python
import time
from datetime import datetime, timedelta

class TimeSeriesStorage:
    """时间序列数据存储"""

    def __init__(self):
        self.data = SkipList()  # timestamp -> value

    def add_point(self, timestamp: float, value: float):
        """添加数据点"""
        self.data.insert(timestamp, value)

    def add_point_now(self, value: float):
        """添加当前时间的数据点"""
        self.add_point(time.time(), value)

    def get_range(self, start_time: float, end_time: float) -> List[tuple[float, float]]:
        """获取时间范围内的数据"""
        return self.data.range_query(start_time, end_time)

    def get_latest(self, n: int) -> List[tuple[float, float]]:
        """获取最新的n个数据点"""
        all_points = self.data.range_query(None, None)
        return all_points[-n:] if n <= len(all_points) else all_points

    def calculate_average(self, start_time: float, end_time: float) -> float:
        """计算时间范围内的平均值"""
        points = self.get_range(start_time, end_time)
        if not points:
            return 0.0
        return sum(value for _, value in points) / len(points)

# 使用示例
storage = TimeSeriesStorage()

# 模拟添加传感器数据
base_time = time.time() - 3600  # 1小时前
for i in range(3600):  # 每秒一个数据点
    timestamp = base_time + i
    value = 20 + 5 * math.sin(i * 0.1) + random.uniform(-1, 1)  # 模拟温度数据
    storage.add_point(timestamp, value)

# 查询最近10分钟的数据
recent_time = time.time() - 600
recent_data = storage.get_range(recent_time, time.time())
avg_temp = storage.calculate_average(recent_time, time.time())

print(f"Recent data points: {len(recent_data)}")
print(f"Average temperature (last 10 min): {avg_temp:.2f}°C")
```

## 总结

Python版本的跳表实现展现了该语言在算法实现方面的优势：

### 优势：
1. **开发效率高**：代码简洁，易于理解和修改
2. **灵活性强**：动态类型，支持多种数据类型
3. **调试方便**：丰富的调试工具和错误信息
4. **生态丰富**：大量第三方库支持

### 劣势：
1. **性能较低**：相比Rust/C++有明显性能差距
2. **内存开销**：Python对象有额外的内存开销
3. **并发限制**：GIL限制了真正的并行执行
4. **类型安全**：运行时才能发现类型错误

### 适用场景：
- **教学和学术**：算法演示和教学
- **快速原型**：验证算法可行性
- **中小规模应用**：性能要求不高的场景
- **数据处理**：作为数据处理管道的一部分

Python跳表虽然性能不如系统语言版本，但其简洁的实现和丰富的功能使其成为学习和使用跳表数据结构的优秀选择。对于大多数应用场景，Python版本的跳表已经足够高效，同时还提供了更好的开发体验。