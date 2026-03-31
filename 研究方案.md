# Star-schema HIN 动态社区搜索维护研究报告

## 研究背景

本课题关注的是 **Star-schema 异构信息网络（Star-schema HINs）** 上的动态社区搜索维护问题。

在 Star-schema HIN 中，存在一类中心节点类型（center type，如 author）和多类属性节点类型（attribute type，如 paper、venue、topic）。基于元路径（meta-path）的社区模型 `(k, P)-core` 是当前最主流的静态社区搜索框架，CM-tree 是其对应的压缩索引结构。

**研究空白：** 当 HIN 发生动态更新时，如何高效维护 `(k, P)-core` 与 CM-tree，目前没有系统性的解决方案。已有工作要么只针对静态图，要么只针对普通同构图的动态维护，无法直接应用于 Star-schema HIN 的多元路径动态社区维护场景。

本课题完整链路如下：

```
原子更新 → 受影响范围定位 → (k,P)-core 增量维护 → CM-tree 局部修复
```

---

## 核心创新点一：受影响范围快速定位机制

### 一句话描述

提出一种**共享式半路径索引（Half-path Signature Index, HSI）**，将一次 center-node 原子更新快速映射为少量受影响的 meta-path projected-edge 变化，避免全量 path instance 重枚举。

### 问题来源

在 Star-schema HIN 中，一次 center 节点的插入/删除，或 center-attribute 边的插入/删除，会同时影响：

- 多个元路径 `P`
- 每个 `P` 下大量 path instances
- 多个 center-center 的 `P-neighbor` 关系
- 后续多个 `(k,P)-core` 的正确性

如果没有专门的定位机制，动态维护算法需要重新枚举所有受影响 path instances，再构造受影响 projected graph，开销退化成近似全量重算。

> **核心目标：先解决"更新影响谁"，再谈"怎么维护"。**

### 具体做法

#### 1. 原子更新建模

将 Star-schema HIN 上的原子更新形式化为四类：

| 更新类型 | 描述 |
|---|---|
| `insertCenter(c)` | 插入一个新 center 节点 |
| `deleteCenter(c)` | 删除一个 center 节点 |
| `insertEdge(c, a)` | 插入 center-attribute 边 |
| `deleteEdge(c, a)` | 删除 center-attribute 边 |

> 复杂的事务型更新可视为若干原子更新的批处理，算法分析以单个 atomic event 为单位。

#### 2. 半路径分解

对于对称元路径：

```
P = C → A₁ → A₂ → ... → Aᵣ → ... → A₂ → A₁ → C
```

将其拆分为：

```
P = φ · φ⁻¹
```

其中 `φ` 是从 center 出发的**半路径（half-path）**。

这一分解的核心价值是：
- 不同元路径可能共享同一前缀半路径
- 一次更新影响的半路径有限，进而精确确定受影响的完整路径集合

#### 3. 共享半路径索引（HSI）

HSI 维护三类信息：

```
HSI = {
  center → 半路径签名映射,     // 某 center 在半路径 φ 下的局部结构签名
  签名 → center 倒排表,         // 哪些 center 具有相同/兼容签名
  签名变化 → 受影响 meta-path 映射  // 某类签名变化影响哪些 P
}
```

#### 4. 从 path instance 变化压缩到 projected-edge 变化

对每个 meta-path `P`，维护导出图：

```
G_P = (V_C, E_P, w_P)
```

其中：
- `V_C`：所有 center 节点
- `(u,v) ∈ E_P`：u 与 v 之间存在至少一个 P-instance
- `w_P(u,v)`：u 与 v 之间的 path instance 支持数

关键观察：`(k,P)-core` 的结构只依赖 projected graph 中边的存在性，不依赖精确支持数。  
因此，只需关注以下两类"结构性"变化：

- `w_P(u,v): 0 → 1`：新增一条 projected edge
- `w_P(u,v): 1 → 0`：删除一条 projected edge

其他支持数变化不触发结构性维护，大幅压缩了维护代价。

#### 5. 完整流程

```
给定原子更新 δ：
  1. 更新受影响 center 的局部属性信息
  2. 重新计算其半路径签名
  3. 查询 HSI 倒排表，得到可能受影响的 center 集合
  4. 推导受影响 meta-path 集合 P_aff
  5. 只对这些 meta-path 和相关 center pairs 更新 w_P(u,v)
  6. 将支持度变化转化为 projected-edge insert/delete events

输出：受影响 meta-path 集合 + 每个 P 的变更边集合 ΔE_P
```

### 理论价值

- 定义了 Star-schema HIN 的原子更新模型
- 将原图更新与 projected graph 更新建立了形式化桥梁
- 把 path-instance 级维护压缩为 projected-edge 级维护
- 显著减少跨 meta-path 的重复枚举

### 英文贡献表述

> We propose an impact localization mechanism for atomic updates in star-schema HINs. By introducing a shared half-path index (HSI), the method transforms local changes in the original HIN into a compact set of affected meta-path projected-edge changes, avoiding full path-instance re-enumeration.

---

