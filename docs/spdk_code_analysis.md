# SPDK代码详细分析文档

## 1. 概述

SPDK（Storage Performance Development Kit）是Intel开发的高性能存储开发工具包。在Ceph中，SPDK被集成作为可选的块设备后端，用于直接访问NVMe设备，提供比传统内核驱动更高的I/O性能。

### 1.1 SPDK核心特性

- **用户空间驱动**：驱动程序运行在用户空间，避免内核态切换开销
- **轮询模式**：使用轮询而非中断来检查I/O完成，减少延迟和抖动
- **零拷贝**：直接内存访问，减少数据拷贝
- **多队列支持**：充分利用NVMe的多队列特性
- **基于DPDK**：使用DPDK提供的内存管理和PCIe设备访问

### 1.2 在Ceph中的用途

在Ceph中，SPDK主要用于：
- **BlueStore后端**：作为BlueStore的可选块设备后端（`NVMEDevice`）
- **高性能存储**：为需要极致性能的场景提供用户空间NVMe驱动
- **可选启用**：通过编译时选项`HAVE_SPDK`控制是否启用

## 2. 目录结构

```
src/spdk/
├── include/spdk/        # 公共头文件
│   ├── nvme.h          # NVMe驱动API
│   ├── env.h           # 环境抽象层
│   ├── bdev.h          # 块设备抽象层
│   └── ...
├── lib/                # 核心库
│   ├── nvme/           # NVMe驱动实现
│   ├── env_dpdk/       # DPDK环境实现
│   ├── bdev/           # 块设备抽象
│   ├── virtio/         # Virtio支持
│   └── vhost/          # vhost支持
├── module/             # 模块实现
│   ├── bdev/           # 块设备模块
│   └── event/          # 事件框架模块
├── app/                # 应用程序
│   ├── spdk_tgt/       # SPDK目标程序
│   ├── nvmf_tgt/       # NVMe-oF目标
│   └── vhost/          # vhost应用
├── dpdk/               # DPDK子模块
├── isa-l/              # Intel存储加速库
├── intel-ipsec-mb/     # Intel IPsec多缓冲区库
└── ocf/                # Open CAS Framework
```

## 3. 核心架构

### 3.1 用户空间驱动原理

#### 3.1.1 设备绑定

SPDK使用UIO或VFIO驱动来绑定PCIe设备：

1. **卸载内核驱动**：将设备从内核驱动（如`nvme`驱动）解绑
2. **绑定UIO/VFIO**：将设备绑定到UIO或VFIO驱动
3. **映射BAR空间**：将PCIe设备的BAR（Base Address Register）映射到用户空间
4. **直接MMIO访问**：通过内存映射I/O直接访问设备寄存器

#### 3.1.2 内存管理

SPDK使用DPDK提供的内存管理：

- **大页内存**：使用大页内存（hugepage）提高TLB效率
- **DMA安全内存**：分配DMA安全的内存用于I/O缓冲区
- **NUMA感知**：支持NUMA节点的内存分配

#### 3.1.3 轮询模式

- **无中断**：不使用硬件中断
- **主动轮询**：应用程序主动轮询完成队列
- **低延迟**：避免中断处理的开销和上下文切换

### 3.2 环境抽象层（Environment）

**文件位置**: `lib/env_dpdk/env.c`

环境抽象层提供：
- **内存管理**：`spdk_malloc()`, `spdk_dma_malloc()`等
- **PCIe设备访问**：设备枚举和绑定
- **线程管理**：线程抽象和CPU亲和性
- **时间服务**：高精度时间戳

**关键函数**：
```c
// 初始化环境
int spdk_env_init(struct spdk_env_opts *opts);

// DMA安全内存分配
void *spdk_dma_malloc(size_t size, size_t align, uint64_t *phys_addr);
void *spdk_dma_zmalloc(size_t size, size_t align, uint64_t *phys_addr);
void spdk_dma_free(void *buf);

// 虚拟地址到物理地址转换
uint64_t spdk_vtophys(void *buf, uint64_t *size);
```

### 3.3 NVMe驱动

**文件位置**: `lib/nvme/nvme.c`, `lib/nvme/nvme_ctrlr_cmd.c`, `lib/nvme/nvme_ns_cmd.c`

#### 3.3.1 控制器管理

