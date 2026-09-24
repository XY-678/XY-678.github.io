---
layout: default
title: "ZK-ACORN：面向可验证混合向量检索的研究 Idea"
date: 2026-09-24 00:00:00 +0800
description: "从 ACORN、可验证 ANN 与零知识证明出发，整理 ZK-ACORN 的研究现状、系统设计与下一步路线。"
categories: [向量检索, ACORN, Zero-Knowledge]
---

> 研究主题：将 ACORN 的 Predicate-Aware Approximate Nearest Neighbor Search 与 Zero-Knowledge Proofs 结合，构建“可验证、可审计、可隐藏部分数据”的混合向量检索系统。  
> 文献检索截止：2026-09-22。  
> 说明：本文区分“已有论文已经实现的内容”和“可以继续研究的内容”。其中关于 ZK-ACORN 的系统设计属于研究设想，不是 ACORN 原论文已经实现的功能。

---

## 1. 研究背景

现代向量数据库通常不仅执行向量相似度搜索，还需要同时处理结构化属性条件，例如：

```text
price < 500
year >= 2024
category = "paper"
keyword contains "AI"
```

一个查询可以表示为：

$$
q=(x_q,p_q)
$$

其中：

- `x_q`：查询向量；
- `p_q`：结构化谓词。

数据库中每条记录可以写成：

$$
e_i=(x_i,a_i)
$$

其中：

- `x_i`：向量 embedding；
- `a_i`：结构化 metadata。

满足 predicate 的向量集合为：

$$
X_p=\{x_i\mid p(a_i)=True\}
$$

Hybrid Search 的目标是：

$$
KNN(x_q,X_p)
$$

也就是：

> 在满足属性谓词的记录中，寻找与查询向量最接近的 Top-K。

---

## 2. ACORN 已经解决了什么

ACORN 是一种基于 HNSW 的 filtered / hybrid ANN 方法。

它的核心目标是：

> 不为每一种 predicate 单独建立一张 HNSW，而是构造一张 predicate-agnostic 图，使查询时由合法节点形成的 predicate subgraph 尽量仍然具备 HNSW 的可导航性。

ACORN 的主要机制包括：

1. Predicate subgraph traversal；
2. ACORN-γ 的邻居扩展；
3. 使用 `Mγ` 提高过滤后节点 degree；
4. 使用 `Mβ` compression 减少图存储；
5. 查询时通过 two-hop expansion 恢复被压缩的候选；
6. predicate filtering；
7. selectivity 很低时回退 pre-filtering。

ACORN 的核心直觉是：

$$
M\gamma s\approx M
$$

其中：

- `M`：普通 HNSW 的邻居规模；
- `γ`：ACORN 的 neighbor expansion factor；
- `s`：predicate selectivity。

因此常见配置思路是：

$$
\gamma\approx\frac{1}{s_{\min}}
$$

---

## 3. ACORN 没有解决什么

ACORN 解决的是“搜索效率与 recall”问题。

它默认搜索服务器是可信的。用户通常需要相信：

```text
服务器确实使用了正确的数据集
服务器没有偷偷修改索引
服务器确实执行了规定的 ACORN 算法
服务器没有提前终止搜索
服务器没有故意跳过某些候选
predicate 判断是正确的
two-hop recovery 是正确的
距离计算是正确的
返回结果确实来自这次搜索
```

ACORN 原论文并没有设计密码学证明机制来验证这些性质。

这就是 ZK-ACORN 可以切入的位置。

---

## 4. 目前相关方向已经实现到什么程度

### 4.1 ACORN：Filtered ANN / Hybrid Search

论文：

**ACORN: Performant and Predicate-Agnostic Search Over Vector Embeddings and Structured Data**

发表：

- PACMMOD / SIGMOD 2024；
- 基于 HNSW；
- 官方实现基于 FAISS C++。

它已经实现：

```text
HNSW-based hybrid search
predicate subgraph traversal
arbitrary / high-cardinality predicates
Mγ neighbor expansion
Mβ compression
two-hop recovery
ACORN-1
ACORN-γ
```

但没有实现：

