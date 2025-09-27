# Mini-LSM 最佳实践指南：从开发到生产

## 概述

本文档提供了使用Mini-LSM存储引擎的最佳实践指南，涵盖开发、测试、部署和性能优化等各个方面。通过遵循这些实践，可以构建高性能、高可靠性的存储系统。

## 开发环境配置

### 1. 项目设置

```rust
// Cargo.toml
[package]
name = "mini-lsm-app"
version = "0.1.0"
edition = "2021"

[dependencies]
mini-lsm = "0.1"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
log = "0.4"
env_logger = "0.10"
thiserror = "1.0"
anyhow = "1.0"
metrics = "0.21"
tracing = "0.1"

[dev-dependencies]
criterion = "0.5"
tempfile = "3.8"
proptest = "1.4"
```

### 2. 配置管理

```rust
// src/config.rs
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AppConfig {
    pub storage: StorageConfig,
    pub logging: LoggingConfig,
    pub monitoring: MonitoringConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StorageConfig {
    pub data_dir: PathBuf,
    pub block_size: usize,
    pub target_sst_size: usize,
    pub num_mem_tables: usize,
    pub enable_wal: bool,
    pub sync_wal: bool,
    pub compaction_strategy: CompactionStrategy,
    pub max_concurrent_compactions: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LoggingConfig {
    pub level: String,
    pub file: Option<PathBuf>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MonitoringConfig {
    pub enable_metrics: bool,
    pub metrics_port: u16,
    pub health_check_interval_secs: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum CompactionStrategy {
    SimpleLeveled {
        l0_threshold: usize,
        l1_threshold: usize,
    },
    Tiered {
        tier_size_ratio: f64,
        min_files_to_compact: usize,
    },
    Leveled {
        level_size_multiplier: f64,
        l0_threshold: usize,
    },
}

impl Default for AppConfig {
    fn default() -> Self {
        Self {
            storage: StorageConfig::default(),
            logging: LoggingConfig::default(),
            monitoring: MonitoringConfig::default(),
        }
    }
}

impl StorageConfig {
    pub fn validate(&self) -> Result<(), anyhow::Error> {
        if self.block_size == 0 {
            return Err(anyhow::anyhow!("Block size must be greater than 0"));
        }
        if self.target_sst_size < self.block_size {
            return Err(anyhow::anyhow!("Target SST size must be greater than block size"));
        }
        if self.num_mem_tables == 0 {
            return Err(anyhow::anyhow!("Number of memtables must be greater than 0"));
        }
        Ok(())
    }
}

impl Default for StorageConfig {
    fn default() -> Self {
        Self {
            data_dir: PathBuf::from("./data"),
            block_size: 4 * 1024, // 4KB
            target_sst_size: 4 * 1024 * 1024, // 4MB
            num_mem_tables: 3,
            enable_wal: true,
            sync_wal: true,
            compaction_strategy: CompactionStrategy::Leveled {
                level_size_multiplier: 10.0,
                l0_threshold: 4,
            },
            max_concurrent_compactions: 2,
        }
    }
}
```

### 3. 应用程序结构

```rust
// src/main.rs
use mini_lsm::{LsmStorage, LsmStorageOptions};
use std::sync::Arc;
use tokio::signal;

mod config;
mod error;
mod metrics;
mod service;

use config::AppConfig;
use service::StorageService;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 加载配置
    let config = load_config()?;

    // 初始化日志
    init_logging(&config.logging);

    // 创建存储引擎
    let storage = create_storage(&config.storage)?;

    // 创建服务
    let service = StorageService::new(storage);

    // 启动服务
    service.start().await?;

    // 等待关闭信号
    tokio::select! {
        _ = signal::ctrl_c() => {
            log::info!("Received Ctrl+C, shutting down...");
        }
        _ = tokio::signal::unix::signal(signal::unix::SignalKind::terminate())
            .unwrap()
            .recv() => {
            log::info!("Received terminate signal, shutting down...");
        }
    }

    // 优雅关闭
    service.shutdown().await?;

    Ok(())
}

fn load_config() -> Result<AppConfig, anyhow::Error> {
    // 从环境变量或配置文件加载配置
    Ok(AppConfig::default())
}

fn init_logging(config: &config::LoggingConfig) {
    use std::env::set_var;

    set_var("RUST_LOG", &config.level);

    if let Some(file) = &config.file {
        // 配置文件日志
    } else {
        env_logger::init();
    }
}

fn create_storage(config: &config::StorageConfig) -> Result<Arc<LsmStorage>, anyhow::Error> {
    config.validate()?;

    let options = LsmStorageOptions {
        block_size: config.block_size,
        target_sst_size: config.target_sst_size,
        num_mem_tables: config.num_mem_tables,
        enable_wal: config.enable_wal,
        sync_wal: config.sync_wal,
        compaction_strategy: convert_compaction_strategy(&config.compaction_strategy),
    };

    let storage = LsmStorage::open(&config.data_dir, options)?;
    Ok(Arc::new(storage))
}

fn convert_compaction_strategy(strategy: &config::CompactionStrategy) -> mini_lsm::CompactionStrategy {
    use config::CompactionStrategy::*;
    use mini_lsm::CompactionStrategy as LsmCompactionStrategy;

    match strategy {
        SimpleLeveled { l0_threshold, l1_threshold } => {
            LsmCompactionStrategy::SimpleLeveled {
                l0_threshold: *l0_threshold,
                l1_threshold: *l1_threshold,
            }
        }
        Tiered { tier_size_ratio, min_files_to_compact } => {
            LsmCompactionStrategy::Tiered {
                tier_size_ratio: *tier_size_ratio,
                min_files_to_compact: *min_files_to_compact,
            }
        }
        Leveled { level_size_multiplier, l0_threshold } => {
            LsmCompactionStrategy::Leveled {
                level_size_multiplier: *level_size_multiplier,
                l0_threshold: *l0_threshold,
            }
        }
    }
}
```