**关键数据结构**：
```c
struct spdk_nvme_ctrlr;  // NVMe控制器
struct spdk_nvme_ns;     // 命名空间（Namespace）
struct spdk_nvme_qpair;  // 队列对（Queue Pair）
```

**控制器初始化流程**：
1. **探测设备**：`spdk_nvme_probe()`
2. **附加控制器**：`attach_cb`回调
3. **获取命名空间**：`spdk_nvme_ctrlr_get_ns()`
4. **创建队列对**：`spdk_nvme_ctrlr_alloc_io_qpair()`

#### 3.3.2 I/O操作

**读取操作**：
```c
int spdk_nvme_ns_cmd_read(
    struct spdk_nvme_ns *ns,
    struct spdk_nvme_qpair *qpair,
    void *buffer,
    uint64_t lba,
    uint32_t lba_count,
    spdk_nvme_cmd_cb cb_fn,
    void *cb_arg,
    uint32_t io_flags);
```

**写入操作**：
```c
int spdk_nvme_ns_cmd_write(
    struct spdk_nvme_ns *ns,
    struct spdk_nvme_qpair *qpair,
    void *buffer,
    uint64_t lba,
    uint32_t lba_count,
    spdk_nvme_cmd_cb cb_fn,
    void *cb_arg,
    uint32_t io_flags);
```

**完成处理**：
```c
int spdk_nvme_qpair_process_completions(
    struct spdk_nvme_qpair *qpair,
    uint32_t max_completions);
```

#### 3.3.3 传输类型

SPDK支持多种NVMe传输类型：
- **PCIe** (`SPDK_NVME_TRANSPORT_PCIE`): 本地PCIe设备
- **RDMA** (`SPDK_NVME_TRANSPORT_RDMA`): NVMe over Fabrics (RDMA)
- **TCP** (`SPDK_NVME_TRANSPORT_TCP`): NVMe over Fabrics (TCP)
- **FC** (`SPDK_NVME_TRANSPORT_FC`): NVMe over Fabrics (Fibre Channel)

### 3.4 块设备抽象层（BDEV）

**文件位置**: `lib/bdev/bdev.c`, `include/spdk/bdev.h`

BDEV（Block Device）提供统一的块设备抽象：

**关键数据结构**：
```c
struct spdk_bdev;        // 块设备
struct spdk_bdev_io;     // I/O请求
struct spdk_bdev_desc;   // 块设备描述符
```

**BDEV模块类型**：
- **Malloc BDEV**: 内存块设备
- **NVMe BDEV**: NVMe块设备
- **Virtio BDEV**: Virtio块设备
- **压缩BDEV**: 压缩块设备
- **加密BDEV**: 加密块设备
- **逻辑卷BDEV**: LVM逻辑卷

## 4. Ceph中的SPDK集成

### 4.1 NVMEDevice类

**文件位置**: `src/blk/spdk/NVMEDevice.h`, `src/blk/spdk/NVMEDevice.cc`

`NVMEDevice`是Ceph中SPDK NVMe驱动的封装类，继承自`BlockDevice`。

#### 4.1.1 类结构

```cpp
class NVMEDevice : public BlockDevice {
  SharedDriverData *driver;  // 共享驱动数据
  std::string name;          // 设备名称
  
public:
  // 设备操作
  int open(const std::string& path) override;
  void close() override;
  
  // I/O操作
  int read(uint64_t off, uint64_t len, bufferlist *pbl, ...) override;
  int write(uint64_t off, bufferlist& bl, ...) override;
  int aio_read(uint64_t off, uint64_t len, bufferlist *pbl, ...) override;
  int aio_write(uint64_t off, bufferlist& bl, ...) override;
  void aio_submit(IOContext *ioc) override;
  
  // 工具方法
  static bool support(const std::string& path);
  int collect_metadata(...) const override;
};
```

#### 4.1.2 关键数据结构

**SharedDriverData**:
```cpp
class SharedDriverData {
  unsigned id;
  spdk_nvme_transport_id trid;    // 传输ID
  spdk_nvme_ctrlr *ctrlr;         // NVMe控制器
  spdk_nvme_ns *ns;               // 命名空间
  uint32_t block_size;
  uint64_t size;
  std::thread admin_thread;       // 管理线程（非PCIe设备）
  std::vector<NVMEDevice*> registered_devices;
};
```

