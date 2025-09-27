# SST (Sorted String Table) 文件格式深度解析

## 概述

SST（Sorted String Table）是LSM-Tree存储引擎中的核心持久化数据结构。它将内存中的数据有序地持久化到磁盘上，支持高效的读写操作。本文将深入探讨SST文件格式的结构设计、编码方式和优化策略。

## SST文件结构

### 整体布局

```
┌─────────────────────────────────────────────────────────────────┐
│                        SST File                                │
├─────────────────────────────────────────────────────────────────┤
│                          Data Blocks                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │   Block 1   │ │   Block 2   │ │   Block 3   │ │   ...    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                         Index Block                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │   Entry 1   │ │   Entry 2   │ │   Entry 3   │ │   ...    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                        Bloom Filter                            │
├─────────────────────────────────────────────────────────────────┤
│                          Meta Block                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │   Magic     │ │   Version   │ │  Block Size │ │ Options  │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                         Meta Offset                            │
└─────────────────────────────────────────────────────────────────┘
```

### 详细结构说明

#### 1. Data Blocks

```
┌─────────────────────────────────────────────────────────────────┐
│                        Data Block                              │
├─────────────────────────────────────────────────────────────────┤
│                          Entries                               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ │
│  │ Key 1   │ │ Value 1 │ │ Key 2   │ │ Value 2 │ │   ...     │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                       Restart Points                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ │
│  │ Offset1 │ │ Offset2 │ │ Offset3 │ │ Offset4 │ │   ...     │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                        Block Metadata                          │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. Index Block

```
┌─────────────────────────────────────────────────────────────────┐
│                        Index Block Entry                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────────────────────────────┐   │
│  │  Key   │ │ Offset  │ │           Size                 │   │
│  └─────────┘ └─────────┘ └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. Bloom Filter

```
┌─────────────────────────────────────────────────────────────────┐
│                       Bloom Filter Block                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ │
│  │  Bit 1  │ │  Bit 2  │ │  Bit 3  │ │  Bit 4  │ │   ...     │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                     Filter Metadata                            │
└─────────────────────────────────────────────────────────────────┘
```

## 键值编码方案

### 前缀压缩

```rust
pub fn encode_key_entry(prev_key: &[u8], current_key: &[u8]) -> Vec<u8> {
    let shared = common_prefix_length(prev_key, current_key);
    let unshared = &current_key[shared..];

    let mut encoded = Vec::new();

    // 共享长度 (Varint32)
    put_varint(&mut encoded, shared as u32);

    // 非共享长度 (Varint32)
    put_varint(&mut encoded, unshared.len() as u32);

    // 非共享部分
    encoded.extend_from_slice(unshared);

    encoded
}

fn common_prefix_length(a: &[u8], b: &[u8]) -> usize {
    a.iter()
        .zip(b.iter())
        .take_while(|(x, y)| x == y)
        .count()
}

fn put_varint(dst: &mut Vec<u8>, mut value: u32) {
    while value >= 0x80 {
        dst.push((value & 0x7f) as u8 | 0x80);
        value >>= 7;
    }
    dst.push(value as u8);
}
```

### 值编码

```rust
pub fn encode_value_entry(value: &[u8]) -> Vec<u8> {
    let mut encoded = Vec::with_capacity(value.len() + 1);

    // 值类型 (1 byte)
    encoded.push(0x01); // 0x01 = Full value

    // 值长度 (Varint32)
    put_varint(&mut encoded, value.len() as u32);

    // 值内容
    encoded.extend_from_slice(value);

    encoded
}
```

### 完整条目编码

```rust
pub struct BlockEntry {
    pub key: Vec<u8>,
    pub value: Vec<u8>,
    pub seq_num: u64,
    pub value_type: ValueType,
}

impl BlockEntry {
    pub fn encode(&self, prev_key: &[u8]) -> Vec<u8> {
        let mut encoded = Vec::new();

        // 键前缀压缩
        let key_shared = common_prefix_length(prev_key, &self.key);
        let key_unshared = &self.key[key_shared..];

        // 共享键长度
        put_varint(&mut encoded, key_shared as u32);

        // 非共享键长度
        put_varint(&mut encoded, key_unshared.len() as u32);

        // 非共享键内容
        encoded.extend_from_slice(key_unshared);

        // 值长度
        put_varint(&mut encoded, self.value.len() as u32);

        // 值内容
        encoded.extend_from_slice(&self.value);

        encoded
    }
}
```