## 数据访问模式

### 1. 基本操作

```rust
// src/storage/operations.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use anyhow::Result;

pub struct StorageOperations {
    storage: Arc<LsmStorage>,
}

impl StorageOperations {
    pub fn new(storage: Arc<LsmStorage>) -> Self {
        Self { storage }
    }

    pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
        let timer = metrics::start_timer("storage.put");
        let result = self.storage.put(key, value);
        metrics::stop_timer(timer);
        metrics::counter("storage.put.count").increment(1);

        result.map_err(|e| anyhow::anyhow!("Failed to put key: {:?}", e))
    }

    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        let timer = metrics::start_timer("storage.get");
        let result = self.storage.get(key);
        metrics::stop_timer(timer);
        metrics::counter("storage.get.count").increment(1);

        match result {
            Ok(Some(value)) => {
                metrics::counter("storage.get.hit").increment(1);
                Ok(Some(value))
            }
            Ok(None) => {
                metrics::counter("storage.get.miss").increment(1);
                Ok(None)
            }
            Err(e) => Err(anyhow::anyhow!("Failed to get key: {:?}", e)),
        }
    }

    pub fn delete(&self, key: &[u8]) -> Result<()> {
        let timer = metrics::start_timer("storage.delete");
        let result = self.storage.delete(key);
        metrics::stop_timer(timer);
        metrics::counter("storage.delete.count").increment(1);

        result.map_err(|e| anyhow::anyhow!("Failed to delete key: {:?}", e))
    }

    pub fn put_batch(&self, batch: &[(Vec<u8>, Vec<u8>)]) -> Result<()> {
        let timer = metrics::start_timer("storage.put_batch");
        let batch_size = batch.len();

        // 批量写入，减少锁竞争
        for (key, value) in batch {
            self.storage.put(key, value)?;
        }

        metrics::stop_timer(timer);
        metrics::counter("storage.put_batch.count").increment(batch_size as u64);

        Ok(())
    }

    pub fn get_batch(&self, keys: &[[u8]]) -> Result<Vec<Option<Vec<u8>>>> {
        let timer = metrics::start_timer("storage.get_batch");
        let batch_size = keys.len();

        let mut results = Vec::with_capacity(batch_size);
        for key in keys {
            results.push(self.storage.get(key)?);
        }

        metrics::stop_timer(timer);
        metrics::counter("storage.get_batch.count").increment(batch_size as u64);

        Ok(results)
    }

    pub fn scan_range(&self, start: &[u8], end: &[u8]) -> Result<Vec<(Vec<u8>, Vec<u8>)>> {
        use std::ops::Bound;

        let timer = metrics::start_timer("storage.scan");
        let iter = self.storage.scan((Bound::Included(start), Bound::Included(end)))?;

        let results: Result<Vec<_>> = iter.map(|(key, value)| {
            Ok((key.data.to_vec(), value.data.to_vec()))
        }).collect();

        metrics::stop_timer(timer);
        metrics::counter("storage.scan.count").increment(1);

        results
    }
}
```

### 2. 事务支持