**SharedDriverQueueData**:
```cpp
class SharedDriverQueueData {
  NVMEDevice *bdev;
  SharedDriverData *driver;
  spdk_nvme_ctrlr *ctrlr;
  spdk_nvme_ns *ns;
  struct spdk_nvme_qpair *qpair;  // 队列对
  uint32_t current_queue_depth;   // 当前队列深度
  bi::slist<data_cache_buf> data_buf_list;  // 数据缓冲区列表
};
```

**Task**:
```cpp
struct Task {
  NVMEDevice *device;
  IOContext *ctx;
  IOCommand command;      // READ_COMMAND, WRITE_COMMAND, FLUSH_COMMAND
  uint64_t offset;
  uint64_t len;
  bufferlist bl;
  std::function<void()> fill_cb;
  IORequest io_request;   // I/O请求信息
  SharedDriverQueueData *queue;
  int ref;                // 引用计数
};
```

#### 4.1.3 设备打开流程

1. **检查设备路径**：通过`SPDK_PREFIX`（"spdk:"）前缀识别
2. **解析传输ID**：从文件读取`spdk_nvme_transport_id`
3. **获取/创建驱动**：通过`NVMEManager::try_get()`获取或创建驱动
4. **初始化环境**：如果是首次使用，初始化SPDK环境（DPDK线程）
5. **探测设备**：调用`spdk_nvme_probe()`探测NVMe设备
6. **注册设备**：将设备注册到`SharedDriverData`

**关键代码**：
```cpp
int NVMEDevice::open(const string& p) {
  // 1. 读取传输ID
  std::ifstream ifs(p);
  std::getline(ifs, val);
  spdk_nvme_transport_id trid;
  spdk_nvme_transport_id_parse(&trid, val.c_str());
  
  // 2. 获取驱动
  manager.try_get(trid, &driver);
  
  // 3. 注册设备
  driver->register_device(this);
  
  // 4. 设置设备属性
  block_size = driver->get_block_size();
  size = driver->get_size();
  name = trid.traddr;
}
```

#### 4.1.4 I/O处理流程

**写入流程**：

1. **创建Task**：`write_split()`将大的写入请求分割成多个Task
2. **分配缓冲区**：从数据缓冲区池分配DMA安全的内存
3. **拷贝数据**：将用户数据拷贝到DMA缓冲区
4. **提交I/O**：调用`spdk_nvme_ns_cmd_writev()`提交写入命令
5. **轮询完成**：`_aio_handle()`轮询完成队列
6. **回调处理**：I/O完成时调用`io_complete()`回调

**关键代码**：
```cpp
void SharedDriverQueueData::_aio_handle(Task *t, IOContext *ioc) {
  while (ioc->num_running) {
    // 轮询完成队列
    if (current_queue_depth) {
      r = spdk_nvme_qpair_process_completions(qpair, max_io_completion);
    }
    
    // 提交新的I/O
    for (; t; t = t->next) {
      if (t->command == IOCommand::WRITE_COMMAND) {
        alloc_buf_from_pool(t, true);  // 分配缓冲区
        spdk_nvme_ns_cmd_writev(
            ns, qpair, lba_off, lba_count, io_complete, t, 0,
            data_buf_reset_sgl, data_buf_next_sge);
      }
    }
  }
}
```

**完成回调**：
```cpp
static void io_complete(void *t, const struct spdk_nvme_cpl *completion) {
  Task *task = static_cast<Task*>(t);
  --queue->current_queue_depth;
  
  if (task->command == IOCommand::WRITE_COMMAND) {
    task->release_segs(queue);  // 释放缓冲区
    if (!--ctx->num_running) {
      task->device->aio_callback(...);  // 调用完成回调
    }
    delete task;
  }
}
```

#### 4.1.5 数据缓冲区管理

**缓冲区池**：
- 使用`spdk_dma_zmalloc()`分配DMA安全的内存
- 默认池大小：1024个缓冲区，每个8KB
- 使用`boost::intrusive::slist`管理空闲缓冲区

**缓冲区分配**：
```cpp
int SharedDriverQueueData::alloc_buf_from_pool(Task *t, bool write) {
  uint64_t count = t->len / data_buffer_size;
  // 从池中分配缓冲区
  for (uint16_t i = 0; i < count; i++) {
    segs[i] = &data_buf_list.front();
    data_buf_list.pop_front();
  }
  // 如果是写入，拷贝数据到缓冲区
  if (write) {
    auto blp = t->bl.begin();
    for (uint16_t i = 0; i < count; ++i) {
      blp.copy(data_buffer_size, static_cast<char*>(segs[i]));
    }
  }
}
```

