# SPDK iSCSI 原理和使用流程分析

## 1. 概述

### 1.1 iSCSI 简介

**iSCSI (Internet Small Computer System Interface)** 是一种基于 TCP/IP 的存储网络协议，允许通过网络传输 SCSI 命令和数据。SPDK iSCSI 是一个高性能的 iSCSI Target 实现，提供用户态、轮询模式的高性能存储访问。

### 1.2 核心功能

SPDK iSCSI 提供以下核心功能：

1. **iSCSI Target 服务**
   - 监听 TCP 端口（默认 3260）
   - 接受来自 iSCSI Initiator 的连接
   - 处理 iSCSI 协议消息

2. **SCSI 命令处理**
   - 将 iSCSI PDU 转换为 SCSI 命令
   - 将 SCSI 响应转换为 iSCSI PDU
   - 支持读写、查询等 SCSI 命令

3. **会话管理**
   - iSCSI 会话建立和维护
   - 多连接会话支持
   - 会话参数协商

4. **连接管理**
   - TCP 连接建立和关闭
   - 连接状态管理
   - 错误恢复

5. **认证和安全**
   - CHAP 认证支持
   - 访问控制（Initiator Group）

### 1.3 适用场景

- **SAN 存储**：通过网络提供块存储服务
- **虚拟化存储**：为虚拟机提供存储后端
- **远程存储**：通过网络访问远程存储设备
- **企业存储**：构建企业级存储解决方案

## 2. 架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│              iSCSI Initiator (客户端)                    │
│  - 发起 iSCSI 连接                                       │
│  - 发送 SCSI 命令                                         │
└─────────────────────────────────────────────────────────┘
                        ↓ TCP/IP 网络
┌─────────────────────────────────────────────────────────┐
│              SPDK iSCSI Target                           │
│  ├── 网络层 (TCP/IP Socket)                             │
│  │   - TCP 连接管理                                      │
│  │   - PDU 接收和发送                                     │
│  ├── iSCSI 协议层                                        │
│  │   - PDU 解析和构建                                     │
│  │   - 会话管理 (Session)                                │
│  │   - 连接管理 (Connection)                             │
│  │   - 任务管理 (Task)                                   │
│  ├── SCSI 层                                            │
│  │   - SCSI 命令处理                                      │
│  │   - SCSI 设备管理                                      │
│  │   - SCSI LUN 管理                                     │
│  └── BDEV 层                                            │
│      - 块设备访问                                         │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│              底层存储设备                                  │
│  - NVMe BDEV / Malloc BDEV / AIO BDEV 等               │
└─────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### 2.2.1 Portal Group（门户组）

**功能**：定义 iSCSI Target 监听的网络地址和端口

**结构**：
```c
struct spdk_iscsi_portal_grp {
    int                     tag;            // 组标签
    bool                    disable_chap;   // 禁用 CHAP
    bool                    require_chap;   // 要求 CHAP
    bool                    mutual_chap;    // 双向 CHAP
    int32_t                 chap_group;     // CHAP 组
    TAILQ_HEAD(, spdk_iscsi_portal) head;  // 门户列表
};
```

**Portal（门户）**：
```c
struct spdk_iscsi_portal {
    struct spdk_iscsi_portal_grp *group;    // 所属组
    char                        host[MAX_PORTAL_ADDR + 1];  // 主机地址
    char                        port[MAX_PORTAL_PORT + 1];  // 端口号
    struct spdk_sock            *sock;      // 套接字
    struct spdk_poller          *acceptor_poller;  // 接受器轮询器
};
```

#### 2.2.2 Initiator Group（发起方组）

**功能**：定义允许访问 iSCSI Target 的 Initiator 列表

**结构**：
```c
struct spdk_iscsi_init_grp {
    int                         tag;        // 组标签
    TAILQ_HEAD(, spdk_iscsi_initiator_name) initiator_head;  // 发起方名称列表
    TAILQ_HEAD(, spdk_iscsi_initiator_netmask) netmask_head; // 网络掩码列表
};
```

**访问控制**：
- 按 Initiator 名称（IQN）控制
- 按 IP 地址/网络掩码控制