```text
Zero-Knowledge Proof
cryptographic verification
proof of correct execution
private predicate proof
Merkleized index commitment
```

论文：

https://arxiv.org/abs/2403.04871

代码：

https://github.com/guestrin-lab/ACORN

---

### 4.2 ANNProof：可验证 HNSW，但不是零知识路线

论文：

**ANNProof: Building a verifiable and efficient outsourced approximate nearest neighbor search system on blockchain**

发表于 Future Generation Computer Systems，2024。

ANNProof 主要使用：

```text
Merkle HNSW node tree
Merkle vector identifier tree
authenticated data structures
blockchain / smart contract
```

它解决的是：

> 如何让用户验证 outsourced HNSW ANN 查询所使用的数据和索引没有被篡改。

它的重要意义在于：

$$
\boxed{
HNSW + authenticated\ verification
}
$$

已经被证明是可以做的。

但它并不是本文设想的 ZK-ACORN，因为它没有提供类似 zk-SNARK / zk-STARK 的零知识执行证明，也不以隐藏私有索引为核心目标。

论文：

https://doi.org/10.1016/j.future.2024.03.002

---

### 4.3 V3DB：已经实现 ANN 的零知识执行证明，但索引是 IVF-PQ

论文：

**V3DB: Audit-on-Demand Zero-Knowledge Proofs for Verifiable Vector Search over Committed Snapshots**

VLDB 2026。

V3DB 做了一件非常关键的事情：

> 对一个 committed vector database snapshot，证明服务器返回的 ANN 结果确实是按照公开定义的 IVF-PQ 查询流程执行得到的。

其核心性质是：

$$
\boxed{
\text{证明 ANN semantics 被正确执行}
}
$$

而不是：

$$
\boxed{
\text{证明 ANN 结果等于 Exact KNN}
}
$$

V3DB 已经实现：

```text
committed dataset snapshot
zero-knowledge ANN proof
IVF-PQ execution semantics
private index / corpus hiding
succinct verification
Plonky2 prototype
```

但它没有解决：

```text
HNSW traversal
ACORN traversal
arbitrary predicate filtering
Mβ two-hop recovery
filtered HNSW
```

论文：

https://arxiv.org/abs/2603.03065

代码：

https://github.com/Relaxed-System-Lab/V3DB

---

### 4.4 Atlas：已经实现 HNSW 的零知识搜索证明

论文：

**Atlas: Efficient Verifiable Semantic Search**

arXiv 2026-09。

Atlas 是目前与 ZK-ACORN 最接近的工作。

Atlas 的目标是：

> 服务端在 committed HNSW index 上执行查询，并生成零知识证明，证明返回结果确实来自指定的 HNSW 搜索过程，同时不向 verifier 暴露索引本身。

Atlas 已经解决：

```text
HNSW data-dependent traversal
committed index
proof of HNSW execution
zero-knowledge graph search
large-scale HNSW proving
```

其关键技术包括：

1. 把数据库相关的大量工作移动到 preprocessing；
2. 把 HNSW 重构为 fixed-size-state procedure；
3. 使用 timestep-tagged batching 合并 traversal proof。

根据作者摘要：

- SIFT1M 单查询 proving time 小于 1 秒；
- 100M 向量规模约 2 秒。

因此：

$$
\boxed{
\text{“HNSW 能不能做 ZK execution proof”已经不再是空白问题}
}
$$

论文：

https://arxiv.org/abs/2609.11841

---

### 4.5 RACORN-1：ACORN 系列仍在继续解决低 selectivity 问题

论文：

**RACORN-1: Adaptive Recall-Preserving Speedup for Low-Selectivity Filtered Vector Search**

arXiv 2026。

它针对 ACORN-1 在低 selectivity 下 predicate subgraph 连接性下降的问题，引入：

```text
Adaptive Search Fallback
filter-failing transient bridge nodes
Adaptive Exact Fallback
```

这说明 filtered HNSW / ACORN 本身仍然是一个活跃研究方向。

对于 ZK-ACORN 来说，RACORN-1 也提示了一个未来问题：

