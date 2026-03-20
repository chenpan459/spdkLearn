# igb_uio模块详细分析文档

## 1. 概述

### 1.1 什么是igb_uio

**igb_uio**是DPDK提供的一个Linux内核模块，用于将PCI设备从内核驱动解绑并绑定到UIO（Userspace I/O）框架，使DPDK/SPDK等用户空间应用能够直接访问PCI设备的硬件资源。

### 1.2 核心功能

1. **PCI设备绑定**：将PCI设备从内核驱动解绑并绑定到UIO框架
2. **BAR映射**：将PCI设备的BAR（Base Address Register）映射到用户空间
3. **中断管理**：支持MSI-X、MSI和Legacy中断模式
4. **SR-IOV支持**：通过sysfs接口管理SR-IOV虚拟功能
5. **用户空间访问**：通过`/dev/uioX`设备文件提供用户空间访问接口

### 1.3 文件位置

```
C:\cp4119\github\devCephMain\cephMain\src\spdk\dpdk\kernel\linux\igb_uio\
├── igb_uio.c      # 主实现文件
├── compat.h        # 内核兼容性头文件
├── Kbuild          # 内核构建文件
└── Makefile        # Makefile
```

## 2. 核心数据结构

### 2.1 rte_uio_pci_dev

```c
/**
 * @struct rte_uio_pci_dev
 * @brief UIO PCI设备的私有信息结构
 */
struct rte_uio_pci_dev {
	struct uio_info info;        /* UIO信息结构 */
	struct pci_dev *pdev;        /* PCI设备指针 */
	enum rte_intr_mode mode;     /* 中断模式（MSI-X/MSI/LEGACY/NONE） */
	atomic_t refcnt;             /* 引用计数 */
};
```

**字段说明**：
- `info`: UIO框架的信息结构，包含设备名称、版本、回调函数等
- `pdev`: 指向Linux内核PCI设备结构
- `mode`: 当前使用的中断模式
- `refcnt`: 引用计数，用于跟踪有多少个用户空间进程打开了设备

## 3. 模块初始化流程

### 3.1 模块加载

```
用户执行: insmod igb_uio.ko [intr_mode=msix] [wc_activate=0]
    ↓
igbuio_pci_init_module()
    ├─> 检查内核锁定状态
    ├─> 解析中断模式参数
    └─> pci_register_driver(&igbuio_pci_driver)
        └─> 内核PCI子系统扫描设备
            └─> 对每个未绑定的PCI设备调用 igbuio_pci_probe()
```

### 3.2 设备探测（igbuio_pci_probe）

**调用时机**：
- 模块加载时，内核扫描所有PCI设备
- 热插拔设备时
- 手动绑定设备时（通过sysfs）

**执行流程**：

```c
igbuio_pci_probe()
    ├─> 检查是否为PCI桥（忽略桥设备）
    ├─> 分配rte_uio_pci_dev结构
    ├─> pci_enable_device()          // 启用PCI设备
    ├─> pci_set_master()              // 设置总线主控
    ├─> igbuio_setup_bars()          // 设置所有BAR
    │   ├─> 遍历BAR0-BAR5
    │   ├─> 区分内存BAR和I/O端口BAR
    │   └─> 分别调用igbuio_pci_setup_iomem()或igbuio_pci_setup_ioport()
    ├─> pci_set_dma_mask()           // 设置64位DMA掩码
    ├─> pci_set_consistent_dma_mask() // 设置一致性DMA掩码
    ├─> 填充uio_info结构
    │   ├─> name = "igb_uio"
    │   ├─> irqcontrol = igbuio_pci_irqcontrol
    │   ├─> open = igbuio_pci_open
    │   └─> release = igbuio_pci_release
    ├─> sysfs_create_group()         // 创建sysfs属性组（SR-IOV支持）
    ├─> uio_register_device()        // 注册UIO设备
    │   └─> 创建/sys/class/uio/uioX目录
    │   └─> 创建/dev/uioX设备文件（如果启用）
    └─> 执行DMA映射测试（用于IOMMU identity mapping）
```

## 4. 用户空间打开设备流程

### 4.1 打开/dev/uioX设备文件