#### 4.1.6 NVMEManager

**作用**：管理SPDK环境和NVMe设备

**关键功能**：
- **环境初始化**：在独立线程中初始化SPDK环境
- **设备管理**：管理`SharedDriverData`列表
- **设备探测**：处理设备探测请求队列

**初始化流程**：
```cpp
int NVMEManager::try_get(const spdk_nvme_transport_id& trid, ...) {
  // 如果DPDK线程未启动，启动它
  if (!dpdk_thread.joinable()) {
    dpdk_thread = std::thread([this, ...]() {
      struct spdk_env_opts opts;
      spdk_env_opts_init(&opts);
      opts.name = "nvme-device-manager";
      opts.core_mask = coremask_arg.c_str();
      opts.mem_size = mem_size_arg;
      spdk_env_init(&opts);  // 初始化环境
      
      // 处理探测队列
      while (!stopping) {
        if (!probe_queue.empty()) {
          spdk_nvme_probe(..., probe_cb, attach_cb, NULL);
        }
      }
    });
  }
}
```

### 4.2 BlueStore集成

**文件位置**: `src/os/bluestore/BlueStore.cc`

BlueStore通过设备路径前缀识别SPDK设备：

```cpp
// 检查是否是SPDK设备
if (!epath.compare(0, strlen(SPDK_PREFIX), SPDK_PREFIX)) {
  // 处理SPDK设备路径
  string trid = epath.substr(strlen(SPDK_PREFIX));
  // ...
}
```

**设备路径格式**：
- SPDK设备路径格式：`spdk:<transport_id>`
- 例如：`spdk:0000:01:00.0` (PCIe设备)
- Transport ID存储在文件中，由BlueStore读取

## 5. 核心库详解

### 5.1 lib/nvme - NVMe驱动库

#### 5.1.1 控制器管理

**文件**: `lib/nvme/nvme.c`

**关键函数**：
- `spdk_nvme_probe()`: 探测NVMe设备
- `spdk_nvme_ctrlr_alloc_io_qpair()`: 分配I/O队列对
- `spdk_nvme_ctrlr_free_io_qpair()`: 释放I/O队列对
- `spdk_nvme_ctrlr_process_admin_completions()`: 处理管理命令完成

#### 5.1.2 命名空间操作

**文件**: `lib/nvme/nvme_ns_cmd.c`

**关键函数**：
- `spdk_nvme_ns_cmd_read()`: 读取命令
- `spdk_nvme_ns_cmd_write()`: 写入命令
- `spdk_nvme_ns_cmd_writev()`: 分散/聚集写入
- `spdk_nvme_ns_cmd_readv()`: 分散/聚集读取
- `spdk_nvme_ns_cmd_flush()`: 刷新命令

**命令拆分**：
- 大I/O请求会被自动拆分成多个子请求
- 支持PRP（Physical Region Page）和SGL（Scatter Gather List）描述符

#### 5.1.3 队列对处理

**轮询完成**：
```c
int spdk_nvme_qpair_process_completions(
    struct spdk_nvme_qpair *qpair,
    uint32_t max_completions);
```

**特点**：
- 非阻塞轮询
- 可以限制每次处理的最大完成数
- 返回处理的完成数

### 5.2 lib/env_dpdk - DPDK环境实现

**文件**: `lib/env_dpdk/env.c`

**内存管理**：
- 基于DPDK的`rte_malloc`
- 支持NUMA感知
- 支持大页内存
- DMA安全的内存分配

**PCIe设备管理**：
- 设备枚举
- 设备绑定（UIO/VFIO）
- BAR空间映射

### 5.3 lib/virtio - Virtio支持

**文件**: `lib/virtio/virtio.c`

提供Virtio设备支持：
- **Virtio PCI**: PCIe上的Virtio设备
- **Virtio User**: 用户空间Virtio设备（通过vhost-user）

### 5.4 lib/vhost - vhost支持

**文件**: `lib/vhost/vhost.c`

提供vhost支持：
- **vhost-scsi**: SCSI设备模拟
- **vhost-blk**: 块设备模拟
- **vhost-nvme**: NVMe设备模拟

### 5.5 lib/bdev - 块设备抽象