> 如果查询允许 filter-failing node 作为 transient bridge，那么 ZK proof 需要证明“哪些节点只是导航桥，哪些节点是真正合法结果”。

论文：

https://arxiv.org/abs/2607.00768

代码：

https://github.com/naver/racorn

---

## 5. 当前研究版图总结

| 工作 | ANN 索引 | Predicate / Filter | 可验证性 | Zero-Knowledge | 证明搜索执行 |
|---|---|---|---|---|---|
| ACORN | HNSW | 是，核心能力 | 否 | 否 | 否 |
| ANNProof | HNSW | 非核心 | 是 | 否 | 部分验证 |
| V3DB | IVF-PQ | 非 ACORN 式 | 是 | 是 | 是 |
| Atlas | HNSW | 普通 HNSW | 是 | 是 | 是 |
| RACORN-1 | HNSW | 是 | 否 | 否 | 否 |
| **目标：ZK-ACORN** | **ACORN / HNSW** | **是** | **是** | **是** | **是** |

因此，目前已经有人分别解决了：

$$
\text{ACORN filtered ANN}
$$

和：

$$
\text{HNSW zero-knowledge execution proof}
$$

但在本次检索范围内，没有发现一篇公开论文完整实现：

$$
\boxed{
\text{ACORN}
+
\text{arbitrary predicate}
+
\text{zero-knowledge execution proof}
}
$$

这里应严谨表述为“在截至 2026-09-22 的本次公开文献检索中未发现”，而不是声称绝对不存在任何未公开或遗漏的工作。

---

## 6. 推荐的研究题目

### 题目 1

**ZK-ACORN: Zero-Knowledge Verifiable Predicate-Aware Approximate Nearest Neighbor Search**

中文：

**面向谓词感知近似最近邻搜索的零知识可验证 ACORN**

### 题目 2

**Verifiable Filtered Vector Search over Committed HNSW Indexes**

中文：

**基于承诺 HNSW 索引的可验证过滤向量检索**

### 题目 3

如果更强调隐私：

**Privacy-Preserving and Verifiable Predicate-Aware Vector Search**

中文：

**隐私保护与可验证的谓词感知向量检索**

---

## 7. ZK-ACORN 应该证明什么

### 7.1 数据集成员关系

服务器返回节点 `e_i`。

首先证明：

$$
e_i\in D
$$

即：

> 返回的数据确实属于已经承诺的数据库。

可以采用：

```text
Merkle Tree
Vector Commitment
Polynomial Commitment
```

例如：

$$
h_i=H(x_i,a_i,id_i)
$$

再构造：

$$
Root_D
$$

作为数据库 commitment。

---

### 7.2 索引成员关系

不仅数据集要 committed，ACORN graph 也应被 committed。

对于节点 `v`：

$$
N(v)
$$

表示它的邻居表。

可以构造：

$$
Root_G
$$

用于承诺：

```text
node id
vector / vector commitment
metadata commitment
level
neighbor lists
```

从而证明：

> 查询中使用的边确实属于查询前已经确定好的 ACORN index。

---

### 7.3 Predicate Satisfaction

对于合法搜索节点或最终返回结果，需要证明：

$$
p_q(a_i)=True
$$

例如：

```text
price < 500
year >= 2024
```

可以在 ZK circuit 中证明 range / equality / Boolean combination。

如果 metadata 是私有的，可以只证明条件成立，而不暴露：

```text
真实 price
真实 year
其他字段
```

---

## 8. 距离计算证明

ACORN 最终依赖向量距离。

例如 L2：

$$
d^2(x_q,x_i)
=
\sum_{j=1}^{d}
(x_{qj}-x_{ij})^2
$$

可以证明：

$$
z_i=d^2(x_q,x_i)
$$

然后验证 candidate ordering。

但主流 ZK proof system 通常工作在有限域，而 embedding 往往是 float。

因此必须解决：

```text
float -> fixed point
float -> integer quantization
overflow bound
distance approximation error
```

这会是实际工程中非常重要的一块。

---

## 9. 最重要的区别：证明 ACORN 正确执行，不等于证明 Exact Top-K

ACORN 是 ANN。

ANN 本来就不保证：