```rust
// src/storage/transaction.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use anyhow::Result;
use std::collections::HashMap;

pub struct Transaction {
    storage: Arc<LsmStorage>,
    operations: Vec<TransactionOperation>,
    isolation_level: IsolationLevel,
}

#[derive(Debug, Clone)]
pub enum TransactionOperation {
    Put { key: Vec<u8>, value: Vec<u8> },
    Delete { key: Vec<u8> },
}

#[derive(Debug, Clone)]
pub enum IsolationLevel {
    ReadCommitted,
    RepeatableRead,
    Serializable,
}

impl Transaction {
    pub fn new(storage: Arc<LsmStorage>) -> Self {
        Self {
            storage,
            operations: Vec::new(),
            isolation_level: IsolationLevel::ReadCommitted,
        }
    }

    pub fn with_isolation_level(storage: Arc<LsmStorage>, level: IsolationLevel) -> Self {
        Self {
            storage,
            operations: Vec::new(),
            isolation_level: level,
        }
    }

    pub fn put(&mut self, key: Vec<u8>, value: Vec<u8>) -> Result<()> {
        // 检查是否已存在该键的操作
        if let Some(pos) = self.operations.iter().position(|op| match op {
            TransactionOperation::Put { key: k, .. } => k == &key,
            TransactionOperation::Delete { key: k } => k == &key,
        }) {
            // 替换现有操作
            self.operations[pos] = TransactionOperation::Put { key, value };
        } else {
            self.operations.push(TransactionOperation::Put { key, value });
        }
        Ok(())
    }

    pub fn delete(&mut self, key: Vec<u8>) -> Result<()> {
        // 检查是否已存在该键的操作
        if let Some(pos) = self.operations.iter().position(|op| match op {
            TransactionOperation::Put { key: k, .. } => k == &key,
            TransactionOperation::Delete { key: k } => k == &key,
        }) {
            // 替换现有操作
            self.operations[pos] = TransactionOperation::Delete { key };
        } else {
            self.operations.push(TransactionOperation::Delete { key });
        }
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        match self.isolation_level {
            IsolationLevel::ReadCommitted => self.get_read_committed(key),
            IsolationLevel::RepeatableRead => self.get_repeatable_read(key),
            IsolationLevel::Serializable => self.get_serializable(key),
        }
    }

    fn get_read_committed(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 首先在事务操作中查找
        for op in self.operations.iter().rev() {
            match op {
                TransactionOperation::Put { key: k, value } if k == key => {
                    return Ok(Some(value.clone()));
                }
                TransactionOperation::Delete { key: k } if k == key => {
                    return Ok(None);
                }
                _ => {}
            }
        }

        // 如果没有找到，查询存储引擎
        self.storage.get(key).map_err(|e| anyhow::anyhow!("Failed to get key: {:?}", e))
    }

    fn get_repeatable_read(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 在可重复读隔离级别下，需要维护快照
        // 这里简化实现，实际应该维护事务开始时的快照
        self.get_read_committed(key)
    }

    fn get_serializable(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 在可串行化隔离级别下，需要更严格的并发控制
        // 这里简化实现
        self.get_read_committed(key)
    }

    pub fn commit(self) -> Result<()> {
        let timer = metrics::start_timer("storage.transaction.commit");

        // 执行所有操作
        for op in self.operations {
            match op {
                TransactionOperation::Put { key, value } => {
                    self.storage.put(&key, &value)?;
                }
                TransactionOperation::Delete { key } => {
                    self.storage.delete(&key)?;
                }
            }
        }

        metrics::stop_timer(timer);
        metrics::counter("storage.transaction.commit").increment(1);

        Ok(())
    }

    pub fn rollback(self) -> Result<()> {
        // 回滚操作，在这个简单实现中直接丢弃操作
        metrics::counter("storage.transaction.rollback").increment(1);
        Ok(())
    }
}
```

### 3. 缓存层

```rust
// src/storage/cache.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use anyhow::Result;
use std::collections::HashMap;
use std::time::{Duration, Instant};
use parking_lot::RwLock;

pub struct CacheConfig {
    pub max_size: usize,
    pub ttl: Duration,
    pub enable_metrics: bool,
}

impl Default for CacheConfig {
    fn default() -> Self {
        Self {
            max_size: 1000,
            ttl: Duration::from_secs(300), // 5 minutes
            enable_metrics: true,
        }
    }
}

pub struct CacheEntry<T> {
    pub value: T,
    pub created_at: Instant,
}

pub struct StorageCache {
    cache: RwLock<HashMap<Vec<u8>, CacheEntry<Vec<u8>>>>,
    config: CacheConfig,
    storage: Arc<LsmStorage>,
}

impl StorageCache {
    pub fn new(storage: Arc<LsmStorage>, config: CacheConfig) -> Self {
        Self {
            cache: RwLock::new(HashMap::new()),
            config,
            storage,
        }
    }

    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        // 首先检查缓存
        {
            let cache = self.cache.read();
            if let Some(entry) = cache.get(key) {
                if entry.created_at.elapsed() < self.config.ttl {
                    if self.config.enable_metrics {
                        metrics::counter("storage.cache.hit").increment(1);
                    }
                    return Ok(Some(entry.value.clone()));
                }
            }
        }

        // 缓存未命中，查询存储
        let result = self.storage.get(key)?;

        if let Some(value) = &result {
            // 更新缓存
            let mut cache = self.cache.write();
            cache.insert(
                key.to_vec(),
                CacheEntry {
                    value: value.clone(),
                    created_at: Instant::now(),
                },
            );

            // 检查缓存大小
            if cache.len() > self.config.max_size {
                self.evict_lru(&mut cache);
            }
        }

        if self.config.enable_metrics {
            metrics::counter("storage.cache.miss").increment(1);
        }

        Ok(result)
    }

    pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
        // 写入存储
        self.storage.put(key, value)?;

        // 更新缓存
        {
            let mut cache = self.cache.write();
            cache.insert(
                key.to_vec(),
                CacheEntry {
                    value: value.to_vec(),
                    created_at: Instant::now(),
                },
            );

            // 检查缓存大小
            if cache.len() > self.config.max_size {
                self.evict_lru(&mut cache);
            }
        }

        Ok(())
    }

    pub fn delete(&self, key: &[u8]) -> Result<()> {
        // 从存储删除
        self.storage.delete(key)?;

        // 从缓存删除
        {
            let mut cache = self.cache.write();
            cache.remove(key);
        }

        Ok(())
    }

    pub fn clear(&self) {
        let mut cache = self.cache.write();
        cache.clear();
    }

    fn evict_lru(&self, cache: &mut HashMap<Vec<u8>, CacheEntry<Vec<u8>>>) {
        // 简单的LRU淘汰策略
        if let Some((&oldest_key, _)) = cache.iter()
            .min_by_key(|(_, entry)| entry.created_at) {
            cache.remove(oldest_key);
        }
    }

    pub fn stats(&self) -> CacheStats {
        let cache = self.cache.read();
        CacheStats {
            size: cache.len(),
            max_size: self.config.max_size,
            hit_count: metrics::counter("storage.cache.hit").get(),
            miss_count: metrics::counter("storage.cache.miss").get(),
        }
    }
}

pub struct CacheStats {
    pub size: usize,
    pub max_size: usize,
    pub hit_count: u64,
    pub miss_count: u64,
}
```

