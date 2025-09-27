# Mini-LSM 实现深度解析：核心代码分析

## 概述

本文将深入分析Mini-LSM项目的核心实现代码，通过实际的代码示例来理解LSM-Tree存储引擎的各个组件是如何协同工作的。我们将重点关注代码架构、关键算法实现和性能优化技巧。

## 核心数据结构分析

### 1. 键值对结构

```rust
// mini-lsm/src/key.rs
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct KeySlice<'a> {
    pub data: &'a [u8],
    pub timestamp: u64,
}

impl<'a> KeySlice<'a> {
    pub fn new(data: &'a [u8], timestamp: u64) -> Self {
        Self { data, timestamp }
    }

    pub fn encode(self) -> Vec<u8> {
        let mut encoded = Vec::with_capacity(self.data.len() + 8);
        encoded.extend_from_slice(self.data);
        encoded.extend_from_slice(&self.timestamp.to_be_bytes());
        encoded
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ValueSlice<'a> {
    pub data: &'a [u8],
}

impl<'a> ValueSlice<'a> {
    pub fn new(data: &'a [u8]) -> Self {
        Self { data }
    }
}
```

### 2. 块结构

```rust
// mini-lsm/src/block.rs
pub struct Block {
    pub data: Vec<u8>,
    pub offsets: Vec<u16>,
}

impl Block {
    pub fn new() -> Self {
        Self {
            data: Vec::new(),
            offsets: Vec::new(),
        }
    }

    pub fn add(&mut self, key: &[u8], value: &[u8]) -> Result<()> {
        let offset = self.data.len() as u16;
        self.offsets.push(offset);

        // 编码键值对
        self.encode_entry(key, value)
    }

    fn encode_entry(&mut self, key: &[u8], value: &[u8]) -> Result<()> {
        // 键长度 (varint)
        put_varint(&mut self.data, key.len() as u32);
        // 键内容
        self.data.extend_from_slice(key);
        // 值长度 (varint)
        put_varint(&mut self.data, value.len() as u32);
        // 值内容
        self.data.extend_from_slice(value);

        Ok(())
    }

    pub fn find(&self, key: &[u8]) -> Option<&[u8]> {
        // 二分查找
        let mut left = 0;
        let mut right = self.offsets.len();

        while left < right {
            let mid = (left + right) / 2;
            let offset = self.offsets[mid] as usize;
            let (entry_key, entry_value) = self.decode_entry_at(offset);

            match entry_key.cmp(key) {
                Ordering::Equal => return Some(entry_value),
                Ordering::Less => left = mid + 1,
                Ordering::Greater => right = mid,
            }
        }

        None
    }

    fn decode_entry_at(&self, offset: usize) -> (&[u8], &[u8]) {
        let mut pos = offset;

        // 解码键长度
        let key_len = decode_varint(&self.data[pos..]) as usize;
        pos += varint_len(&self.data[pos..]);

        // 解码键内容
        let key = &self.data[pos..pos + key_len];
        pos += key_len;

        // 解码值长度
        let value_len = decode_varint(&self.data[pos..]) as usize;
        pos += varint_len(&self.data[pos..]);

        // 解码值内容
        let value = &self.data[pos..pos + value_len];

        (key, value)
    }
}
```

### 3. 内存表实现