**调用路径**：
```
DPDK应用
    ↓
open("/dev/uio0", O_RDWR)
    ↓
内核VFS层
    ↓
igbuio_pci_open()
```

**igbuio_pci_open()执行流程**：

```c
igbuio_pci_open()
    ├─> atomic_inc_return(&udev->refcnt)  // 增加引用计数
    │   └─> 如果不是第一个打开者，直接返回
    ├─> pci_set_master(dev)               // 设置总线主控
    └─> igbuio_pci_enable_interrupts()    // 使能中断
        ├─> 尝试MSI-X中断
        │   ├─> pci_enable_msix() 或 pci_alloc_irq_vectors()
        │   └─> 成功则设置mode = RTE_INTR_MODE_MSIX
        ├─> 如果MSI-X失败，尝试MSI中断
        │   ├─> pci_enable_msi() 或 pci_alloc_irq_vectors()
        │   └─> 成功则设置mode = RTE_INTR_MODE_MSI
        ├─> 如果MSI失败，尝试Legacy中断
        │   ├─> pci_intx_mask_supported() 检查硬件支持
        │   └─> 成功则设置mode = RTE_INTR_MODE_LEGACY
        ├─> 如果都失败，使用无中断模式
        │   └─> mode = RTE_INTR_MODE_NONE
        └─> request_irq()                   // 注册中断处理函数
```

### 4.2 中断使能流程详解

#### 4.2.1 MSI-X模式（首选）

```c
// 旧内核API
pci_enable_msix(udev->pdev, &msix_entry, 1)
    └─> 分配1个MSI-X向量
    └─> 返回中断向量号

// 新内核API（4.8+）
pci_alloc_irq_vectors(udev->pdev, 1, 1, PCI_IRQ_MSIX)
    └─> 分配1个MSI-X向量
    └─> pci_irq_vector(udev->pdev, 0) 获取中断号
```

**特点**：
- 每个中断向量独立
- 自动屏蔽，无需共享
- 性能最好

#### 4.2.2 MSI模式

```c
// 旧内核API
pci_enable_msi(udev->pdev)
    └─> 分配1个MSI向量
    └─> 返回中断号（在pdev->irq中）

// 新内核API
pci_alloc_irq_vectors(udev->pdev, 1, 1, PCI_IRQ_MSI)
    └─> 分配1个MSI向量
    └─> pci_irq_vector(udev->pdev, 0) 获取中断号
```

**特点**：
- 单个中断向量
- 自动屏蔽
- 性能较好

#### 4.2.3 Legacy模式

```c
pci_intx_mask_supported(udev->pdev)
    └─> 检查硬件是否支持INTx掩码
    └─> 通过读写PCI_COMMAND寄存器测试

// 如果支持，使用Legacy中断
udev->info.irq = udev->pdev->irq
udev->info.irq_flags = IRQF_SHARED | IRQF_NO_THREAD
```

**特点**：
- 传统的中断方式
- 可能共享中断
- 需要硬件支持掩码
- 性能较差

## 5. 中断处理流程

### 5.1 中断发生

```
硬件设备产生中断
    ↓
内核中断子系统
    ↓
igbuio_pci_irqhandler()
    ├─> 检查中断模式
    ├─> Legacy模式：pci_check_and_mask_intx() 检查并屏蔽
    ├─> MSI/MSI-X模式：自动屏蔽（硬件特性）
    └─> uio_event_notify() 通知UIO框架
        └─> 唤醒等待在/dev/uioX上的用户空间进程
```

### 5.2 用户空间中断控制

**通过ioctl或write系统调用控制中断**：

```c
// 用户空间使能中断
write(uio_fd, &value, sizeof(value))  // value = 1
    ↓
igbuio_pci_irqcontrol(info, 1)
    ├─> MSI-X/MSI模式：
    │   ├─> pci_msi_unmask_irq() 或 igbuio_mask_irq()
    │   └─> 取消屏蔽中断
    └─> Legacy模式：
        └─> pci_intx(pdev, 1)  // 使能INTx中断

// 用户空间禁用中断
write(uio_fd, &value, sizeof(value))  // value = 0
    ↓
igbuio_pci_irqcontrol(info, 0)
    ├─> MSI-X/MSI模式：
    │   ├─> pci_msi_mask_irq() 或 igbuio_mask_irq()
    │   └─> 屏蔽中断
    └─> Legacy模式：
        └─> pci_intx(pdev, 0)  // 禁用INTx中断
```