$$
R=ExactKNN(X_p)
$$

因此 ZK-ACORN 合理的证明目标应该是：

$$
\boxed{
R=ACORN(q,D,G,\theta)
}
$$

其中：

- `D`：committed dataset；
- `G`：committed ACORN index；
- `θ`：ACORN 参数；
- `q`：查询。

也就是说：

> 返回结果与“按照规定 ACORN 算法正确执行得到的结果”一致。

而不是证明整个合法集合上的绝对 Exact Top-K。

这和 V3DB、Atlas 所采用的 execution correctness 思路一致。

---

## 10. ZK-ACORN 最有研究价值的新增证明

Atlas 已经解决普通 HNSW traversal 的 ZK proof。

因此不能把：

> “证明 HNSW 被正确执行”

作为主要创新。

真正新增的问题应集中在以下部分。

### 10.1 Predicate-Aware GET_NEIGHBORS

普通 HNSW：

```text
GET_NEIGHBORS(v)
```

ACORN：

```text
GET_NEIGHBORS(v, predicate)
```

需要证明：

1. 读取了正确的 committed neighbor list；
2. 执行了正确的 predicate；
3. 没有漏掉合法邻居；
4. 没有加入非法邻居；
5. 截断逻辑符合 ACORN 规则。

---

### 10.2 ACORN two-hop recovery

ACORN compression 会删掉一部分 direct edge。

例如：

```text
v -> d
```

可能被删除。

但保证存在某个保留节点：

```text
v -> c
c -> d
```

查询时通过 neighbor expansion 重新发现 `d`。

因此 ZKP 要证明：

$$
c\in N(v)
$$

并且：

$$
d\in N(c)
$$

同时证明 `d` 确实是在 ACORN 规定的 expansion 阶段被恢复出来的。

---

### 10.3 Filter-before-distance

ACORN 的性能优势之一来自：

```text
先 predicate check
↓
只对合法候选做昂贵距离计算
```

因此 proof system 可以尝试利用这个特性：

$$
\boxed{
\text{非法节点不进入高维 distance circuit}
}
$$

这可能成为降低 proving cost 的重要优化点。

---

## 11. 一个完整的 ZK-ACORN 架构

```text
                     数据库
                       |
             embeddings + metadata
                       |
                 建立 ACORN
                       |
           -----------------------
           |                     |
     Dataset Commitment     Graph Commitment
           |                     |
        Root_D                 Root_G
           \                     /
            ------ public -------
                     |
             query q=(xq,pq)
                     |
                     v
                ACORN Search
                     |
              execution trace
                     |
                 ZK Prover
                     |
          -----------------------
          |                     |
       Top-K R                Proof π
          |                     |
          --------> Verifier <--
```

Verifier 检查：

$$
Verify(Root_D,Root_G,q,R,\pi)=True
$$

成功意味着：

> 对 committed dataset 和 committed ACORN index，服务器按照指定的 ACORN 查询语义正确执行，并返回了结果 R。

---

## 12. Public Input 与 Private Witness 如何设计

### Public Input

```text
dataset root
graph root
query commitment / query
predicate description / predicate commitment
ACORN parameters
Top-K result IDs
```

例如：

```text
M
gamma
M_beta
efSearch
K
metric type
```

### Private Witness

```text
embedding vectors
metadata
visited nodes
neighbor lists
Merkle paths
candidate queue states
distance values
two-hop expansion nodes
intermediate search states
```

如果希望进一步保护 query：

```text
query vector
predicate parameters
```

也可以部分进入 private witness。

---

## 13. 第一个可落地版本：不要直接证明完整 ACORN

如果一开始就证明：

```text
完整 HNSW hierarchy
+
ACORN candidate queue
+
Mγ
+
Mβ
+
two-hop
+
predicate
+
distance
+
termination
```

工作量会非常大。

推荐先做一个 Minimal Viable ZK-ACORN。

---

## 14. MVP 版本建议

第一版只证明：

$$
\boxed{
\text{Returned Result Membership}
+
\text{Predicate Satisfaction}
+
\text{Distance Correctness}
}
$$