## 监控和运维

### 1. 性能监控

```rust
// src/monitoring/metrics.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use anyhow::Result;
use prometheus::{Counter, Histogram, Gauge, Registry};

pub struct StorageMetrics {
    registry: Registry,

    // 操作计数器
    pub put_count: Counter,
    pub get_count: Counter,
    pub delete_count: Counter,
    pub scan_count: Counter,

    // 延迟指标
    pub put_latency: Histogram,
    pub get_latency: Histogram,
    pub delete_latency: Histogram,
    pub scan_latency: Histogram,

    // 存储统计
    pub total_size: Gauge,
    pub total_keys: Gauge,
    pub memtable_count: Gauge,
    pub sstable_count: Gauge,

    // 合并统计
    pub compaction_count: Counter,
    pub compaction_latency: Histogram,

    // 缓存统计
    pub cache_hit_count: Counter,
    pub cache_miss_count: Counter,
    pub cache_size: Gauge,
}

impl StorageMetrics {
    pub fn new() -> Result<Self> {
        let registry = Registry::new();

        Ok(Self {
            put_count: Counter::new("storage_put_count_total", "Total number of put operations")?,
            get_count: Counter::new("storage_get_count_total", "Total number of get operations")?,
            delete_count: Counter::new("storage_delete_count_total", "Total number of delete operations")?,
            scan_count: Counter::new("storage_scan_count_total", "Total number of scan operations")?,

            put_latency: Histogram::new("storage_put_latency_seconds", "Put operation latency")?,
            get_latency: Histogram::new("storage_get_latency_seconds", "Get operation latency")?,
            delete_latency: Histogram::new("storage_delete_latency_seconds", "Delete operation latency")?,
            scan_latency: Histogram::new("storage_scan_latency_seconds", "Scan operation latency")?,

            total_size: Gauge::new("storage_total_size_bytes", "Total storage size in bytes")?,
            total_keys: Gauge::new("storage_total_keys", "Total number of keys")?,
            memtable_count: Gauge::new("storage_memtable_count", "Number of memtables")?,
            sstable_count: Gauge::new("storage_sstable_count", "Number of SSTables")?,

            compaction_count: Counter::new("storage_compaction_count_total", "Total number of compactions")?,
            compaction_latency: Histogram::new("storage_compaction_latency_seconds", "Compaction latency")?,

            cache_hit_count: Counter::new("storage_cache_hit_count_total", "Total number of cache hits")?,
            cache_miss_count: Counter::new("storage_cache_miss_count_total", "Total number of cache misses")?,
            cache_size: Gauge::new("storage_cache_size", "Cache size")?,
        })
    }

    pub fn update_from_storage(&self, storage: &LsmStorage) -> Result<()> {
        let stats = storage.get_stats();

        self.total_size.set(stats.total_size as f64);
        self.total_keys.set(stats.total_keys as f64);
        self.memtable_count.set(stats.mem_table_count as f64);
        self.sstable_count.set(stats.sstable_count as f64);

        Ok(())
    }

    pub fn start_timer(&self, operation: &str) -> Timer {
        match operation {
            "put" => Timer::new(&self.put_latency),
            "get" => Timer::new(&self.get_latency),
            "delete" => Timer::new(&self.delete_latency),
            "scan" => Timer::new(&self.scan_latency),
            "compaction" => Timer::new(&self.compaction_latency),
            _ => Timer::new(&self.put_latency),
        }
    }
}

pub struct Timer {
    start: std::time::Instant,
    histogram: Histogram,
}

impl Timer {
    pub fn new(histogram: &Histogram) -> Self {
        Self {
            start: std::time::Instant::now(),
            histogram: histogram.clone(),
        }
    }
}

impl Drop for Timer {
    fn drop(&mut self) {
        let duration = self.start.elapsed();
        self.histogram.observe(duration.as_secs_f64());
    }
}
```

### 2. 健康检查