```rust
// mini-lsm/src/mem_table.rs
pub struct MemTable {
    map: BTreeMap<KeySlice, ValueSlice>,
    approximate_size: usize,
}

impl MemTable {
    pub fn new() -> Self {
        Self {
            map: BTreeMap::new(),
            approximate_size: 0,
        }
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> Result<()> {
        let key_slice = KeySlice::new(key, 0);
        let value_slice = ValueSlice::new(value);

        // 计算新条目的大小
        let new_size = key.len() + value.len();

        // 插入到BTreeMap中
        self.map.insert(key_slice, value_slice);

        // 更新 approximate_size
        self.approximate_size += new_size;

        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> Result<Option<ValueSlice>> {
        let key_slice = KeySlice::new(key, 0);

        // 在BTreeMap中查找
        match self.map.get(&key_slice) {
            Some(value) => Ok(Some(value.clone())),
            None => Ok(None),
        }
    }

    pub fn delete(&mut self, key: &[u8]) -> Result<()> {
        let key_slice = KeySlice::new(key, 0);

        if let Some(old_value) = self.map.remove(&key_slice) {
            self.approximate_size -= key.len() + old_value.data.len();
        }

        Ok(())
    }

    pub fn scan(&self, range: (Bound<&[u8]>, Bound<&[u8]>)) -> MemTableIterator {
        let start_key = match range.0 {
            Bound::Included(key) => Bound::Included(KeySlice::new(key, 0)),
            Bound::Excluded(key) => Bound::Excluded(KeySlice::new(key, 0)),
            Bound::Unbounded => Bound::Unbounded,
        };

        let end_key = match range.1 {
            Bound::Included(key) => Bound::Included(KeySlice::new(key, 0)),
            Bound::Excluded(key) => Bound::Excluded(KeySlice::new(key, 0)),
            Bound::Unbounded => Bound::Unbounded,
        };

        MemTableIterator {
            inner: self.map.range((start_key, end_key)),
        }
    }

    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }

    pub fn approximate_size(&self) -> usize {
        self.approximate_size
    }
}

pub struct MemTableIterator<'a> {
    inner: btree_map::Range<'a, KeySlice<'a>, ValueSlice<'a>>,
}

impl<'a> Iterator for MemTableIterator<'a> {
    type Item = (KeySlice<'a>, ValueSlice<'a>);

    fn next(&mut self) -> Option<Self::Item> {
        self.inner.next().map(|(k, v)| (*k, *v))
    }
}
```

## 存储引擎核心实现

### 1. 存储引擎结构

```rust
// mini-lsm/src/lsm_storage.rs
pub struct LsmStorage {
    // 内存表
    mem_table: Arc<RwLock<MemTable>>,
    imm_mem_tables: Vec<Arc<MemTable>>,

    // SST文件
    sstables: Vec<SsTable>,
    next_sst_id: u64,

    // 配置
    options: Arc<LsmStorageOptions>,

    // 合并控制器
    compaction_controller: Arc<CompactionController>,

    // WAL
    wal: Option<Arc<Wal>>,

    // 统计信息
    stats: Arc<RwLock<StorageStats>>,
}

pub struct LsmStorageOptions {
    pub block_size: usize,
    pub target_sst_size: usize,
    pub num_mem_tables: usize,
    pub enable_wal: bool,
    pub compaction_strategy: CompactionStrategy,
}

#[derive(Debug)]
pub struct StorageStats {
    pub total_keys: u64,
    pub total_size: u64,
    pub mem_table_count: usize,
    pub sstable_count: usize,
    pub read_count: u64,
    pub write_count: u64,
    pub compaction_count: u64,
}
```

### 2. 写入操作实现