#### 2.2.3 Target Node（目标节点）

**功能**：定义 iSCSI Target 的配置和 LUN 映射

**结构**：
```c
struct spdk_iscsi_tgt_node {
    char                    name[MAX_TARGET_NAME + 1];  // 目标名称 (IQN)
    char                    alias[MAX_TARGET_NAME + 1]; // 别名
    bool                    disable_chap;               // 禁用 CHAP
    bool                    require_chap;               // 要求 CHAP
    bool                    header_digest;              // 头摘要
    bool                    data_digest;                // 数据摘要
    int                     queue_depth;                // 队列深度
    struct spdk_scsi_dev    *dev;                       // SCSI 设备
    TAILQ_HEAD(, spdk_iscsi_pg_map) pg_map_head;       // 门户组映射列表
};
```

**LUN 映射**：
- Target Node 可以包含多个 LUN
- 每个 LUN 映射到一个 BDEV

#### 2.2.4 Connection（连接）

**功能**：管理单个 TCP 连接

**结构**：
```c
struct spdk_iscsi_conn {
    int                         id;                     // 连接 ID
    struct spdk_sock            *sock;                  // TCP 套接字
    struct spdk_iscsi_sess      *sess;                  // 所属会话
    struct spdk_iscsi_portal    *portal;                // 所属门户
    enum iscsi_connection_state state;                  // 连接状态
    int                         login_phase;            // 登录阶段
    struct spdk_iscsi_pdu       *pdu_in_progress;       // 正在处理的 PDU
    struct iscsi_chap_auth      auth;                   // CHAP 认证信息
    uint32_t                    StatSN;                 // 状态序列号
    struct spdk_iscsi_lun       *luns[SPDK_SCSI_DEV_MAX_LUN];  // LUN 数组
};
```

**连接状态**：
- `ISCSI_CONN_STATE_RUNNING`：运行中
- `ISCSI_CONN_STATE_EXITING`：正在退出
- `ISCSI_CONN_STATE_EXITED`：已退出

#### 2.2.5 Session（会话）

**功能**：管理一个 iSCSI 会话（可能包含多个连接）

**结构**：
```c
struct spdk_iscsi_sess {
    uint32_t                connections;                // 连接数量
    struct spdk_iscsi_conn  **conns;                    // 连接数组
    struct spdk_scsi_port   *initiator_port;            // 发起方端口
    uint64_t                isid;                       // 发起方会话标识符
    uint16_t                tsih;                       // Target 会话处理句柄
    struct spdk_iscsi_tgt_node *target;                 // 目标节点
    int                     queue_depth;                // 队列深度
    uint32_t                ExpCmdSN;                   // 预期命令序列号
    uint32_t                MaxCmdSN;                   // 最大命令序列号
};
```

**会话类型**：
- `SESSION_TYPE_NORMAL`：普通会话
- `SESSION_TYPE_DISCOVERY`：发现会话

#### 2.2.6 Task（任务）

**功能**：管理一个 SCSI 命令执行

**结构**：
```c
struct spdk_iscsi_task {
    struct spdk_scsi_task   scsi;                       // SCSI 任务
    struct spdk_iscsi_conn  *conn;                      // 关联的连接
    struct spdk_iscsi_pdu   *pdu;                       // 关联的 PDU
    uint32_t                tag;                        // 任务标签
    uint32_t                outstanding_r2t;            // 未完成的 R2T 数量
    uint32_t                bytes_completed;            // 已完成的字节数
    uint32_t                next_r2t_offset;            // 下一个 R2T 偏移量
    bool                    is_r2t_active;              // R2T 是否激活
};
```

#### 2.2.7 PDU（协议数据单元）

**功能**：封装 iSCSI 协议消息

**结构**：
```c
struct spdk_iscsi_pdu {
    struct iscsi_bhs        bhs;                        // 基本头段
    uint8_t                 *data_buf;                  // 数据缓冲区
    size_t                  data_segment_len;           // 数据段长度
    uint32_t                cmd_sn;                     // 命令序列号
    uint8_t                 header_digest[ISCSI_DIGEST_LEN];  // 头摘要
    uint8_t                 data_digest[ISCSI_DIGEST_LEN];    // 数据摘要
    struct spdk_iscsi_task  *task;                      // 关联的任务
    struct spdk_iscsi_conn  *conn;                      // 关联的连接
};
```

