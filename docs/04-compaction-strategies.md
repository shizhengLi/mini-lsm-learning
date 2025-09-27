# Compaction策略深度解析：LSM-Tree的性能核心

## 概述

Compaction（合并）是LSM-Tree存储引擎中最关键的操作之一，它负责合并和重写SST文件，控制空间放大、写放大和读放大之间的平衡。本文将深入探讨不同的Compaction策略及其实现细节。

## Compaction的重要性

### 三大放大因子

```
┌─────────────────────────────────────────────────────────────────┐
│                    LSM-Tree Compaction Trade-offs               │
├─────────────────────────────────────────────────────────────────┤
│                                                               │
│  Space Amplification = Total Storage / Actual Data Size        │
│  Write Amplification = Bytes Written / Bytes Received          │
│  Read Amplification = Files Read per Query                     │
│                                                               │
├─────────────────────────────────────────────────────────────────┤
│                Different Strategies, Different Trade-offs      │
└─────────────────────────────────────────────────────────────────┘
```

### Compaction的目标

1. **控制空间放大**: 删除过期的数据，回收存储空间
2. **优化读取性能**: 减少查询时需要检查的SST文件数量
3. **平衡写入性能**: 避免频繁的Compaction操作影响写入吞吐量
4. **维护数据有序性**: 保证数据在SST文件中的有序性

## Compaction策略分类

### 1. Simple Leveled Compaction

#### 原理
```
┌─────────────────────────────────────────────────────────────────┐
│                 Simple Leveled Compaction                      │
├─────────────────────────────────────────────────────────────────┤
│                                                               │
│  L0: [SST1] [SST2] [SST3]  (Small files, overlapping keys)   │
│  L1: [SST4] [SST5] [SST6]  (Larger files, non-overlapping)    │
│  L2: [SST7] [SST8] [SST9]  (Even larger files)                │
│                                                               │
│  Strategy: Merge L0 files into L1 when L0 threshold reached  │
│                                                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 实现代码

```rust
pub struct SimpleLeveledCompaction {
    l0_threshold: usize,
    l1_threshold: usize,
    level_size_multiplier: f64,
}

impl CompactionStrategy for SimpleLeveledCompaction {
    fn pick_compaction(&self, manifest: &Manifest) -> Option<CompactionTask> {
        // 检查L0是否需要合并
        if self.need_l0_compaction(manifest) {
            return Some(self.pick_l0_compaction(manifest));
        }

        // 检查L1是否需要合并
        if self.need_l1_compaction(manifest) {
            return Some(self.pick_l1_compaction(manifest));
        }

        None
    }

    fn apply_compaction(&self, task: CompactionTask) -> Result<()> {
        // 执行合并操作
        self.execute_compaction(task)
    }
}

impl SimpleLeveledCompaction {
    fn need_l0_compaction(&self, manifest: &Manifest) -> bool {
        let l0_files = manifest.get_level_files(0);
        l0_files.len() >= self.l0_threshold
    }

    fn need_l1_compaction(&self, manifest: &Manifest) -> bool {
        let l1_files = manifest.get_level_files(1);
        let l1_size: u64 = l1_files.iter().map(|f| f.size).sum();
        l1_size >= self.l1_threshold as u64
    }

    fn pick_l0_compaction(&self, manifest: &Manifest) -> CompactionTask {
        let l0_files = manifest.get_level_files(0);
        let l1_files = manifest.get_level_files(1);

        // 选择所有L0文件
        let input_files = l0_files.clone();

        // 确定需要合并的L1文件
        let key_range = self.get_key_range(&l0_files);
        let overlapping_l1_files = self.find_overlapping_files(&l1_files, &key_range);

        CompactionTask {
            input_level: 0,
            output_level: 1,
            input_files,
            output_files: Vec::new(),
            target_size: self.calculate_target_size(1),
        }
    }

    fn execute_compaction(&self, task: CompactionTask) -> Result<()> {
        // 1. 创建合并迭代器
        let mut merge_iter = self.create_merge_iterator(&task.input_files)?;

        // 2. 创建新的SST文件
        let mut new_files = Vec::new();
        let mut current_size = 0;
        let mut current_writer = None;

        // 3. 遍历合并后的数据
        while let Some((key, value)) = merge_iter.next() {
            if current_writer.is_none() || current_size >= task.target_size {
                // 创建新的SST文件
                if let Some(writer) = current_writer.take() {
                    new_files.push(writer.finish()?);
                }
                current_writer = Some(self.create_sst_writer(task.output_level)?);
                current_size = 0;
            }

            current_writer.as_mut().unwrap().write(key, value)?;
            current_size += key.len() + value.len();
        }

        // 4. 完成最后一个文件
        if let Some(writer) = current_writer {
            new_files.push(writer.finish()?);
        }

        // 5. 更新manifest
        self.update_manifest(&task, &new_files)?;

        // 6. 删除旧文件
        self.delete_old_files(&task.input_files)?;

        Ok(())
    }
}
```

### 2. Tiered Compaction (Universal Compaction)

#### 原理
```
┌─────────────────────────────────────────────────────────────────┐
│                    Tiered Compaction                           │
├─────────────────────────────────────────────────────────────────┤
│                                                               │
│  Tier 1: [SST1] [SST2] [SST3] (Size: 100MB each)              │
│  Tier 2: [SST4] [SST5]       (Size: 300MB each)                │
│  Tier 3: [SST6]              (Size: 900MB)                     │
│                                                               │
│  Strategy: Merge all files in a tier when threshold reached   │
│                                                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 实现代码