## 核心创新点二：`(k,P)-core` 与 CM-tree 联合增量维护机制

### 一句话描述

提出一种**联合增量维护框架（DynaCM）**，在受影响 projected graph 上批量维护 `(k,P)-core`，并对 CM-tree 执行懒惰失效与局部修复，避免全量重算与整索引重建。

### 问题来源

即使创新点一已经输出了受影响的 `ΔE_P`，后续仍然有两个难点：

**难点 A：如何维护多个 P 上的 `(k,P)-core`**

一次原子更新通常影响多个元路径（如 APA、APTPA、APVPA），独立逐路径重算代价很高。

**难点 B：如何维护 CM-tree**

CM-tree 是静态索引结构，现有工作只解决了静态构建，没有给出：
- 更新后哪些节点失效
- 如何局部修复
- 如何避免整棵树重建

> **核心目标：把 projected-edge changes 高效传递到 `(k,P)-core` 和 CM-tree，只修复真正受影响的区域。**

### 具体做法

#### 1. 每个受影响 P 上的导出图维护

对每个受影响元路径 `P`，维护以下结构：

```
G_P      = (V_C, E_P)         // 导出图
core_P(v)                      // 顶点 v 在 G_P 上的 core number
ord_P(v)                       // 对应的 k-order
adj_P(v)                       // 导出图邻接
ΔE_P⁺ / ΔE_P⁻                 // 本次更新触发的边插入/删除集合
```

#### 2. Batch Order-P Maintenance

传统 order-based core maintenance 只处理普通图的单边更新，而本课题面对的是：
- 多个 projected graphs 同时变化
- 每个图上可能一次变更多条边
- 这些变更来自同一次 HIN 原子更新

为此，提出 **Batch Order-P Maintenance**：

```
对每个受影响 P：
  1. 汇总本次变更边集 ΔE_P = ΔE_P⁺ ∪ ΔE_P⁻
  2. 计算受影响顶点闭包 V_P^aff（变更边端点 + 相邻 shell 顶点）
  3. 只在 V_P^aff 上执行 order-based 批量局部维护
  4. 更新受影响顶点的 core_P 与 ord_P
```

**关键性质（可作为定理）：**

> 对一次原子更新引起的批量 projected-edge changes，可能发生 core number 变化的顶点集合局限于变更边端点诱导出的局部 shell 闭包，不需要遍历整个导出图。

#### 3. Delta-CM：CM-tree 懒惰失效与局部修复

对 CM-tree 引入两阶段动态维护机制：

**第一阶段：标脏（Mark Dirty）**

```
对每个受影响 P：
  1. 找到 CM-tree 中对应节点
  2. 将其标记为 dirty
  3. 根据嵌套关系，将相关祖先/后代节点也标记
```

**第二阶段：局部修复（Lazy Repair）**

```
只对 dirty subtree 做修复：
  1. 重新验证受影响节点的 (k,P)-core 信息
  2. 如果节点结果未变 → 停止向下传播（剪枝）
  3. 如果节点失效 → 修复其子树结构
  4. 如果节点社区变空 → 整棵子树直接剪枝
```

#### 4. 基于嵌套关系的三类剪枝

| 剪枝类型 | 触发条件 | 效果 |
|---|---|---|
| **父节点稳定剪枝** | 父节点社区结果未变 | 大量子节点无需修复 |
| **空节点下推剪枝** | 某节点结果为空 | 后代节点直接失效 |
| **边界不变剪枝** | 某节点核心顶点边界不变 | 下游子节点复用旧结果 |

#### 5. 完整算法框架 HandleAtomicUpdate

```
算法：HandleAtomicUpdate(δ)

输入：Star-schema HIN H，meta-path 集合 P，projected graphs {G_P}，CM-tree T，原子更新 δ
输出：更新后的 {G_P}，(k,P)-core，修复后的 T

Step 1  受影响范围定位（创新点一）
  - 更新局部 center/attribute 信息
  - 更新半路径签名索引 HSI
  - 得到受影响 meta-path 集合 P_aff
  - 得到每个 P 的变更边集合 ΔE_P

Step 2  projected graph 维护
  - 根据 ΔE_P 更新每个 G_P

Step 3  (k,P)-core 批量维护（Batch Order-P）
  - 计算 V_P^aff
  - 只在 V_P^aff 上执行 order-based 批量局部维护
  - 更新 core numbers 与 order

Step 4  CM-tree 局部修复（Delta-CM）
  - 标记 dirty 节点
  - 对 dirty subtree 懒惰修复
  - 利用三类嵌套剪枝减少冗余计算
```

### 理论价值

| 层面 | 贡献 |
|---|---|
| 动态 `(k,P)-core` 维护 | 将 order-based maintenance 推广到 meta-path induced projected graph |
| 批量更新适配 | 支持一次 HIN 原子更新引发的多边、多路径变化 |
| 索引联合维护 | 同步维护 core 结果与 CM-tree，避免整索引重建 |

### 英文贡献表述