对于返回的每个结果 `r_i`：

### Proof 1

$$
r_i\in D
$$

通过 Merkle membership。

### Proof 2

$$
p_q(a_i)=True
$$

通过 ZK predicate circuit。

### Proof 3

$$
d_i=dist(x_q,x_i)
$$

通过 ZK distance circuit。

### Proof 4

$$
d_1\le d_2\le\cdots\le d_K
$$

证明返回结果内部排序正确。

这个版本还不能证明：

> 服务器没有漏掉某个应该访问的 ACORN 节点。

但已经可以形成一个独立原型。

---

## 15. 第二阶段：证明单步 ACORN Neighbor Lookup

下一步实现：

$$
\boxed{
ZK\_GET\_NEIGHBORS
}
$$

输入：

```text
node v
level l
predicate p
committed graph root
```

证明：

1. `N(v)` 属于 committed graph；
2. 对直接邻居正确做 predicate check；
3. 如果有 compression，则执行规定的 two-hop expansion；
4. 恢复出的邻居属于对应节点的真实 neighbor list；
5. 输出合法 neighbor set。

这是一个非常合适的单独 benchmark。

---

## 16. 第三阶段：优先证明 ACORN-1

推荐首先做：

$$
\boxed{
ZK\text{-}ACORN\text{-}1
}
$$

原因：

ACORN-1 construction 更接近普通 HNSW。

它主要把额外复杂度放在 search-time neighbor expansion：

```text
1-hop
+
2-hop
↓
predicate filtering
↓
search
```

因此可以更直接复用 Atlas 的：

```text
HNSW commitment
HNSW traversal proof
fixed-state search
```

只扩展：

```text
predicate proof
two-hop proof
```

研究风险明显更低。

---

## 17. 第四阶段：完整 ACORN-γ

等 ACORN-1 proof 跑通以后，再扩展到 ACORN-γ。

需要额外证明：

```text
expanded neighbor list commitment
Mγ structure
Mβ compression
compressed-edge recoverability
```

这一阶段更有研究价值，但实现复杂度也明显更高。

---

## 18. 是否要证明 ACORN construction

建议第一篇工作不要一开始证明 construction。

可以先假设：

> ACORN index 在离线阶段已经正确构建，并对其最终状态进行 commitment。

然后查询阶段证明：

$$
\boxed{
\text{search correctness on committed ACORN index}
}
$$

等 query proof 完成以后，再考虑：

$$
\boxed{
\text{verifiable ACORN index construction}
}
$$

即证明：

```text
random level assignment 合法
neighbor candidate collection 合法
compression 合法
final graph root 由原 dataset 正确生成
```

这是一个更大的后续课题。

---

## 19. 可以考虑的技术路线

具体 proving framework 不建议一开始锁死，但可以调研：
```text
Plonky2
Halo2
Nova / folding schemes
Spartan
STARK-style systems
custom lookup arguments
```

V3DB 的开源实现基于 Rust / Plonky2，因此非常适合作为“如何把 ANN operation 放进 ZK”的工程参考。

Atlas 更适合作为：

```text
graph search execution proof
data-dependent control flow
HNSW state transformation
```

的算法参考。

ACORN 官方 C++ 代码则作为：

```text
filtered HNSW ground truth
```

---

## 20. ZK 电路中的几个主要难点

### 20.1 Random Access

HNSW / ACORN 本质上大量做：

```text
根据 node id 随机读取 neighbor list
```

而 ZK circuit 不擅长 RAM-like random access。

需要考虑：

```text
Merkle authentication
lookup argument
memory consistency argument
preprocessing
```

---

### 20.2 Priority Queue

HNSW / ACORN 使用动态 candidate queue。

需要证明：

```text
当前弹出的节点确实是 candidate 中距离最近的
结果队列淘汰的确实是当前最差结果
```

如果直接在 circuit 中做完整 heap，非常昂贵。

可以研究：

```text
sorted multiset
permutation argument
boundary condition
batched comparison
```

---

### 20.3 Floating Point

Embedding 常见：

```text
float32
float16
```

ZK circuit 中最好转换成：