## 6. BAR资源映射

### 6.1 内存BAR映射（igbuio_pci_setup_iomem）

**功能**：将PCI设备的内存BAR映射到UIO资源

**流程**：
```c
igbuio_pci_setup_iomem()
    ├─> pci_resource_start()  // 获取BAR物理起始地址
    ├─> pci_resource_len()    // 获取BAR长度
    ├─> 如果wc_activate == 0:
    │   └─> ioremap(addr, len)  // 映射到内核虚拟地址
    └─> 填充uio_info->mem[n]:
        ├─> name = "BAR0"等
        ├─> addr = 物理地址
        ├─> internal_addr = 内核虚拟地址（如果映射）
        ├─> size = BAR长度
        └─> memtype = UIO_MEM_PHYS
```

**用户空间访问**：
- 通过`mmap()`系统调用映射`/sys/class/uio/uioX/maps/mapY`
- 或直接映射`/sys/bus/pci/devices/.../resourceY`

### 6.2 I/O端口BAR映射（igbuio_pci_setup_ioport）

**功能**：将PCI设备的I/O端口BAR映射到UIO资源

**流程**：
```c
igbuio_pci_setup_ioport()
    ├─> pci_resource_start()  // 获取I/O端口起始地址
    ├─> pci_resource_len()      // 获取I/O端口长度
    └─> 填充uio_info->port[n]:
        ├─> name = "BAR0"等
        ├─> start = I/O端口起始地址
        ├─> size = I/O端口长度
        └─> porttype = UIO_PORT_X86
```

**用户空间访问**：
- 通过`iopl()`或`ioperm()`获取I/O端口访问权限
- 使用`inb()/outb()`等函数访问I/O端口

## 7. DPDK/SPDK如何调用igb_uio

### 7.1 设备绑定流程

#### 7.1.1 手动绑定（通过脚本）

```bash
# 1. 加载igb_uio模块
insmod igb_uio.ko [intr_mode=msix] [wc_activate=0]

# 2. 解绑内核驱动
echo "0000:01:00.0" > /sys/bus/pci/drivers/ixgbe/unbind

# 3. 绑定到igb_uio
echo "0000:01:00.0" > /sys/bus/pci/drivers/igb_uio/bind
    ↓
内核PCI子系统
    ↓
igbuio_pci_probe()
```

#### 7.1.2 通过DPDK工具绑定

```bash
# 使用dpdk-devbind.py工具
./usertools/dpdk-devbind.py --bind=igb_uio 01:00.0
    ↓
读取/sys/bus/pci/devices/.../driver
    ↓
写入/sys/bus/pci/drivers/igb_uio/bind
    ↓
内核PCI子系统
    ↓
igbuio_pci_probe()
```

### 7.2 DPDK EAL初始化流程

```
DPDK应用启动
    ↓
rte_eal_init()
    ├─> rte_eal_pci_init()
    │   ├─> pci_scan()  // 扫描PCI总线
    │   │   └─> 读取/sys/bus/pci/devices/
    │   │       └─> 检查设备是否绑定到igb_uio
    │   └─> pci_probe()  // 探测设备
    │       └─> 对每个设备调用驱动probe函数
    └─> 对于绑定到igb_uio的设备：
        └─> pci_uio_alloc_resource()
            ├─> pci_get_uio_dev()  // 查找UIO设备
            │   └─> 读取/sys/bus/pci/devices/.../uio/uioX
            ├─> open("/dev/uioX", O_RDWR)  // 打开UIO设备
            │   └─> igbuio_pci_open()  // 内核调用
            ├─> open("/sys/class/uio/uioX/device/config", O_RDWR)  // 打开配置空间
            └─> 保存文件描述符
```

### 7.3 BAR映射流程（DPDK侧）

```
DPDK应用
    ↓
pci_uio_map_resource_by_index()
    ├─> 打开/sys/bus/pci/devices/.../resourceY
    │   └─> 如果wc_activate，尝试打开resourceY_wc
    ├─> mmap() 映射资源文件
    │   └─> 内核VFS层
    │       └─> 映射到igb_uio设置的物理地址
    └─> 保存映射地址
```

