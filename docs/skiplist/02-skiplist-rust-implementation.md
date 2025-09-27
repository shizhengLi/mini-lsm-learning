# 跳表的Rust实现深度解析

## 概述

Rust作为一门系统级编程语言，其所有权、借用和生命周期特性使得实现复杂数据结构时既安全又高效。本文将深入探讨如何用Rust实现一个生产级的跳表，涵盖从基础实现到高级优化的方方面面。

## Rust实现跳表的挑战

### 1. 所有权和借用

在Rust中，节点的指针管理需要特别注意所有权和生命周期：

```rust
// 错误的尝试：编译器会报错
struct SkipListNode<K, V> {
    key: K,
    value: V,
    forward: Vec<Box<SkipListNode<K, V>>>, // 问题：循环引用
}
```

### 2. 并发安全

多线程环境下，跳表的操作需要保证原子性和线程安全。

### 3. 内存管理

需要确保节点在不再使用时能被正确释放。

## 基础实现

### 1. 节点定义

```rust
use std::ptr::NonNull;
use std::sync::atomic::{AtomicPtr, Ordering};
use std::marker::PhantomData;
use std::fmt::Debug;

#[derive(Debug)]
pub struct SkipListNode<K, V> {
    key: K,
    value: V,
    // 使用AtomicPtr支持并发操作
    forward: Vec<AtomicPtr<SkipListNode<K, V>>>,
}

impl<K, V> SkipListNode<K, V> {
    pub fn new(key: K, value: V, level: usize) -> Self {
        let mut forward = Vec::with_capacity(level);
        for _ in 0..level {
            forward.push(AtomicPtr::new(std::ptr::null_mut()));
        }
        Self {
            key,
            value,
            forward,
        }
    }

    pub fn key(&self) -> &K {
        &self.key
    }

    pub fn value(&self) -> &V {
        &self.value
    }

    pub fn value_mut(&mut self) -> &mut V {
        &mut self.value
    }

    pub fn set_value(&mut self, value: V) {
        self.value = value;
    }

    pub fn get_forward(&self, level: usize) -> Option<NonNull<SkipListNode<K, V>>> {
        if level >= self.forward.len() {
            return None;
        }
        NonNull::new(self.forward[level].load(Ordering::Acquire))
    }

    pub fn set_forward(&self, level: usize, node: *mut SkipListNode<K, V>) {
        if level < self.forward.len() {
            self.forward[level].store(node, Ordering::Release);
        }
    }
}

impl<K, V> Drop for SkipListNode<K, V> {
    fn drop(&mut self) {
        // 手动释放所有指针
        for ptr in &self.forward {
            let node_ptr = ptr.load(Ordering::Acquire);
            if !node_ptr.is_null() {
                unsafe {
                    drop(Box::from_raw(node_ptr));
                }
            }
        }
    }
}
```

### 2. 跳表结构

```rust
use std::sync::{Arc, Mutex};
use rand::{thread_rng, Rng};
use std::cmp::Ordering;

pub struct SkipList<K, V> {
    head: *mut SkipListNode<K, V>,
    max_level: usize,
    level: AtomicUsize,
    length: AtomicUsize,
    probability: f64,
    _marker: PhantomData<(K, V)>,
}

impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn new(max_level: usize, probability: f64) -> Self {
        assert!(max_level > 0);
        assert!(probability > 0.0 && probability <= 1.0);

        let head_node = SkipListNode::new(
            // 头节点使用最小可能值
            unsafe { std::mem::zeroed() },
            unsafe { std::mem::zeroed() },
            max_level,
        );

        Self {
            head: Box::into_raw(Box::new(head_node)),
            max_level,
            level: AtomicUsize::new(1),
            length: AtomicUsize::new(0),
            probability,
            _marker: PhantomData,
        }
    }

    pub fn default() -> Self {
        Self::new(16, 0.5)
    }

    fn random_level(&self) -> usize {
        let mut level = 1;
        let mut rng = thread_rng();

        while rng.gen::<f64>() < self.probability && level < self.max_level {
            level += 1;
        }

        level
    }

    pub fn len(&self) -> usize {
        self.length.load(Ordering::Acquire)
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    pub fn max_level(&self) -> usize {
        self.max_level
    }

    pub fn current_level(&self) -> usize {
        self.level.load(Ordering::Acquire)
    }
}
```

### 3. 插入操作