**PDU 类型**：
- Login Request/Response：登录请求/响应
- Logout Request/Response：登出请求/响应
- SCSI Command/Response：SCSI 命令/响应
- Data-In/Data-Out：数据输入/输出
- R2T（Ready To Transfer）：准备传输
- NOP-In/NOP-Out：心跳包

## 3. 核心原理

### 3.1 iSCSI 协议栈

```
┌─────────────────────────────────────────────────────────┐
│       应用层 (Application)                               │
│       - 文件系统 / 数据库 / 应用                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│       SCSI 层 (SCSI)                                     │
│       - SCSI 命令 (READ/WRITE/INQUIRY)                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│       iSCSI 层 (iSCSI Protocol)                         │
│       - PDU 封装 SCSI 命令                                │
│       - 会话管理                                          │
│       - 序列号管理                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│       TCP/IP 层 (TCP/IP)                                 │
│       - TCP 连接                                          │
│       - 数据包传输                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│       网络层 (Network)                                    │
│       - 以太网 / IP 网络                                  │
└─────────────────────────────────────────────────────────┘
```

### 3.2 iSCSI 会话建立流程

```
┌─────────────────────────────────────────────────────────┐
│  1. TCP 连接建立                                         │
│     Initiator → Target (TCP SYN/SYN-ACK/ACK)           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. iSCSI 登录阶段 (Security Negotiation)               │
│     Login Request (CSG=0, NSG=0)                        │
│     Login Response (CSG=0, NSG=0)                       │
│     - 参数协商                                            │
│     - CHAP 认证（如果启用）                               │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. iSCSI 登录阶段 (Operational Negotiation)            │
│     Login Request (CSG=1, NSG=1)                        │
│     Login Response (CSG=1, NSG=1)                       │
│     - 会话参数协商                                        │
│     - 连接参数协商                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. iSCSI 全功能阶段 (Full Feature Phase)               │
│     Login Request (CSG=3, NSG=3)                        │
│     Login Response (CSG=3, NSG=3, T=1)                  │
│     - 会话建立完成                                        │
│     - 可以发送 SCSI 命令                                  │
└─────────────────────────────────────────────────────────┘
```

### 3.3 SCSI 命令处理流程

#### 3.3.1 读命令流程

```
┌─────────────────────────────────────────────────────────┐
│  1. 接收 SCSI Command PDU                                │
│     - 解析 PDU 头部                                       │
│     - 提取 SCSI 命令 (CDB)                                │
│     - 创建 Task                                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 提交到 SCSI 层                                        │
│     - 将 Task 提交到 SCSI LUN                             │
│     - SCSI 层处理命令                                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 读取数据                                              │
│     - 从 BDEV 读取数据                                    │
│     - 数据返回                                            │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 发送 Data-In PDU                                     │
│     - 构建 Data-In PDU                                   │
│     - 发送数据到 Initiator                                │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  5. 发送 SCSI Response PDU                               │
│     - 发送命令完成响应                                     │
│     - 更新序列号                                          │
└─────────────────────────────────────────────────────────┘
```

#### 3.3.2 写命令流程

```
┌─────────────────────────────────────────────────────────┐
│  1. 接收 SCSI Command PDU                                │
│     - 解析 PDU 头部                                       │
│     - 提取 SCSI 命令 (CDB)                                │
│     - 创建 Task                                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 处理数据                                              │
│     - 如果包含立即数据，接收 Data-Out PDU                  │
│     - 如果需要更多数据，发送 R2T PDU                       │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 接收 Data-Out PDU                                    │
│     - 接收写入数据                                        │
│     - 继续接收直到数据完整                                 │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 写入数据                                              │
│     - 将数据写入 BDEV                                     │
│     - 等待写入完成                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  5. 发送 SCSI Response PDU                               │
│     - 发送命令完成响应                                     │
│     - 更新序列号                                          │
└─────────────────────────────────────────────────────────┘
```

### 3.4 R2T 机制（Ready To Transfer）