```rust
// src/monitoring/health.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use anyhow::Result;

pub struct HealthChecker {
    storage: Arc<LsmStorage>,
    config: HealthConfig,
}

#[derive(Debug, Clone)]
pub struct HealthConfig {
    pub max_response_time_ms: u64,
    pub max_memtable_count: usize,
    pub max_sstable_count: usize,
    pub min_free_space_bytes: u64,
}

impl Default for HealthConfig {
    fn default() -> Self {
        Self {
            max_response_time_ms: 100,
            max_memtable_count: 10,
            max_sstable_count: 1000,
            min_free_space_bytes: 1024 * 1024 * 1024, // 1GB
        }
    }
}

#[derive(Debug)]
pub struct HealthStatus {
    pub is_healthy: bool,
    pub details: Vec<HealthDetail>,
}

#[derive(Debug)]
pub struct HealthDetail {
    pub component: String,
    pub status: HealthComponentStatus,
    pub message: String,
}

#[derive(Debug, PartialEq)]
pub enum HealthComponentStatus {
    Healthy,
    Warning,
    Critical,
}

impl HealthChecker {
    pub fn new(storage: Arc<LsmStorage>, config: HealthConfig) -> Self {
        Self { storage, config }
    }

    pub async fn check_health(&self) -> Result<HealthStatus> {
        let mut details = Vec::new();
        let mut is_healthy = true;

        // 检查存储响应时间
        let response_time = self.check_response_time().await?;
        if response_time > self.config.max_response_time_ms {
            details.push(HealthDetail {
                component: "response_time".to_string(),
                status: HealthComponentStatus::Critical,
                message: format!("Response time {}ms exceeds threshold {}ms",
                    response_time, self.config.max_response_time_ms),
            });
            is_healthy = false;
        } else if response_time > self.config.max_response_time_ms / 2 {
            details.push(HealthDetail {
                component: "response_time".to_string(),
                status: HealthComponentStatus::Warning,
                message: format!("Response time {}ms is approaching threshold", response_time),
            });
        } else {
            details.push(HealthDetail {
                component: "response_time".to_string(),
                status: HealthComponentStatus::Healthy,
                message: format!("Response time {}ms is normal", response_time),
            });
        }

        // 检查内存表数量
        let stats = self.storage.get_stats();
        if stats.mem_table_count > self.config.max_memtable_count {
            details.push(HealthDetail {
                component: "memtable_count".to_string(),
                status: HealthComponentStatus::Critical,
                message: format!("Memtable count {} exceeds threshold {}",
                    stats.mem_table_count, self.config.max_memtable_count),
            });
            is_healthy = false;
        }

        // 检查SST文件数量
        if stats.sstable_count > self.config.max_sstable_count {
            details.push(HealthDetail {
                component: "sstable_count".to_string(),
                status: HealthComponentStatus::Warning,
                message: format!("SSTable count {} is high", stats.sstable_count),
            });
        }

        // 检查磁盘空间
        let free_space = self.check_disk_space().await?;
        if free_space < self.config.min_free_space_bytes {
            details.push(HealthDetail {
                component: "disk_space".to_string(),
                status: HealthComponentStatus::Critical,
                message: format!("Free space {} bytes is below threshold {}",
                    free_space, self.config.min_free_space_bytes),
            });
            is_healthy = false;
        }

        Ok(HealthStatus { is_healthy, details })
    }

    async fn check_response_time(&self) -> Result<u64> {
        let start = std::time::Instant::now();

        // 执行一个简单的GET操作来测试响应时间
        let test_key = b"health_check_test_key";
        let _ = self.storage.get(test_key);

        Ok(start.elapsed().as_millis() as u64)
    }

    async fn check_disk_space(&self) -> Result<u64> {
        use std::fs;

        // 获取数据目录的磁盘空间
        let metadata = fs::metadata(".")?;
        let free_space = metadata.available_space();

        Ok(free_space)
    }
}
```

### 3. 备份和恢复