### 7.4 中断处理流程（DPDK侧）

```
DPDK应用
    ↓
rte_intr_callback_register()
    ├─> 注册中断回调函数
    └─> 添加到中断源列表
    ↓
中断处理线程
    ├─> epoll_wait() 等待中断
    │   └─> 监听/dev/uioX文件描述符
    ├─> read(uio_fd) 读取中断计数
    │   └─> 阻塞直到中断发生
    │   └─> igbuio_pci_irqhandler() 唤醒
    └─> 调用注册的回调函数
```

**详细流程**：

```c
// DPDK中断处理线程
while (1) {
    epoll_wait(epfd, events, ...)  // 等待事件
        ↓
    中断发生
        ↓
    igbuio_pci_irqhandler()
        ↓
    uio_event_notify()
        ↓
    唤醒read()调用
        ↓
    read(uio_fd, &count, sizeof(count))  // 读取中断计数
        ↓
    调用用户注册的回调函数
}
```

### 7.5 SPDK如何使用igb_uio

**SPDK通过env_dpdk封装DPDK功能**：

```
SPDK应用
    ↓
spdk_env_init()
    ├─> rte_eal_init()  // 初始化DPDK EAL
    │   └─> 扫描PCI设备
    │       └─> 发现绑定到igb_uio的设备
    └─> 通过SPDK PCI抽象层访问设备
        ├─> spdk_pci_device_attach()
        │   └─> rte_eal_hotplug_add()  // 热插拔添加
        └─> spdk_pci_device_map_bar()
            └─> 使用DPDK的BAR映射功能
```

## 8. 用户空间与igb_uio的交互

### 8.1 设备文件接口

**设备文件**：`/dev/uioX`（X为UIO设备编号）

**主要操作**：

1. **open()** - 打开设备
   - 触发`igbuio_pci_open()`
   - 使能中断

2. **read()** - 读取中断
   - 阻塞等待中断
   - 返回中断计数（uint32_t）

3. **write()** - 控制中断
   - 写入1：使能中断
   - 写入0：禁用中断
   - 触发`igbuio_pci_irqcontrol()`

4. **mmap()** - 映射BAR
   - 映射`/sys/class/uio/uioX/maps/mapY`
   - 或映射`/sys/bus/pci/devices/.../resourceY`

5. **close()** - 关闭设备
   - 触发`igbuio_pci_release()`
   - 禁用中断

### 8.2 sysfs接口

#### 8.2.1 设备信息

```
/sys/class/uio/uioX/
├── name              # 设备名称（"igb_uio"）
├── version           # 驱动版本（"0.1"）
├── dev               # 设备号（major:minor）
└── maps/
    ├── map0/         # BAR0映射信息
    │   ├── name      # "BAR0"
    │   ├── addr      # 物理地址
    │   └── size      # 大小
    └── map1/         # BAR1映射信息（如果有）
```

#### 8.2.2 配置空间访问

```
/sys/class/uio/uioX/device/config
    └─> 直接访问PCI配置空间
        └─> 通过pread/pwrite操作
```

#### 8.2.3 SR-IOV管理

```
/sys/bus/pci/devices/.../max_vfs
    ├─> 读取：显示当前VF数量
    └─> 写入：设置VF数量
        ├─> 0：禁用SR-IOV
        └─> N：启用SR-IOV并设置N个VF
```

## 9. 完整调用流程图

### 9.1 模块加载和设备绑定

```
┌─────────────────────────────────────────────────────────┐
│  1. 加载igb_uio模块                                      │
│     insmod igb_uio.ko [intr_mode=msix]                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 模块初始化                                           │
│     igbuio_pci_init_module()                            │
│     └─> pci_register_driver()                           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 绑定PCI设备                                          │
│     方式1: echo "0000:01:00.0" > .../bind               │
│     方式2: dpdk-devbind.py --bind=igb_uio 01:00.0       │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 内核调用探测函数                                     │
│     igbuio_pci_probe()                                   │
│     ├─> pci_enable_device()                             │
│     ├─> igbuio_setup_bars()                             │
│     ├─> uio_register_device()                           │
│     └─> 创建/dev/uioX和/sys/class/uio/uioX              │
└─────────────────────────────────────────────────────────┘
```