```rust
impl LsmStorage {
    pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
        // 1. 写入WAL（如果启用）
        if let Some(wal) = &self.wal {
            wal.write(key, value)?;
        }

        // 2. 写入内存表
        {
            let mut mem_table = self.mem_table.write().unwrap();
            mem_table.put(key, value)?;
        }

        // 3. 更新统计信息
        {
            let mut stats = self.stats.write().unwrap();
            stats.write_count += 1;
            stats.total_keys += 1;
            stats.total_size += key.len() + value.len();
        }

        // 4. 检查是否需要冻结内存表
        self.check_and_freeze_mem_table()?;

        Ok(())
    }

    pub fn delete(&self, key: &[u8]) -> Result<()> {
        // 1. 写入WAL（如果启用）
        if let Some(wal) = &self.wal {
            wal.write_delete(key)?;
        }

        // 2. 从内存表中删除
        {
            let mut mem_table = self.mem_table.write().unwrap();
            mem_table.delete(key)?;
        }

        // 3. 更新统计信息
        {
            let mut stats = self.stats.write().unwrap();
            stats.write_count += 1;
        }

        // 4. 检查是否需要冻结内存表
        self.check_and_freeze_mem_table()?;

        Ok(())
    }

    fn check_and_freeze_mem_table(&self) -> Result<()> {
        // 检查当前内存表是否已满
        let mem_table = self.mem_table.read().unwrap();
        let should_freeze = mem_table.approximate_size() >= self.options.target_sst_size;
        drop(mem_table);

        if should_freeze {
            self.freeze_mem_table()?;
        }

        Ok(())
    }

    fn freeze_mem_table(&self) -> Result<()> {
        // 获取当前内存表
        let mut current_mem_table = self.mem_table.write().unwrap();
        let frozen_mem_table = std::mem::replace(&mut *current_mem_table, MemTable::new());
        drop(current_mem_table);

        // 添加到不可变内存表列表
        {
            let mut imm_tables = self.imm_mem_tables.lock().unwrap();
            imm_tables.push(Arc::new(frozen_mem_table));
        }

        // 触发后台合并
        self.trigger_background_compaction()?;

        Ok(())
    }
}
```

### 3. 读取操作实现

```rust
impl LsmStorage {
    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 更新读取统计
        {
            let mut stats = self.stats.write().unwrap();
            stats.read_count += 1;
        }

        // 1. 首先在当前内存表中查找
        {
            let mem_table = self.mem_table.read().unwrap();
            if let Some(value) = mem_table.get(key)? {
                return Ok(Some(value.data.to_vec()));
            }
        }

        // 2. 在不可变内存表中查找
        {
            let imm_tables = self.imm_mem_tables.lock().unwrap();
            for imm_table in imm_tables.iter() {
                if let Some(value) = imm_table.get(key)? {
                    return Ok(Some(value.data.to_vec()));
                }
            }
        }

        // 3. 在SST文件中查找
        for sstable in &self.sstables {
            if let Some(value) = sstable.get(key)? {
                return Ok(Some(value));
            }
        }

        Ok(None)
    }

    pub fn scan(&self, range: (Bound<&[u8]>, Bound<&[u8]>)) -> Result<ScanIterator> {
        // 创建合并迭代器
        let mut iterators = Vec::new();

        // 添加内存表迭代器
        {
            let mem_table = self.mem_table.read().unwrap();
            iterators.push(Box::new(mem_table.scan(range.clone())) as Box<dyn Iterator<Item = _>>);
        }

        // 添加不可变内存表迭代器
        {
            let imm_tables = self.imm_mem_tables.lock().unwrap();
            for imm_table in imm_tables.iter() {
                iterators.push(Box::new(imm_table.scan(range.clone())) as Box<dyn Iterator<Item = _>>);
            }
        }

        // 添加SST表迭代器
        for sstable in &self.sstables {
            iterators.push(Box::new(sstable.scan(range.clone())?) as Box<dyn Iterator<Item = _>>);
        }

        // 创建合并迭代器
        let merge_iter = MergeIterator::new(iterators);

        Ok(ScanIterator { inner: merge_iter })
    }
}
```

### 4. 合并控制器实现

```rust
// mini-lsm/src/compact.rs
pub struct CompactionController {
    strategy: Box<dyn CompactionStrategy>,
    manifest: Arc<Manifest>,
    thread_pool: ThreadPool,
}

impl CompactionController {
    pub fn new(strategy: Box<dyn CompactionStrategy>, manifest: Arc<Manifest>) -> Self {
        Self {
            strategy,
            manifest,
            thread_pool: ThreadPool::new(4),
        }
    }

    pub fn schedule_compaction(&self) -> Result<()> {
        if let Some(task) = self.strategy.pick_compaction(&self.manifest) {
            self.execute_compaction(task)?;
        }
        Ok(())
    }

    fn execute_compaction(&self, task: CompactionTask) -> Result<()> {
        let manifest = self.manifest.clone();
        let strategy = self.strategy.as_ref();

        // 在线程池中执行合并
        self.thread_pool.execute(move || {
            if let Err(e) = strategy.apply_compaction(task) {
                eprintln!("Compaction failed: {}", e);
            }
        });

        Ok(())
    }
}

pub trait CompactionStrategy: Send + Sync {
    fn pick_compaction(&self, manifest: &Manifest) -> Option<CompactionTask>;
    fn apply_compaction(&self, task: CompactionTask) -> Result<()>;
}

#[derive(Debug)]
pub struct CompactionTask {
    pub input_level: usize,
    pub output_level: usize,
    pub input_files: Vec<SstFile>,
    pub output_files: Vec<SstFile>,
    pub target_size: usize,
}
```