```rust
// src/backup/backup.rs
use mini_lsm::LsmStorage;
use std::sync::Arc;
use std::path::{Path, PathBuf};
use anyhow::Result;
use std::fs;
use tokio::io::AsyncReadExt;
use tokio::fs::File;

pub struct BackupManager {
    storage: Arc<LsmStorage>,
    backup_dir: PathBuf,
    config: BackupConfig,
}

#[derive(Debug, Clone)]
pub struct BackupConfig {
    pub max_backups: usize,
    pub compression_enabled: bool,
    pub checksum_enabled: bool,
}

impl Default for BackupConfig {
    fn default() -> Self {
        Self {
            max_backups: 5,
            compression_enabled: true,
            checksum_enabled: true,
        }
    }
}

impl BackupManager {
    pub fn new(storage: Arc<LsmStorage>, backup_dir: PathBuf, config: BackupConfig) -> Self {
        Self { storage, backup_dir, config }
    }

    pub async fn create_backup(&self) -> Result<String> {
        let timestamp = chrono::Utc::now().format("%Y%m%d_%H%M%S");
        let backup_name = format!("backup_{}", timestamp);
        let backup_path = self.backup_dir.join(&backup_name);

        // 创建备份目录
        tokio::fs::create_dir_all(&backup_path).await?;

        // 创建备份清单
        let manifest = self.create_backup_manifest().await?;

        // 复制数据文件
        self.copy_data_files(&backup_path).await?;

        // 创建备份元数据
        self.create_backup_metadata(&backup_path, &manifest).await?;

        // 清理旧备份
        self.cleanup_old_backups().await?;

        Ok(backup_name)
    }

    async fn create_backup_manifest(&self) -> Result<BackupManifest> {
        let stats = self.storage.get_stats();

        Ok(BackupManifest {
            timestamp: chrono::Utc::now(),
            total_keys: stats.total_keys,
            total_size: stats.total_size,
            memtable_count: stats.mem_table_count,
            sstable_count: stats.sstable_count,
            version: "1.0".to_string(),
        })
    }

    async fn copy_data_files(&self, backup_path: &Path) -> Result<()> {
        // 获取数据目录中的所有文件
        let data_dir = self.get_data_directory();
        let mut entries = tokio::fs::read_dir(&data_dir).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_file() {
                let dest_path = backup_path.join(path.file_name().unwrap());
                tokio::fs::copy(&path, &dest_path).await?;
            }
        }

        Ok(())
    }

    async fn create_backup_metadata(&self, backup_path: &Path, manifest: &BackupManifest) -> Result<()> {
        let metadata_path = backup_path.join("backup.json");
        let metadata_json = serde_json::to_string_pretty(manifest)?;
        tokio::fs::write(&metadata_path, metadata_json).await?;

        Ok(())
    }

    async fn cleanup_old_backups(&self) -> Result<()> {
        if !self.backup_dir.exists() {
            return Ok(());
        }

        let mut backups = Vec::new();
        let mut entries = tokio::fs::read_dir(&self.backup_dir).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_dir() && path.file_name().unwrap().to_string_lossy().starts_with("backup_") {
                if let Ok(metadata) = entry.metadata().await {
                    if let Ok(created) = metadata.created() {
                        backups.push((path, created));
                    }
                }
            }
        }

        // 按创建时间排序
        backups.sort_by(|a, b| b.1.cmp(&a.1));

        // 删除多余的备份
        for (backup_path, _) in backups.iter().skip(self.config.max_backups) {
            tokio::fs::remove_dir_all(backup_path).await?;
        }

        Ok(())
    }

    pub async fn restore_backup(&self, backup_name: &str) -> Result<()> {
        let backup_path = self.backup_dir.join(backup_name);

        if !backup_path.exists() {
            return Err(anyhow::anyhow!("Backup {} not found", backup_name));
        }

        // 验证备份完整性
        self.verify_backup_integrity(&backup_path).await?;

        // 停止存储服务
        // TODO: Implement graceful shutdown

        // 备份当前数据
        self.backup_current_data().await?;

        // 恢复数据
        self.restore_data_files(&backup_path).await?;

        // 重启存储服务
        // TODO: Implement service restart

        Ok(())
    }

    async fn verify_backup_integrity(&self, backup_path: &Path) -> Result<()> {
        let metadata_path = backup_path.join("backup.json");
        let metadata_content = tokio::fs::read_to_string(&metadata_path).await?;
        let manifest: BackupManifest = serde_json::from_str(&metadata_content)?;

        // 验证数据文件
        let mut file_count = 0;
        let mut entries = tokio::fs::read_dir(backup_path).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_file() && path.file_name().unwrap() != "backup.json" {
                file_count += 1;
            }
        }

        if file_count == 0 {
            return Err(anyhow::anyhow!("No data files found in backup"));
        }

        Ok(())
    }

    async fn backup_current_data(&self) -> Result<()> {
        let timestamp = chrono::Utc::now().format("%Y%m%d_%H%M%S");
        let backup_name = format!("pre_restore_backup_{}", timestamp);
        self.create_backup().await?;
        Ok(())
    }

    async fn restore_data_files(&self, backup_path: &Path) -> Result<()> {
        let data_dir = self.get_data_directory();

        // 清空数据目录
        if data_dir.exists() {
            tokio::fs::remove_dir_all(&data_dir).await?;
        }
        tokio::fs::create_dir_all(&data_dir).await?;

        // 复制备份文件
        let mut entries = tokio::fs::read_dir(backup_path).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_file() && path.file_name().unwrap() != "backup.json" {
                let dest_path = data_dir.join(path.file_name().unwrap());
                tokio::fs::copy(&path, &dest_path).await?;
            }
        }

        Ok(())
    }

    fn get_data_directory(&self) -> PathBuf {
        // 返回数据目录路径
        PathBuf::from("./data")
    }

    pub async fn list_backups(&self) -> Result<Vec<BackupInfo>> {
        let mut backups = Vec::new();

        if !self.backup_dir.exists() {
            return Ok(backups);
        }

        let mut entries = tokio::fs::read_dir(&self.backup_dir).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_dir() && path.file_name().unwrap().to_string_lossy().starts_with("backup_") {
                if let Ok(metadata) = entry.metadata().await {
                    if let Ok(created) = metadata.created() {
                        let size = self.calculate_backup_size(&path).await?;
                        backups.push(BackupInfo {
                            name: path.file_name().unwrap().to_string_lossy().to_string(),
                            created_at: created,
                            size,
                        });
                    }
                }
            }
        }

        backups.sort_by(|a, b| b.created_at.cmp(&a.created_at));
        Ok(backups)
    }

    async fn calculate_backup_size(&self, backup_path: &Path) -> Result<u64> {
        let mut total_size = 0;
        let mut entries = tokio::fs::read_dir(backup_path).await?;

        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            if path.is_file() {
                let metadata = entry.metadata().await?;
                total_size += metadata.len();
            }
        }

        Ok(total_size)
    }
}

#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct BackupManifest {
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub total_keys: u64,
    pub total_size: u64,
    pub memtable_count: usize,
    pub sstable_count: usize,
    pub version: String,
}

#[derive(Debug)]
pub struct BackupInfo {
    pub name: String,
    pub created_at: std::time::SystemTime,
    pub size: u64,
}
```

## 测试策略

### 1. 单元测试

