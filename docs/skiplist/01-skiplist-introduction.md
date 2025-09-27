# 跳表(Skip List)：概率平衡的艺术

## 概述

跳表(Skip List)是一种概率数据结构，由William Pugh在1990年提出。它通过在有序链表的基础上添加多级索引，实现了类似平衡树的性能，但实现起来却简单得多。跳表在Redis、LevelDB、RocksDB等知名系统中都有广泛应用。

## 跳表的本质：有序链表的进化

### 传统有序链表的限制

让我们先来看一个传统的有序链表：

```
┌───┐    ┌───┐    ┌───┐    ┌───┐    ┌───┐
│ 1 │───►│ 3 │───►│ 7 │───►│ 9 │───►│12 │
└───┘    └───┘    └───┘    └───┘    └───┘
```

在有序链表中，查找一个元素需要从头开始逐个比较，时间复杂度为O(n)。当数据量很大时，这种线性查找的效率就显得很低。

### 跳表的创新思想

跳表的核心思想是：**如果我们能够"跳过"一些节点，是不是就能更快地找到目标？**

这就像在一本书中查找内容，我们通常会先看目录，找到对应的章节，然后再在该章节中详细查找。跳表就是基于这个思想构建的：

```
Level 3:  ┌─────┐       ┌─────┐
          │  7  ├──────►│ 12  │
          └─────┘       └─────┘
                    ▲

Level 2:  ┌─────┐ ┌─────┐ ┌─────┐
          │  3  ├─►│  7  ├─►│ 12  │
          └─────┘ └─────┘ └─────┘
                          ▲

Level 1:  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
          │  1  ├─►│  3  ├─►│  7  ├─►│ 9  ├─►│ 12 │
          └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
```

通过这种多级索引结构，我们可以：
1. 从最高层开始查找
2. 如果当前节点的值大于目标，就下降到下一层
3. 重复这个过程直到找到目标

## 跳表的工作原理

### 基本概念

跳表由多层链表组成：
- **底层(Level 1)**：包含所有节点的完整有序链表
- **上层(Level 2, 3, ...)**：稀疏的索引层，包含部分节点的指针

### 节点结构

每个跳表节点包含：
- 键值对 (Key-Value)
- 指针数组，指向不同层次的下一个节点
```
┌─────────────────────────────────────┐
│            SkipListNode             │
├─────────────────────────────────────┤
│ Key: 42                             │
│ Value: "Answer to Life"             │
├─────────────────────────────────────┤
│ forward[0] ──► next node on level 1  │
│ forward[1] ──► next node on level 2  │
│ forward[2] ──► next node on level 3  │
│ ...                                 │
└─────────────────────────────────────┘
```

### 查找操作

让我们通过一个具体的例子来理解查找过程：

假设我们要在以下跳表中查找值 **8**：

```
Level 3:  ┌─────┐       ┌─────┐
          │  7  ├──────►│ 12  │
          └─────┘       └─────┘

Level 2:  ┌─────┐ ┌─────┐ ┌─────┐
          │  3  ├─►│  7  ├─►│ 12  │
          └─────┘ └─────┘ └─────┘

Level 1:  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
          │  1  ├─►│  3  ├─►│  7  ├─►│ 9  ├─►│ 12 │
          └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
```

**查找过程**：
1. 从Level 3开始，当前节点是7
2. 7 < 8，尝试前进，但下一个节点是12 > 8
3. 下降到Level 2，当前节点仍然是7
4. 在Level 2上，下一个节点是12 > 8，继续下降到Level 1
5. 在Level 1上，从7前进到9
6. 9 > 8，发现8不存在

### 插入操作

插入操作的关键在于确定新节点的"高度"（层数）。

#### 随机高度算法

跳表使用概率算法来确定节点高度：

```python
import random

def random_level(max_level=16, p=0.5):
    level = 1
    while random.random() < p and level < max_level:
        level += 1
    return level
```

这个算法确保：
- 有50%的概率节点只有1层
- 有25%的概率节点有2层
- 有12.5%的概率节点有3层
- 以此类推...

#### 插入步骤

假设要插入值 **8**，随机得到高度为2：

```
Before:
Level 3:  ┌─────┐       ┌─────┐
          │  7  ├──────►│ 12  │
          └─────┘       └─────┘

Level 2:  ┌─────┐ ┌─────┐ ┌─────┐
          │  3  ├─►│  7  ├─►│ 12  │
          └─────┘ └─────┘ └─────┘

Level 1:  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
          │  1  ├─►│  3  ├─►│  7  ├─►│ 9  ├─►│ 12 │
          └─────┘ └─────┘ └─────┘ └─────┘ └─────┘

After:
Level 3:  ┌─────┐       ┌─────┐
          │  7  ├──────►│ 12  │
          └─────┘       └─────┘

Level 2:  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
          │  3  ├─►│  7  ├─►│  8  ├─►│ 12  │
          └─────┘ └─────┘ └─────┘ └─────┘

Level 1:  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
          │  1  ├─►│  3  ├─►│  7  ├─►│ 8  ├─►│ 9  ├─►│ 12 │
          └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
```

### 删除操作

删除操作相对简单：
1. 查找要删除的节点
2. 在每一层中更新指针，跳过要删除的节点
3. 释放节点内存

## 性能分析

### 时间复杂度

- **查找**: O(log n) - 平均情况
- **插入**: O(log n) - 平均情况
- **删除**: O(log n) - 平均情况

最坏情况下（所有节点都只有1层），性能退化为O(n)，但概率极低。

### 空间复杂度

每个节点平均包含 2 个指针（包含Level 1的指针），所以空间复杂度为O(n)。

### 与平衡树的对比