### 5. WAL实现

```rust
// mini-lsm/src/wal.rs
pub struct Wal {
    file: File,
    path: PathBuf,
    sync_on_write: bool,
}

impl Wal {
    pub fn create(path: &Path) -> Result<Self> {
        let file = File::create(path)?;
        Ok(Self {
            file,
            path: path.to_path_buf(),
            sync_on_write: true,
        })
    }

    pub fn write(&self, key: &[u8], value: &[u8]) -> Result<()> {
        let mut entry = Vec::new();

        // 写入类型 (1 byte)
        entry.push(0x01); // PUT操作

        // 写入键长度 (varint)
        put_varint(&mut entry, key.len() as u32);

        // 写入键内容
        entry.extend_from_slice(key);

        // 写入值长度 (varint)
        put_varint(&mut entry, value.len() as u32);

        // 写入值内容
        entry.extend_from_slice(value);

        // 写入CRC校验和 (4 bytes)
        let crc = crc32c::crc32c(&entry);
        entry.extend_from_slice(&crc.to_le_bytes());

        // 写入文件
        self.file.write_all(&entry)?;

        // 同步到磁盘
        if self.sync_on_write {
            self.file.sync_all()?;
        }

        Ok(())
    }

    pub fn write_delete(&self, key: &[u8]) -> Result<()> {
        let mut entry = Vec::new();

        // 写入类型 (1 byte)
        entry.push(0x02); // DELETE操作

        // 写入键长度 (varint)
        put_varint(&mut entry, key.len() as u32);

        // 写入键内容
        entry.extend_from_slice(key);

        // 写入CRC校验和 (4 bytes)
        let crc = crc32c::crc32c(&entry);
        entry.extend_from_slice(&crc.to_le_bytes());

        // 写入文件
        self.file.write_all(&entry)?;

        // 同步到磁盘
        if self.sync_on_write {
            self.file.sync_all()?;
        }

        Ok(())
    }

    pub fn recover(&mut self) -> Result<Vec<WalEntry>> {
        let mut entries = Vec::new();
        let mut buffer = Vec::new();

        // 读取整个文件
        self.file.seek(SeekFrom::Start(0))?;
        self.file.read_to_end(&mut buffer)?;

        let mut pos = 0;
        while pos < buffer.len() {
            // 读取条目
            if let Some((entry, next_pos)) = self.decode_entry(&buffer, pos)? {
                entries.push(entry);
                pos = next_pos;
            } else {
                break;
            }
        }

        Ok(entries)
    }

    fn decode_entry(&self, buffer: &[u8], pos: usize) -> Result<Option<(WalEntry, usize)>> {
        if pos + 5 > buffer.len() {
            return Ok(None);
        }

        let entry_type = buffer[pos];
        pos += 1;

        // 读取键长度
        let key_len = decode_varint(&buffer[pos..]) as usize;
        pos += varint_len(&buffer[pos..]);

        if pos + key_len > buffer.len() {
            return Ok(None);
        }

        let key = &buffer[pos..pos + key_len];
        pos += key_len;

        let entry = match entry_type {
            0x01 => {
                // PUT操作
                let value_len = decode_varint(&buffer[pos..]) as usize;
                pos += varint_len(&buffer[pos..]);

                if pos + value_len + 4 > buffer.len() {
                    return Ok(None);
                }

                let value = &buffer[pos..pos + value_len];
                pos += value_len;

                // 读取CRC
                let crc_bytes = &buffer[pos..pos + 4];
                let expected_crc = u32::from_le_bytes(crc_bytes.try_into().unwrap());
                pos += 4;

                // 验证CRC
                let actual_crc = crc32c::crc32c(&buffer[pos - (key_len + value_len + 1)..pos - 4]);
                if actual_crc != expected_crc {
                    return Err(Error::CorruptedWal);
                }

                WalEntry::Put(key.to_vec(), value.to_vec())
            }
            0x02 => {
                // DELETE操作
                if pos + 4 > buffer.len() {
                    return Ok(None);
                }

                // 读取CRC
                let crc_bytes = &buffer[pos..pos + 4];
                let expected_crc = u32::from_le_bytes(crc_bytes.try_into().unwrap());
                pos += 4;

                // 验证CRC
                let actual_crc = crc32c::crc32c(&buffer[pos - (key_len + 1)..pos - 4]);
                if actual_crc != expected_crc {
                    return Err(Error::CorruptedWal);
                }

                WalEntry::Delete(key.to_vec())
            }
            _ => return Err(Error::InvalidWalEntry),
        };

        Ok(Some((entry, pos)))
    }
}

pub enum WalEntry {
    Put(Vec<u8>, Vec<u8>),
    Delete(Vec<u8>),
}
```

