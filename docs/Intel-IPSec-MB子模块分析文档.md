# Intel-IPSec-MB子模块作用与设计原理分析文档

## 目录
1. [概述](#概述)
2. [模块架构](#模块架构)
3. [核心功能模块](#核心功能模块)
4. [多缓冲区设计原理](#多缓冲区设计原理)
5. [性能优化策略](#性能优化策略)
6. [在SPDK中的应用](#在spdk中的应用)

---

## 概述

### Intel-IPSec-MB简介

**Intel Multi-Buffer Crypto for IPsec Library (Intel-IPSec-MB)** 是Intel开发的高性能IPSec加密库，专门针对Intel处理器优化的软件实现。该库提供了IPSec核心加密处理功能，在Intel处理器上提供业界领先的性能。

### 核心特性

- **多缓冲区处理**: 同时处理多个加密作业，提高吞吐量
- **SIMD优化**: 充分利用SSE、AVX、AVX2、AVX512指令集
- **硬件加速**: 利用AES-NI、PCLMULQDQ等硬件指令
- **乱序执行**: 支持乱序(out-of-order)调度，提高CPU利用率
- **多种算法**: 支持多种加密和完整性算法

### 主要功能

1. **加密算法**: AES-GCM, AES-CBC, AES-CTR, AES-ECB, DES, 3DES等
2. **完整性算法**: HMAC-SHA1/224/256/384/512, AES-XCBC, AES-CMAC, AES-GMAC等
3. **组合模式**: 加密+完整性验证的组合操作
4. **无线算法**: KASUMI, ZUC, SNOW3G (3GPP标准)

---

## 模块架构

### 目录结构

```
intel-ipsec-mb/
├── sse/              # SSE优化实现
├── avx/              # AVX优化实现
├── avx2/             # AVX2优化实现
├── avx512/           # AVX512优化实现
├── no-aesni/         # 无AES-NI的软件实现
├── include/          # 内部头文件
├── LibTestApp/       # 测试应用
├── LibPerfApp/       # 性能测试应用
├── intel-ipsec-mb.h  # 公共API头文件
└── mb_mgr_code.h     # 多缓冲区管理器代码
```

### 架构设计特点

1. **多版本实现**: 每个算法提供多个SIMD版本
   - SSE版本: 使用SSE4.1指令集
   - AVX版本: 使用AVX指令集
   - AVX2版本: 使用AVX2指令集
   - AVX512版本: 使用AVX512指令集
   - VAES版本: 使用VAES和VPCLMULQDQ扩展（AVX512）

2. **运行时选择**: 根据CPU特性自动选择最优实现

3. **汇编优化**: 关键路径使用汇编语言实现

---

## 核心功能模块

### 1. 多缓冲区管理器 (Multi-Buffer Manager)

#### 作用

多缓冲区管理器是Intel-IPSec-MB的核心组件，负责管理多个加密作业的调度和执行。

#### 核心概念

**Lane (通道)**: 每个lane可以处理一个独立的加密作业
- SSE: 4个lanes (HMAC-SHA1/256)
- AVX: 4-8个lanes
- AVX2: 8-16个lanes
- AVX512: 16-32个lanes

**Job (作业)**: 表示一个加密/完整性验证任务

#### 关键数据结构

```c
// 多缓冲区管理器
struct MB_MGR {
    // AES乱序调度器
    MB_MGR_AES_OOO aes_ooo;
    
    // HMAC乱序调度器
    MB_MGR_HMAC_SHA_1_OOO hmac_sha_1_ooo;
    MB_MGR_HMAC_SHA_256_OOO hmac_sha_256_ooo;
    MB_MGR_HMAC_SHA_512_OOO hmac_sha_512_ooo;
    
    // 作业数组
    JOB_AES_HMAC jobs[MAX_JOBS];
    
    // 函数指针（根据CPU特性设置）
    get_next_job_t get_next_job;
    submit_job_t submit_job;
    flush_job_t flush_job;
    get_completed_job_t get_completed_job;
};

// AES乱序调度器
struct MB_MGR_AES_OOO {
    AES_ARGS args;                    // AES参数
    uint16_t lens[16];                // 每个lane的数据长度
    uint64_t unused_lanes;            // 未使用的lane列表
    JOB_AES_HMAC *job_in_lane[16];    // 每个lane的作业指针
    uint64_t num_lanes_inuse;         // 正在使用的lane数量
};

// 作业结构
struct JOB_AES_HMAC {
    const void *aes_enc_key_expanded;  // 加密密钥（已扩展）
    const void *aes_dec_key_expanded;  // 解密密钥（已扩展）
    const uint8_t *src;                // 源数据
    uint8_t *dst;                      // 目标数据
    uint64_t msg_len_to_cipher_in_bytes;  // 加密数据长度
    uint64_t msg_len_to_hash_in_bytes;    // 哈希数据长度
    const uint8_t *iv;                 // 初始化向量
    uint8_t *auth_tag_output;          // 认证标签输出
    JOB_STS status;                    // 作业状态
    JOB_CIPHER_MODE cipher_mode;       // 加密模式
    JOB_HASH_ALG hash_alg;             // 哈希算法
    // ...
};
```

#### 关键API

```c
// 分配多缓冲区管理器
MB_MGR *alloc_mb_mgr(uint64_t flags);

// 获取下一个可用作业
JOB_AES_HMAC *get_next_job(MB_MGR *state);

// 提交作业
JOB_AES_HMAC *submit_job(MB_MGR *state);

// 刷新作业（强制完成）
JOB_AES_HMAC *flush_job(MB_MGR *state);

// 获取已完成的作业
JOB_AES_HMAC *get_completed_job(MB_MGR *state);
```

#### 工作流程

```
1. 获取作业: job = get_next_job(mgr)
2. 填充作业: 设置job的各个字段（密钥、数据、IV等）
3. 提交作业: submit_job(mgr)  // 将job添加到队列
4. 处理作业: 库内部使用SIMD指令并行处理多个作业
5. 获取结果: completed_job = get_completed_job(mgr)
6. 检查状态: if (completed_job->status == STS_COMPLETED)
```

---

### 2. 加密算法模块

#### 支持的加密算法

**AES系列**:
- **AES-GCM**: Galois/Counter Mode（认证加密）
- **AES-CBC**: Cipher Block Chaining
- **AES-CTR**: Counter Mode
- **AES-ECB**: Electronic Codebook
- **AES-CCM**: Counter with CBC-MAC

**其他算法**:
- **DES/3DES**: 数据加密标准（遗留支持）
- **KASUMI-F8**: 3GPP加密算法
- **ZUC-EEA3**: 中国标准加密算法
- **SNOW3G-UEA2**: 3GPP加密算法

#### AES-GCM实现原理

**GCM模式特点**:
- 同时提供加密和认证
- 使用CTR模式进行加密
- 使用GHASH进行认证

**关键数据结构**:
```c
// GCM密钥数据
struct gcm_key_data {
    uint8_t expanded_keys[16 * 15];   // 扩展密钥
    union {
        // SSE/AVX版本
        struct {
            uint8_t shifted_hkey[16 * 8];      // 预计算的哈希密钥
            uint8_t shifted_hkey_k[16 * 8];    // Karatsuba乘法密钥
        } sse_avx;
        // AVX2/AVX512版本
        struct {
            uint8_t shifted_hkey[16 * 8];
        } avx2_avx512;
        // VAES版本
        struct {
            uint8_t shifted_hkey[16 * 48];     // 更多预计算密钥
        } vaes_avx512;
    } ghash_keys;
};

// GCM上下文
struct gcm_context_data {
    uint8_t aad_hash[16];              // AAD哈希值
    uint64_t aad_length;               // AAD长度
    uint64_t in_length;                // 输入长度
    uint8_t current_counter[16];       // 当前计数器
    // ...
};
```

**GCM处理流程**:
```
1. 密钥扩展: aes_gcm_pre_128(key, key_data)
   - 扩展AES密钥
   - 预计算GHASH密钥

2. 初始化: aes_gcm_init(key_data, ctx, iv, aad, aad_len)
   - 初始化计数器
   - 处理AAD（Additional Authenticated Data）

3. 加密/解密: aes_gcm_enc_dec_update(key_data, ctx, dst, src, len)
   - CTR模式加密/解密
   - 同时更新GHASH

4. 完成: aes_gcm_enc_dec_finalize(key_data, ctx, auth_tag, tag_len)
   - 生成认证标签
```

**性能优化**:
- **预计算**: 预计算GHASH所需的密钥表
- **并行处理**: 一次处理多个块（by8表示一次8个块）
- **硬件加速**: 使用PCLMULQDQ指令加速GF(2^128)乘法

---

### 3. 完整性算法模块

#### 支持的完整性算法

**HMAC系列**:
- **HMAC-SHA1-96**: SHA1的HMAC，96位截断
- **HMAC-SHA2-224/256/384/512**: SHA2系列的HMAC

**AES系列**:
- **AES-XCBC-96**: AES扩展CBC-MAC
- **AES-CMAC-96**: AES CMAC
- **AES-GMAC**: GCM的认证部分

**其他**:
- **HMAC-MD5-96**: MD5的HMAC（遗留支持）
- **KASUMI-F9**: 3GPP完整性算法
- **ZUC-EIA3**: 中国标准完整性算法
- **SNOW3G-UIA2**: 3GPP完整性算法

#### HMAC实现原理

**HMAC算法**:
```
HMAC(K, m) = H((K ⊕ opad) || H((K ⊕ ipad) || m))

其中:
- K: 密钥
- m: 消息
- H: 哈希函数（SHA1/SHA2等）
- opad: 外部填充（0x5c重复）
- ipad: 内部填充（0x36重复）
```

**多缓冲区HMAC**:
```c
// HMAC-SHA1乱序调度器
struct MB_MGR_HMAC_SHA_1_OOO {
    SHA1_ARGS args;                    // SHA1参数数组
    uint16_t lens[16];                 // 每个lane的长度
    uint64_t unused_lanes;             // 未使用的lane
    HMAC_SHA1_LANE_DATA ldata[16];     // 每个lane的数据
    uint32_t num_lanes_inuse;          // 使用的lane数
};

// Lane数据
struct HMAC_SHA1_LANE_DATA {
    uint8_t extra_block[2 * 64 + 8];   // 额外块（用于填充）
    JOB_AES_HMAC *job_in_lane;         // 作业指针
    uint8_t outer_block[64];           // 外部块
    uint32_t outer_done;               // 外部块处理标志
    // ...
};
```

**处理流程**:
```
1. 提交作业: submit_job_hmac_sha1(mgr)
   - 将作业分配到可用lane
   - 处理内部哈希（K ⊕ ipad || m）

2. 刷新作业: flush_job_hmac_sha1(mgr)
   - 完成所有lane的处理
   - 处理外部哈希（K ⊕ opad || H_internal）
   - 生成认证标签

3. 获取结果: get_completed_job(mgr)
   - 返回已完成的作业
```

**性能优化**:
- **并行处理**: 同时处理多个HMAC作业
- **硬件加速**: 使用SHA-NI指令（如果可用）
- **批量处理**: 一次处理多个块

---

### 4. 组合操作 (Chained Operations)

#### 作用

支持加密和完整性验证的组合操作，这是IPSec的常见需求。

#### 组合模式

**CIPHER_HASH**: 先加密后哈希
```
加密 → 完整性验证
```

**HASH_CIPHER**: 先哈希后加密
```
完整性验证 → 加密
```

#### 实现原理

```c
// 作业结构中的组合字段
struct JOB_AES_HMAC {
    JOB_CHAIN_ORDER chain_order;       // 组合顺序
    uint64_t cipher_start_src_offset_in_bytes;  // 加密起始偏移
    uint64_t hash_start_src_offset_in_bytes;    // 哈希起始偏移
    // ...
};

// 处理流程
void process_chained_job(JOB_AES_HMAC *job)
{
    if (job->chain_order == CIPHER_HASH) {
        // 1. 先加密
        aes_encrypt(job);
        // 2. 后哈希（对加密后的数据）
        hmac_compute(job);
    } else {
        // 1. 先哈希
        hmac_compute(job);
        // 2. 后加密
        aes_encrypt(job);
    }
}
```

---

## 多缓冲区设计原理

### 核心思想

**多缓冲区技术**通过同时处理多个独立的加密作业来提高CPU利用率，充分利用SIMD指令集的并行能力。

### 设计原理

#### 1. Lane分配机制

**原理**: 将多个作业分配到不同的lane，使用SIMD指令并行处理。

**实现**:
```c
// 获取下一个可用lane
int get_unused_lane(MB_MGR_AES_OOO *mgr)
{
    // unused_lanes是一个位图或列表
    // 每个nibble/byte表示一个lane索引
    int lane = extract_lane(mgr->unused_lanes);
    mgr->unused_lanes = remove_lane(mgr->unused_lanes, lane);
    return lane;
}

// 分配作业到lane
void assign_job_to_lane(MB_MGR_AES_OOO *mgr, JOB_AES_HMAC *job, int lane)
{
    mgr->job_in_lane[lane] = job;
    mgr->lens[lane] = job->msg_len_to_cipher_in_bytes;
    mgr->num_lanes_inuse++;
}
```

#### 2. 批量处理

**原理**: 当lane填满时，使用SIMD指令批量处理所有lane。

**示例**: AES-CBC加密（AVX版本，8个lane）
```c
void aes_cbc_enc_128_x8(MB_MGR_AES_OOO *mgr)
{
    // 加载8个lane的IV
    __m256i iv0 = _mm256_loadu_si256((__m256i*)mgr->args.IV[0]);
    __m256i iv1 = _mm256_loadu_si256((__m256i*)mgr->args.IV[1]);
    // ...
    
    // 加载8个lane的数据
    __m256i data0 = _mm256_loadu_si256((__m256i*)mgr->args.in[0]);
    // ...
    
    // 并行加密（使用AES-NI指令）
    __m256i encrypted0 = _mm256_aesenc_epi128(data0, key);
    // ...
    
    // 存储结果
    _mm256_storeu_si256((__m256i*)mgr->args.out[0], encrypted0);
    // ...
}
```

#### 3. 乱序执行

**原理**: 作业可以乱序完成，不需要按照提交顺序。

**优势**:
- 提高CPU利用率
- 减少等待时间
- 适应不同大小的作业

**实现**:
```c
// 提交作业（不立即执行）
JOB_AES_HMAC *submit_job(MB_MGR *mgr)
{
    JOB_AES_HMAC *job = get_next_job(mgr);
    
    // 分配lane
    int lane = get_unused_lane(&mgr->aes_ooo);
    assign_job_to_lane(&mgr->aes_ooo, job, lane);
    
    // 如果lane满了，批量处理
    if (mgr->aes_ooo.num_lanes_inuse >= NUM_LANES) {
        process_batch(mgr);
    }
    
    return job;
}

// 处理批次
void process_batch(MB_MGR_AES_OOO *mgr)
{
    // 使用SIMD指令并行处理所有lane
    aes_cbc_enc_128_x8(mgr);
    
    // 标记作业完成
    for (int i = 0; i < NUM_LANES; i++) {
        if (mgr->job_in_lane[i]) {
            mgr->job_in_lane[i]->status = STS_COMPLETED_AES;
            mgr->job_in_lane[i] = NULL;
        }
    }
    
    // 清空lane
    mgr->num_lanes_inuse = 0;
    mgr->unused_lanes = ALL_LANES_FREE;
}
```

#### 4. 刷新机制

**原理**: 强制处理所有待处理的作业，即使lane未满。

**实现**:
```c
JOB_AES_HMAC *flush_job(MB_MGR *mgr)
{
    // 处理所有部分填充的lane
    if (mgr->aes_ooo.num_lanes_inuse > 0) {
        // 用NULL填充空lane
        fill_empty_lanes(mgr);
        // 处理批次
        process_batch(mgr);
    }
    
    return get_completed_job(mgr);
}
```

---

## 性能优化策略

### 1. SIMD向量化

#### 原理

使用SIMD指令同时处理多个数据元素。

#### 实现层次

**SSE (4 lanes)**:
```c
// 一次处理4个128位数据
__m128i data0 = _mm_loadu_si128((__m128i*)src[0]);
__m128i data1 = _mm_loadu_si128((__m128i*)src[1]);
__m128i data2 = _mm_loadu_si128((__m128i*)src[2]);
__m128i data3 = _mm_loadu_si128((__m128i*)src[3]);

// 并行AES加密
__m128i enc0 = _mm_aesenc_si128(data0, key);
__m128i enc1 = _mm_aesenc_si128(data1, key);
__m128i enc2 = _mm_aesenc_si128(data2, key);
__m128i enc3 = _mm_aesenc_si128(data3, key);
```

**AVX2 (8 lanes)**:
```c
// 一次处理8个128位数据（使用256位寄存器）
__m256i data01 = _mm256_loadu_si256((__m256i*)src[0]);
__m256i data23 = _mm256_loadu_si256((__m256i*)src[2]);
// ...

// 并行处理
__m256i enc01 = _mm256_aesenc_epi128(data01, key);
// ...
```

**AVX512 (16 lanes)**:
```c
// 一次处理16个128位数据（使用512位寄存器）
__m512i data0_3 = _mm512_loadu_si512(src);
// ...

// 并行处理
__m512i enc0_3 = _mm512_aesenc_epi128(data0_3, key);
// ...
```

### 2. 硬件指令利用

#### AES-NI指令

**AESENC/AESDEC**: AES加密/解密轮
```c
// 单轮AES加密
__m128i aesenc(__m128i data, __m128i round_key);
```

**AESKEYGENASSIST**: AES密钥生成
```c
// 生成下一轮密钥
__m128i aeskeygenassist(__m128i key, int round);
```

#### PCLMULQDQ指令

**GF(2^128)乘法**: 用于GCM的GHASH
```c
// 64位×64位→128位结果（在GF(2^128)中）
__m128i _mm_clmulepi64_si128(__m128i a, __m128i b, int imm8);
```

#### SHA-NI指令（如果可用）

**SHA1/SHA256加速**: 硬件SHA指令
```c
// SHA1消息调度
__m128i _mm_sha1msg1_epu32(__m128i a, __m128i b);
__m128i _mm_sha1msg2_epu32(__m128i a, __m128i b);
```

### 3. 预计算优化

#### GCM密钥预计算

**原理**: 预计算GHASH所需的所有密钥表。

**实现**:
```c
void aes_gcm_precomp_128_sse(struct gcm_key_data *key_data)
{
    // 预计算 HashKey, HashKey^2, ..., HashKey^8
    // 用于加速GHASH计算
    uint8_t hkey[16];
    // ... 计算hkey ...
    
    for (int i = 0; i < 8; i++) {
        // 计算 HashKey^(i+1) << 1 mod poly
        ghash_multiply(hkey, key_data->ghash_keys.sse_avx.shifted_hkey + i*16);
    }
}
```

### 4. 内存对齐优化

#### 对齐要求

- **16字节对齐**: SSE操作
- **32字节对齐**: AVX操作
- **64字节对齐**: AVX512操作

#### 实现

```c
// 对齐的数据结构
DECLARE_ALIGNED(uint8_t buffer[256], 32);  // 32字节对齐

// 对齐检查
if ((uintptr_t)ptr & 0x1F) {
    // 处理未对齐情况
    // 或要求调用者提供对齐的缓冲区
}
```

### 5. 缓存优化

#### 数据布局

- **紧凑布局**: 相关数据放在一起
- **对齐到缓存行**: 64字节对齐
- **预取**: 使用硬件预取指令

---

## 在SPDK中的应用

### 使用场景

1. **IPSec加密**: 在NVMe over Fabrics等场景中提供IPSec加密支持
2. **数据完整性**: 提供数据完整性验证
3. **高性能加密**: 需要高吞吐量的加密场景

### 集成方式

```c
// SPDK中可能的集成示例
#include "intel-ipsec-mb.h"

// 初始化
MB_MGR *mgr = alloc_mb_mgr(0);

// 加密作业
JOB_AES_HMAC *job = get_next_job(mgr);
job->cipher_mode = GCM;
job->hash_alg = AES_GMAC;
job->aes_enc_key_expanded = expanded_key;
job->src = plaintext;
job->dst = ciphertext;
job->iv = iv;
job->msg_len_to_cipher_in_bytes = len;
job->msg_len_to_hash_in_bytes = len;

// 提交
submit_job(mgr);

// 获取结果
JOB_AES_HMAC *completed = get_completed_job(mgr);
if (completed->status == STS_COMPLETED) {
    // 使用加密后的数据
}
```

### 性能影响

- **高吞吐**: 可达数十Gbps的加密吞吐量
- **低延迟**: 批量处理减少延迟
- **CPU效率**: 充分利用SIMD指令，提高CPU利用率

---

## 算法支持矩阵

### 加密算法

| 算法 | SSE | AVX | AVX2 | AVX512 | VAES |
|------|-----|-----|------|--------|------|
| AES128-GCM | by8 | by8 | by8 | by8 | by48 |
| AES256-GCM | by8 | by8 | by8 | by8 | by48 |
| AES128-CBC | x4 | x8 | - | - | x16 |
| AES256-CBC | x4 | x8 | - | - | x16 |
| AES128-CTR | by4 | by8 | - | - | by16 |
| AES256-CTR | by4 | by8 | - | - | by16 |
| DES/3DES | - | - | - | x16 | - |

### 完整性算法

| 算法 | SSE | AVX | AVX2 | AVX512 |
|------|-----|-----|------|--------|
| HMAC-SHA1-96 | x4 | x4 | x8 | x16 |
| HMAC-SHA256-128 | x4 | x4 | x8 | x16 |
| HMAC-SHA512-256 | x2 | x2 | x4 | x8 |
| AES-XCBC-96 | x4 | x8 | - | - |
| AES-GMAC | by8 | by8 | by8 | by8 |

**说明**:
- **byY**: 单缓冲区，一次处理Y个块
- **xY**: 多缓冲区，同时处理Y个缓冲区

---

## 安全考虑

### 安全选项

1. **SAFE_DATA**: 清除敏感数据（密钥、IV等）
2. **SAFE_PARAM**: 参数验证
3. **SAFE_LOOKUP**: 常量时间查找（防止时序攻击）

### 编译选项

```bash
# 启用安全选项
make SAFE_DATA=y SAFE_PARAM=y SAFE_LOOKUP=y
```

### 算法建议

- **避免使用**: DES, 3DES, HMAC-MD5（遗留算法）
- **推荐使用**: AES-GCM, HMAC-SHA256/512

---

## 总结

Intel-IPSec-MB是SPDK中重要的加密加速组件，通过以下方式提供高性能：

1. **多缓冲区技术**: 同时处理多个作业，提高吞吐量
2. **SIMD优化**: 充分利用现代CPU的并行计算能力
3. **硬件加速**: 利用AES-NI、PCLMULQDQ等硬件指令
4. **乱序执行**: 提高CPU利用率
5. **预计算优化**: 减少运行时计算开销

这些优化使得Intel-IPSec-MB能够提供比标准实现高数倍到数十倍的性能，是SPDK实现高性能加密的关键组件之一。

---

**文档版本**: 1.0  
**最后更新**: 2024年