```rust
pub struct TieredCompaction {
    tier_size_ratio: f64,
    max_tiers: usize,
    min_files_to_compact: usize,
}

impl CompactionStrategy for TieredCompaction {
    fn pick_compaction(&self, manifest: &Manifest) -> Option<CompactionTask> {
        let tiers = self.organize_into_tiers(manifest);

        for (tier_index, tier) in tiers.iter().enumerate() {
            if self.should_compact_tier(tier) {
                return Some(self.create_tier_compaction_task(tier_index, tier));
            }
        }

        None
    }
}

impl TieredCompaction {
    fn organize_into_tiers(&self, manifest: &Manifest) -> Vec<Vec<SstFile>> {
        let mut tiers = Vec::new();
        let mut current_tier = Vec::new();
        let mut current_max_size = 0;

        for level in 0..manifest.max_level() {
            let files = manifest.get_level_files(level);

            for file in files {
                if current_tier.is_empty() || file.size <= current_max_size * self.tier_size_ratio {
                    current_tier.push(file);
                    current_max_size = current_max_size.max(file.size);
                } else {
                    if !current_tier.is_empty() {
                        tiers.push(current_tier);
                    }
                    current_tier = vec![file];
                    current_max_size = file.size;
                }
            }
        }

        if !current_tier.is_empty() {
            tiers.push(current_tier);
        }

        tiers
    }

    fn should_compact_tier(&self, tier: &[SstFile]) -> bool {
        tier.len() >= self.min_files_to_compact
    }

    fn create_tier_compaction_task(&self, tier_index: usize, tier: &[SstFile]) -> CompactionTask {
        let total_size: u64 = tier.iter().map(|f| f.size).sum();
        let output_level = tier_index + 1;

        CompactionTask {
            input_level: tier_index,
            output_level,
            input_files: tier.to_vec(),
            output_files: Vec::new(),
            target_size: (total_size as f64 * self.tier_size_ratio) as u64,
        }
    }
}
```

### 3. Leveled Compaction

#### 原理
```
┌─────────────────────────────────────────────────────────────────┐
│                    Leveled Compaction                          │
├─────────────────────────────────────────────────────────────────┤
│                                                               │
│  L0: [SST1] [SST2] [SST3]  (Overlapping allowed)              │
│  L1: [SST4] [SST5] [SST6]  (Non-overlapping, 100MB total)     │
│  L2: [SST7] [SST8]       (Non-overlapping, 1GB total)         │
│  L3: [SST9]              (Non-overlapping, 10GB total)         │
│                                                               │
│  Strategy: Pick files from Ln and merge with Ln+1             │
│           when Ln size exceeds threshold                      │
│                                                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 实现代码

```rust
pub struct LeveledCompaction {
    level_size_multiplier: f64,
    max_levels: usize,
    l0_threshold: usize,
    max_compaction_bytes: u64,
}

impl CompactionStrategy for LeveledCompaction {
    fn pick_compaction(&self, manifest: &Manifest) -> Option<CompactionTask> {
        // 优先处理L0
        if let Some(task) = self.pick_l0_compaction(manifest) {
            return Some(task);
        }

        // 检查其他层级
        for level in 1..self.max_levels {
            if let Some(task) = self.pick_level_compaction(manifest, level) {
                return Some(task);
            }
        }

        None
    }
}

impl LeveledCompaction {
    fn pick_l0_compaction(&self, manifest: &Manifest) -> Option<CompactionTask> {
        let l0_files = manifest.get_level_files(0);

        if l0_files.len() < self.l0_threshold {
            return None;
        }

        // 选择最老的文件
        let input_files = l0_files.iter()
            .take(self.l0_threshold)
            .cloned()
            .collect();

        // 计算与L1的重叠范围
        let key_range = self.get_key_range(&input_files);
        let l1_files = manifest.get_level_files(1);
        let overlapping_l1_files = self.find_overlapping_files(&l1_files, &key_range);

        Some(CompactionTask {
            input_level: 0,
            output_level: 1,
            input_files,
            output_files: Vec::new(),
            target_size: self.calculate_level_size(1),
        })
    }