## 工具函数和辅助类

### 1. 变长整数编码

```rust
// mini-lsm/src/utils.rs
pub fn put_varint(dst: &mut Vec<u8>, mut value: u32) {
    while value >= 0x80 {
        dst.push((value & 0x7f) as u8 | 0x80);
        value >>= 7;
    }
    dst.push(value as u8);
}

pub fn decode_varint(data: &[u8]) -> u32 {
    let mut result = 0;
    let mut shift = 0;

    for &byte in data.iter().take(5) {
        result |= ((byte & 0x7f) as u32) << shift;
        shift += 7;

        if byte & 0x80 == 0 {
            break;
        }
    }

    result
}

pub fn varint_len(data: &[u8]) -> usize {
    for (i, &byte) in data.iter().enumerate().take(5) {
        if byte & 0x80 == 0 {
            return i + 1;
        }
    }
    5
}
```

### 2. 错误处理

```rust
// mini-lsm/src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Key not found")]
    KeyNotFound,

    #[error("MemTable is full")]
    MemTableFull,

    #[error("SSTable error: {0}")]
    SsTable(String),

    #[error("Compaction error: {0}")]
    Compaction(String),

    #[error("WAL corrupted")]
    CorruptedWal,

    #[error("Invalid WAL entry")]
    InvalidWalEntry,

    #[error("Manifest corrupted")]
    CorruptedManifest,

    #[error("Storage error: {0}")]
    Storage(String),
}

pub type Result<T> = std::result::Result<T, Error>;
```

### 3. 迭代器实现

```rust
// mini-lsm/src/iterators.rs
pub struct MergeIterator {
    iterators: Vec<Box<dyn Iterator<Item = (KeySlice, ValueSlice)>>>,
    heap: BinaryHeap<HeapItem>,
}

struct HeapItem {
    key: KeySlice,
    value: ValueSlice,
    iterator_index: usize,
}

impl Ord for HeapItem {
    fn cmp(&self, other: &Self) -> Ordering {
        // 比较键，如果相同则比较时间戳（新的在前）
        self.key.cmp(&other.key)
    }
}

impl PartialOrd for HeapItem {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for HeapItem {
    fn eq(&self, other: &Self) -> bool {
        self.key == other.key
    }
}

impl Eq for HeapItem {}

impl MergeIterator {
    pub fn new(iterators: Vec<Box<dyn Iterator<Item = (KeySlice, ValueSlice)>>>) -> Self {
        let mut heap = BinaryHeap::new();

        // 初始化堆
        for (i, iter) in iterators.iter().enumerate() {
            if let Some((key, value)) = iter.next() {
                heap.push(HeapItem {
                    key: key.clone(),
                    value: value.clone(),
                    iterator_index: i,
                });
            }
        }

        Self {
            iterators,
            heap,
        }
    }
}

impl Iterator for MergeIterator {
    type Item = (KeySlice, ValueSlice);

    fn next(&mut self) -> Option<Self::Item> {
        if self.heap.is_empty() {
            return None;
        }

        // 获取最小的元素
        let current = self.heap.pop().unwrap();

        // 从对应的迭代器中获取下一个元素
        if let Some((key, value)) = self.iterators[current.iterator_index].next() {
            self.heap.push(HeapItem {
                key,
                value,
                iterator_index: current.iterator_index,
            });
        }

        Some((current.key, current.value))
    }
}
```