## Block组织结构

### Block构建器

```rust
pub struct BlockBuilder {
    data: Vec<u8>,
    offsets: Vec<u16>,
    last_key: Vec<u8>,
    restart_interval: usize,
    entry_count: usize,
}

impl BlockBuilder {
    pub fn new(restart_interval: usize) -> Self {
        Self {
            data: Vec::new(),
            offsets: Vec::new(),
            last_key: Vec::new(),
            restart_interval,
            entry_count: 0,
        }
    }

    pub fn add(&mut self, key: &[u8], value: &[u8]) -> Result<()> {
        let start_offset = self.data.len();

        // 检查是否需要重启点
        if self.entry_count % self.restart_interval == 0 {
            // 添加重启点偏移量
            self.offsets.push(start_offset as u16);
            // 重置 last_key
            self.last_key.clear();
        }

        // 编码键值对
        if self.last_key.is_empty() {
            // 第一个条目，不压缩
            self.encode_first_entry(key, value);
        } else {
            // 前缀压缩
            self.encode_compressed_entry(key, value);
        }

        self.last_key.clear();
        self.last_key.extend_from_slice(key);
        self.entry_count += 1;

        Ok(())
    }

    fn encode_first_entry(&mut self, key: &[u8], value: &[u8]) {
        // 共享长度 = 0
        put_varint(&mut self.data, 0);

        // 键长度
        put_varint(&mut self.data, key.len() as u32);

        // 键内容
        self.data.extend_from_slice(key);

        // 值长度
        put_varint(&mut self.data, value.len() as u32);

        // 值内容
        self.data.extend_from_slice(value);
    }

    fn encode_compressed_entry(&mut self, key: &[u8], value: &[u8]) {
        let shared = common_prefix_length(&self.last_key, key);
        let unshared = &key[shared..];

        // 共享长度
        put_varint(&mut self.data, shared as u32);

        // 非共享键长度
        put_varint(&mut self.data, unshared.len() as u32);

        // 非共享键内容
        self.data.extend_from_slice(unshared);

        // 值长度
        put_varint(&mut self.data, value.len() as u32);

        // 值内容
        self.data.extend_from_slice(value);
    }

    pub fn build(self) -> Block {
        // 添加重启点数量
        let restart_count = self.offsets.len();

        let mut block_data = self.data;

        // 添加重启点偏移量
        for offset in &self.offsets {
            block_data.extend_from_slice(&offset.to_le_bytes());
        }

        // 添加重启点数量
        block_data.extend_from_slice(&(restart_count as u32).to_le_bytes());

        Block {
            data: block_data,
            offsets: self.offsets,
        }
    }
}
```

### Block迭代器

```rust
pub struct BlockIterator<'a> {
    data: &'a [u8],
    restarts: &'a [u8],
    current_offset: usize,
    last_key: Vec<u8>,
}

impl<'a> BlockIterator<'a> {
    pub fn new(block: &'a Block) -> Self {
        let restarts_offset = block.data.len() - (block.offsets.len() + 1) * 4;
        let restarts = &block.data[restarts_offset..];

        Self {
            data: &block.data[..restarts_offset],
            restarts,
            current_offset: 0,
            last_key: Vec::new(),
        }
    }

    pub fn seek_to_first(&mut self) {
        self.current_offset = 0;
        self.decode_next();
    }

    pub fn seek_to_key(&mut self, key: &[u8]) {
        // 使用重启点进行二分查找
        let restart_count = self.get_restart_count();

        let mut left = 0;
        let mut right = restart_count;

        while left < right {
            let mid = (left + right) / 2;
            let restart_offset = self.get_restart_offset(mid);

            if let Some((restart_key, _)) = self.decode_entry_at(restart_offset) {
                if restart_key.as_slice() < key {
                    left = mid + 1;
                } else {
                    right = mid;
                }
            }
        }

        // 线性搜索
        if left > 0 {
            self.current_offset = self.get_restart_offset(left - 1);
        } else {
            self.current_offset = 0;
        }

        while let Some((entry_key, entry_value)) = self.decode_next() {
            if entry_key.as_slice() >= key {
                break;
            }
        }
    }

    fn decode_next(&mut self) -> Option<(Vec<u8>, Vec<u8>)> {
        if self.current_offset >= self.data.len() {
            return None;
        }

        let (key, value, next_offset) = self.decode_entry_at(self.current_offset)?;
        self.current_offset = next_offset;
        self.last_key = key.clone();

        Some((key, value))
    }

    fn decode_entry_at(&self, offset: usize) -> Option<(Vec<u8>, Vec<u8>, usize)> {
        let mut pos = offset;

        // 解码共享长度
        let shared = decode_varint(&self.data[pos..])? as usize;
        pos += varint_len(&self.data[pos..]);

        // 解码非共享键长度
        let unshared_len = decode_varint(&self.data[pos..])? as usize;
        pos += varint_len(&self.data[pos..]);

        // 解码非共享键内容
        let key_start = pos;
        let key_end = pos + unshared_len;
        if key_end > self.data.len() {
            return None;
        }
        let unshared_key = &self.data[key_start..key_end];
        pos = key_end;

        // 解码值长度
        let value_len = decode_varint(&self.data[pos..])? as usize;
        pos += varint_len(&self.data[pos..]);

        // 解码值内容
        let value_start = pos;
        let value_end = pos + value_len;
        if value_end > self.data.len() {
            return None;
        }
        let value = &self.data[value_start..value_end];
        pos = value_end;

        // 重建完整键
        let mut full_key = if shared > 0 {
            self.last_key[..shared].to_vec()
        } else {
            Vec::new()
        };
        full_key.extend_from_slice(unshared_key);

        Some((full_key, value.to_vec(), pos))
    }
}
```

