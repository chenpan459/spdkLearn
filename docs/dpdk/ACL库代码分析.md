# DPDK `lib/acl` 代码分析

本文档分析目录 `spdk/dpdk/lib/acl` 的实现，重点回答三个问题：

1. 这个库在编译时如何组织；
2. 规则如何从“用户规则”转换为“运行时查表结构”；
3. 分类（classify）阶段如何在标量/SIMD路径执行。

---

## 1. 功能定位

`lib/acl` 是 DPDK 的多字段分类器（Access Control List classifier）库。它不是简单线性匹配，而是把规则集预构建为多棵 trie + 压缩转移表，在运行时按字节推进状态机，返回每个 category 的最高优先级命中结果。

对外 API 在 `rte_acl.h`，典型使用顺序：

- `rte_acl_create()` 创建上下文；
- `rte_acl_add_rules()` 加规则；
- `rte_acl_build()` 构建运行时结构；
- `rte_acl_classify()` 或 `rte_acl_classify_alg()` 做批量分类。

---

## 2. 编译规则（`meson.build`）

`lib/acl/meson.build` 的规则可以概括为：**核心逻辑 + 架构加速后端**。

- 基础源码始终编译：`acl_bld.c`、`acl_gen.c`、`acl_run_scalar.c`、`rte_acl.c`、`tb_mem.c`。
- x86 额外编译 `acl_run_sse.c`，并把 `acl_run_avx2.c`、`acl_run_avx512.c` 放到 AVX2/AVX512 专用对象构建链。
- ARM 编译 `acl_run_neon.c`（并加 `-flax-vector-conversions`）。
- PPC64 编译 `acl_run_altivec.c`。
- Windows 平台直接禁用该库。

这意味着：

- API 层统一在 `rte_acl.c`；
- 真正的 classify 内核由 `classify_fns[]` 在运行时选择；
- 某 ISA 不可用时，`rte_acl.c` 提供 `-ENOTSUP` 的 dummy fallback，保证链接完整。

---

## 3. 核心数据结构

### 3.1 用户可见结构（`rte_acl.h`）

- `rte_acl_field_def`：描述字段类型、大小、偏移、输入分组；
- `rte_acl_rule_data`：`category_mask` + `priority` + `userdata`；
- `rte_acl_rule`：规则头 + N 个字段；
- `rte_acl_config`：构建参数（字段定义、category 数、`max_size` 限制）。

关键约束：

- `categories` 必须为 1 或 `RTE_ACL_RESULTS_MULTIPLIER` 的倍数；
- 输入数据字段为网络序（MSB）；
- 规则优先级范围受 `RTE_ACL_MIN_PRIORITY` / `RTE_ACL_MAX_PRIORITY` 约束。

### 3.2 内部结构（`acl.h`）

- `rte_acl_ctx`：ACL 上下文主对象，保存规则区、trie 元信息、转移表、匹配表、默认算法等；
- `rte_acl_node`：构建期节点，包含 bitset、子指针集合、节点类型（SINGLE/QRANGE/DFA/MATCH）；
- `rte_acl_trie`：每棵 trie 的根索引、规则数、data_index 映射信息；
- `trans_table`：运行时核心查表数组（`uint64_t` transition entries）。

`acl.h` 里明确了 transition 编码思想：每个状态转移项同时编码 **节点类型 + 节点地址 + 节点类型特定信息**，运行时可以少分支快速推进。

---

## 4. 规则构建阶段（`acl_bld.c`）

`rte_acl_build()` 的主流程：

1. 参数校验（字段数、category 数、字段类型/大小合法性）；
2. 清理旧构建结果（`acl_build_reset`）；
3. 以 `node_max` 启发式循环尝试构建（必要时减半重试）；
4. `acl_bld()` 生成构建期 trie；
5. `rte_acl_gen()` 生成运行时压缩结构；
6. 写回 `ctx->trie[] / data_indexes / first_load_sz / config`。

### 4.1 构建期内存策略

- `tb_mem.c` 提供临时内存池 `tb_alloc`；
- 内存不足时不是返回 NULL，而是 `siglongjmp(pool->fail, -ENOMEM)`；
- `acl_bld()` 用 `sigsetjmp` 接住失败并统一错误返回；
- 构建结束统一 `tb_free_pool` 释放临时内存。

该策略减少了大量深层错误传递分支，适合构建期大量短生命周期对象。