```rust
// tests/unit_tests.rs
use mini_lsm::{LsmStorage, LsmStorageOptions};
use tempfile::tempdir;

#[tokio::test]
async fn test_basic_operations() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions::default();
    let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

    // 测试put操作
    storage.put(b"key1", b"value1").unwrap();
    storage.put(b"key2", b"value2").unwrap();

    // 测试get操作
    assert_eq!(storage.get(b"key1").unwrap(), Some(b"value1".to_vec()));
    assert_eq!(storage.get(b"key2").unwrap(), Some(b"value2".to_vec()));
    assert_eq!(storage.get(b"key3").unwrap(), None);

    // 测试delete操作
    storage.delete(b"key1").unwrap();
    assert_eq!(storage.get(b"key1").unwrap(), None);
}

#[tokio::test]
async fn test_batch_operations() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions::default();
    let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

    let batch = vec![
        (b"batch_key1".to_vec(), b"batch_value1".to_vec()),
        (b"batch_key2".to_vec(), b"batch_value2".to_vec()),
        (b"batch_key3".to_vec(), b"batch_value3".to_vec()),
    ];

    // 批量写入
    for (key, value) in &batch {
        storage.put(key, value).unwrap();
    }

    // 验证批量读取
    for (key, expected_value) in batch {
        assert_eq!(storage.get(&key).unwrap(), Some(expected_value));
    }
}

#[tokio::test]
async fn test_range_scan() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions::default();
    let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

    // 插入测试数据
    for i in 0..100 {
        let key = format!("key_{:03}", i);
        let value = format!("value_{:03}", i);
        storage.put(key.as_bytes(), value.as_bytes()).unwrap();
    }

    // 范围扫描
    use std::ops::Bound;
    let iter = storage.scan((Bound::Included(b"key_010"), Bound::Included(b"key_020"))).unwrap();

    let mut count = 0;
    for (key, value) in iter {
        let key_str = String::from_utf8(key.data.to_vec()).unwrap();
        let value_str = String::from_utf8(value.data.to_vec()).unwrap();

        assert!(key_str >= "key_010" && key_str <= "key_020");
        assert_eq!(key_str.replace("key_", "value_"), value_str);
        count += 1;
    }

    assert_eq!(count, 11); // 包含边界值
}

#[tokio::test]
async fn test_concurrent_operations() {
    use tokio::task::JoinSet;

    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions::default();
    let storage = std::sync::Arc::new(LsmStorage::open(temp_dir.path(), options).unwrap());

    let mut join_set = JoinSet::new();

    // 启动多个并发写入任务
    for task_id in 0..10 {
        let storage_clone = storage.clone();
        join_set.spawn(async move {
            for i in 0..100 {
                let key = format!("task_{}_key_{}", task_id, i);
                let value = format!("task_{}_value_{}", task_id, i);
                storage_clone.put(key.as_bytes(), value.as_bytes()).unwrap();
            }
        });
    }

    // 等待所有任务完成
    while let Some(result) = join_set.join_next().await {
        result.unwrap();
    }

    // 验证数据
    for task_id in 0..10 {
        for i in 0..100 {
            let key = format!("task_{}_key_{}", task_id, i);
            let expected_value = format!("task_{}_value_{}", task_id, i);
            assert_eq!(storage.get(key.as_bytes()).unwrap(), Some(expected_value.into_bytes()));
        }
    }
}
```

### 2. 集成测试

```rust
// tests/integration_tests.rs
use mini_lsm::{LsmStorage, LsmStorageOptions};
use tempfile::tempdir;
use std::time::Duration;
use tokio::time::sleep;

#[tokio::test]
async fn test_persistence_across_restarts() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions::default();

    // 第一次启动，写入数据
    {
        let storage = LsmStorage::open(temp_dir.path(), options.clone()).unwrap();
        storage.put(b"persistent_key", b"persistent_value").unwrap();

        // 强制刷盘
        storage.flush().unwrap();
    }

    // 第二次启动，验证数据持久化
    {
        let storage = LsmStorage::open(temp_dir.path(), options).unwrap();
        assert_eq!(storage.get(b"persistent_key").unwrap(), Some(b"persistent_value".to_vec()));
    }
}

#[tokio::test]
async fn test_compaction_during_heavy_writes() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions {
        target_sst_size: 1024, // 小尺寸，触发合并
        ..Default::default()
    };

    let storage = std::sync::Arc::new(LsmStorage::open(temp_dir.path(), options).unwrap());

    // 大量写入触发合并
    for i in 0..1000 {
        let key = format!("compaction_key_{}", i);
        let value = vec![0u8; 100]; // 较大的值
        storage.put(key.as_bytes(), &value).unwrap();
    }

    // 等待合并完成
    sleep(Duration::from_secs(2)).await;

    // 验证数据仍然存在
    for i in 0..1000 {
        let key = format!("compaction_key_{}", i);
        assert!(storage.get(key.as_bytes()).unwrap().is_some());
    }
}

#[tokio::test]
async fn test_memory_usage_under_load() {
    let temp_dir = tempdir().unwrap();
    let options = LsmStorageOptions {
        target_sst_size: 1024 * 1024, // 1MB
        num_mem_tables: 2,
        ..Default::default()
    };

    let storage = std::sync::Arc::new(LsmStorage::open(temp_dir.path(), options).unwrap());

    // 监控内存使用
    let initial_stats = storage.get_stats();
    println!("Initial stats: {:?}", initial_stats);

    // 大量写入
    for i in 0..5000 {
        let key = format!("memory_test_key_{}", i);
        let value = format!("memory_test_value_{}", i);
        storage.put(key.as_bytes(), value.as_bytes()).unwrap();

        // 定期检查统计信息
        if i % 1000 == 0 {
            let stats = storage.get_stats();
            println!("After {} writes: {:?}", i, stats);
        }
    }

    let final_stats = storage.get_stats();
    println!("Final stats: {:?}", final_stats);

    // 验证内存表数量合理
    assert!(final_stats.mem_table_count <= 5); // 包括不可变内存表
}
```

### 3. 性能测试