    fn pick_level_compaction(&self, manifest: &Manifest, level: usize) -> Option<CompactionTask> {
        let files = manifest.get_level_files(level);
        let level_size: u64 = files.iter().map(|f| f.size).sum();
        let target_size = self.calculate_level_size(level);

        if level_size <= target_size {
            return None;
        }

        // 选择最大的文件进行合并
        let mut files_vec: Vec<_> = files.iter().collect();
        files_vec.sort_by(|a, b| b.size.cmp(&a.size));

        let input_file = files_vec[0].clone();
        let input_files = vec![input_file];

        // 计算与下一层的重叠范围
        let key_range = self.get_key_range(&input_files);
        let next_level_files = manifest.get_level_files(level + 1);
        let overlapping_next_level_files = self.find_overlapping_files(&next_level_files, &key_range);

        Some(CompactionTask {
            input_level: level,
            output_level: level + 1,
            input_files,
            output_files: Vec::new(),
            target_size: self.calculate_level_size(level + 1),
        })
    }

    fn calculate_level_size(&self, level: usize) -> u64 {
        let base_size = 10 * 1024 * 1024; // 10MB
        (base_size as f64 * self.level_size_multiplier.powi(level as i32 - 1)) as u64
    }
}
```

## 合并执行器

### 合并迭代器

```rust
pub struct MergeIterator {
    iterators: Vec<Box<dyn Iterator<Item = (KeySlice, ValueSlice)>>>,
    current_key: Option<KeySlice>,
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

impl MergeIterator {
    pub fn new(iterators: Vec<Box<dyn Iterator<Item = (KeySlice, ValueSlice)>>>) -> Self {
        let mut heap = BinaryHeap::new();
        let mut initial_items = Vec::new();

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
            current_key: None,
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
        let mut current = self.heap.pop().unwrap();

        // 跳过重复的键（保留最新的版本）
        while let Some(next) = self.heap.peek() {
            if next.key.data == current.key.data {
                // 如果是相同的键，比较时间戳
                if next.key.timestamp > current.key.timestamp {
                    // 下一个元素更新，跳过当前元素
                    current = self.heap.pop().unwrap();
                } else {
                    // 跳过下一个元素
                    self.heap.pop().unwrap();
                }
            } else {
                break;
            }
        }

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

### 合并任务调度

```rust
pub struct CompactionScheduler {
    strategy: Box<dyn CompactionStrategy>,
    manifest: Arc<Manifest>,
    pending_tasks: Vec<CompactionTask>,
    max_concurrent_compactions: usize,
}

impl CompactionScheduler {
    pub fn new(strategy: Box<dyn CompactionStrategy>, manifest: Arc<Manifest>) -> Self {
        Self {
            strategy,
            manifest,
            pending_tasks: Vec::new(),
            max_concurrent_compactions: 2,
        }
    }

    pub fn schedule_compactions(&mut self) -> Vec<CompactionTask> {
        let mut tasks = Vec::new();

        // 检查是否有pending tasks
        if !self.pending_tasks.is_empty() {
            tasks.extend(self.pending_tasks.drain(..));
        }

        // 生成新的compaction任务
        while let Some(task) = self.strategy.pick_compaction(&self.manifest) {
            if tasks.len() >= self.max_concurrent_compactions {
                self.pending_tasks.push(task);
                break;
            }
            tasks.push(task);
        }

        tasks
    }

    pub async fn execute_compaction(&self, task: CompactionTask) -> Result<()> {
        // 在单独的线程池中执行合并
        let manifest = self.manifest.clone();
        let strategy = self.strategy.as_ref();

        tokio::spawn(async move {
            if let Err(e) = strategy.apply_compaction(task) {
                eprintln!("Compaction failed: {}", e);
            }
        });

        Ok(())
    }
}
```

## 性能监控和调优

### 性能指标

```rust
pub struct CompactionStats {
    pub compaction_count: u64,
    pub total_bytes_written: u64,
    pub total_bytes_read: u64,
    pub average_duration_ms: u64,
    pub write_amplification: f64,
    pub space_amplification: f64,
    pub read_amplification: f64,
}

pub struct CompactionMonitor {
    stats: CompactionStats,
    history: Vec<CompactionEvent>,
}

impl CompactionMonitor {
    pub fn record_compaction(&mut self, event: CompactionEvent) {
        self.stats.compaction_count += 1;
        self.stats.total_bytes_written += event.bytes_written;
        self.stats.total_bytes_read += event.bytes_read;
        self.stats.average_duration_ms =
            (self.stats.average_duration_ms * (self.stats.compaction_count - 1) + event.duration_ms) /
            self.stats.compaction_count;

        self.history.push(event);
    }

    pub fn calculate_amplification_factors(&self, manifest: &Manifest) -> (f64, f64, f64) {
        // 计算写入放大
        let total_user_writes = self.stats.total_bytes_written;
        let write_amplification = self.stats.total_bytes_written as f64 / total_user_writes as f64;

        // 计算空间放大
        let total_storage: u64 = manifest.get_total_size();
        let actual_data = manifest.get_actual_data_size();
        let space_amplification = total_storage as f64 / actual_data as f64;

        // 计算读取放大
        let read_amplification = manifest.get_average_levels_read();

        (write_amplification, space_amplification, read_amplification)
    }
}
```

### 自适应调优

```rust
pub struct AdaptiveCompaction {
    strategy: Box<dyn CompactionStrategy>,
    monitor: CompactionMonitor,
    config: CompactionConfig,
}

impl AdaptiveCompaction {
    pub fn adapt_strategy(&mut self) {
        let (write_amp, space_amp, read_amp) = self.monitor.calculate_amplification_factors();

        // 根据性能指标调整策略
        if write_amp > self.config.max_write_amplification {
            self.adjust_for_write_performance();
        }

        if space_amp > self.config.max_space_amplification {
            self.adjust_for_space_efficiency();
        }

        if read_amp > self.config.max_read_amplification {
            self.adjust_for_read_performance();
        }
    }

    fn adjust_for_write_performance(&self) {
        // 增加合并阈值，减少合并频率
        if let Some(leveled) = self.strategy.as_any().downcast_ref::<LeveledCompaction>() {
            // 调整层级大小倍数
        }
    }

    fn adjust_for_space_efficiency(&self) {
        // 降低合并阈值，增加合并频率
    }

    fn adjust_for_read_performance(&self) {
        // 更积极地合并，减少SST文件数量
    }
}
```

## 最佳实践

### 1. 策略选择指南

```rust
pub fn recommend_strategy(workload: &Workload) -> CompactionStrategyType {
    match workload {
        Workload::WriteHeavy => CompactionStrategyType::Tiered,
        Workload::ReadHeavy => CompactionStrategyType::Leveled,
        Workload::Mixed => CompactionStrategyType::SimpleLeveled,
        Workload::SpaceConstrained => CompactionStrategyType::Leveled,
    }
}
```

### 2. 参数调优

```rust
pub struct CompactionParams {
    pub l0_threshold: usize,
    pub level_size_multiplier: f64,
    pub target_file_size: u64,
    pub max_compaction_bytes: u64,
}

impl Default for CompactionParams {
    fn default() -> Self {
        Self {
            l0_threshold: 4,
            level_size_multiplier: 10.0,
            target_file_size: 2 * 1024 * 1024, // 2MB
            max_compaction_bytes: 64 * 1024 * 1024, // 64MB
        }
    }
}

pub fn optimize_params(params: &mut CompactionParams, stats: &CompactionStats) {
    // 根据历史统计调整参数
    if stats.write_amplification > 20.0 {
        params.l0_threshold = (params.l0_threshold * 12) / 10;
    }

    if stats.space_amplification > 2.0 {
        params.level_size_multiplier = params.level_size_multiplier * 0.9;
    }
}
```

### 3. 监控和告警

```rust
pub struct CompactionAlerts {
    pub max_compaction_duration_ms: u64,
    pub max_write_amplification: f64,
    pub max_space_amplification: f64,
}

impl CompactionAlerts {
    pub fn check_alerts(&self, stats: &CompactionStats) -> Vec<Alert> {
        let mut alerts = Vec::new();

        if stats.average_duration_ms > self.max_compaction_duration_ms {
            alerts.push(Alert::CompactionTooSlow);
        }

        if stats.write_amplification > self.max_write_amplification {
            alerts.push(Alert::HighWriteAmplification);
        }

        if stats.space_amplification > self.max_space_amplification {
            alerts.push(Alert::HighSpaceAmplification);
        }

        alerts
    }
}
```

## 总结

Compaction策略是LSM-Tree存储引擎的核心，不同的策略有不同的性能特征。通过理解各种策略的原理和实现，我们可以根据具体的应用场景选择合适的策略，并通过监控和调优来优化系统性能。

关键要点：
1. **策略选择**: 根据工作负载特性选择合适的合并策略
2. **参数调优**: 根据性能指标动态调整合并参数
3. **性能监控**: 实时监控三大放大因子
4. **自适应优化**: 根据运行时状态调整策略
5. **并发控制**: 合理控制并发合并任务的数量

通过深入理解Compaction策略，我们可以构建一个高性能、高可靠性的LSM-Tree存储引擎。