### 9.2 DPDK应用使用设备

```
┌─────────────────────────────────────────────────────────┐
│  1. DPDK应用启动                                         │
│     rte_eal_init()                                       │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  2. 扫描PCI设备                                          │
│     pci_scan()                                           │
│     └─> 读取/sys/bus/pci/devices/.../driver             │
│         └─> 检查是否为"igb_uio"                          │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  3. 打开UIO设备                                          │
│     open("/dev/uioX", O_RDWR)                           │
│     └─> igbuio_pci_open()                                │
│         └─> igbuio_pci_enable_interrupts()              │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  4. 映射BAR资源                                          │
│     mmap(/sys/.../resourceY)                             │
│     └─> 映射到用户空间虚拟地址                           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  5. 注册中断回调                                         │
│     rte_intr_callback_register()                         │
│     └─> epoll_ctl() 添加到epoll监听                     │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  6. 中断处理循环                                         │
│     while (1) {                                          │
│         epoll_wait()  // 等待中断                        │
│         read(uio_fd)  // 读取中断计数                    │
│         调用回调函数                                      │
│     }                                                    │
└─────────────────────────────────────────────────────────┘
```

### 9.3 中断处理详细流程

```
硬件中断发生
    ↓
┌─────────────────────────────────────────────────────────┐
│  内核中断处理                                            │
│  igbuio_pci_irqhandler()                                │
│  ├─> Legacy模式: pci_check_and_mask_intx()             │
│  └─> uio_event_notify()                                 │
│      └─> 唤醒等待在/dev/uioX上的read()调用              │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  DPDK中断处理线程                                        │
│  read(uio_fd) 返回（不再阻塞）                          │
│  └─> 读取中断计数                                       │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  调用用户回调函数                                        │
│  callback_fn(callback_arg)                              │
│  └─> 用户定义的中断处理逻辑                              │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  重新使能中断（如果需要）                                │
│  write(uio_fd, &value=1, sizeof(value))                 │
│  └─> igbuio_pci_irqcontrol(info, 1)                    │
│      └─> pci_msi_unmask_irq() 或 pci_intx()            │
└─────────────────────────────────────────────────────────┘
```

## 10. 关键函数详解

### 10.1 igbuio_pci_probe()

**功能**：PCI设备探测和初始化

**详细步骤**：

1. **设备检查**
   ```c
   if (pci_is_bridge(dev))
       return -ENODEV;  // 忽略PCI桥设备
   ```

2. **分配设备结构**
   ```c
   udev = kzalloc(sizeof(struct rte_uio_pci_dev), GFP_KERNEL);
   ```

3. **启用PCI设备**
   ```c
   pci_enable_device(dev);  // 启用I/O和内存空间
   pci_set_master(dev);     // 设置总线主控（允许DMA）
   ```

4. **设置BAR资源**
   ```c
   igbuio_setup_bars(dev, &udev->info);
   // 遍历所有BAR，区分内存和I/O端口
   ```

5. **设置DMA掩码**
   ```c
   pci_set_dma_mask(dev, DMA_BIT_MASK(64));           // 64位DMA
   pci_set_consistent_dma_mask(dev, DMA_BIT_MASK(64)); // 一致性DMA
   ```

6. **填充UIO信息**
   ```c
   udev->info.name = "igb_uio";
   udev->info.irqcontrol = igbuio_pci_irqcontrol;
   udev->info.open = igbuio_pci_open;
   udev->info.release = igbuio_pci_release;
   ```

7. **注册UIO设备**
   ```c
   uio_register_device(&dev->dev, &udev->info);
   // 创建/sys/class/uio/uioX
   // 可能创建/dev/uioX（如果启用）
   ```

### 10.2 igbuio_pci_enable_interrupts()

**功能**：按照优先级使能中断

**中断模式选择逻辑**：

```
首选：MSI-X
    ↓ 失败
次选：MSI
    ↓ 失败
再次：Legacy INTx
    ↓ 失败
最后：无中断模式
```

**实现细节**：