### 4.2 多 trie 与规则切分

`acl_build_tries()` 会根据规则特征和野值（wildness）做规则集拆分，可能生成多棵 trie（上限 `RTE_ACL_MAX_TRIES`）。这样做的目的：

- 降低单棵 trie 的节点爆炸；
- 减小运行时状态转移表体积；
- 让并行 classify 更容易把多 trie 映射到固定宽度执行槽位。

---

## 5. 运行时结构生成（`acl_gen.c`）

`acl_gen.c` 的职责是把构建期节点图转换为紧凑的运行时数组：

- 统计并判定节点类型（MATCH/SINGLE/QRANGE/DFA）；
- 计算每类节点在 `trans_table` 中的布局索引；
- 生成 match results 区和 transition 区；
- 做 DFA group-of-64 的压缩与重排，减少重复块；
- 校验总内存不超过 `cfg->max_size`（超限会返回错误，如 `-ERANGE` 路径触发上层重试）。

结果是 `ctx->trans_table` + `ctx->match_index` + 每 trie 的 `root_index/data_index`，供 classify 直接访问。

---

## 6. 分类执行路径（`acl_run*.c` + `acl_run.h`）

### 6.1 调度入口（`rte_acl.c`）

- `rte_acl_classify()` 调用 `rte_acl_classify_alg(ctx->alg)`；
- `rte_acl_set_ctx_classify()` 可显式设算法；
- `RTE_ACL_CLASSIFY_DEFAULT` 会自动选择“当前平台最佳算法”（x86 优先 AVX512/AVX2/SSE，ARM 优先 NEON，PPC 优先 ALTIVEC，最后回退 scalar）。

算法选择同时检查两类条件：

- 编译时是否具备该 ISA 代码（如 AVX512 编译支持）；
- 运行时 CPU flag 与 `rte_vect_get_max_simd_bitwidth()` 是否满足。

### 6.2 统一执行框架（`acl_run.h`）

`acl_run.h` 抽象了所有后端共享的“并行 trie 遍历框架”：

- `acl_flow_data`：管理批量 packet、当前 trie、活动槽位；
- `completion`：保存每个包的中间优先级与结果；
- `acl_start_next_trie()`：填充空槽为“下一棵 trie”；
- `acl_match_check()`：命中 MATCH 节点时合并优先级并续填槽位。

也就是说，不同 ISA 主要差在“每轮能并行跑几个槽、如何算 transition”。

### 6.3 标量路径（`acl_run_scalar.c`）

- 每次并行处理 2 个搜索槽（`MAX_SEARCHES_SCALAR = 2`）；
- 每轮按 4 字节推进（`GET_NEXT_4BYTES` + 4 次 transition）；
- `scalar_transition()` 根据节点类型（DFA vs QRANGE/SINGLE）计算下个地址；
- 命中后用 `resolve_priority_scalar()` 按 category 做优先级比较更新。

---

## 7. 结果语义与优先级

`rte_acl_classify*` 的输出是“每包每 category 一个 `userdata` 结果”。如果多个规则同时命中同一 category，选择 **priority 最大** 的规则。

多 trie 场景下，`completion.priority[]` 记录运行中最优值，直到该包的所有 trie 都完成才稳定输出。

---

## 8. 代码分工速查

| 文件 | 主要职责 |
|---|---|
| `rte_acl.h` | 对外 API 与规则/配置定义 |
| `rte_acl.c` | 上下文生命周期、算法选择、API 调度 |
| `acl_bld.c` | 规则分析、构建期 trie 生成 |
| `acl_gen.c` | 构建期节点 -> 运行时转移表压缩布局 |
| `acl_run_scalar.c` | 标量 classify 内核 |
| `acl_run_sse.c` / `acl_run_avx2.c` / `acl_run_avx512.c` | x86 SIMD classify |
| `acl_run_neon.c` | ARM NEON classify |
| `acl_run_altivec.c` | PPC ALTIVEC classify |
| `acl_run.h` | 后端共享运行时框架与辅助函数 |
| `tb_mem.c` | 构建期临时内存池 |

---

## 9. 一句话总结

`lib/acl` 的设计是“**构建期重、运行期轻**”：先把规则集离线压缩成多 trie + 紧凑 transition 表，运行时再用标量或 SIMD 后端做批量字节状态机遍历，并按 category 维护最高优先级命中结果，从而在复杂多字段匹配场景获得稳定吞吐。