## SST构建过程

### SST构建器

```rust
pub struct SstBuilder {
    file: File,
    block_builder: BlockBuilder,
    data_blocks: Vec<(Vec<u8>, usize)>, // (first_key, offset)
    block_size: usize,
    estimated_size: usize,
}

impl SstBuilder {
    pub fn new(file: File, block_size: usize) -> Self {
        Self {
            file,
            block_builder: BlockBuilder::new(16),
            data_blocks: Vec::new(),
            block_size,
            estimated_size: 0,
        }
    }

    pub fn add(&mut self, key: &[u8], value: &[u8]) -> Result<()> {
        // 检查当前block是否已满
        if self.block_builder.estimated_size() >= self.block_size {
            self.finish_block()?;
        }

        self.block_builder.add(key, value)?;
        Ok(())
    }

    fn finish_block(&mut self) -> Result<()> {
        if self.block_builder.entry_count() == 0 {
            return Ok(());
        }

        // 构建block
        let block = self.block_builder.build();
        let block_data = block.data;

        // 获取第一个键作为索引
        let first_key = self.block_builder.first_key().to_vec();

        // 写入文件
        let offset = self.file.metadata()?.len() as usize;
        self.file.write_all(&block_data)?;

        // 记录block信息
        self.data_blocks.push((first_key, offset));
        self.estimated_size += block_data.len();

        // 重置block builder
        self.block_builder = BlockBuilder::new(16);

        Ok(())
    }

    pub fn finish(mut self) -> Result<()> {
        // 完成最后一个block
        self.finish_block()?;

        // 构建并写入index block
        let index_block = self.build_index_block()?;
        let index_offset = self.file.metadata()?.len() as usize;
        self.file.write_all(&index_block)?;

        // 构建并写入bloom filter
        let bloom_filter = self.build_bloom_filter()?;
        let bloom_offset = self.file.metadata()?.len() as usize;
        self.file.write_all(&bloom_filter)?;

        // 构建并写入meta block
        let meta_block = self.build_meta_block(index_offset, bloom_offset)?;
        let meta_offset = self.file.metadata()?.len() as usize;
        self.file.write_all(&meta_block)?;

        // 写入meta offset
        self.file.write_all(&(meta_offset as u64).to_le_bytes())?;

        Ok(())
    }

    fn build_index_block(&self) -> Result<Vec<u8>> {
        let mut index_builder = BlockBuilder::new(1);

        for (first_key, offset) in &self.data_blocks {
            index_builder.add(first_key, &offset.to_le_bytes())?;
        }

        Ok(index_builder.build().data)
    }

    fn build_bloom_filter(&self) -> Result<Vec<u8>> {
        let mut filter = BloomFilter::new(self.data_blocks.len());

        for (first_key, _) in &self.data_blocks {
            filter.add(first_key);
        }

        Ok(filter.encode())
    }

    fn build_meta_block(&self, index_offset: usize, bloom_offset: usize) -> Result<Vec<u8>> {
        let mut meta = Vec::new();

        // Magic number
        meta.extend_from_slice(b"MINI\0\0\0\0");

        // Version
        meta.push(1);

        // Index block offset
        meta.extend_from_slice(&(index_offset as u64).to_le_bytes());

        // Bloom filter offset
        meta.extend_from_slice(&(bloom_offset as u64).to_le_bytes());

        // Block count
        meta.extend_from_slice(&(self.data_blocks.len() as u32).to_le_bytes());

        meta
    }
}
```