```text
integer
fixed point
quantized vector
```

需要分析：

$$
\text{quantization error}
$$

是否改变 ACORN traversal。

这可以形成一个实验问题：

> ZK-friendly quantization 对 ACORN recall / search path 的影响是多少？

---

### 20.4 Arbitrary Predicate

ACORN 的卖点是 predicate-agnostic。

但 ZK circuit 不容易真正支持：

```text
任意 Python / SQL predicate
```

因此第一版最好限定 predicate grammar，例如：

```text
equality
range
AND
OR
IN
```

形成一个：

```text
ZK Predicate DSL
```

之后再扩展：

```text
contains
regex
```

---

## 21. 一个合理的研究问题定义

可以把核心问题写成：

> 给定一个 committed dataset 和 committed ACORN index，如何让一个不可信服务器在执行 predicate-aware ANN search 后，生成一个 succinct zero-knowledge proof，使 verifier 能够确认服务器严格遵守指定的 ACORN 查询语义，同时不需要重新执行搜索，也不需要获得私有索引和 metadata？

形式化：

公开：

$$
Root_D,\ Root_G,\ q,\ \theta,\ R
$$

私有 witness：

$$
W
$$

证明关系：

$$
\mathcal{R}(Root_D,Root_G,q,\theta,R;W)=1
$$

当且仅当：

$$
R=ACORN(D,G,q,\theta)
$$

并且：

$$
Commit(D)=Root_D
$$

$$
Commit(G)=Root_G
$$

---

## 22. 可以提出的主要贡献

### Contribution 1

首个面向 predicate-aware graph ANN 的 ZK execution proof framework。

注意论文中需要谨慎写成：

> “据我们的文献检索……”

不能在没有系统综述的情况下直接声称 first。

### Contribution 2

ZK-friendly ACORN neighbor expansion。

把：

```text
predicate check
+
1-hop
+
2-hop recovery
```

转换成高效证明关系。

### Contribution 3

Proof-aware filtering optimization。

利用 ACORN：

```text
filter before distance
```

的特点，减少高维 distance proof 数量。

### Contribution 4

Committed metadata + private predicate verification。

证明：

$$
p(a_i)=True
$$

但不公开：

```text
a_i
```

的完整内容。

### Contribution 5

完整性能分析。

比较：

```text
plaintext ACORN
ZK-ACORN
Atlas-style HNSW proof
V3DB-style vector proof
```

指标：

```text
proving time
verification time
proof size
memory
QPS
Recall@K
index commitment size
preprocessing time
```

---

## 23. 最推荐的推进路线

### Phase 0：复现基础系统

目标：

```text
跑通 HNSW
跑通 ACORN-1
跑通 ACORN-γ
理解官方 C++ 实现
```

数据集先用：

```text
SIFT1M
```

不要一开始做 LAION 25M。

### Phase 1：学习 ZK ANN

重点阅读 / 复现：

```text
V3DB
Atlas
```

先理解：

```text
commitment
Merkle proof
execution trace
ZK circuit
prover / verifier
```

### Phase 2：Predicate Proof Prototype

实现：

```text
metadata commitment
range proof
equality proof
AND / OR
```

输入：

```text
private metadata
public predicate
```

输出：

```text
proof that predicate(metadata) == true
```

### Phase 3：Vector Distance Proof

实现：

```text
fixed-point embedding
L2 distance
distance comparison
Top-K internal ordering
```

评估不同维度：

```text
128
256
512
768
```

### Phase 4：ZK GET_NEIGHBORS

这是第一项真正 ACORN-specific 的工作。

证明：

```text
neighbor list membership
two-hop expansion
predicate filtering
truncation
```

### Phase 5：ZK-ACORN-1 Search

把：

```text
Atlas-style HNSW traversal
+
ZK GET_NEIGHBORS
```

组合。

得到：

$$
\boxed{
ZK\text{-}ACORN\text{-}1
}
$$

### Phase 6：ACORN-γ + Mβ Compression

再实现：

```text
compressed ACORN graph commitment
Mβ direct region
two-hop recoverable region
```

并评估 compression 对：

```text
proof size
proving time
memory
```