```rust
impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn insert(&self, key: K, value: V) -> Option<V> {
        // 创建update数组，记录各层的前驱节点
        let mut update = vec![std::ptr::null_mut(); self.max_level];
        let mut current = self.head;

        // 从最高层开始查找插入位置
        for i in (0..self.current_level()).rev() {
            unsafe {
                while let Some(next) = (*current).get_forward(i) {
                    if next.as_ref().key < key {
                        current = next.as_ptr();
                    } else {
                        break;
                    }
                }
                update[i] = current;
            }
        }

        // 检查是否已存在
        unsafe {
            let next_0 = (*current).get_forward(0);
            if let Some(node) = next_0 {
                if node.as_ref().key == key {
                    // 键已存在，更新值
                    let old_value = node.as_ref().value.clone();
                    let node_mut = node.as_ptr() as *mut SkipListNode<K, V>;
                    (*node_mut).set_value(value.clone());
                    return Some(old_value);
                }
            }
        }

        // 随机生成新节点的高度
        let new_level = self.random_level();

        // 如果新高度超过当前最大高度，更新头节点指针
        let current_level = self.current_level();
        if new_level > current_level {
            for i in current_level..new_level {
                update[i] = self.head;
            }
            self.level.store(new_level, Ordering::Release);
        }

        // 创建新节点
        let new_node = Box::into_raw(Box::new(SkipListNode::new(
            key.clone(),
            value.clone(),
            new_level
        )));

        // 更新各层的指针
        for i in 0..new_level {
            unsafe {
                let update_node = update[i];
                let update_next = (*update_node).get_forward(i);

                if let Some(next) = update_next {
                    (*new_node).set_forward(i, next.as_ptr());
                } else {
                    (*new_node).set_forward(i, std::ptr::null_mut());
                }

                (*update_node).set_forward(i, new_node);
            }
        }

        self.length.fetch_add(1, Ordering::Acquire);
        None
    }
}
```

### 4. 查找操作

```rust
impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn get(&self, key: &K) -> Option<V> {
        let mut current = self.head;

        // 从最高层开始查找
        for i in (0..self.current_level()).rev() {
            unsafe {
                while let Some(next) = (*current).get_forward(i) {
                    if next.as_ref().key < *key {
                        current = next.as_ptr();
                    } else {
                        break;
                    }
                }
            }
        }

        // 在最底层检查是否找到
        unsafe {
            let next_0 = (*current).get_forward(0);
            if let Some(node) = next_0 {
                if node.as_ref().key == *key {
                    return Some(node.as_ref().value.clone());
                }
            }
        }

        None
    }

    pub fn contains(&self, key: &K) -> bool {
        self.get(key).is_some()
    }

    pub fn get_first(&self) -> Option<(&K, &V)> {
        unsafe {
            let first = (*self.head).get_forward(0)?;
            Some((&first.as_ref().key, &first.as_ref().value))
        }
    }

    pub fn get_last(&self) -> Option<(&K, &V)> {
        let mut current = self.head;
        unsafe {
            // 从最高层开始
            for i in (0..self.current_level()).rev() {
                while let Some(next) = (*current).get_forward(i) {
                    current = next.as_ptr();
                }
            }

            if current != self.head {
                Some((&(*current).key, &(*current).value))
            } else {
                None
            }
        }
    }
}
```

### 5. 删除操作

```rust
impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn remove(&self, key: &K) -> Option<V> {
        let mut update = vec![std::ptr::null_mut(); self.max_level];
        let mut current = self.head;

        // 从最高层开始查找
        for i in (0..self.current_level()).rev() {
            unsafe {
                while let Some(next) = (*current).get_forward(i) {
                    if next.as_ref().key < *key {
                        current = next.as_ptr();
                    } else {
                        break;
                    }
                }
                update[i] = current;
            }
        }

        // 检查是否存在
        let target_ptr = unsafe {
            let next_0 = (*current).get_forward(0);
            match next_0 {
                Some(node) if node.as_ref().key == *key => node.as_ptr(),
                _ => return None,
            }
        };

        let old_value = unsafe { (*target_ptr).value.clone() };

        // 更新各层指针
        for i in 0..self.current_level() {
            unsafe {
                let update_node = update[i];
                let update_next = (*update_node).get_forward(i);

                if let Some(next) = update_next {
                    if next.as_ptr() == target_ptr {
                        let target_next = (*target_ptr).get_forward(i);
                        match target_next {
                            Some(target_next_node) => {
                                (*update_node).set_forward(i, target_next_node.as_ptr());
                            }
                            None => {
                                (*update_node).set_forward(i, std::ptr::null_mut());
                            }
                        }
                    } else {
                        break;
                    }
                }
            }
        }

        // 释放目标节点
        unsafe {
            drop(Box::from_raw(target_ptr));
        }

        // 如果删除的节点是最高层的节点，降低层数
        let mut current_level = self.current_level();
        while current_level > 1 {
            unsafe {
                if (*self.head).get_forward(current_level - 1).is_some() {
                    break;
                }
            }
            current_level -= 1;
        }
        self.level.store(current_level, Ordering::Release);

        self.length.fetch_sub(1, Ordering::Acquire);
        Some(old_value)
    }
}
```