> We develop a joint incremental maintenance framework (DynaCM) for multi-meta-path (k,P)-cores and CM-tree. The framework performs batch core maintenance on affected projected graphs and repairs only the dirty region of the index, avoiding whole-graph recomputation and whole-index reconstruction.

---

## 两个创新点的关系

两个创新点前后衔接，构成完整的动态维护链路：

```
原子更新 δ
    ↓
[创新点一] 受影响范围定位（HSI）
    → 受影响 meta-path 集合 P_aff
    → 每个 P 的变更边集合 ΔE_P
    ↓
[创新点二] 联合增量维护（DynaCM）
    → Batch Order-P：维护 (k,P)-core
    → Delta-CM：局部修复 CM-tree
    ↓
维护完成，结果与全量重建一致
```

**创新点一** 解决"更新影响谁"  
**创新点二** 解决"受影响后怎么维护"

---

## 实验设计

### 数据集

| 数据集 | 说明 | 规模 |
|---|---|---|
| DBLP | 学术文献网络，含 author/paper/venue/topic | 中等 |
| IMDB | 电影信息网络，含 movie/actor/director | 中等 |
| Foursquare | 地点签到网络，含 venue/city/user/date | 中等 |

### Baseline 设置

| Baseline | 描述 |
|---|---|
| B1 全量重建 | 每次更新后重新构造所有 G_P、(k,P)-core 和 CM-tree |
| B2 独立逐路径维护 | 对每个受影响 P 独立维护，无共享索引、无 CM-tree 修复 |
| B3 仅维护 core | 只做 (k,P)-core 维护，不做 CM-tree 动态修复 |
| **B4 DynaCM（本方法）** | HSI + Batch Order-P + Delta-CM 完整方法 |

### 评估指标

**效率指标**

| 指标 | 对应验证 |
|---|---|
| 受影响范围定位耗时 | 创新点一效率 |
| projected-core 维护耗时 | 创新点二 core 维护效率 |
| CM-tree 修复耗时 | 创新点二索引修复效率 |
| 总更新耗时 | 整体方法效率 |

**冗余消除指标**

| 指标 | 对应验证 |
|---|---|
| 受影响 meta-path 数量 | HSI 定位精准度 |
| path instance 枚举减少率 | 创新点一压缩效果 |
| dirty 节点数 / 修复节点数 | Delta-CM 局部修复程度 |
| 剪枝节点数 | 嵌套剪枝有效性 |

**正确性指标**

每次更新后，与 B1 全量重建结果对比：
- `(k,P)-core` 顶点集一致率（应为 100%）
- CM-tree 查询结果一致率（应为 100%）

### 更新类型

分别测试：
1. center node 插入
2. center node 删除
3. center-attribute edge 插入
4. center-attribute edge 删除
5. 批量事务更新（若干原子更新的组合）

---

## 论文贡献总结

### 中文版

1. **提出面向 Star-schema HIN 原子更新的受影响范围定位机制**，通过共享半路径索引（HSI）将原图局部变化压缩映射为少量 meta-path 导出边变化，避免全量 path instance 重枚举。

2. **提出 `(k,P)-core` 与 CM-tree 的联合增量维护机制（DynaCM）**，在受影响导出图上进行批量 core 维护，并对 CM-tree 执行局部修复，从而避免整图重算与整索引重建。

### 英文版

1. We propose an impact localization mechanism for atomic updates in star-schema HINs. By introducing a shared half-path index (HSI), our method transforms local updates in the original HIN into a compact set of affected meta-path projected-edge changes, avoiding full path-instance re-enumeration.

2. We propose a joint incremental maintenance framework (DynaCM) for multi-meta-path `(k,P)`-cores and CM-tree. The framework performs batch core maintenance on affected projected graphs and repairs only the dirty region of the index, avoiding whole-graph recomputation and whole-index reconstruction.

---

## 研究范围与收敛建议

为保证论文聚焦，建议将研究范围收敛如下：

### 本课题覆盖
- Star-schema HIN
- 有界长度对称 meta-path（长度 ≤ L）
- center-centric 原子更新
- 标准 `(k,P)-core`（basic 模型）
- CM-tree 局部修复

### 暂不覆盖
- influence / fairness / attribute ranking 等扩展模型
- 任意 schema HIN
- motif-generalized community
- GNN 融合方法
- 复杂权重模型

---

## 参考文献

1. Yangqin Jiang, Yixiang Fang, et al. **Effective Community Search over Large Star-Schema Heterogeneous Information Networks.** PVLDB, 2022.

2. Yixiang Fang, Yixing Yang, et al. **Effective and Efficient Community Search over Large Heterogeneous Information Networks.** PVLDB, 2020.

3. Yikai Zhang, Jeffrey Xu Yu, et al. **A Fast Order-Based Approach for Core Maintenance.** ICDE, 2017.

4. Xun Jian, Yue Wang, Lei Chen. **Effective and Efficient Relational Community Detection and Search in Large Dynamic Heterogeneous Information Networks.** PVLDB, 2020.

5. Alexander Zhou, et al. **Influential Community Search over Large Heterogeneous Information Networks.** PVLDB, 2023.
