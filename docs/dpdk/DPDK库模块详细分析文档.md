# DPDK库模块详细分析文档

## 目录
1. [概述](#概述)
2. [核心基础库](#核心基础库)
3. [数据结构库](#数据结构库)
4. [网络处理库](#网络处理库)
5. [安全与加密库](#安全与加密库)
6. [事件与调度库](#事件与调度库)
7. [工具与辅助库](#工具与辅助库)
8. [模块依赖关系](#模块依赖关系)

---

## 概述

DPDK的`lib`目录包含了所有核心库模块，这些库提供了从底层环境抽象到高层应用接口的完整功能栈。根据构建顺序和依赖关系，这些库可以分为以下几个层次：

1. **基础层**: EAL、kvargs、telemetry
2. **数据结构层**: ring、rcu、mempool、mbuf、stack
3. **网络层**: net、ethdev、pci、meter
4. **应用层**: acl、hash、lpm、pipeline等

---

## 核心基础库

### 1. librte_eal (Environment Abstraction Layer)

**路径**: `lib/librte_eal/`

**功能**: DPDK的核心环境抽象层，提供平台无关的底层服务

**主要功能模块**:
- **内存管理**: 大页内存分配、NUMA感知、内存段管理
- **CPU核心管理**: 逻辑核心抽象、核心绑定、线程管理
- **PCI设备管理**: 设备枚举、BAR映射、配置空间访问
- **定时器服务**: 高精度定时、周期性定时
- **日志系统**: 分级日志、日志回调
- **中断管理**: 中断注册和处理

**实现原理**:
```
初始化流程:
rte_eal_init()
    ├── 创建运行时目录 (/var/run/dpdk)
    ├── 解析命令行参数
    ├── 初始化大页内存系统
    │   ├── 读取 /sys/kernel/mm/hugepages
    │   ├── 分配大页内存
    │   └── 映射到用户空间
    ├── 初始化内存分配器
    ├── 初始化PCI子系统
    │   ├── 扫描PCI总线
    │   ├── 发现设备
    │   └── 加载驱动
    ├── 初始化定时器
    └── 初始化中断系统
```

**关键数据结构**:
- `struct rte_config`: 全局配置结构
- `struct rte_mem_config`: 内存配置结构
- `struct lcore_config`: 逻辑核心配置

**平台相关实现**:
- `linux/`: Linux平台特定实现
- `freebsd/`: FreeBSD平台实现
- `windows/`: Windows平台实现
- `x86/`, `arm/`, `ppc/`: 架构特定优化

---

### 2. librte_kvargs

**路径**: `lib/librte_kvargs/`

**功能**: 键值对参数解析库

**主要功能**:
- 解析字符串格式的键值对参数
- 支持多种数据类型转换
- 参数验证和提取

**实现原理**:
- 使用字符串解析和哈希表存储
- 支持格式: `key1=value1,key2=value2`
- 提供类型安全的参数提取接口

**使用场景**:
- EAL参数解析
- 设备参数配置
- 应用配置解析

---

### 3. librte_telemetry

**路径**: `lib/librte_telemetry/`

**功能**: 遥测和监控数据查询

**主要功能**:
- 提供运行时数据查询接口
- 支持JSON格式数据输出
- 统计信息收集

**实现原理**:
- 维护全局数据注册表
- 通过回调函数收集数据
- JSON序列化输出

---

## 数据结构库

### 4. librte_ring

**路径**: `lib/librte_ring/`

**功能**: 无锁环形队列实现

**主要功能**:
- FIFO队列操作
- 多生产者/多消费者支持
- 批量入队/出队
- 多种同步模式

**实现原理**:

**数据结构**:
```c
struct rte_ring {
    char name[RTE_RING_NAMESIZE];
    uint32_t size;                    // 必须是2的幂
    uint32_t mask;                    // size - 1
    struct rte_ring_headtail prod;     // 生产者头尾指针
    struct rte_ring_headtail cons;     // 消费者头尾指针
    void *ring[];                     // 元素数组
};
```

**无锁算法**:
1. **CAS操作**: 使用原子操作更新head/tail指针
2. **内存屏障**: 确保操作顺序
3. **批量操作**: 一次处理多个元素，减少CAS次数

**同步模式**:
- **SP/SC**: 单生产者/单消费者（最快）
- **MP/MC**: 多生产者/多消费者（通用）
- **MP_RTS/MC_RTS**: Relaxed Tail Sync模式
- **MP_HTS/MC_HTS**: Head-Tail Sync模式

**性能优化**:
- 缓存行对齐
- 批量操作减少函数调用
- 无锁设计避免锁竞争

**使用场景**:
- 数据包队列
- 内存池后端
- 生产者-消费者模式
- 线程间通信

---

### 5. librte_mempool

**路径**: `lib/librte_mempool/`

**功能**: 固定大小对象的内存池管理

**主要功能**:
- 快速分配/释放固定大小对象
- 每核心缓存优化
- NUMA感知分配
- 多种后端支持

**实现原理**:

**数据结构**:
```c
struct rte_mempool {
    char name[RTE_MEMPOOL_NAMESIZE];
    struct rte_ring *ring;             // 后端存储
    uint32_t size;                     // 对象数量
    uint32_t cache_size;               // 每核心缓存大小
    uint32_t elt_size;                 // 对象大小
    struct rte_mempool_cache *local_cache[RTE_MAX_LCORE];
    void *pool_data;                   // 对象池
};
```

**分配流程**:
```
rte_mempool_get()
    ├── 检查本地缓存
    │   ├── 有对象 → 直接返回（快速路径，~10 cycles）
    │   └── 无对象 → 继续
    ├── 从ring批量获取对象（如32个）
    ├── 填充本地缓存
    └── 返回一个对象
```

**释放流程**:
```
rte_mempool_put()
    ├── 检查本地缓存是否满
    │   ├── 未满 → 放入本地缓存（快速路径）
    │   └── 已满 → 批量刷新到ring
    └── 更新统计信息
```

**后端实现**:
- **Ring后端**: 使用rte_ring存储对象指针
- **Stack后端**: LIFO栈结构
- **Bucket后端**: 桶式分配器
- **硬件后端**: Octeontx等专用硬件

**性能优化**:
- 每核心缓存减少跨核心访问
- 批量操作减少CAS开销
- NUMA感知优化内存访问

---

### 6. librte_mbuf

**路径**: `lib/librte_mbuf/`

**功能**: 数据包缓冲区管理

**主要功能**:
- 数据包元数据管理
- 零拷贝支持
- 数据包链（分片/重组）
- 网络层信息存储

**实现原理**:

**数据结构**:
```c
struct rte_mbuf {
    struct rte_mempool *pool;          // 所属内存池
    void *buf_addr;                   // 数据缓冲区地址
    uint16_t buf_len;                 // 缓冲区长度
    uint16_t data_off;                // 数据偏移
    uint32_t pkt_len;                 // 数据包总长度
    uint16_t data_len;                // 当前段长度
    // 网络层信息
    struct rte_ether_addr *eth_addr;
    uint16_t vlan_tci;
    // ... 更多字段
};
```

**零拷贝机制**:
- mbuf只存储元数据和指针
- 实际数据在独立缓冲区中
- 支持数据包链（next指针）
- 引用计数管理

**使用场景**:
- 数据包接收/发送
- 数据包处理
- 协议解析

---

### 7. librte_rcu

**路径**: `lib/librte_rcu/`

**功能**: Read-Copy-Update同步机制

**主要功能**:
- 无锁读取
- 延迟更新
- 垃圾回收

**实现原理**:
- 基于RCU算法实现
- 支持多读者单写者场景
- 延迟释放机制

---

### 8. librte_stack

**路径**: `lib/librte_stack/`

**功能**: 无锁栈数据结构

**主要功能**:
- LIFO操作
- 多生产者/多消费者支持
- 无锁实现

**实现原理**:
- 类似ring但使用栈语义
- 基于原子操作实现

---

## 网络处理库

### 9. librte_net

**路径**: `lib/librte_net/`

**功能**: 网络协议定义和工具函数

**主要功能**:
- 以太网、IP、TCP、UDP等协议头定义
- 字节序转换
- 校验和计算
- 协议解析工具

**实现原理**:
- 标准网络协议头结构体定义
- 内联函数优化
- SIMD加速校验和计算

---

### 10. librte_ethdev

**路径**: `lib/librte_ethdev/`

**功能**: 以太网设备抽象和管理

**主要功能**:
- 以太网设备管理
- 队列配置（RX/TX队列）
- 流控制和速率限制
- 统计信息收集
- 设备热插拔支持

**实现原理**:

**设备抽象**:
```c
struct rte_eth_dev {
    struct rte_eth_dev_data *data;    // 设备数据
    const struct eth_dev_ops *dev_ops; // 设备操作函数
    // ...
};
```

**队列管理**:
- RX队列: 接收数据包队列
- TX队列: 发送数据包队列
- 每个队列独立管理
- 支持多队列

**驱动接口**:
- PMD (Poll Mode Driver)接口
- 轮询模式接收/发送
- 批量操作优化

---

### 11. librte_pci

**路径**: `lib/librte_pci/`

**功能**: PCI设备访问和管理

**主要功能**:
- PCI设备枚举
- BAR空间映射
- 配置空间访问
- 设备驱动管理

**实现原理**:
- 通过sysfs或直接IO访问PCI配置空间
- mmap映射BAR空间到用户空间
- 设备驱动注册和匹配

---

### 12. librte_meter

**路径**: `lib/librte_meter/`

**功能**: 流量计量和整形

**主要功能**:
- 令牌桶算法
- 流量速率限制
- 流量整形
- 多种计量算法

**实现原理**:
- 基于令牌桶算法
- 支持单速率和双速率
- 时间戳更新机制

---

## 查找与分类库

### 13. librte_hash

**路径**: `lib/librte_hash/`

**功能**: 高性能哈希表

**主要功能**:
- Cuckoo哈希算法
- 快速查找和插入
- 支持键值对存储
- 多种哈希函数

**实现原理**:

**Cuckoo哈希**:
- 使用两个哈希函数
- 冲突时重新定位
- 支持扩展表

**数据结构**:
```c
struct rte_hash {
    char name[RTE_HASH_NAMESIZE];
    uint32_t entries;                 // 条目数
    uint32_t key_len;                 // 键长度
    rte_hash_function hash_func;      // 哈希函数
    // ...
};
```

**性能优化**:
- 缓存友好的数据结构
- 批量查找支持
- SIMD优化

---

### 14. librte_lpm

**路径**: `lib/librte_lpm/`

**功能**: IPv4长前缀匹配（路由查找）

**主要功能**:
- IPv4路由查找
- 最长前缀匹配
- 快速查找算法

**实现原理**:

**数据结构**:
- **Tbl24**: 24位前缀表（16M条目）
- **Tbl8**: 8位扩展表（256条目/组）

**查找算法**:
```
1. 提取IP地址前24位
2. 查找Tbl24表
3. 如果valid_group=1，查找Tbl8
4. 返回next_hop
```

**内存优化**:
- 两级表结构减少内存占用
- 动态分配Tbl8组
- 缓存对齐优化

---

### 15. librte_acl

**路径**: `lib/librte_acl/`

**功能**: 访问控制列表（规则匹配）

**主要功能**:
- 多字段规则匹配
- SIMD优化
- 支持IPv4/IPv6
- 优先级匹配

**实现原理**:

**Trie树构建**:
1. **规则分类**: 根据通配符比例分类
2. **Trie构建**: 构建多级Trie树
3. **节点合并**: 合并相同节点
4. **SIMD优化**: 使用SSE/AVX指令

**匹配流程**:
```
rte_acl_classify()
    ├── 加载输入数据包字段
    ├── SIMD并行匹配多个规则
    ├── 遍历Trie树
    └── 返回匹配结果和优先级
```

**SIMD实现**:
- `acl_run_sse.c`: SSE优化
- `acl_run_avx2.c`: AVX2优化
- `acl_run_neon.c`: ARM NEON优化
- `acl_run_scalar.c`: 标量实现

**关键文件**:
- `acl_bld.c`: Trie树构建
- `acl_gen.c`: 代码生成
- `rte_acl.c`: 公共接口

---

### 16. librte_rib

**路径**: `lib/librte_rib/`

**功能**: 路由信息库（Radix树）

**主要功能**:
- IPv4/IPv6路由表
- Radix树实现
- 路由插入/删除/查找

**实现原理**:
- 基于Radix树（压缩Trie树）
- 支持最长前缀匹配
- 动态节点分配

---

### 17. librte_fib

**路径**: `lib/librte_fib/`

**功能**: 转发信息库

**主要功能**:
- 基于RIB构建FIB
- 快速转发查找
- 支持多种算法

**实现原理**:
- 依赖librte_rib
- 优化转发路径
- 缓存优化

---

### 18. librte_member

**路径**: `lib/librte_member/`

**功能**: 集合 membership测试

**主要功能**:
- 布隆过滤器
- 集合成员查询
- 流分类

**实现原理**:
- 基于哈希的集合表示
- 支持误报但无漏报
- 内存高效

---

## 安全与加密库

### 19. librte_cryptodev

**路径**: `lib/librte_cryptodev/`

**功能**: 加密设备抽象

**主要功能**:
- 对称加密（AES、DES等）
- 非对称加密（RSA、ECC等）
- 认证（HMAC、CMAC等）
- 硬件加速支持

**实现原理**:
- 设备抽象层
- 队列管理
- 会话管理
- 硬件/软件驱动

---

### 20. librte_security

**路径**: `lib/librte_security/`

**功能**: 安全协议支持

**主要功能**:
- IPsec协议
- 安全关联管理
- 加密/解密流程

**实现原理**:
- 与cryptodev集成
- 协议状态机
- 安全策略管理

---

### 21. librte_ipsec

**路径**: `lib/librte_ipsec/`

**功能**: IPsec协议实现

**主要功能**:
- IPsec封装/解封装
- SA查找和管理
- 与security库集成

**实现原理**:
- 依赖security和cryptodev
- 协议处理流程
- 性能优化

---

### 22. librte_compressdev

**路径**: `lib/librte_compressdev/`

**功能**: 压缩设备抽象

**主要功能**:
- 数据压缩/解压缩
- 多种压缩算法
- 硬件加速支持

**实现原理**:
- 类似cryptodev的设备抽象
- 队列和会话管理
- 硬件/软件驱动

---

## 事件与调度库

### 23. librte_eventdev

**路径**: `lib/librte_eventdev/`

**功能**: 事件驱动框架

**主要功能**:
- 事件调度
- 事件队列管理
- 事件端口
- 硬件调度器支持

**实现原理**:
- 事件抽象模型
- 调度器接口
- 软件/硬件实现

---

### 24. librte_distributor

**路径**: `lib/librte_distributor/`

**功能**: 数据包分发器

**主要功能**:
- 负载均衡分发
- 流保持（同一流到同一核心）
- 批量分发

**实现原理**:
- 基于哈希的流分类
- 核心选择算法
- SIMD优化匹配

**关键文件**:
- `rte_distributor_match_sse.c`: SSE优化
- `rte_distributor_match_generic.c`: 通用实现

---

### 25. librte_timer

**路径**: `lib/librte_timer/`

**功能**: 定时器管理

**主要功能**:
- 周期性定时器
- 一次性定时器
- 高精度定时

**实现原理**:
- 基于EAL定时器
- 定时器链表管理
- 回调机制

---

### 26. librte_jobstats

**路径**: `lib/librte_jobstats/`

**功能**: 作业统计和调度

**主要功能**:
- 作业执行时间统计
- 作业调度优化
- 周期性作业管理

**实现原理**:
- 时间统计
- 自适应周期调整
- 性能监控

---

## 数据包处理库

### 27. librte_gro

**路径**: `lib/librte_gro/`

**功能**: 通用接收卸载（GRO）

**主要功能**:
- TCP/IP数据包合并
- 减少数据包数量
- 提高处理效率

**实现原理**:
- 流识别
- 数据包合并算法
- 超时处理

---

### 28. librte_gso

**路径**: `lib/librte_gso/`

**功能**: 通用分段卸载（GSO）

**主要功能**:
- 数据包分段
- TCP/UDP分段
- 隧道分段

**实现原理**:
- 大包分段算法
- 协议头处理
- 校验和更新

---

### 29. librte_ip_frag

**路径**: `lib/librte_ip_frag/`

**功能**: IP分片和重组

**主要功能**:
- IP分片
- IP重组
- 分片表管理

**实现原理**:
- 分片表数据结构
- 超时清理
- 内存管理

---

## 管道框架库

### 30. librte_port

**路径**: `lib/librte_port/`

**功能**: 端口抽象

**主要功能**:
- 输入/输出端口
- 多种端口类型
- 数据包处理接口

**实现原理**:
- 端口抽象接口
- 多种实现（ring、ethdev等）
- 统一处理接口

---

### 31. librte_table

**路径**: `lib/librte_table/`

**功能**: 表查找抽象

**主要功能**:
- 表查找接口
- 多种表类型（hash、lpm等）
- 批量查找

**实现原理**:
- 表抽象接口
- 多种后端实现
- 统一查找接口

---

### 32. librte_pipeline

**路径**: `lib/librte_pipeline/`

**功能**: 数据包处理管道

**主要功能**:
- 管道定义
- 端口连接
- 表查找
- 动作执行

**实现原理**:
- 管道抽象模型
- 节点连接
- 数据流处理

---

### 33. librte_flow_classify

**路径**: `lib/librte_flow_classify/`

**功能**: 流分类

**主要功能**:
- 基于规则的流分类
- 表查找集成
- 分类结果处理

**实现原理**:
- 依赖table库
- 规则匹配
- 分类动作

---

## 虚拟化库

### 34. librte_vhost

**路径**: `lib/librte_vhost/`

**功能**: Vhost用户空间实现

**主要功能**:
- Vhost-user协议
- Virtio设备支持
- 虚拟化加速
- vDPA支持

**实现原理**:

**Vhost-user协议**:
- Unix域套接字通信
- 消息协议处理
- 内存映射管理

**关键组件**:
- `vhost_user.c`: 协议实现
- `virtio_net.c`: Virtio网络设备
- `vdpa.c`: vDPA支持
- `iotlb.c`: IOMMU TLB管理

**数据结构**:
```c
struct rte_vhost_memory {
    uint32_t nregions;
    struct rte_vhost_mem_region regions[];
};
```

**内存管理**:
- 共享内存映射
- IOMMU支持
- 地址转换

---

## 其他工具库

### 35. librte_kni

**路径**: `lib/librte_kni/`

**功能**: 内核网络接口

**主要功能**:
- 用户空间到内核的接口
- 虚拟网络设备
- 与内核网络栈交互

**实现原理**:
- 内核模块支持
- 共享内存通信
- 设备抽象

---

### 36. librte_pdump

**路径**: `lib/librte_pdump/`

**功能**: 数据包抓取

**主要功能**:
- 数据包捕获
- 调试支持
- 流量分析

**实现原理**:
- 数据包复制
- 过滤机制
- 输出接口

---

### 37. librte_power

**路径**: `lib/librte_power/`

**功能**: 电源管理

**主要功能**:
- CPU频率调整
- 电源状态管理
- 性能优化

**实现原理**:
- 与内核交互
- 频率调节接口
- 策略管理

---

### 38. librte_bbdev

**路径**: `lib/librte_bbdev/`

**功能**: 基带设备抽象

**主要功能**:
- 5G基带处理
- 编码/解码
- 硬件加速

**实现原理**:
- 设备抽象层
- 队列管理
- 硬件驱动

---

### 39. librte_rawdev

**路径**: `lib/librte_rawdev/`

**功能**: 原始设备抽象

**主要功能**:
- 原始设备访问
- FPGA设备支持
- 自定义设备接口

**实现原理**:
- 设备抽象接口
- 队列管理
- 设备驱动

---

### 40. librte_sched

**路径**: `lib/librte_sched/`

**功能**: 数据包调度器

**主要功能**:
- 层次化调度
- QoS支持
- 流量整形

**实现原理**:
- 层次化队列结构
- 调度算法
- 速率控制

---

### 41. librte_reorder

**路径**: `lib/librte_reorder/`

**功能**: 数据包重排序

**主要功能**:
- 乱序数据包重排
- 序列号管理
- 超时处理

**实现原理**:
- 序列号跟踪
- 缓冲区管理
- 超时机制

---

### 42. librte_efd

**路径**: `lib/librte_efd/`

**功能**: 可扩展完美哈希

**主要功能**:
- 完美哈希表
- 无冲突查找
- 动态扩展

**实现原理**:
- 完美哈希算法
- 动态扩展机制
- 内存优化

---

### 43. librte_metrics

**路径**: `lib/librte_metrics/`

**功能**: 指标收集

**主要功能**:
- 性能指标收集
- 统计信息管理
- 指标查询

**实现原理**:
- 指标注册机制
- 数据收集
- 查询接口

---

### 44. librte_bitratestats

**路径**: `lib/librte_bitratestats/`

**功能**: 比特率统计

**主要功能**:
- 流量速率统计
- 时间窗口统计
- 性能监控

**实现原理**:
- 时间窗口管理
- 统计计算
- 依赖metrics库

---

### 45. librte_latencystats

**路径**: `lib/librte_latencystats/`

**功能**: 延迟统计

**主要功能**:
- 延迟测量
- 统计信息
- 性能分析

**实现原理**:
- 时间戳记录
- 延迟计算
- 统计汇总

---

### 46. librte_cmdline

**路径**: `lib/librte_cmdline/`

**功能**: 命令行解析

**主要功能**:
- 命令行参数解析
- 交互式命令行
- 命令补全

**实现原理**:
- 词法分析
- 语法解析
- 命令执行

---

### 47. librte_cfgfile

**路径**: `lib/librte_cfgfile/`

**功能**: 配置文件解析

**主要功能**:
- 配置文件读取
- 参数解析
- 配置管理

**实现原理**:
- 文件解析
- 键值对提取
- 类型转换

---

### 48. librte_bpf

**路径**: `lib/librte_bpf/`

**功能**: eBPF支持

**主要功能**:
- eBPF程序加载
- eBPF执行
- JIT编译

**实现原理**:
- eBPF虚拟机
- JIT编译器（x86、ARM64）
- 程序验证

**关键文件**:
- `bpf_exec.c`: 解释器
- `bpf_jit_x86.c`: x86 JIT
- `bpf_jit_arm64.c`: ARM64 JIT
- `bpf_validate.c`: 验证器

---

### 49. librte_graph

**路径**: `lib/librte_graph/`

**功能**: 图数据处理框架

**主要功能**:
- 图节点定义
- 图执行
- 数据流处理

**实现原理**:
- 图抽象模型
- 节点连接
- 执行引擎

---

### 50. librte_node

**路径**: `lib/librte_node/`

**功能**: 图节点实现

**主要功能**:
- 预定义节点类型
- 节点处理函数
- 与graph库集成

**实现原理**:
- 节点接口定义
- 多种节点实现
- 数据流处理

---

## 模块依赖关系

### 依赖层次图

```
第一层（基础）:
    kvargs
    telemetry
    eal (所有库的基础)

第二层（数据结构）:
    ring
    rcu (依赖ring)
    mempool (依赖ring)
    mbuf (依赖mempool)
    stack

第三层（网络基础）:
    net
    pci
    ethdev (依赖mbuf, mempool, net, pci)
    meter

第四层（查找和分类）:
    hash (依赖ring)
    lpm
    acl
    rib
    fib (依赖rib)
    member
    efd (依赖hash)

第五层（安全）:
    cryptodev
    security
    ipsec (依赖net, crypto, security)
    compressdev

第六层（事件和调度）:
    timer
    eventdev (依赖timer)
    distributor
    jobstats

第七层（数据包处理）:
    gro
    gso
    ip_frag

第八层（管道框架）:
    port
    table
    pipeline (依赖port, table)
    flow_classify (依赖table)

第九层（虚拟化）:
    vhost
    kni

第十层（工具）:
    cmdline
    cfgfile
    metrics
    bitratestats (依赖metrics)
    latencystats
    pdump
    power
    bbdev
    rawdev
    sched
    reorder
    bpf
    graph
    node
```

### 关键依赖说明

1. **EAL是所有库的基础**: 提供内存、CPU、设备等基础服务
2. **Ring是核心数据结构**: 被mempool、hash等多个库使用
3. **Mempool和Mbuf**: 网络处理的基础
4. **Ethdev**: 网络应用的核心，依赖多个底层库
5. **Pipeline框架**: 构建复杂数据包处理流程

---

## 总结

DPDK的lib目录包含了40多个功能模块，从底层环境抽象到高层应用接口，形成了完整的功能栈：

1. **基础层**: 提供平台抽象和基础服务
2. **数据结构层**: 提供高性能数据结构和算法
3. **网络层**: 提供网络处理功能
4. **应用层**: 提供高级应用接口和框架

每个模块都经过精心设计，注重性能优化，支持SIMD加速、无锁设计、NUMA感知等特性，为高性能网络应用提供了强大的基础。

---

**文档版本**: 1.0  
**最后更新**: 2024年