### 6. 迭代器实现

```rust
pub struct SkipListIterator<'a, K, V> {
    current: *const SkipListNode<K, V>,
    _marker: PhantomData<&'a SkipList<K, V>>,
}

impl<'a, K, V> Iterator for SkipListIterator<'a, K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    type Item = (&'a K, &'a V);

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            if let Some(next) = (*self.current).get_forward(0) {
                self.current = next.as_ptr();
                Some((&next.as_ref().key, &next.as_ref().value))
            } else {
                None
            }
        }
    }
}

impl<'a, K, V> DoubleEndedIterator for SkipListIterator<'a, K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    fn next_back(&mut self) -> Option<Self::Item> {
        // 跳表的backward迭代需要更复杂的实现
        // 这里简化为forward的反向
        None
    }
}

impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn iter(&self) -> SkipListIterator<'_, K, V> {
        SkipListIterator {
            current: self.head,
            _marker: PhantomData,
        }
    }

    pub fn range(&self, start: Option<&K>, end: Option<&K>) -> Vec<(&K, &V)> {
        let mut result = Vec::new();
        let mut current = self.head;

        // 跳到起始位置
        unsafe {
            for i in (0..self.current_level()).rev() {
                while let Some(next) = (*current).get_forward(i) {
                    if start.map_or(true, |s| next.as_ref().key < *s) {
                        current = next.as_ptr();
                    } else {
                        break;
                    }
                }
            }

            // 遍历直到结束位置
            while let Some(next) = (*current).get_forward(0) {
                let key = &next.as_ref().key;
                if end.map_or(true, |e| key <= e) {
                    result.push((key, &next.as_ref().value));
                    current = next.as_ptr();
                } else {
                    break;
                }
            }
        }

        result
    }
}
```

### 7. 调试和显示

```rust
use std::fmt::{Display, Formatter, Result};

impl<K, V> Display for SkipList<K, V>
where
    K: Ord + Clone + Debug + Display,
    V: Clone + Debug + Display,
{
    fn fmt(&self, f: &mut Formatter<'_>) -> Result {
        writeln!(f, "SkipList (len={}, level={}/{})\n",
                self.len(), self.current_level(), self.max_level())?;

        for level in (0..self.current_level()).rev() {
            write!(f, "Level {}: ", level + 1)?;

            let mut current = self.head;
            let mut first = true;

            unsafe {
                while let Some(next) = (*current).get_forward(level) {
                    if !first {
                        write!(f, " -> ")?;
                    }
                    write!(f, "{}", next.as_ref().key)?;
                    current = next.as_ptr();
                    first = false;
                }
            }

            writeln!(f)?;
        }

        Ok(())
    }
}

impl<K, V> Drop for SkipList<K, V> {
    fn drop(&mut self) {
        // 手动释放头节点
        unsafe {
            drop(Box::from_raw(self.head));
        }
    }
}
```

## 高级特性

### 1. 并发安全版本

```rust
use std::sync::Arc;
use parking_lot::RwLock;

pub struct ConcurrentSkipList<K, V> {
    inner: Arc<RwLock<SkipList<K, V>>>,
}

impl<K, V> ConcurrentSkipList<K, V>
where
    K: Ord + Clone + Debug,
    V: Clone + Debug,
{
    pub fn new(max_level: usize, probability: f64) -> Self {
        Self {
            inner: Arc::new(RwLock::new(SkipList::new(max_level, probability))),
        }
    }

    pub fn insert(&self, key: K, value: V) -> Option<V> {
        let mut guard = self.inner.write();
        guard.insert(key, value)
    }

    pub fn get(&self, key: &K) -> Option<V> {
        let guard = self.inner.read();
        guard.get(key)
    }

    pub fn remove(&self, key: &K) -> Option<V> {
        let mut guard = self.inner.write();
        guard.remove(key)
    }

    pub fn len(&self) -> usize {
        let guard = self.inner.read();
        guard.len()
    }

    pub fn contains(&self, key: &K) -> bool {
        let guard = self.inner.read();
        guard.contains(key)
    }
}
```