| 特性 | 跳表 | 平衡树 |
|------|------|--------|
| 实现复杂度 | 简单 | 复杂 |
| 平均性能 | O(log n) | O(log n) |
| 最坏性能 | O(n) | O(log n) |
| 平衡方法 | 概率性 | 旋转操作 |
| 内存开销 | O(n) | O(n) |
| 缓存友好性 | 一般 | 较好 |

## 为什么选择跳表？

### 优势

1. **实现简单**：相比红黑树等平衡树，跳表的实现要简单得多
2. **无锁操作**：跳表更适合并发环境，容易实现无锁操作
3. **性能稳定**：概率性平衡避免了最坏情况的发生
4. **内存效率**：相比平衡树，内存开销更小

### 劣势

1. **最坏情况**：理论上存在O(n)的最坏情况（虽然概率极低）
2. **缓存不友好**：多级结构可能导致缓存不命中
3. **随机性**：性能依赖于随机数生成器的质量

## 跳表的应用场景

### 1. 内存数据库

**Redis** 的 SortedSet 底层就使用了跳表：

```redis
ZADD myzset 1 "one"
ZADD myzset 2 "two"
ZADD myzset 3 "three"
ZRANGE myzset 0 -1
```

### 2. 存储引擎

**LevelDB** 和 **RocksDB** 使用跳表作为 MemTable 的实现：

```cpp
// LevelDB 中的跳表节点
template <typename Key, class Comparator>
class SkipList {
 private:
  struct Node {
    const Key key;
    // 访问下一层的指针数组
    std::atomic<Node*> next_[1];
  };
};
```

### 3. 并发数据结构

跳表的层次结构使其特别适合并发环境：

```java
// Java ConcurrentSkipListMap
ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
map.put(1, "one");
map.put(2, "two");
String value = map.get(1);
```

## 跳表的变体

### 1. 确定性跳表

使用确定性算法而非随机算法来平衡跳表，避免了随机性带来的不确定性。

### 2. 跳表哈希

结合跳表和哈希表的优势，提供更好的性能。

### 3. 内存优化跳表

针对内存使用进行优化的跳表变体。

## 实现一个简单的跳表

让我们用Python实现一个简单的跳表来加深理解：

```python
import random

class SkipListNode:
    def __init__(self, key, value, max_level):
        self.key = key
        self.value = value
        self.forward = [None] * max_level  # 各层的指针

class SkipList:
    def __init__(self, max_level=16, p=0.5):
        self.max_level = max_level
        self.p = p
        self.level = 1  # 当前最大层数
        self.header = SkipListNode(None, None, max_level)

    def random_level(self):
        level = 1
        while random.random() < self.p and level < self.max_level:
            level += 1
        return level

    def insert(self, key, value):
        # 创建update数组，记录各层的前驱节点
        update = [None] * self.max_level
        current = self.header

        # 从最高层开始查找插入位置
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        # 如果key已存在，更新value
        if current.forward[0] and current.forward[0].key == key:
            current.forward[0].value = value
            return

        # 随机生成新节点的高度
        new_level = self.random_level()

        # 如果新高度超过当前最大高度，更新header指针
        if new_level > self.level:
            for i in range(self.level, new_level):
                update[i] = self.header
            self.level = new_level

        # 创建新节点
        new_node = SkipListNode(key, value, self.max_level)

        # 更新各层的指针
        for i in range(new_level):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node

    def search(self, key):
        current = self.header

        # 从最高层开始查找
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]

        # 在最底层检查是否找到
        if current.forward[0] and current.forward[0].key == key:
            return current.forward[0].value
        return None

    def delete(self, key):
        update = [None] * self.max_level
        current = self.header

        # 从最高层开始查找
        for i in range(self.level - 1, -1, -1):
            while current.forward[i] and current.forward[i].key < key:
                current = current.forward[i]
            update[i] = current

        # 检查是否存在
        target = current.forward[0]
        if not target or target.key != key:
            return False

        # 更新各层指针
        for i in range(self.level):
            if update[i].forward[i] != target:
                break
            update[i].forward[i] = target.forward[i]

        # 如果删除的节点是最高层的节点，降低层数
        while self.level > 1 and self.header.forward[self.level - 1] is None:
            self.level -= 1

        return True

    def display(self):
        print("Skip List Structure:")
        for i in range(self.level):
            print(f"Level {i + 1}:", end=" ")
            current = self.header.forward[i]
            while current:
                print(f"{current.key}", end=" -> ")
                current = current.forward[i]
            print("None")
```

### 使用示例

```python
# 创建跳表
sl = SkipList()

# 插入数据
for i in range(1, 11):
    sl.insert(i, f"value_{i}")

# 显示结构
sl.display()

# 查找测试
print("\nSearch tests:")
print(f"Search 5: {sl.search(5)}")
print(f"Search 15: {sl.search(15)}")

# 删除测试
print("\nDelete tests:")
print(f"Delete 7: {sl.delete(7)}")
print(f"Search 7 after deletion: {sl.search(7)}")

# 显示删除后的结构
print("\nAfter deletion:")
sl.display()
```

## 总结

跳表是一种优雅的概率数据结构，它用相对简单的实现提供了与平衡树相当的性能。通过理解跳表的工作原理，我们不仅能够掌握一个实用的数据结构，还能体会到概率算法在计算机科学中的妙用。

关键要点：
1. **多级索引**：通过多级链表实现快速查找
2. **概率平衡**：使用随机算法避免复杂的旋转操作
3. **简单实现**：相比平衡树，实现复杂度大大降低
4. **广泛应用**：在Redis、LevelDB等系统中都有成功应用

跳表的设计思想告诉我们，有时候"好"的解决方案不一定是最复杂的。通过概率性和简单性的结合，我们可以创造出既实用又优雅的数据结构。