```c
// MSI-X尝试
if (pci_alloc_irq_vectors(dev, 1, 1, PCI_IRQ_MSIX) == 1) {
    udev->info.irq = pci_irq_vector(dev, 0);
    udev->mode = RTE_INTR_MODE_MSIX;
    break;
}

// MSI尝试（如果MSI-X失败）
if (pci_alloc_irq_vectors(dev, 1, 1, PCI_IRQ_MSI) == 1) {
    udev->info.irq = pci_irq_vector(dev, 0);
    udev->mode = RTE_INTR_MODE_MSI;
    break;
}

// Legacy尝试（如果MSI失败）
if (pci_intx_mask_supported(dev)) {
    udev->info.irq = dev->irq;
    udev->mode = RTE_INTR_MODE_LEGACY;
    udev->info.irq_flags = IRQF_SHARED | IRQF_NO_THREAD;
    break;
}

// 注册中断处理函数
if (udev->info.irq != UIO_IRQ_NONE)
    request_irq(udev->info.irq, igbuio_pci_irqhandler,
                udev->info.irq_flags, udev->info.name, udev);
```

### 10.3 igbuio_pci_irqcontrol()

**功能**：从用户空间控制中断的使能/禁用

**实现**：

```c
// MSI-X/MSI模式
if (mode == MSIX || mode == MSI) {
    if (irq_state == 1)
        pci_msi_unmask_irq(irq);  // 取消屏蔽
    else
        pci_msi_mask_irq(irq);    // 屏蔽
}

// Legacy模式
if (mode == LEGACY)
    pci_intx(pdev, !!irq_state);  // 使能/禁用INTx
```

### 10.4 igbuio_setup_bars()

**功能**：设置所有PCI BAR到UIO资源

**实现逻辑**：

```c
for (i = 0; i < 6; i++) {  // BAR0-BAR5
    if (BAR有效) {
        flags = pci_resource_flags(dev, i);
        if (flags & IORESOURCE_MEM) {
            // 内存BAR
            igbuio_pci_setup_iomem(dev, info, iom++, i, "BARx");
        } else if (flags & IORESOURCE_IO) {
            // I/O端口BAR
            igbuio_pci_setup_ioport(dev, info, iop++, i, "BARx");
        }
    }
}
```

## 11. DPDK如何使用igb_uio

### 11.1 设备发现

**位置**：`dpdk/drivers/bus/pci/linux/pci_uio.c`

**关键函数**：`pci_get_uio_dev()`

```c
// 查找UIO设备
snprintf(dirname, "%s/.../uio", sysfs_path);
dir = opendir(dirname);
// 查找uioX目录
// 返回UIO编号
```

### 11.2 打开UIO设备

**关键函数**：`pci_uio_alloc_resource()`

```c
// 1. 查找UIO设备
uio_num = pci_get_uio_dev(dev, dirname, ...);

// 2. 打开UIO设备文件
dev->intr_handle.fd = open("/dev/uioX", O_RDWR);
    └─> 触发 igbuio_pci_open()

// 3. 打开配置空间文件
dev->intr_handle.uio_cfg_fd = open("/sys/.../config", O_RDWR);
```

### 11.3 映射BAR资源

**关键函数**：`pci_uio_map_resource_by_index()`

```c
// 1. 打开资源文件
fd = open("/sys/.../resourceY", O_RDWR);
// 或（如果wc_activate）
fd = open("/sys/.../resourceY_wc", O_RDWR);

// 2. 映射到用户空间
mapaddr = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    └─> 映射到igb_uio设置的物理地址

// 3. 保存映射地址
dev->mem_resource[res_idx].addr = mapaddr;
```

### 11.4 中断处理

**关键函数**：`rte_intr_callback_register()`

```c
// 1. 注册回调
rte_intr_callback_register(intr_handle, callback_fn, callback_arg);

// 2. 添加到epoll监听
epoll_ctl(epfd, EPOLL_CTL_ADD, intr_handle->fd, ...);

// 3. 中断处理线程
while (1) {
    epoll_wait(epfd, events, ...);  // 等待中断
    read(intr_handle->fd, &count, sizeof(count));  // 读取中断计数
    // 调用回调函数
}
```

## 12. SPDK如何使用igb_uio

### 12.1 初始化流程

