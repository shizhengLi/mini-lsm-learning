# Mini-LSM 项目概述与架构设计

## 项目简介

Mini-LSM 是一个教育性质的项目，旨在通过一周的学习和编码，从零开始构建一个简单的键值存储引擎。该项目采用Rust语言实现，深入讲解了LSM-Tree（Log-Structured Merge-Tree）存储引擎的核心原理和实现细节。

### 项目目标

- 深入理解LSM-Tree存储引擎的工作原理
- 学习现代数据库系统的核心组件和设计模式
- 掌握Rust语言在系统编程中的应用
- 实践数据库系统的性能优化技术

### 技术栈

- **编程语言**: Rust
- **存储引擎**: 自实现的LSM-Tree
- **核心特性**: Memtable、SST、Compaction、WAL、MVCC

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     Mini-LSM Storage Engine                  │
├─────────────────────────────────────────────────────────────┤
│                      Application API                         │
├─────────────────────────────────────────────────────────────┤
│                   LsmStorage (Core Engine)                   │
├─────────────────────────────────────────────────────────────┤
│  MemTable (Active)  │  MemTable (Immutable)  │  SST Files    │
├─────────────────────────────────────────────────────────────┤
│  Write-Ahead Log (WAL) │   Block Cache   │   Bloom Filter  │
├─────────────────────────────────────────────────────────────┤
│                  File System Operations                      │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件

#### 1. LsmStorage - 存储引擎核心
- 负责协调所有存储操作
- 管理内存表和SST文件的生命周期
- 处理读写请求的路由和合并

#### 2. MemTable - 内存表
- **Active MemTable**: 当前活跃的内存表，接收新的写入操作
- **Immutable MemTable**: 不可变内存表，等待刷写到磁盘
- 基于跳表(SkipList)或有序数据结构实现，保证键的有序性

#### 3. SST (Sorted String Table) - 有序字符串表
- 磁盘上的持久化数据结构
- 分层存储，支持范围查询
- 包含数据块、索引块、布隆过滤器等

#### 4. Compaction - 合并机制
- **Simple Leveled Compaction**: 简单的分层合并策略
- **Tiered Compaction**: 分层合并策略
- **Leveled Compaction**: LevelDB风格的分层合并

#### 5. WAL (Write-Ahead Log) - 预写日志
- 保证数据持久性和崩溃恢复
- 记录所有写入操作
- 支持批量写入优化

#### 6. MVCC (Multi-Version Concurrency Control) - 多版本并发控制
- 支持事务和快照隔离
- 时间戳键编码
- 水印和垃圾回收

## 数据流程

### 写入流程

```
Client Write Request
         ↓
   Write to WAL
         ↓
   Write to Active MemTable
         ↓
   Check MemTable Size
         ↓
   [If Full] → Switch to Immutable MemTable
         ↓
   [Background] → Compact to SST
```

### 读取流程

```
Client Read Request
         ↓
   Query Active MemTable
         ↓
   Query Immutable MemTables
         ↓
   Query SST Files (L0 → Ln)
         ↓
   Merge Results from All Sources
         ↓
   Return to Client
```

### 合并流程

```
SST Size Threshold Reached
         ↓
   Select Compaction Strategy
         ↓
   Read SST Files for Compaction
         ↓
   Merge and Sort Data
         ↓
   Write New SST Files
         ↓
   Update Manifest
         ↓
   Delete Old SST Files
```

## 核心数据结构

### Key-Value 结构

```rust
pub struct KeySlice {
    pub data: &[u8],
    pub timestamp: u64,
}

pub struct ValueSlice {
    pub data: &[u8],
}
```

### Block 结构

```rust
pub struct Block {
    pub data: Vec<u8>,
    pub offsets: Vec<u16>,
}
```

### SST 结构

```
┌─────────────────────────────────────────┐
│              SST File                   │
├─────────────────────────────────────────┤
│              Block 1                    │
│              Block 2                    │
│              Block 3                    │
│              ...                        │
├─────────────────────────────────────────┤
│              Index Block                │
├─────────────────────────────────────────┤
│              Bloom Filter               │
├─────────────────────────────────────────┤
│              Meta Block                 │
└─────────────────────────────────────────┘
```

## 性能特性

### 写入优化
- **内存缓冲**: 通过MemTable实现快速写入
- **批量合并**: 通过Compaction减少写放大
- **顺序写入**: WAL和SST都采用顺序写入模式

### 读取优化
- **层次化查询**: 从内存到磁盘逐层查询
- **布隆过滤器**: 快速判断键是否存在
- **块缓存**: 缓存热点数据块
- **稀疏索引**: 减少磁盘I/O

### 并发控制
- **读写分离**: 读取操作不阻塞写入
- **MVCC**: 支持多版本并发控制
- **快照隔离**: 提供一致性的快照读取

## 学习路径

### 第一周：存储格式 + 引擎框架
1. **MemTable**: 实现内存表结构
2. **Merge Iterator**: 实现合并迭代器
3. **Block**: 实现数据块结构
4. **SST**: 实现有序字符串表
5. **Read Path**: 实现读取路径
6. **Write Path**: 实现写入路径
7. **SST优化**: 前缀键编码 + 布隆过滤器

### 第二周：合并 + 持久化
1. **Compaction实现**: 实现合并机制
2. **Simple Leveled Compaction**: 简单分层合并策略
3. **Tiered Compaction**: 分层合并策略
4. **Leveled Compaction**: LevelDB风格合并
5. **Manifest**: 元数据管理
6. **WAL**: 预写日志
7. **批量写入和校验和**: 批量优化

### 第三周：多版本并发控制
1. **时间戳键编码**: MVCC基础
2. **快照读取**: 内存表和时间戳
3. **快照读取**: 事务API
4. **水印和垃圾回收**: 生命周期管理
5. **事务和OCC**: 并发控制
6. **可序列化快照隔离**: 高级隔离级别
7. **合并过滤器**: 优化合并过程

## 总结

Mini-LSM项目通过循序渐进的方式，让学习者深入理解现代存储引擎的设计原理和实现细节。从基础的内存表到复杂的多版本并发控制，每个组件都经过精心设计，既保证了教育的完整性，又具备实用性。

通过实现这个项目，学习者将掌握：
- LSM-Tree的核心原理
- 数据库系统的设计模式
- Rust系统编程技能
- 存储引擎的性能优化技术
- 并发控制和事务处理

这个项目为深入学习现代数据库系统打下了坚实的基础。