**原理**：
- 当写命令的数据长度超过 `FirstBurstLength` 时，Target 需要发送 R2T 请求数据
- Initiator 收到 R2T 后，发送 Data-Out PDU
- 支持多个未完成的 R2T（`MaxOutstandingR2T`）

**流程**：
```
写命令（数据长度 > FirstBurstLength）
  ↓
发送 R2T PDU（指定偏移和长度）
  ↓
接收 Data-Out PDU（包含数据）
  ↓
如果还有数据未接收，发送下一个 R2T
  ↓
所有数据接收完成，执行写入
```

## 4. I/O 处理流程

### 4.1 PDU 接收流程

```
┌─────────────────────────────────────────────────────────┐
│  1. TCP Socket 接收数据                                   │
│     - 从 Socket 读取数据                                  │
│     - 解析 PDU 头部（48 字节）                            │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 解析 PDU 类型                                         │
│     - Login Request/Response                             │
│     - SCSI Command                                       │
│     - Data-In/Data-Out                                   │
│     - NOP-In/NOP-Out                                     │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 验证 PDU                                              │
│     - 验证头摘要（如果启用）                               │
│     - 验证数据摘要（如果启用）                             │
│     - 验证序列号                                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 处理 PDU                                              │
│     - 根据 PDU 类型分发处理                                │
│     - 创建或查找关联的 Task                               │
└─────────────────────────────────────────────────────────┘
```

### 4.2 SCSI 命令处理流程

```
┌─────────────────────────────────────────────────────────┐
│  1. 解析 SCSI Command PDU                                │
│     - 提取 CDB (Command Descriptor Block)                │
│     - 提取 LUN ID                                        │
│     - 提取任务标签                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 查找 SCSI LUN                                         │
│     - 根据 Target Node 和 LUN ID 查找 LUN                │
│     - 验证访问权限                                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 创建 SCSI Task                                        │
│     - 分配 Task 结构体                                    │
│     - 关联 Connection 和 PDU                              │
│     - 设置完成回调                                         │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 提交到 SCSI 层                                        │
│     - 调用 spdk_scsi_dev_queue_task()                    │
│     - SCSI 层处理命令                                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  5. SCSI 层执行                                           │
│     - 解析 CDB 操作码                                     │
│     - 执行相应操作（读写/查询等）                          │
│     - 调用 BDEV I/O 接口                                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  6. 完成回调                                              │
│     - Task 完成时调用回调                                  │
│     - 构建 Response PDU                                   │
│     - 发送响应到 Initiator                                │
└─────────────────────────────────────────────────────────┘
```

## 5. 使用流程

### 5.1 初始化流程

#### 步骤 1：初始化 SPDK 环境

```c
// 初始化 SPDK 环境
struct spdk_env_opts opts;
spdk_env_opts_init(&opts);
opts.name = "iscsi_tgt";
opts.shm_id = 0;
spdk_env_init(&opts);
```

#### 步骤 2：初始化 iSCSI 子系统

```c
void iscsi_init_cb(void *cb_arg, int rc)
{
    if (rc) {
        printf("iSCSI initialization failed: %d\n", rc);
        return;
    }
    printf("iSCSI subsystem initialized\n");
}

// 初始化 iSCSI 子系统
spdk_iscsi_init(iscsi_init_cb, NULL);
```

**初始化步骤**：
1. 分配内存池（PDU、Session、Task）
2. 解析配置文件（如果存在）
3. 初始化 Portal Group
4. 初始化 Initiator Group
5. 初始化 Target Node
6. 打开 Portal（监听 TCP 端口）

#### 步骤 3：创建 Portal Group

```c
// 创建 Portal Group
struct spdk_iscsi_portal *portal = iscsi_portal_create("0.0.0.0", "3260");
struct spdk_iscsi_portal_grp *pg = iscsi_portal_grp_create(1);
iscsi_portal_grp_add_portal(pg, portal);
iscsi_portal_grp_register(pg);
iscsi_portal_grp_open(pg);  // 开始监听
```

#### 步骤 4：创建 Initiator Group