## 使用示例

### 1. 基本使用

```rust
// 创建存储引擎
let options = LsmStorageOptions {
    block_size: 4096,
    target_sst_size: 4 * 1024 * 1024, // 4MB
    num_mem_tables: 3,
    enable_wal: true,
    compaction_strategy: CompactionStrategy::Leveled,
};

let storage = Arc::new(LsmStorage::open("/path/to/db", options)?);

// 写入数据
storage.put(b"key1", b"value1")?;
storage.put(b"key2", b"value2")?;
storage.put(b"key3", b"value3")?;

// 读取数据
if let Some(value) = storage.get(b"key1")? {
    println!("Found value: {:?}", value);
}

// 范围查询
let iter = storage.scan((Bound::Included(b"key1"), Bound::Included(b"key3")))?;
for (key, value) in iter {
    println!("Key: {:?}, Value: {:?}", key.data, value.data);
}

// 删除数据
storage.delete(b"key2")?;
```

### 2. 批量操作

```rust
// 批量写入
let batch = vec![
    (b"batch_key1".to_vec(), b"batch_value1".to_vec()),
    (b"batch_key2".to_vec(), b"batch_value2".to_vec()),
    (b"batch_key3".to_vec(), b"batch_value3".to_vec()),
];

storage.put_batch(&batch)?;

// 批量读取
let keys = vec![b"batch_key1", b"batch_key2", b"batch_key3"];
let values = storage.get_batch(&keys)?;

for (key, value) in keys.iter().zip(values.iter()) {
    if let Some(val) = value {
        println!("Key: {:?}, Value: {:?}", key, val);
    }
}
```

### 3. 监控和统计

```rust
// 获取统计信息
let stats = storage.get_stats();
println!("Total keys: {}", stats.total_keys);
println!("Total size: {} bytes", stats.total_size);
println!("Read count: {}", stats.read_count);
println!("Write count: {}", stats.write_count);

// 监控性能
let monitor = storage.create_performance_monitor();
monitor.start_monitoring();

// 执行一些操作
for i in 0..1000 {
    storage.put(&format!("key_{}", i).as_bytes(), &format!("value_{}", i).as_bytes())?;
}

// 获取性能报告
let report = monitor.get_report();
println!("Average read latency: {:?}", report.avg_read_latency);
println!("Average write latency: {:?}", report.avg_write_latency);
println!("Compaction count: {}", report.compaction_count);
```

## 总结

通过深入分析Mini-LSM的实现代码，我们可以看到：

1. **模块化设计**: 每个组件都有清晰的职责分工，便于维护和扩展
2. **性能优化**: 通过内存表、批量操作、合并策略等技术优化性能
3. **错误处理**: 完善的错误处理机制确保系统可靠性
4. **并发控制**: 合理的锁机制保证线程安全
5. **可扩展性**: 通过trait和配置参数支持不同的策略和配置

这个实现不仅展示了LSM-Tree的核心原理，也体现了现代Rust系统编程的最佳实践。通过理解这些代码，我们可以更好地掌握存储引擎的设计和实现技术。