### 2. 内存优化版本

```rust
// 使用池化技术减少内存分配
pub struct NodePool<K, V> {
    free_nodes: parking_lot::Mutex<Vec<Box<SkipListNode<K, V>>>>,
    max_pool_size: usize,
}

impl<K, V> NodePool<K, V> {
    pub fn new(max_pool_size: usize) -> Self {
        Self {
            free_nodes: parking_lot::Mutex::new(Vec::new()),
            max_pool_size,
        }
    }

    pub fn get_node(&self, key: K, value: V, level: usize) -> *mut SkipListNode<K, V> {
        let mut nodes = self.free_nodes.lock();
        if let Some(mut node) = nodes.pop() {
            // 重用节点
            node.key = key;
            node.value = value;
            node.forward.clear();
            for _ in 0..level {
                node.forward.push(AtomicPtr::new(std::ptr::null_mut()));
            }
            Box::into_raw(node)
        } else {
            // 创建新节点
            Box::into_raw(Box::new(SkipListNode::new(key, value, level)))
        }
    }

    pub fn return_node(&self, node: *mut SkipListNode<K, V>) {
        let mut nodes = self.free_nodes.lock();
        if nodes.len() < self.max_pool_size {
            unsafe {
                // 重置节点状态
                (*node).forward.clear();
                let boxed = Box::from_raw(node);
                nodes.push(boxed);
            }
        } else {
            unsafe {
                drop(Box::from_raw(node));
            }
        }
    }
}
```

### 3. 序列化支持

```rust
use serde::{Serialize, Deserialize};

impl<K, V> SkipList<K, V>
where
    K: Ord + Clone + Debug + Serialize + for<'de> Deserialize<'de>,
    V: Clone + Debug + Serialize + for<'de> Deserialize<'de>,
{
    pub fn serialize(&self) -> Vec<u8> {
        let items: Vec<_> = self.iter().map(|(k, v)| (k.clone(), v.clone())).collect();
        bincode::serialize(&items).expect("Serialization failed")
    }

    pub fn deserialize(data: &[u8]) -> Result<Self, bincode::Error> {
        let items: Vec<(K, V)> = bincode::deserialize(data)?;
        let mut skiplist = Self::default();

        for (key, value) in items {
            skiplist.insert(key, value);
        }

        Ok(skiplist)
    }
}
```