的影响。

### Phase 7：Private Metadata / Private Predicate

如果前面已经成功，再增加隐私。

例如只公开：

```text
proof that condition is true
```

进一步再考虑：

```text
predicate itself is committed / partially private
```

---

## 24. 推荐的最小论文范围

如果项目时间有限，建议把论文范围控制在：

$$
\boxed{
\text{ZK-ACORN-1 Search over Committed HNSW with Private Metadata Filtering}
}
$$

也就是：

```text
不证明 ACORN construction
不先做 ACORN-γ compression
不做 regex
不做 fully private query
```

只做：

```text
committed HNSW
ACORN-1 two-hop expansion
equality / range predicate
distance verification
search execution proof
```

这个范围已经足够有内容。

---

## 25. 不建议一开始做的内容

以下内容很容易让项目失控：

```text
完整 SQL
regex proof
全隐私 query
homomorphic encrypted vector distance
动态 index update
ACORN construction proof
ACORN-γ + RACORN + Atlas 全部一次实现
亿级数据集
GPU prover optimization
```

应该先把：

$$
\boxed{
\text{filtered graph ANN execution proof}
}
$$

做出来。

---

## 26. 实验设计建议

至少包含四组实验。

### 26.1 Baseline Search Quality

比较：

```text
HNSW
ACORN-1
ACORN-γ
ZK-ACORN 的 plaintext execution
```

指标：

```text
Recall@10
QPS
distance computations
```

### 26.2 Proof Cost

变量：

```text
vector dimension d
K
efSearch
predicate selectivity s
```

指标：

```text
proving time
verification time
proof size
peak memory
```

### 26.3 Selectivity

测试：

```text
s = 50%
20%
10%
5%
1%
0.1%
```

核心问题：

> selectivity 降低时，ACORN search cost 和 ZK proving cost 分别如何变化？

### 26.4 Predicate Complexity

分别测试：

```text
equality
single range
multiple ranges
AND
OR
```

观察：

```text
constraints
proving time
proof size
```

---

## 27. 一个非常值得验证的核心 Hypothesis

ACORN 的性能优化来自：

```text
predicate filter
↓
减少 distance computations
```

而 ZK 中，高维距离证明同样很贵。

因此 ACORN 可能出现“双重收益”：

$$
\boxed{
\text{predicate filtering}
\Rightarrow
\text{少做 plaintext distance}
+
\text{少证明 ZK distance}
}
$$

也就是说：

> ACORN 也许不仅比 post-filter HNSW 查询更快，还可能比“ZK post-filter HNSW”显著更容易证明。

可以定义一个简化的 proof cost 模型：

$$
C_{zk}
=
C_{predicate}
+
N_{dist}\cdot C_{distance}
+
C_{graph}
$$

如果 ACORN 能大幅降低：

$$
N_{dist}
$$

那么即使增加 predicate proof，整体 ZK cost 仍可能下降。

这非常适合作为论文的核心实验假设。

---

## 28. 另一个值得研究的问题：Mβ 与 proof cost

`Mβ` compression 原本为了减少内存。

但对于 ZK 系统，可能出现新的 trade-off：

```text
更少 direct edges
↓
更小 graph commitment / memory
```

但同时：

```text
更多 two-hop proof
```

因此可能存在：

$$
\boxed{
M_\beta
\leftrightarrow
\text{proof cost}
}
$$

可以研究：

> 什么样的 `Mβ` 最适合 ZK-ACORN，而不是 plaintext ACORN？

这可能和 ACORN 原论文中偏向查询性能的参数选择不同。

---

## 29. 新的优化目标

普通 ACORN 优化：

$$
\text{search latency}
+
\text{recall}
+
\text{index size}
$$

ZK-ACORN 还必须考虑：

$$
\text{proving time}
+
\text{verification time}
+
\text{proof size}
$$

所以：

> 最优的 ACORN 参数不一定等于最优的 ZK-ACORN 参数。

这本身也可以成为论文中的系统优化问题。

---

## 30. 当前最推荐的研究路线总结