## SST读取过程

### SST读取器

```rust
pub struct SstReader {
    file: File,
    index_block: Block,
    bloom_filter: BloomFilter,
    meta: SstMeta,
}

impl SstReader {
    pub fn open(file: File) -> Result<Self> {
        let metadata = file.metadata()?;
        let file_size = metadata.len() as usize;

        // 读取meta offset
        file.seek(SeekFrom::End(-8))?;
        let mut meta_offset_bytes = [0u8; 8];
        file.read_exact(&mut meta_offset_bytes)?;
        let meta_offset = u64::from_le_bytes(meta_offset_bytes) as usize;

        // 读取meta block
        file.seek(SeekFrom::Start(meta_offset as u64))?;
        let mut meta_data = Vec::new();
        file.read_to_end(&mut meta_data)?;
        let meta = SstMeta::decode(&meta_data)?;

        // 读取index block
        file.seek(SeekFrom::Start(meta.index_offset as u64))?;
        let mut index_data = vec![0u8; meta.bloom_offset - meta.index_offset];
        file.read_exact(&mut index_data)?;
        let index_block = Block::decode(&index_data)?;

        // 读取bloom filter
        file.seek(SeekFrom::Start(meta.bloom_offset as u64))?;
        let mut bloom_data = vec![0u8; meta_offset - meta.bloom_offset];
        file.read_exact(&mut bloom_data)?;
        let bloom_filter = BloomFilter::decode(&bloom_data)?;

        Ok(Self {
            file,
            index_block,
            bloom_filter,
            meta,
        })
    }

    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 首先检查bloom filter
        if !self.bloom_filter.may_contain(key) {
            return Ok(None);
        }

        // 在index block中查找
        let mut index_iter = BlockIterator::new(&self.index_block);
        index_iter.seek_to_key(key);

        while let Some((index_key, offset_bytes)) = index_iter.next() {
            if index_key.as_slice() > key {
                break;
            }

            let offset = u64::from_le_bytes(offset_bytes.as_slice().try_into().unwrap()) as usize;

            // 读取data block
            let block = self.read_data_block(offset)?;

            // 在block中查找
            let mut block_iter = BlockIterator::new(&block);
            block_iter.seek_to_key(key);

            if let Some((found_key, value)) = block_iter.next() {
                if found_key.as_slice() == key {
                    return Ok(Some(value));
                }
            }
        }

        Ok(None)
    }

    fn read_data_block(&self, offset: u64) -> Result<Block> {
        // 计算block大小
        let next_block_start = self.find_next_block_start(offset)?;
        let block_size = next_block_start - offset;

        // 读取block数据
        self.file.seek(SeekFrom::Start(offset))?;
        let mut block_data = vec![0u8; block_size];
        self.file.read_exact(&mut block_data)?;

        Ok(Block::decode(&block_data))
    }

    fn find_next_block_start(&self, current_offset: u64) -> Result<u64> {
        // 在index block中找到下一个block的偏移量
        let mut index_iter = BlockIterator::new(&self.index_block);
        index_iter.seek_to_first();

        let mut last_offset = current_offset;
        while let Some((_, offset_bytes)) = index_iter.next() {
            let offset = u64::from_le_bytes(offset_bytes.as_slice().try_into().unwrap());
            if offset > current_offset {
                return Ok(offset);
            }
            last_offset = offset;
        }

        // 如果没有找到下一个block，返回文件结束位置
        Ok(self.meta.data_end_offset)
    }
}
```

## 性能优化策略

### 1. 块大小优化

```rust
pub struct SstConfig {
    pub block_size: usize,
    pub block_restart_interval: usize,
    pub index_block_restart_interval: usize,
    pub bloom_filter_bits_per_key: usize,
}

impl Default for SstConfig {
    fn default() -> Self {
        Self {
            block_size: 4 * 1024, // 4KB
            block_restart_interval: 16,
            index_block_restart_interval: 1,
            bloom_filter_bits_per_key: 10,
        }
    }
}
```