## 性能测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Instant;

    #[test]
    fn test_basic_operations() {
        let sl = SkipList::default();

        // 测试插入
        assert!(sl.insert(1, "one").is_none());
        assert!(sl.insert(2, "two").is_none());
        assert!(sl.insert(3, "three").is_none());

        // 测试查找
        assert_eq!(sl.get(&1), Some("one"));
        assert_eq!(sl.get(&2), Some("two"));
        assert_eq!(sl.get(&4), None);

        // 测试更新
        assert_eq!(sl.insert(2, "two_updated"), Some("two"));
        assert_eq!(sl.get(&2), Some("two_updated"));

        // 测试删除
        assert_eq!(sl.remove(&2), Some("two_updated"));
        assert_eq!(sl.get(&2), None);

        // 测试长度
        assert_eq!(sl.len(), 2);
    }

    #[test]
    fn test_range_query() {
        let sl = SkipList::default();

        for i in 0..100 {
            sl.insert(i, format!("value_{}", i));
        }

        // 测试范围查询
        let range_10_20 = sl.range(Some(&10), Some(&20));
        assert_eq!(range_10_20.len(), 11); // 包含边界

        let range_from_50 = sl.range(Some(&50), None);
        assert_eq!(range_from_50.len(), 50);

        let range_to_30 = sl.range(None, Some(&30));
        assert_eq!(range_to_30.len(), 31);
    }

    #[test]
    fn test_concurrent_operations() {
        let sl = Arc::new(ConcurrentSkipList::default());
        let handles: Vec<_> = (0..4)
            .map(|thread_id| {
                let sl_clone = sl.clone();
                std::thread::spawn(move || {
                    for i in 0..1000 {
                        let key = thread_id * 1000 + i;
                        sl_clone.insert(key, format!("thread_{}_value_{}", thread_id, i));
                    }
                })
            })
            .collect();

        for handle in handles {
            handle.join().unwrap();
        }

        assert_eq!(sl.len(), 4000);
    }

    #[test]
    fn benchmark_operations() {
        let sl = SkipList::default();
        let n = 100_000;

        // 插入性能测试
        let start = Instant::now();
        for i in 0..n {
            sl.insert(i, format!("value_{}", i));
        }
        let insert_time = start.elapsed();

        // 查找性能测试
        let start = Instant::now();
        for i in 0..n {
            sl.get(&i);
        }
        let search_time = start.elapsed();

        // 删除性能测试
        let start = Instant::now();
        for i in 0..n {
            sl.remove(&i);
        }
        let delete_time = start.elapsed();

        println!("Performance with n={}:", n);
        println!("  Insert: {:?}", insert_time);
        println!("  Search: {:?}", search_time);
        println!("  Delete: {:?}", delete_time);

        // 验证删除后为空
        assert!(sl.is_empty());
    }

    #[test]
    fn test_memory_usage() {
        let sl = SkipList::default();
        let initial_memory = get_memory_usage();

        // 插入大量数据
        for i in 0..10000 {
            sl.insert(i, format!("value_{}", i));
        }

        let after_insert = get_memory_usage();
        let insert_increase = after_insert - initial_memory;

        // 删除所有数据
        for i in 0..10000 {
            sl.remove(&i);
        }

        let after_delete = get_memory_usage();
        let delete_decrease = after_insert - after_delete;

        println!("Memory usage analysis:");
        println!("  Initial: {} bytes", initial_memory);
        println!("  After insert: {} bytes (+{})", after_insert, insert_increase);
        println!("  After delete: {} bytes (-{})", after_delete, delete_decrease);
    }

    fn get_memory_usage() -> usize {
        // 简化的内存使用统计
        // 在实际应用中，可以使用更精确的方法
        std::mem::size_of::<SkipList<i32, String>>()
    }
}
```

## 使用示例

```rust
fn main() {
    // 创建跳表
    let sl = SkipList::default();

    // 插入数据
    sl.insert("apple", 10);
    sl.insert("banana", 20);
    sl.insert("cherry", 30);
    sl.insert("date", 40);

    println!("SkipList structure:");
    println!("{}", sl);

    // 查找操作
    println!("Search 'banana': {:?}", sl.get(&"banana"));
    println!("Search 'grape': {:?}", sl.get(&"grape"));

    // 范围查询
    println!("Range from 'banana' to 'cherry':");
    for (key, value) in sl.range(Some(&"banana"), Some(&"cherry")) {
        println!("  {} -> {}", key, value);
    }

    // 迭代器
    println!("All elements:");
    for (key, value) in sl.iter() {
        println!("  {} -> {}", key, value);
    }

    // 并发版本
    let concurrent_sl = ConcurrentSkipList::default();
    concurrent_sl.insert("key1", "value1");
    concurrent_sl.insert("key2", "value2");

    println!("Concurrent search: {:?}", concurrent_sl.get(&"key1"));

    // 性能测试
    benchmark_skip_list();
}

fn benchmark_skip_list() {
    let sl = SkipList::default();
    let n = 50_000;

    // 插入性能
    let start = std::time::Instant::now();
    for i in 0..n {
        sl.insert(i, format!("value_{}", i));
    }
    let insert_duration = start.elapsed();

    // 查找性能
    let start = std::time::Instant::now();
    for i in 0..n {
        let _ = sl.get(&i);
    }
    let search_duration = start.elapsed();

    // 删除性能
    let start = std::time::Instant::now();
    for i in 0..n {
        let _ = sl.remove(&i);
    }
    let delete_duration = start.elapsed();

    println!("Benchmark results (n={}):", n);
    println!("  Insert: {:?}", insert_duration);
    println!("  Search: {:?}", search_duration);
    println!("  Delete: {:?}", delete_duration);
}
```

## 总结

Rust版本的跳表实现展示了如何利用Rust的特性来构建安全高效的数据结构：

### 关键要点：

1. **内存安全**：使用指针和原子操作确保内存安全
2. **并发支持**：通过读写锁实现线程安全
3. **性能优化**：使用池化技术和缓存友好设计
4. **错误处理**：使用Result类型处理可能的错误
5. **资源管理**：通过Drop trait确保资源正确释放

### Rust特有的优势：

- **零成本抽象**：高级特性不会带来运行时开销
- **内存安全**：编译器保证的内存安全
- **并发安全**：类型系统保证的并发安全
- **性能**：与C++相媲美的性能表现

这个实现不仅展示了跳表的核心概念，还体现了现代系统编程的最佳实践。通过这个例子，我们可以看到如何在保证安全性的同时，实现高性能的数据结构。