```rust
// benches/performance_tests.rs
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId};
use mini_lsm::{LsmStorage, LsmStorageOptions};
use tempfile::tempdir;

fn benchmark_put(c: &mut Criterion) {
    let mut group = c.benchmark_group("put_operations");

    for size in [100, 1000, 10000].iter() {
        group.bench_with_input(BenchmarkId::new("put", size), size, |b, size| {
            let temp_dir = tempdir().unwrap();
            let options = LsmStorageOptions::default();
            let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

            b.iter(|| {
                for i in 0..*size {
                    let key = format!("benchmark_key_{}", i);
                    let value = format!("benchmark_value_{}", i);
                    storage.put(key.as_bytes(), value.as_bytes()).unwrap();
                }
            });
        });
    }

    group.finish();
}

fn benchmark_get(c: &mut Criterion) {
    let mut group = c.benchmark_group("get_operations");

    for size in [100, 1000, 10000].iter() {
        group.bench_with_input(BenchmarkId::new("get", size), size, |b, size| {
            let temp_dir = tempdir().unwrap();
            let options = LsmStorageOptions::default();
            let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

            // 预先插入数据
            for i in 0..*size {
                let key = format!("benchmark_key_{}", i);
                let value = format!("benchmark_value_{}", i);
                storage.put(key.as_bytes(), value.as_bytes()).unwrap();
            }

            b.iter(|| {
                for i in 0..*size {
                    let key = format!("benchmark_key_{}", i);
                    storage.get(key.as_bytes()).unwrap();
                }
            });
        });
    }

    group.finish();
}

fn benchmark_range_scan(c: &mut Criterion) {
    let mut group = c.benchmark_group("range_scan");

    for size in [1000, 10000, 100000].iter() {
        group.bench_with_input(BenchmarkId::new("scan", size), size, |b, size| {
            let temp_dir = tempdir().unwrap();
            let options = LsmStorageOptions::default();
            let storage = LsmStorage::open(temp_dir.path(), options).unwrap();

            // 预先插入数据
            for i in 0..*size {
                let key = format!("benchmark_key_{:06}", i);
                let value = format!("benchmark_value_{:06}", i);
                storage.put(key.as_bytes(), value.as_bytes()).unwrap();
            }

            b.iter(|| {
                use std::ops::Bound;
                let start = size / 4;
                let end = size * 3 / 4;

                let iter = storage.scan((
                    Bound::Included(format!("benchmark_key_{:06}", start).as_bytes()),
                    Bound::Included(format!("benchmark_key_{:06}", end).as_bytes()),
                )).unwrap();

                let mut count = 0;
                for _ in iter {
                    count += 1;
                }

                assert!(count > 0);
            });
        });
    }

    group.finish();
}

criterion_group!(benches, benchmark_put, benchmark_get, benchmark_range_scan);
criterion_main!(benches);
```

## 部署和运维

### 1. Docker容器化

```dockerfile
# Dockerfile
FROM rust:1.70 as builder

WORKDIR /app

# 复制依赖文件
COPY Cargo.toml Cargo.lock ./
COPY src ./src

# 构建应用
RUN cargo build --release

# 运行时镜像
FROM debian:bullseye-slim

# 安装运行时依赖
RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 复制二进制文件
COPY --from=builder /app/target/release/mini-lsm-app ./mini-lsm-app

# 创建数据目录
RUN mkdir -p /data

# 设置环境变量
ENV RUST_LOG=info
ENV RUST_BACKTRACE=1

# 暴露端口
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# 启动命令
CMD ["./mini-lsm-app"]
```

### 2. Kubernetes部署

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mini-lsm-app
  labels:
    app: mini-lsm-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mini-lsm-app
  template:
    metadata:
      labels:
        app: mini-lsm-app
    spec:
      containers:
      - name: mini-lsm-app
        image: mini-lsm-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: RUST_LOG
          value: "info"
        - name: DATA_DIR
          value: "/data"
        volumeMounts:
        - name: data-volume
          mountPath: /data
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: mini-lsm-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mini-lsm-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mini-lsm-service
spec:
  selector:
    app: mini-lsm-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
```

### 3. 监控配置

```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mini-lsm'
    static_configs:
      - targets: ['mini-lsm-service:8080']
    metrics_path: '/metrics'
    scrape_interval: 5s

# grafana-dashboard.json
{
  "dashboard": {
    "title": "Mini-LSM Monitoring",
    "panels": [
      {
        "title": "Operations per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(storage_put_count_total[5m])",
            "legendFormat": "PUT"
          },
          {
            "expr": "rate(storage_get_count_total[5m])",
            "legendFormat": "GET"
          },
          {
            "expr": "rate(storage_delete_count_total[5m])",
            "legendFormat": "DELETE"
          }
        ]
      },
      {
        "title": "Latency (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(storage_put_latency_seconds_bucket[5m]))",
            "legendFormat": "PUT"
          },
          {
            "expr": "histogram_quantile(0.95, rate(storage_get_latency_seconds_bucket[5m]))",
            "legendFormat": "GET"
          }
        ]
      },
      {
        "title": "Storage Size",
        "type": "graph",
        "targets": [
          {
            "expr": "storage_total_size_bytes",
            "legendFormat": "Total Size"
          }
        ]
      }
    ]
  }
}
```

## 总结

通过遵循这些最佳实践，我们可以构建一个高性能、高可靠性的Mini-LSM存储系统：

1. **配置管理**: 使用结构化配置，支持环境变量和配置文件
2. **性能优化**: 实现缓存层、批量操作、连接池等优化策略
3. **监控运维**: 完善的监控指标、健康检查和备份恢复机制
4. **测试策略**: 全面的单元测试、集成测试和性能测试
5. **部署运维**: 容器化部署、Kubernetes配置和监控集成

这些实践可以帮助我们更好地使用和维护Mini-LSM存储引擎，确保系统在生产环境中的稳定性和性能。