# OCF (Open CAS Framework) 子模块分析文档

## 目录
1. [概述](#概述)
2. [架构设计](#架构设计)
3. [核心概念](#核心概念)
4. [缓存模式](#缓存模式)
5. [策略机制](#策略机制)
6. [I/O处理流程](#io处理流程)
7. [元数据管理](#元数据管理)
8. [环境抽象层](#环境抽象层)
9. [统计信息](#统计信息)
10. [主要API](#主要api)

---

## 概述

### OCF简介

**Open CAS Framework (OCF)** 是一个高性能的块存储缓存元库，用C语言编写。它是完全平台和系统无关的，通过用户提供的环境包装层访问系统API。OCF与软件栈的其余部分紧密集成，提供无缺陷、高性能、低延迟的缓存工具。

### 关键特性

1. **平台无关**: 通过环境抽象层实现平台独立性
2. **高性能**: 优化的缓存算法和数据结构
3. **低延迟**: 零拷贝设计和异步I/O
4. **可扩展**: 支持多个缓存实例和核心设备
5. **灵活配置**: 支持多种缓存模式、驱逐策略、清理策略

### 设计目标

- 提供统一的缓存抽象接口
- 支持多种缓存实现策略
- 最小化性能开销
- 易于集成到现有系统

---

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────┐
│          Application Layer              │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          OCF API Layer                  │
│  (ocf_cache, ocf_core, ocf_io, etc.)   │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          OCF Core Engine                │
│  (Cache Engine, Metadata, Policies)    │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          Environment Abstraction        │
│  (Volume, Queue, Memory, Threading)     │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          System Layer                   │
│  (POSIX, SPDK, etc.)                    │
└─────────────────────────────────────────┘
```

### 目录结构

```
ocf/
├── inc/              # 公共头文件
│   ├── ocf.h         # 主头文件
│   ├── ocf_cache.h  # 缓存API
│   ├── ocf_core.h   # 核心设备API
│   ├── ocf_io.h     # I/O API
│   ├── ocf_volume.h # 卷API
│   ├── ocf_mngt.h   # 管理API
│   ├── ocf_stats.h  # 统计API
│   └── ...
├── src/              # 源代码实现
│   ├── ocf_cache.c  # 缓存实现
│   ├── ocf_core.c   # 核心设备实现
│   ├── ocf_io.c     # I/O实现
│   ├── engine/       # 缓存引擎
│   ├── metadata/     # 元数据管理
│   ├── eviction/    # 驱逐策略
│   ├── cleaning/    # 清理策略
│   ├── promotion/   # 提升策略
│   └── ...
├── env/              # 环境抽象层
│   └── posix/       # POSIX环境实现
├── example/          # 示例代码
└── tests/            # 测试代码
```

---

## 核心概念

### 1. Context (上下文)

**ocf_ctx_t**: OCF上下文对象，管理所有缓存实例

```c
typedef struct ocf_ctx *ocf_ctx_t;
```

**功能**:
- 管理所有缓存实例
- 提供环境抽象接口
- 管理资源分配

### 2. Cache (缓存)

**ocf_cache_t**: 缓存设备对象

```c
typedef struct ocf_cache *ocf_cache_t;
```

**关键属性**:
- 缓存设备卷 (Cache Volume)
- 缓存模式 (Cache Mode)
- 缓存行大小 (Cache Line Size)
- 关联的核心设备列表
- 元数据信息

**缓存信息结构**:
```c
struct ocf_cache_info {
    bool attached;                    // 是否附加了缓存设备
    uint8_t volume_type;              // 缓存卷类型
    uint32_t size;                     // 缓存大小（缓存行数）
    uint32_t occupancy;                // 占用率
    uint32_t dirty;                    // 脏数据量
    ocf_cache_mode_t cache_mode;       // 缓存模式
    ocf_eviction_t eviction_policy;    // 驱逐策略
    ocf_cleaning_t cleaning_policy;    // 清理策略
    ocf_promotion_t promotion_policy;  // 提升策略
    ocf_cache_line_size_t cache_line_size; // 缓存行大小
    uint32_t core_count;               // 核心设备数量
};
```

### 3. Core (核心设备)

**ocf_core_t**: 被缓存的核心设备对象

```c
typedef struct ocf_core *ocf_core_t;
```

**关键属性**:
- 核心设备卷 (Core Volume)
- 核心名称
- 核心UUID
- 关联的缓存实例
- 顺序截断策略

**核心信息结构**:
```c
struct ocf_core_info {
    uint64_t core_size;                // 核心大小（缓存行单位）
    uint64_t core_size_bytes;          // 核心大小（字节）
    uint32_t flushed;                  // 已刷新块数
    uint32_t dirty;                    // 脏块数
    uint32_t dirty_for;                // 脏数据持续时间（秒）
    uint32_t seq_cutoff_threshold;     // 顺序截断阈值
    ocf_seq_cutoff_policy seq_cutoff_policy; // 顺序截断策略
};
```

### 4. Volume (卷)

**ocf_volume_t**: 存储卷抽象

```c
typedef struct ocf_volume *ocf_volume_t;
```

**卷操作接口**:
```c
struct ocf_volume_ops {
    void (*submit_io)(struct ocf_io *io);
    void (*submit_flush)(struct ocf_io *io);
    void (*submit_metadata)(struct ocf_io *io);
    void (*submit_discard)(struct ocf_io *io);
    void (*submit_write_zeroes)(struct ocf_io *io);
    int (*open)(ocf_volume_t volume, void *volume_params);
    void (*close)(ocf_volume_t volume);
    unsigned int (*get_max_io_size)(ocf_volume_t volume);
    uint64_t (*get_length)(ocf_volume_t volume);
};
```

**卷类型**:
- 缓存卷 (Cache Volume): 缓存设备
- 核心卷 (Core Volume): 被缓存的后端设备
- 前端卷 (Front Volume): 暴露给应用的虚拟卷

### 5. IO (I/O请求)

**ocf_io**: I/O请求对象

```c
struct ocf_io {
    uint64_t addr;          // I/O目标地址
    uint64_t flags;         // I/O标志
    uint32_t bytes;         // I/O大小（字节）
    uint32_t io_class;      // I/O类别
    uint32_t dir;           // I/O方向（读/写）
    ocf_queue_t io_queue;   // I/O队列
    ocf_start_io_t start;  // 开始回调
    ocf_handle_io_t handle;// 处理回调
    ocf_end_io_t end;      // 完成回调
    void *priv1;           // 私有数据1
    void *priv2;           // 私有数据2
};
```

### 6. Queue (队列)

**ocf_queue_t**: I/O队列对象

```c
typedef struct ocf_queue *ocf_queue_t;
```

**功能**:
- 管理I/O请求队列
- 提供异步I/O处理
- 支持多线程并发

---

## 缓存模式

OCF支持多种缓存模式，每种模式有不同的写入策略：

### 1. Write-Through (WT) - 写透模式

**特点**:
- 写入同时写入缓存和后端
- 读取命中时从缓存读取
- 数据一致性最好
- 写入延迟较高

**适用场景**:
- 需要强一致性的场景
- 写入性能要求不高的场景

### 2. Write-Back (WB) - 写回模式

**特点**:
- 写入只写入缓存，标记为脏
- 后台异步刷新到后端
- 读取命中时从缓存读取
- 写入延迟最低
- 需要清理策略管理脏数据

**适用场景**:
- 写入性能要求高的场景
- 可以容忍短暂数据不一致的场景

### 3. Write-Around (WA) - 写旁路模式

**特点**:
- 写入直接写入后端，不写入缓存
- 读取命中时从缓存读取
- 避免缓存污染
- 写入性能中等

**适用场景**:
- 写入数据访问模式不规律的场景
- 避免缓存被写入数据污染

### 4. Pass-Through (PT) - 透传模式

**特点**:
- 所有I/O直接透传到后端
- 不使用缓存
- 用于故障恢复或测试

**适用场景**:
- 缓存故障时的降级模式
- 性能测试基准

### 5. Write-Invalidate (WI) - 写无效模式

**特点**:
- 写入时使缓存中对应数据无效
- 读取时从后端读取并填充缓存
- 避免脏数据

**适用场景**:
- 写入后很少再次读取的场景

### 6. Write-Only (WO) - 只写模式

**特点**:
- 只缓存写入数据
- 读取直接从后端读取
- 用于写入加速

**适用场景**:
- 写入密集型工作负载

### 缓存模式定义

```c
typedef enum {
    ocf_cache_mode_wt = 0,      // Write-Through
    ocf_cache_mode_wb,           // Write-Back
    ocf_cache_mode_wa,           // Write-Around
    ocf_cache_mode_pt,           // Pass-Through
    ocf_cache_mode_wi,           // Write-Invalidate
    ocf_cache_mode_wo,           // Write-Only
    ocf_cache_mode_max,
    ocf_cache_mode_default = ocf_cache_mode_wt,
    ocf_cache_mode_none = -1,
} ocf_cache_mode_t;
```

---

## 策略机制

### 1. 驱逐策略 (Eviction Policy)

驱逐策略决定当缓存满时，哪些缓存行被驱逐以腾出空间。

#### LRU (Least Recently Used)

**特点**:
- 最近最少使用的缓存行被驱逐
- 实现简单，效果良好
- 适合大多数工作负载

**实现**:
- 使用双向链表维护访问顺序
- 每次访问时移动到链表头部
- 驱逐时从链表尾部选择

```c
typedef enum {
    ocf_eviction_lru = 0,    // LRU策略
    ocf_eviction_max,
} ocf_eviction_t;
```

### 2. 清理策略 (Cleaning Policy)

清理策略决定何时将脏数据刷新到后端设备。

#### ALRU (Adaptive LRU)

**特点**:
- 基于LRU的自适应清理
- 根据缓存占用率调整清理频率
- 平衡性能和一致性

**参数**:
- 唤醒时间 (Wake Up Time)
- 停滞时间 (Stall Time)
- 刷新阈值 (Flush Threshold)

#### ACP (Always Clean Policy)

**特点**:
- 总是保持缓存干净
- 立即刷新脏数据
- 适合需要强一致性的场景

#### NOP (No Operation)

**特点**:
- 不主动清理
- 依赖显式刷新操作
- 用于测试或特殊场景

```c
typedef enum {
    ocf_cleaning_alru = 0,   // ALRU策略
    ocf_cleaning_acp,        // ACP策略
    ocf_cleaning_nop,        // NOP策略
    ocf_cleaning_max,
} ocf_cleaning_t;
```

### 3. 提升策略 (Promotion Policy)

提升策略决定哪些数据应该被提升到缓存中。

#### NHIT (N-Hit)

**特点**:
- 数据被访问N次后才提升到缓存
- 避免一次性访问污染缓存
- 提高缓存命中率

**参数**:
- 命中阈值 (Hit Threshold)

```c
typedef enum {
    ocf_promotion_nhit = 0,  // N-Hit策略
    ocf_promotion_max,
} ocf_promotion_t;
```

### 4. 顺序截断策略 (Sequential Cutoff Policy)

顺序截断策略决定是否缓存顺序I/O。

#### 策略类型

```c
typedef enum {
    ocf_seq_cutoff_policy_always = 0,  // 总是截断
    ocf_seq_cutoff_policy_full,        // 缓存满时截断
    ocf_seq_cutoff_policy_never,       // 不截断
    ocf_seq_cutoff_policy_max,
    ocf_seq_cutoff_policy_default = ocf_seq_cutoff_policy_full,
} ocf_seq_cutoff_policy;
```

**特点**:
- 顺序I/O通常只访问一次，缓存价值低
- 截断顺序I/O可以避免缓存污染
- 提高随机I/O的缓存命中率

---

## I/O处理流程

### 读取流程

```
1. 应用发起读取请求
   ↓
2. OCF接收I/O请求 (ocf_core_new_io)
   ↓
3. 查找缓存行 (Metadata Lookup)
   ↓
4. 缓存命中?
   ├─ 是 → 从缓存读取 (engine_fast)
   │        ↓
   │      完成I/O
   │
   └─ 否 → 从后端读取 (engine_rd)
           ↓
          填充缓存 (Promotion Policy)
           ↓
          完成I/O
```

### 写入流程 (Write-Back模式)

```
1. 应用发起写入请求
   ↓
2. OCF接收I/O请求
   ↓
3. 查找缓存行
   ↓
4. 缓存命中?
   ├─ 是 → 写入缓存 (engine_wb)
   │        ↓
   │      标记为脏
   │        ↓
   │      完成I/O
   │
   └─ 否 → 分配缓存行 (Eviction Policy)
           ↓
          写入缓存
           ↓
          标记为脏
           ↓
          完成I/O
```

### 写入流程 (Write-Through模式)

```
1. 应用发起写入请求
   ↓
2. OCF接收I/O请求
   ↓
3. 查找缓存行
   ↓
4. 缓存命中?
   ├─ 是 → 写入缓存和后端 (engine_wt)
   │        ↓
   │      等待后端完成
   │        ↓
   │      完成I/O
   │
   └─ 否 → 直接写入后端 (engine_wa)
           ↓
          完成I/O
```

### 缓存引擎

OCF使用不同的引擎处理不同类型的I/O：

- **engine_fast**: 快速路径（缓存命中）
- **engine_rd**: 读取未命中
- **engine_wb**: 写回模式写入
- **engine_wt**: 写透模式写入
- **engine_wa**: 写旁路模式写入
- **engine_wi**: 写无效模式写入
- **engine_wo**: 只写模式写入
- **engine_pt**: 透传模式
- **engine_discard**: 丢弃操作
- **engine_zero**: 写零操作

---

## 元数据管理

### 元数据结构

OCF使用元数据来跟踪缓存状态：

1. **缓存行元数据**:
   - 缓存行状态（有效、脏、无效）
   - 核心设备ID
   - 核心设备地址映射
   - 访问时间戳

2. **分区元数据**:
   - 分区分配信息
   - 分区统计信息

3. **核心元数据**:
   - 核心设备信息
   - 核心设备映射表

### 元数据存储

**动态元数据**:
- 存储在内存中
- 快速访问
- 支持原子操作

**持久化元数据**:
- 存储在缓存设备上
- 支持缓存恢复
- 包含超级块信息

### 元数据操作

```c
// 元数据查找
ocf_metadata_get_core_info(...)

// 元数据更新
ocf_metadata_set_core_info(...)

// 元数据刷新
ocf_metadata_flush(...)
```

---

## 环境抽象层

### 环境接口

OCF通过环境抽象层实现平台无关性：

```c
// 内存管理
env_malloc()
env_free()
env_vmalloc()

// 线程同步
env_mutex_init()
env_mutex_lock()
env_mutex_unlock()

// 原子操作
env_atomic_read()
env_atomic_set()
env_atomic_inc()

// 时间管理
env_get_tick_count()
env_ticks_to_nsec()

// 日志
env_log()
```

### POSIX环境实现

`env/posix/` 目录提供了基于POSIX的环境实现：

- 使用标准C库函数
- 使用pthread进行线程管理
- 使用标准内存分配函数

### SPDK环境集成

OCF可以集成到SPDK环境中：

- 使用SPDK的块设备作为卷
- 使用SPDK的线程模型
- 使用SPDK的内存管理

---

## 统计信息

### 统计类型

OCF提供详细的统计信息：

#### 1. 使用统计 (Usage Statistics)

```c
struct ocf_stats_usage {
    struct ocf_stat occupancy;  // 占用率
    struct ocf_stat free;       // 空闲
    struct ocf_stat clean;      // 干净数据
    struct ocf_stat dirty;      // 脏数据
};
```

#### 2. 请求统计 (Request Statistics)

```c
struct ocf_stats_requests {
    struct ocf_stat rd_hits;           // 读命中
    struct ocf_stat rd_partial_misses; // 读部分未命中
    struct ocf_stat rd_full_misses;    // 读完全未命中
    struct ocf_stat rd_total;          // 读总数
    struct ocf_stat wr_hits;           // 写命中
    struct ocf_stat wr_partial_misses; // 写部分未命中
    struct ocf_stat wr_full_misses;    // 写完全未命中
    struct ocf_stat wr_total;          // 写总数
    struct ocf_stat rd_pt;             // 读透传
    struct ocf_stat wr_pt;              // 写透传
    struct ocf_stat serviced;          // 已服务请求
    struct ocf_stat total;              // 总请求数
};
```

#### 3. 块统计 (Block Statistics)

```c
struct ocf_stats_blocks {
    struct ocf_stat core_volume_rd;    // 从核心卷读取
    struct ocf_stat core_volume_wr;    // 写入核心卷
    struct ocf_stat cache_volume_rd;   // 从缓存卷读取
    struct ocf_stat cache_volume_wr;   // 写入缓存卷
    struct ocf_stat volume_rd;          // 总读取
    struct ocf_stat volume_wr;          // 总写入
};
```

#### 4. 错误统计 (Error Statistics)

```c
struct ocf_stats_errors {
    struct ocf_stat core_volume_rd;    // 核心卷读错误
    struct ocf_stat core_volume_wr;    // 核心卷写错误
    struct ocf_stat cache_volume_rd;   // 缓存卷读错误
    struct ocf_stat cache_volume_wr;   // 缓存卷写错误
    struct ocf_stat total;             // 总错误数
};
```

### 统计API

```c
// 获取缓存统计
int ocf_stats_collect_cache(ocf_cache_t cache, 
                            struct ocf_stats *stats);

// 获取核心统计
int ocf_stats_collect_core(ocf_core_t core, 
                           struct ocf_stats *stats);

// 重置统计
int ocf_stats_reset_cache(ocf_cache_t cache);
int ocf_stats_reset_core(ocf_core_t core);
```

---

## 主要API

### 上下文管理

```c
// 创建上下文
int ocf_mngt_ctx_init(ocf_ctx_t *ctx, const struct ocf_ctx_config *cfg);

// 销毁上下文
void ocf_mngt_ctx_cleanup(ocf_ctx_t ctx);
```

### 缓存管理

```c
// 启动缓存
int ocf_mngt_cache_start(ocf_ctx_t ctx, const struct ocf_mngt_cache_config *cfg,
                        ocf_cache_t *cache);

// 停止缓存
void ocf_mngt_cache_stop(ocf_cache_t cache);

// 附加缓存设备
int ocf_mngt_cache_attach(ocf_cache_t cache, 
                          struct ocf_mngt_cache_attach_config *cfg);

// 分离缓存设备
void ocf_mngt_cache_detach(ocf_cache_t cache);
```

### 核心管理

```c
// 添加核心
int ocf_mngt_core_add(ocf_cache_t cache, 
                     const struct ocf_mngt_core_config *cfg,
                     ocf_core_t *core);

// 移除核心
void ocf_mngt_core_remove(ocf_core_t core);

// 获取核心
int ocf_core_get_by_name(ocf_cache_t cache, const char *name, 
                        size_t name_len, ocf_core_t *core);
```

### I/O操作

```c
// 创建I/O请求
struct ocf_io *ocf_core_new_io(ocf_core_t core, ocf_queue_t queue,
                              uint64_t addr, uint32_t bytes, 
                              uint32_t dir, uint32_t io_class, 
                              uint64_t flags);

// 提交I/O请求
void ocf_volume_submit_io(struct ocf_io *io);

// 完成I/O请求
void ocf_io_put(struct ocf_io *io);
```

### 配置管理

```c
// 设置缓存模式
int ocf_mngt_cache_set_mode(ocf_cache_t cache, ocf_cache_mode_t mode);

// 设置清理策略
int ocf_mngt_cache_set_cleaning_policy(ocf_cache_t cache, 
                                       ocf_cleaning_t type);

// 设置驱逐策略
int ocf_mngt_cache_set_eviction_policy(ocf_cache_t cache, 
                                       ocf_eviction_t type);

// 设置提升策略
int ocf_mngt_core_set_promotion_policy(ocf_core_t core, 
                                       ocf_promotion_t type);
```

---

## 关键定义

### 缓存定义

```c
#define OCF_CACHE_ID_MIN 1
#define OCF_CACHE_ID_MAX 16384
#define OCF_CACHE_SIZE_MIN (20 * MiB)
#define OCF_CACHE_NAME_SIZE 32
```

### 核心定义

```c
#define OCF_CORE_MAX OCF_CONFIG_MAX_CORES
#define OCF_CORE_ID_MIN 0
#define OCF_CORE_ID_MAX (OCF_CORE_MAX - 1)
#define OCF_CORE_NAME_SIZE 32
```

### 缓存行大小

```c
typedef enum {
    ocf_cache_line_size_4 = 0,   // 4 KiB
    ocf_cache_line_size_8,       // 8 KiB
    ocf_cache_line_size_16,      // 16 KiB
    ocf_cache_line_size_32,      // 32 KiB
    ocf_cache_line_size_64,      // 64 KiB
    ocf_cache_line_size_max,
} ocf_cache_line_size_t;
```

---

## 状态管理

### 缓存状态

```c
typedef enum {
    ocf_cache_state_running = 0,      // 运行中
    ocf_cache_state_stopping = 1,     // 停止中
    ocf_cache_state_initializing = 2, // 初始化中
    ocf_cache_state_incomplete = 3,   // 不完整（有非活跃核心）
    ocf_cache_state_max,
} ocf_cache_state_t;
```

### 核心状态

```c
typedef enum {
    ocf_core_state_active = 0,    // 活跃
    ocf_core_state_inactive,      // 非活跃
    ocf_core_state_max,
} ocf_core_state_t;
```

---

## 并发控制

### 锁机制

OCF使用多种锁机制保证并发安全：

1. **缓存管理锁**: 保护缓存配置操作
2. **元数据锁**: 保护元数据访问
3. **I/O队列锁**: 保护I/O队列操作
4. **缓存行锁**: 保护单个缓存行操作

### 异步操作

OCF大量使用异步操作：

- 异步I/O提交
- 异步回调完成
- 异步清理操作
- 异步元数据更新

---

## 总结

OCF是一个功能完整、高性能的块存储缓存框架：

1. **架构设计**: 通过环境抽象层实现平台无关性
2. **核心概念**: Cache、Core、Volume、IO、Queue
3. **缓存模式**: 支持6种缓存模式（WT、WB、WA、PT、WI、WO）
4. **策略机制**: 支持多种驱逐、清理、提升策略
5. **I/O处理**: 优化的I/O路径和引擎
6. **元数据管理**: 高效的元数据存储和访问
7. **统计信息**: 详细的性能和使用统计
8. **并发控制**: 完善的锁机制和异步操作

OCF的设计使其能够轻松集成到各种存储系统中，包括SPDK，提供高性能的缓存加速功能。

---

**文档版本**: 1.0  
**最后更新**: 2024年