```c
// 创建 Initiator Group
char *initiator_names[] = {"iqn.2016-06.io.initiator:1"};
char *initiator_masks[] = {"ANY"};
iscsi_init_grp_create_from_initiator_list(
    1,  // tag
    1,  // num_initiator_names
    initiator_names,
    1,  // num_initiator_masks
    initiator_masks
);
```

#### 步骤 5：创建 Target Node

```c
// 创建 Target Node
const char *target_name = "iqn.2016-06.io.spdk:target1";
int pg_tag_list[] = {1};  // Portal Group tags
int ig_tag_list[] = {1};  // Initiator Group tags
const char *bdev_name_list[] = {"Malloc0"};  // BDEV 名称
int lun_id_list[] = {0};  // LUN IDs

struct spdk_iscsi_tgt_node *target = iscsi_tgt_node_construct(
    0,                    // target_index
    target_name,          // name
    "Target 1",           // alias
    pg_tag_list,          // pg_tag_list
    ig_tag_list,          // ig_tag_list
    1,                    // num_maps
    bdev_name_list,       // bdev_name_list
    lun_id_list,          // lun_id_list
    1,                    // num_luns
    64,                   // queue_depth
    false,                // disable_chap
    false,                // require_chap
    false,                // mutual_chap
    0,                    // chap_group
    false,                // header_digest
    false                 // data_digest
);
```

### 5.2 运行流程

#### iSCSI 会话建立

```
1. Portal 接受 TCP 连接
   └─> 创建 Connection
   
2. 接收 Login Request PDU
   └─> 进入登录阶段处理
   
3. 参数协商
   ├─> 安全协商阶段（CSG=0）
   ├─> 操作协商阶段（CSG=1）
   └─> 全功能阶段（CSG=3）
   
4. 发送 Login Response PDU (T=1)
   └─> 会话建立完成
```

#### SCSI 命令处理

```
1. 接收 SCSI Command PDU
   └─> 解析命令，创建 Task
   
2. 提交到 SCSI 层
   └─> SCSI 层处理命令
   
3. BDEV I/O 操作
   └─> 读写底层设备
   
4. 完成回调
   ├─> 构建 Response PDU
   └─> 发送响应
```

### 5.3 配置管理（JSON-RPC）

#### 创建 Portal Group

```json
{
  "jsonrpc": "2.0",
  "method": "iscsi_create_portal_group",
  "id": 1,
  "params": {
    "tag": 1,
    "portals": [
      {
        "host": "0.0.0.0",
        "port": "3260"
      }
    ]
  }
}
```

#### 创建 Initiator Group

```json
{
  "jsonrpc": "2.0",
  "method": "iscsi_create_initiator_group",
  "id": 2,
  "params": {
    "tag": 1,
    "initiators": ["iqn.2016-06.io.initiator:1"],
    "netmasks": ["ANY"]
  }
}
```

#### 创建 Target Node

```json
{
  "jsonrpc": "2.0",
  "method": "iscsi_create_target_node",
  "id": 3,
  "params": {
    "name": "iqn.2016-06.io.spdk:target1",
    "alias_name": "Target 1",
    "pg_ig_maps": [
      {
        "pg_tag": 1,
        "ig_tag": 1
      }
    ],
    "luns": [
      {
        "lun_id": 0,
        "bdev_name": "Malloc0"
      }
    ],
    "queue_depth": 64
  }
}
```

### 5.4 清理流程

```c
void iscsi_fini_cb(void *arg)
{
    printf("iSCSI subsystem finalized\n");
}

// 清理 iSCSI 子系统
spdk_iscsi_fini(iscsi_fini_cb, NULL);
```

**清理步骤**：
1. 关闭所有 Portal（停止监听）
2. 关闭所有活跃连接
3. 清理所有 Target Node
4. 清理所有 Portal Group 和 Initiator Group
5. 释放内存池和资源

## 6. 关键配置参数