**文件**: `lib/bdev/bdev.c`

提供统一的块设备接口：
- **BDEV模块系统**：插件化的块设备后端
- **I/O队列**：管理I/O请求队列
- **QoS控制**：I/O限速

## 6. 模块系统

### 6.1 module/bdev - 块设备模块

各种块设备后端实现：
- **bdev_nvme**: NVMe块设备
- **bdev_malloc**: 内存块设备
- **bdev_virtio**: Virtio块设备
- **bdev_compress**: 压缩块设备
- **bdev_crypto**: 加密块设备
- **bdev_raid**: RAID块设备

### 6.2 module/event - 事件框架

事件驱动的应用框架：
- **反应器模式**：事件循环
- **异步I/O**：基于事件的异步I/O
- **线程模型**：每个线程一个反应器

## 7. 性能特性

### 7.1 零拷贝

- 使用DMA安全的内存
- 减少数据拷贝次数
- 直接内存访问

### 7.2 多队列

- 每个线程一个队列对
- 无锁设计
- 充分利用NVMe多队列特性

### 7.3 轮询模式

- 无中断延迟
- 可预测的延迟
- 适合低延迟应用

### 7.4 CPU亲和性

- 绑定CPU核心
- 减少上下文切换
- 提高缓存命中率

## 8. 配置和使用

### 8.1 编译配置

**启用SPDK支持**：
- 编译时选项：`HAVE_SPDK`
- 需要DPDK库
- 需要SPDK库

### 8.2 运行时配置

**Ceph配置选项**：
- `bluestore_spdk_coremask`: SPDK使用的CPU核心掩码
- `bluestore_spdk_mem`: SPDK内存大小
- `bluestore_spdk_max_io_completion`: 最大I/O完成数
- `bluestore_spdk_io_sleep`: I/O轮询休眠时间（微秒）

### 8.3 设备路径

**SPDK设备路径格式**：
```
spdk:<transport_id>
```

**Transport ID格式**：
- PCIe: `PCIe:0000:01:00.0`
- RDMA: `RDMA:192.168.1.1:4420`
- TCP: `TCP:192.168.1.1:4420`

**示例**：
```
/dev/nvme0n1 -> spdk:PCIe:0000:01:00.0
```

## 9. 数据流程图

### 9.1 写入流程

```
应用程序
    ↓
BlueStore::_do_write()
    ↓
NVMEDevice::aio_write()
    ↓
write_split() [创建Task]
    ↓
NVMEDevice::aio_submit()
    ↓
SharedDriverQueueData::_aio_handle()
    ↓
alloc_buf_from_pool() [分配DMA缓冲区]
    ↓ [拷贝数据到缓冲区]
spdk_nvme_ns_cmd_writev() [提交写入命令]
    ↓
spdk_nvme_qpair_process_completions() [轮询完成]
    ↓
io_complete() [完成回调]
    ↓
释放缓冲区，调用上层回调
```

### 9.2 读取流程

```
应用程序
    ↓
BlueStore::_do_read()
    ↓
NVMEDevice::aio_read()
    ↓
make_read_tasks() [创建Task]
    ↓
NVMEDevice::aio_submit()
    ↓
SharedDriverQueueData::_aio_handle()
    ↓
alloc_buf_from_pool() [分配DMA缓冲区]
    ↓
spdk_nvme_ns_cmd_readv() [提交读取命令]
    ↓
spdk_nvme_qpair_process_completions() [轮询完成]
    ↓
io_complete() [完成回调]
    ↓
fill_cb() [拷贝数据到用户缓冲区]
    ↓
释放缓冲区，调用上层回调
```

## 10. 关键代码路径

### 10.1 设备初始化

```
NVMEDevice::open()
  → NVMEManager::try_get()
    → [启动DPDK线程]
      → spdk_env_init()
      → spdk_nvme_probe()
        → probe_cb()
        → attach_cb()
          → NVMEManager::register_ctrlr()
            → new SharedDriverData()
```

### 10.2 I/O提交

```
NVMEDevice::aio_write()
  → write_split()
    → new Task()
    → ioc_append_task()
  → NVMEDevice::aio_submit()
    → SharedDriverQueueData::_aio_handle()
      → alloc_buf_from_pool()
      → spdk_nvme_ns_cmd_writev()
      → [轮询完成队列]
```

### 10.3 I/O完成