### 2. 缓存策略

```rust
pub struct BlockCache {
    cache: LruCache<(u64, u64), Block>, // (file_id, offset) -> Block
    capacity: usize,
}

impl BlockCache {
    pub fn get(&mut self, file_id: u64, offset: u64) -> Option<Block> {
        self.cache.get(&(file_id, offset)).cloned()
    }

    pub fn put(&mut self, file_id: u64, offset: u64, block: Block) {
        self.cache.put((file_id, offset), block);
    }
}
```

### 3. 前缀压缩优化

```rust
pub fn calculate_compression_ratio(entries: &[(Vec<u8>, Vec<u8>)]) -> f64 {
    let mut compressed_size = 0;
    let mut uncompressed_size = 0;
    let mut last_key = Vec::new();

    for (key, value) in entries {
        uncompressed_size += key.len() + value.len();

        // 计算压缩后的大小
        let shared = common_prefix_length(&last_key, key);
        let unshared = key.len() - shared;

        compressed_size += varint_len(shared as u32);
        compressed_size += varint_len(unshared as u32);
        compressed_size += unshared;
        compressed_size += varint_len(value.len() as u32);
        compressed_size += value.len();

        last_key = key.clone();
    }

    compressed_size as f64 / uncompressed_size as f64
}
```

## 最佳实践

### 1. 合理设置参数

```rust
// 推荐：根据工作负载调整块大小
fn optimize_block_size(key_size: usize, value_size: usize) -> usize {
    let avg_entry_size = key_size + value_size;

    // 目标：每个block包含16-64个条目
    let target_entries = 32;
    (avg_entry_size * target_entries).max(1024).min(64 * 1024)
}

// 推荐：根据数据特性调整重启点间隔
fn optimize_restart_interval(key_size: usize, block_size: usize) -> usize {
    let entries_per_block = block_size / (key_size + 128); // 假设平均值大小为128

    // 目标：每16-64个条目设置一个重启点
    (entries_per_block / 32).max(1).min(64)
}
```

### 2. 错误处理和恢复

```rust
pub enum SstError {
    Io(io::Error),
    Corrupted(String),
    InvalidFormat(String),
}

impl From<io::Error> for SstError {
    fn from(err: io::Error) -> Self {
        SstError::Io(err)
    }
}

pub fn validate_sst_file(file: &mut File) -> Result<(), SstError> {
    let metadata = file.metadata()?;
    let file_size = metadata.len() as usize;

    // 检查文件大小
    if file_size < 8 {
        return Err(SstError::Corrupted("File too small".to_string()));
    }

    // 读取并验证meta offset
    file.seek(SeekFrom::End(-8))?;
    let mut meta_offset_bytes = [0u8; 8];
    file.read_exact(&mut meta_offset_bytes)?;
    let meta_offset = u64::from_le_bytes(meta_offset_bytes) as usize;

    if meta_offset >= file_size {
        return Err(SstError::Corrupted("Invalid meta offset".to_string()));
    }

    // 读取并验证meta block
    file.seek(SeekFrom::Start(meta_offset as u64))?;
    let mut meta_data = vec![0u8; file_size - meta_offset - 8];
    file.read_exact(&mut meta_data)?;

    let meta = SstMeta::decode(&meta_data)?;

    // 验证各块的偏移量
    if meta.index_offset >= meta.bloom_offset {
        return Err(SstError::Corrupted("Invalid index offset".to_string()));
    }

    if meta.bloom_offset >= meta_offset {
        return Err(SstError::Corrupted("Invalid bloom offset".to_string()));
    }

    Ok(())
}
```

## 总结

SST文件格式是LSM-Tree存储引擎的核心，其设计直接影响整个系统的性能。通过合理的数据结构选择、高效的编码方式和智能的优化策略，可以构建一个高性能的SST实现。

关键要点：
1. **分层结构**: Data blocks、Index block、Bloom filter、Meta block各司其职
2. **前缀压缩**: 显著减少存储空间和I/O开销
3. **重启点**: 平衡压缩效率和查询性能
4. **Bloom filter**: 快速过滤不存在的键
5. **缓存策略**: 优化热点数据访问

通过深入理解SST文件格式的设计原理，我们可以更好地理解LSM-Tree存储引擎的工作机制，并为实际系统设计提供参考。