```
SPDK应用
    ↓
spdk_env_init()
    ├─> rte_eal_init()  // 初始化DPDK EAL
    │   └─> 扫描PCI设备
    │       └─> 发现绑定到igb_uio的设备
    └─> pci_env_init()  // SPDK PCI环境初始化
        └─> scan_pci_bus()  // 扫描PCI总线
            └─> 对每个设备调用驱动probe
```

### 12.2 设备附加流程

```
SPDK应用
    ↓
spdk_pci_device_attach()
    ├─> rte_eal_hotplug_add("pci", bdf, "")
    │   └─> 如果设备未绑定，触发绑定
    │       └─> igbuio_pci_probe()
    └─> pci_device_init()
        └─> 调用驱动回调函数
```

### 12.3 BAR访问

```
SPDK应用
    ↓
spdk_pci_device_map_bar()
    ├─> 使用DPDK的BAR映射
    │   └─> pci_uio_map_resource_by_index()
    └─> 返回用户空间虚拟地址
```

## 13. 关键sysfs路径

### 13.1 设备绑定

```
/sys/bus/pci/devices/0000:01:00.0/
├── driver -> ../../drivers/igb_uio  # 当前绑定的驱动
├── uio/
│   └── uio0/                        # UIO设备目录
│       ├── name                     # "igb_uio"
│       ├── version                  # "0.1"
│       └── maps/
│           └── map0/                # BAR映射信息
└── resource0, resource1, ...        # BAR资源文件
```

### 13.2 UIO设备信息

```
/sys/class/uio/uio0/
├── name                             # 设备名称
├── version                          # 驱动版本
├── dev                              # 设备号
├── device/                          # 指向PCI设备
└── maps/
    ├── map0/                        # 第一个BAR
    │   ├── name                     # "BAR0"
    │   ├── addr                     # 物理地址
    │   └── size                     # 大小
    └── map1/                        # 第二个BAR（如果有）
```

## 14. 模块参数

### 14.1 intr_mode

**参数名**：`intr_mode`

**可选值**：
- `msix`（默认）：使用MSI-X中断
- `msi`：使用MSI中断
- `legacy`：使用Legacy INTx中断

**使用示例**：
```bash
insmod igb_uio.ko intr_mode=msix
```

### 14.2 wc_activate

**参数名**：`wc_activate`

**可选值**：
- `0`（默认）：禁用写合并（Write Combining）
- 非0：启用写合并

**使用示例**：
```bash
insmod igb_uio.ko wc_activate=1
```

**说明**：
- 写合并可以提高某些写操作的性能
- 启用时，BAR不会映射到内核虚拟地址（internal_addr = NULL）
- 用户空间直接映射物理地址

## 15. 错误处理

### 15.1 常见错误

1. **设备已被其他驱动绑定**
   - 错误：`Device or resource busy`
   - 解决：先解绑原驱动

2. **内核锁定**
   - 错误：`kernel lock down is enabled`
   - 解决：禁用安全启动或内核锁定

3. **中断分配失败**
   - 错误：所有中断模式都失败
   - 解决：检查硬件支持，使用无中断模式

4. **BAR映射失败**
   - 错误：`Cannot map resource`
   - 解决：检查设备状态，确保设备已启用

## 16. 性能考虑

### 16.1 中断模式性能

- **MSI-X**：性能最好，推荐使用
- **MSI**：性能较好
- **Legacy**：性能较差，可能共享中断

### 16.2 写合并（Write Combining）

- 启用WC可以提高写操作性能
- 但需要硬件支持
- 可能影响某些设备的兼容性

## 17. 总结

igb_uio是DPDK/SPDK框架的关键组件，它：

1. **提供用户空间访问**：通过UIO框架将PCI设备暴露给用户空间
2. **管理中断**：支持多种中断模式，自动选择最佳模式
3. **映射资源**：将PCI BAR映射到用户空间，实现零拷贝访问
4. **支持SR-IOV**：通过sysfs接口管理虚拟功能

**调用关系**：
- 内核PCI子系统 → igb_uio模块 → UIO框架
- DPDK EAL → 打开/dev/uioX → igb_uio回调函数
- 用户空间应用 → mmap()映射BAR → 直接访问硬件

igb_uio是DPDK高性能的基础，它消除了内核上下文切换的开销，使DPDK应用能够以接近硬件的速度访问PCI设备。