```text
ACORN
↓
复现 ACORN-1
↓
阅读 Atlas
↓
阅读并跑通 V3DB
↓
实现 metadata commitment
↓
实现 predicate ZK
↓
实现 vector distance ZK
↓
实现 ZK GET_NEIGHBORS
↓
实现 ZK-ACORN-1 traversal
↓
评估 selectivity 对 proof cost 的影响
↓
再考虑 ACORN-γ / Mβ
```

不要反过来一开始就设计一个“万能 ZK vector database”。

---

## 31. 最终研究定位

最合适的定位不是：

> “给 ACORN 加一个零知识证明。”

而应该是：

$$
\boxed{
\text{Designing Efficient Zero-Knowledge Proofs for Predicate-Aware Graph ANN Search}
}
$$

ACORN 是其中最合适的算法载体。

研究问题可以表述为：

1. HNSW 的 ZK proof 已经由 Atlas 推进；
2. ACORN 提供 predicate-aware graph traversal；
3. 新问题是如何证明：

```text
filtered neighbor expansion
predicate correctness
two-hop recovery
distance computation
search-state transition
```

并让证明成本保持可接受。

---

## 32. 一句话概括 Idea

$$
\boxed{
\text{ZK-ACORN 的目标，是让不可信服务器能够证明：}
}
$$

$$
\boxed{
\text{它在一个已承诺的 ACORN 图上，严格按照指定 predicate 执行了 ANN 搜索，}
}
$$

$$
\boxed{
\text{且无需向验证者公开不必要的向量、metadata 和索引内部状态。}
}
$$

---

## 33. 参考文献与项目

### ACORN

Liana Patel, Peter Kraft, Carlos Guestrin, Matei Zaharia.  
ACORN: Performant and Predicate-Agnostic Search Over Vector Embeddings and Structured Data.  
PACMMOD / SIGMOD 2024.

Paper:

https://arxiv.org/abs/2403.04871

Code:

https://github.com/guestrin-lab/ACORN

### ANNProof

Lingling Lu et al.  
ANNProof: Building a verifiable and efficient outsourced approximate nearest neighbor search system on blockchain.  
Future Generation Computer Systems, 2024.

https://doi.org/10.1016/j.future.2024.03.002

### V3DB

Zipeng Qiu, Wenjie Qu, Jiaheng Zhang, Binhang Yuan.  
V3DB: Audit-on-Demand Zero-Knowledge Proofs for Verifiable Vector Search over Committed Snapshots.  
VLDB 2026.

Paper:

https://arxiv.org/abs/2603.03065

Code:

https://github.com/Relaxed-System-Lab/V3DB

### Atlas

Nikolay Avramov, Hidde Lycklama, Alexander Viand, Anwar Hithnawi.  
Atlas: Efficient Verifiable Semantic Search.  
arXiv 2026.

https://arxiv.org/abs/2609.11841

### RACORN-1

Yoonseok Kim, Gyusik Choe.  
RACORN-1: Adaptive Recall-Preserving Speedup for Low-Selectivity Filtered Vector Search.  
arXiv 2026.

Paper:

https://arxiv.org/abs/2607.00768

Code:

https://github.com/naver/racorn

---

## 34. 当前结论

截至本次检索：

```text
ACORN 已实现 predicate-aware ANN
Atlas 已实现 HNSW ZK execution proof
V3DB 已实现 IVF-PQ ZK ANN proof
ANNProof 已实现 HNSW authenticated verification
RACORN-1 已进一步改进 filtered HNSW
```

仍值得进一步研究的交叉点是：

$$
\boxed{
\text{Predicate-Aware HNSW / ACORN}
+
\text{Zero-Knowledge Execution Proof}
}
$$

第一阶段最合理、最可控的目标是：

$$
\boxed{
\text{ZK-ACORN-1}
}
$$

即先证明：

```text
committed HNSW
+
ACORN-1 two-hop expansion
+
predicate filtering
+
distance computation
+
search execution
```

在这个基础上，再扩展到：

```text
ACORN-γ
Mβ compression
private predicate
dynamic index
RACORN-style fallback
```

这样研究路线更稳，也更容易逐步形成可验证的研究成果。