### 6.1 全局配置

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `MaxSessions` | 最大会话数 | 128 |
| `MaxConnectionsPerSession` | 每个会话的最大连接数 | 2 |
| `MaxConnections` | 最大连接数 | 1024 |
| `MaxQueueDepth` | 最大队列深度 | 64 |
| `DefaultTime2Wait` | 默认等待时间2（秒） | 2 |
| `DefaultTime2Retain` | 默认保留时间2（秒） | 20 |
| `FirstBurstLength` | 首次突发长度（字节） | 8192 |
| `ImmediateData` | 启用立即数据 | true |
| `ErrorRecoveryLevel` | 错误恢复级别 | 0 |

### 6.2 Target Node 配置

| 配置项 | 说明 |
|--------|------|
| `name` | 目标名称（IQN 格式） |
| `alias` | 目标别名 |
| `queue_depth` | 队列深度 |
| `disable_chap` | 禁用 CHAP |
| `require_chap` | 要求 CHAP |
| `mutual_chap` | 双向 CHAP |
| `header_digest` | 启用头摘要 |
| `data_digest` | 启用数据摘要 |

### 6.3 Portal Group 配置

| 配置项 | 说明 |
|--------|------|
| `tag` | 组标签 |
| `host` | 主机地址（IP 地址） |
| `port` | 端口号（默认 3260） |

## 7. 认证和安全

### 7.1 CHAP 认证

**原理**：
- CHAP（Challenge-Handshake Authentication Protocol）提供认证机制
- Target 发送挑战（Challenge）给 Initiator
- Initiator 使用密钥和挑战计算响应（Response）
- Target 验证响应

**配置**：
```c
// 创建认证组
iscsi_add_auth_group(1, &auth_group);

// 添加认证密钥
iscsi_auth_group_add_secret(
    auth_group,
    "user1",     // 用户名
    "secret1",   // 密钥
    NULL,        // 双向认证用户名（可选）
    NULL         // 双向认证密钥（可选）
);

// 在 Target Node 或 Portal Group 中启用 CHAP
iscsi_target_node_set_chap_params(target, false, true, false, 1);
```

### 7.2 访问控制

**Initiator Group**：
- 按 Initiator 名称（IQN）控制
- 按 IP 地址/网络掩码控制

**示例**：
```c
// 允许特定 Initiator
char *initiator_names[] = {"iqn.2016-06.io.initiator:1"};

// 允许特定 IP 范围
char *initiator_masks[] = {"192.168.1.0/24"};

// 允许所有
char *initiator_masks[] = {"ANY"};
```

## 8. 性能优化

### 8.1 批量处理

- 批量处理 PDU 接收和发送
- 减少系统调用开销

### 8.2 内存池

- 使用内存池分配 PDU 和 Task
- 减少内存分配开销

### 8.3 轮询模式

- 使用 SPDK 轮询模式处理网络 I/O
- 消除中断开销

### 8.4 零拷贝

- 尽量减少数据拷贝
- 直接使用 DMA 缓冲区

## 9. 错误处理

### 9.1 连接错误

- 检测 TCP 连接断开
- 自动清理连接资源

### 9.2 协议错误

- 验证 PDU 格式
- 验证序列号
- 发送 Reject PDU（如果无效）

### 9.3 SCSI 错误

- 处理 SCSI 错误状态
- 发送 SCSI Response 和 Sense Data

## 10. 总结

### 10.1 核心特性

1. **高性能**：用户态、轮询模式
2. **标准协议**：兼容 iSCSI 协议标准
3. **灵活配置**：支持 Portal Group、Initiator Group、Target Node
4. **安全认证**：支持 CHAP 认证
5. **访问控制**：基于 Initiator Group 的访问控制

### 10.2 架构优势

- **模块化设计**：Portal Group、Initiator Group、Target Node 分离
- **高性能**：轮询模式、内存池、零拷贝
- **灵活配置**：支持动态配置和管理

### 10.3 使用场景

- **SAN 存储**：通过网络提供块存储服务
- **虚拟化存储**：为虚拟机提供存储后端
- **远程存储**：通过网络访问远程存储设备

## 11. 参考资料

- SPDK iSCSI 源码：`lib/iscsi/`
- SPDK iSCSI API：`include/spdk/iscsi.h`
- SPDK SCSI 子模块：`lib/scsi/`
- iSCSI 协议规范：RFC 3720, RFC 3721
- SCSI 协议规范：SAM-5, SPC-5