```
[硬件完成I/O]
  → spdk_nvme_qpair_process_completions()
    → io_complete()
      → Task::release_segs()
      → [调用上层回调]
```

## 11. 内存管理

### 11.1 DMA安全内存

- **分配**：`spdk_dma_zmalloc()`
- **对齐**：页对齐（4KB）
- **NUMA感知**：根据CPU核心选择NUMA节点

### 11.2 缓冲区池

- **预分配**：启动时预分配缓冲区池
- **复用**：I/O完成后缓冲区回到池中
- **大小**：默认1024个缓冲区，每个8KB

### 11.3 大页内存

- **配置**：通过DPDK配置大页内存
- **好处**：减少TLB缺失，提高性能
- **大小**：通常2MB或1GB

## 12. 线程模型

### 12.1 DPDK线程

- **独立线程**：SPDK环境运行在独立线程中
- **CPU绑定**：可以绑定到特定CPU核心
- **轮询循环**：持续轮询I/O完成

### 12.2 应用线程

- **每个线程一个队列对**：避免锁竞争
- **线程本地存储**：`thread_local SharedDriverQueueData`
- **异步I/O**：非阻塞I/O操作

## 13. 错误处理

### 13.1 设备错误

- **控制器错误**：检测控制器状态
- **命名空间错误**：检测命名空间状态
- **传输错误**：处理网络传输错误

### 13.2 I/O错误

- **完成状态检查**：`spdk_nvme_cpl_is_error()`
- **错误代码**：从完成结构体获取错误码
- **重试机制**：由上层应用处理重试

## 14. 性能优化

### 14.1 队列深度

- **最大队列深度**：队列大小-1（避免溢出）
- **动态调整**：根据负载调整
- **背压机制**：队列满时等待

### 14.2 I/O合并

- **请求合并**：合并相邻的I/O请求
- **向量I/O**：使用`writev`/`readv`减少系统调用

### 14.3 轮询优化

- **批量处理**：一次处理多个完成
- **自适应休眠**：队列空时短暂休眠
- **CPU占用控制**：通过配置控制CPU使用

## 15. 限制和注意事项

### 15.1 设备独占

- SPDK设备被绑定后，内核无法访问
- 需要确保设备未被其他程序使用

### 15.2 内存要求

- 需要大页内存
- DMA缓冲区占用内存
- 配置足够的内存大小

### 15.3 CPU使用

- 轮询模式占用CPU
- 需要专用CPU核心
- 不适合CPU资源紧张的场景

### 15.4 兼容性

- 需要特定硬件（NVMe设备）
- 需要特定的内核版本和驱动
- 不同传输类型有不同的要求

## 16. 调试和监控

### 16.1 日志

- SPDK日志级别可配置
- Ceph日志集成SPDK日志
- 使用`dout`输出调试信息

### 16.2 性能统计

- I/O延迟统计
- 吞吐量统计
- 队列深度统计

### 16.3 工具

- `spdk_tgt`: SPDK目标程序
- `spdk_lspci`: 列出PCIe设备
- `spdk_top`: 性能监控工具

## 17. 总结

SPDK在Ceph中作为可选的高性能块设备后端，主要特点：

1. **高性能**：用户空间驱动，轮询模式，零拷贝
2. **低延迟**：无中断，直接设备访问
3. **可扩展**：多队列，多线程
4. **可选**：编译时和运行时可选
5. **集成**：通过NVMEDevice类与BlueStore集成

适用场景：
- 需要极致性能的NVMe存储
- 低延迟要求高的应用
- CPU资源充足的系统
- 专用的高性能存储节点

## 18. 参考代码位置

- **Ceph集成**: `cephMain/src/blk/spdk/NVMEDevice.h/cc`
- **SPDK核心**: `cephMain/src/spdk/lib/nvme/`
- **环境抽象**: `cephMain/src/spdk/lib/env_dpdk/`
- **块设备抽象**: `cephMain/src/spdk/lib/bdev/`
- **头文件**: `cephMain/src/spdk/include/spdk/`
- **BlueStore集成**: `cephMain/src/os/bluestore/BlueStore.cc`

## 19. 相关文档

- SPDK官方文档: http://www.spdk.io/doc/
- NVMe规范: http://nvmexpress.org/
- DPDK文档: http://dpdk.org/
- Ceph BlueStore文档: Ceph官方文档
