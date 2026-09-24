---
layout: default
title: "ACORN 与 HNSW 学习笔记"
date: 2026-09-24 15:00:00 +0800
description: "系统梳理 HNSW 原理、ACORN 构建与搜索、关键参数、复杂度和容易混淆的问题。"
categories: [向量检索, HNSW, ACORN]
---

> 论文：**ACORN: Performant and Predicate-Agnostic Search Over Vector Embeddings and Structured Data**  
> arXiv:2403.04871v1  
> 本文档重点：HNSW 实现原理、ACORN 的构建与搜索、关键公式、复杂度、参数含义，以及学习过程中容易混淆的问题。

---

# 1. 论文要解决的问题

现代检索系统经常同时包含两类数据：

1. **向量数据**：文本、图片、视频等经过模型编码后得到的 embedding。
2. **结构化属性**：价格、时间、类别、关键词、标签等。

一条数据可以写成：

$$
e_i=(x_i,a_i)
$$

其中：

- $x_i\in\mathbb{R}^d$：向量表示；
- $a_i$：结构化属性。

查询写成：

$$
q=(x_q,p_q)
$$

其中：

- $x_q$：查询向量；
- $p_q$：属性谓词，例如 `price < 500`、`year between 2020 and 2024`、`keyword contains AI`。

定义：

$$
X_p=\{x_i\mid p(a_i)=True\}
$$

表示**整个数据库中所有满足谓词 $p$ 的向量集合**。

Hybrid Search 的目标是：

$$
\text{在 }X_p\text{ 中找到距离 }x_q\text{ 最近的 }K\text{ 个向量}
$$

即：

$$
KNN(x_q,X_p)
$$

注意：$X_p$ 不是“先做 KNN 后再筛出来的集合”，而是从整个数据库的定义域中，由属性条件决定的合法向量子集。

---

# 2. 三个重要符号：$n$、$s$、$K$

论文中：

$$
n=\text{数据库中的总向量数量}
$$

$$
s=\frac{|X_p|}{n}
$$

$s$ 称为 predicate selectivity，即满足谓词的数据比例。

因此：

$$
|X_p|=sn
$$

例如：

- 数据库有 $n=1,000,000$ 个向量；
- 有 $80,000$ 个满足属性条件；

则：

$$
s=\frac{80,000}{1,000,000}=0.08
$$

而：

$$
K=\text{最终需要返回的最近邻数量}
$$

例如 Top-10 时：

$$
K=10
$$

---

# 3. 为什么需要 ANN

最直接的 KNN 方法，是把查询向量 $x_q$ 和数据库所有向量都计算距离。

例如欧氏距离：

$$
d(x_q,x_i)=\sqrt{\sum_{j=1}^{d}(x_{qj}-x_{ij})^2}
$$

如果数据库中有 $n$ 个向量，则一次查询大约需要检查 $n$ 个点：

$$
O(n)
$$

这称为 Exact KNN。

当 $n$ 达到百万、千万甚至亿级时，逐个扫描太慢，因此实际系统常使用：

$$
ANN=\text{Approximate Nearest Neighbor}
$$

HNSW 就是一种非常经典的图式 ANN 索引。

---

# 4. HNSW 的核心思想

HNSW 全称：

**Hierarchical Navigable Small World**

可以拆成：

- Hierarchical：多层；
- Navigable：可以通过图进行高效导航；
- Small World：少量长距离连接可以快速跨越大范围区域。

HNSW 的整体思路可以概括为：

$$
\boxed{
\text{多层邻近图}
+
\text{Greedy Routing}
+
\text{候选集搜索}
+
\text{Neighbor Pruning}
}
$$

---

# 5. HNSW 的图结构

HNSW 把每个向量看作一个图节点。

记图为：

$$
G=(V,E)
$$

其中：

- $V$：所有向量节点；
- $E$：节点之间的邻近边。

距离比较近的向量倾向于互相连边。

一个简化示意：

```text
A ----- B
 \     /
   C
   |
   D ----- E
   |
F--G
   |
   H ----- I
```

如果查询点 $Q$ 靠近 $I$，图搜索不需要扫描所有节点，而是沿着图逐步逼近目标区域。

---

# 6. 为什么 HNSW 要分层

如果只有一层图，数据规模非常大时，从远处逐步走到目标区域仍然可能需要很多跳。

所以 HNSW 增加层级结构：

```text
Level 3:          A ---------------- G
                  |                  |
Level 2:          A ----- C -------- G ------ I
                  |       |          |        |
Level 1:          A -- B--C -- D ----G -- H---I
                  |   |   |    |     |    |   |
Level 0:          A-B-C-D-E-F-G-H-I-...
```

可以类比：

```text
高层：高速公路
中层：城市主干道
底层：普通道路
```

搜索时：

1. 先在高层快速靠近目标区域；
2. 再逐层下降；
3. 在 Level 0 做更精细的局部搜索。

---

# 7. HNSW 中节点层高如何生成

**每个新插入节点 $v$ 会随机生成一个最高层：**
$$
l(v)
$$

如果：

$$
l(v)=2
$$

则节点存在于：

```text
Level 2
Level 1
Level 0
```

但不存在于：

```text
Level 3
Level 4
...
```

HNSW 使用指数衰减的层级采样机制，高层节点越来越少。

论文中使用：

$$
m_L=\frac{1}{\ln M}
$$

DD作为层级分布相关参数。

直觉是：

```text
Level 0：几乎所有节点
Level 1：少一些
Level 2：更少
Level 3：非常少
```

因此高层天然形成稀疏的长距离导航网络。

---

# 8. HNSW 的几个核心参数

## 8.1 $M$

$M$ 控制每个节点保留的邻居数量级。

粗略理解：

$$
M=\text{节点的主要连接度}
$$

$M$ 较大：

- 图更密；
- 内存增加；
- 建图更慢；
- 搜索时检查更多邻居；
- Recall 通常更高。

$M$ 较小：

- 图更稀疏；
- 内存更省；
- 但可能更容易错过有效路径。

论文中还指出，Level 0 的 degree bound 通常可以提高到：

$$
2M
$$

---

## 8.2 $ef_c$

$ef_c$ 是 construction 阶段的搜索宽度参数。

它控制插入新节点时，底层搜索能够维护和探索多大的候选范围。

不要把它理解成：

```text
最终节点一定有 efc 条边
```

也不要简单理解成：

```text
先固定得到 efc 个候选，再从中做所有后续操作
```

更准确地说，它控制构建阶段搜索的广度，而最终保留多少邻居，还要看 $M$、pruning 以及 ACORN 中的 $M\gamma$ 等机制。

> 注：原 Typora 文档此处引用了一张本地示意图，博客版本暂未包含该本地图片。

例如上述图片，将新节点v插入，假设随机生成的最高层为2，且当efc=2时第二层会将C点排除，这样就忽略了最佳点E，所以efc的代表了构建阶段搜索的广度。

---

## 8.3 $ef_s$

$ef_s$ 是 search 阶段的动态候选列表大小。

一般：

$$
ef_s\uparrow
\Rightarrow
Recall\uparrow
$$

但同时：

$$
ef_s\uparrow
\Rightarrow
\text{距离计算次数增加}
\Rightarrow
QPS\downarrow
$$

---

## 8.4 $K$

$K$ 是最终返回的最近邻个数。

例如：

$$
K=10
$$

表示最终返回 Top-10。

要区分：

$$
K=\text{最终输出数量}
$$

$$
ef_s=\text{为了找到这些结果愿意探索多大的候选空间}
$$

通常：

$$
ef_s\ge K
$$

---

# 9. HNSW 构建过程

假设现有 HNSW 的最高层：

$$
L=4
$$

新节点 $v$ 随机生成：

$$
l(v)=2
$$

那么插入过程分成两个阶段。

---

## 9.1 第一阶段：从 $L$ 到 $l(v)+1$

也就是：

```text
Level 4
Level 3
```

此时新节点 $v$ 不存在于这些层，因此：

- 不建边；
- 不为 $v$ 选择邻居；
- 只进行 greedy navigation；
- 找到一个更接近 $v$ 的节点，作为下一层入口。

可以写成：

$$
L\rightarrow l(v)+1:
\quad
\text{只导航}
$$

例如：

```text
Level 4:
入口 A
↓
greedy search
↓
得到 C	

Level 3:
从 C 开始
↓
greedy search
↓
得到 F
```

然后 $F$ 成为 Level 2 的入口。

---

## 9.2 第二阶段：从 $l(v)$ 到 Level 0

从：

$$
l(v)=2
$$

开始，新节点真正进入图。

因此：

```text
Level 2：搜索候选 -> pruning -> 建边
Level 1：搜索候选 -> pruning -> 建边
Level 0：搜索候选 -> pruning -> 建边
```

普通 HNSW 在每层会通过 construction search 找到一批候选，然后使用 RNG-based pruning 选择最多约 $M$ 个邻居。

---

# 10. HNSW 的 greedy navigation 会不会检查入口点距离

会。

假设某层入口点是：

$$
e
$$

查询目标是 $q$。

算法先有：

$$
d(q,e)
$$

然后检查 $e$ 的邻居。

若某个邻居 $u$ 满足：

$$
d(q,u)<d(q,e)
$$

就可能向 $u$ 移动。

例如：

$$
d(q,A)=10
$$

$$
d(q,B)=12
$$

$$
d(q,C)=7
$$

从 $A$ 开始：

```text
A 的邻居：B、C

B：12 > 10，不改善
C：7 < 10，改善
```

于是：

$$
A\rightarrow C
$$

继续检查 $C$ 的邻居。

所以高层 greedy search 的直觉是：

$$
\boxed{
不断寻找比当前节点更接近目标的邻居
}
$$

---

# 11. *HNSW 的 SEARCH-**LAYER***

在 Level 0，纯 greedy 容易陷入局部最优，所以 HNSW 会维护一组候选，而不是只保留一个当前最优节点。

可以用三个集合理解：

- `visited`：已经访问过的节点；
- `C`：待扩展候选；
- `W`：当前较好的结果集合。

核心逻辑：

```text
初始化：
C = {entry point}
W = {entry point}

循环：
1. 从 C 中取距离 query 最近的节点 c
2. 查看 c 的邻居
3. 对未访问邻居计算距离
4. 若它足够好，则加入 C 和 W
5. 如果 W 太大，则删除最远节点
6. 当最好的待扩展候选都已经不可能改善结果时停止
```

---

# 12. HNSW 为什么不能简单选最近的 $M$ 个邻居

假设：

```text
             d


        v  a b c
```

$a,b,c$ 都离 $v$ 非常近，但集中在同一个方向。

如果只选最近的三个：

```text
v -> a
v -> b
v -> c
```

图的方向多样性可能很差。

以后若目标位于 $d$ 的方向，可能缺少有效的跨区域连接。

因此 HNSW 会使用基于 RNG 思想的 pruning，避免所有邻居都挤在一个很小的局部区域。

---

# 13. HNSW 的 RNG pruning

考虑：

```text
      b
     / \
    a---v
```

假设：

$$
d(v,a)<d(v,b)
$$

同时：

$$
d(a,b)<d(v,b)
$$

HNSW 会认为：

```text
v -> a -> b
```

已经足够。

因此直接边：

$$
v-b
$$

可以被删除。

这相当于删除三角形中冗余的长边。

---

# 14. HNSW 搜索复杂度为什么写成 $O(\log n+K)$

论文在分析中把普通 HNSW 的无过滤搜索复杂度写成：

$$
O(\log n+K)
$$

直觉来源：

1. HNSW 的层数随数据规模对数增长；
2. 高层每层通过少量 greedy steps 导航；
3. 因此定位过程近似为：

$$
O(\log n)
$$

4. 最后返回 $K$ 个结果：

$$
O(K)
$$

因此：

$$
O(\log n+K)
$$

这是一种用于论文比较的理想化/期望复杂度表达，不意味着所有数据集和所有参数设置下都严格只访问 $\log n$ 个节点。

实际性能仍然受到：

- $M$；
- $ef_s$；
- 向量维度 $d$；
- 图质量；
- 数据分布；

等因素影响。

---

# 15. Hybrid Search 的两个传统基线

## 15.1 Pre-filtering

先做属性过滤：

$$
X_p=\{x_i\mid p(a_i)=True\}
$$

再对 $X_p$ 做距离搜索。

如果直接扫描 $X_p$：

$$
|X_p|=sn
$$

复杂度约：

$$
O(sn+K)
$$

优点：

- 结果准确；
- predicate 很严格时可能很划算。

缺点：

- 当 $sn$ 很大时，需要处理大量合法向量。

---

## 15.2 Post-filtering

先在整个向量集合 $X$ 上做 ANN：

```text
HNSW search
↓
得到近邻候选
↓
predicate filter
```

问题：

查询向量附近的节点不一定满足 predicate。

如果 selectivity 很低，就要扩大 ANN 搜索范围。

论文给出的直观分析：

- 正相关、合法结果就在 query 附近时：

$$
O(\log n+K)
$$

- 合法节点近似均匀分布时：

$$
O\left(\log n+\frac{K}{s}\right)
$$

- 最坏情况下可能接近：

$$
O(n)
$$

---

# 16. Query Correlation

论文非常重视 query correlation。

假设 query 是“狗”的 embedding。

如果 predicate 是：

```text
category = dog
```

满足 predicate 的节点本来就靠近 query。

这是 positive correlation。

如果 predicate 是：

```text
category = airplane
```

合法节点可能距离 query 很远。

这是 negative correlation。

Post-filtering 在 negative correlation 下会非常难受，因为它会先大量搜索 query 附近的非法节点。

---

# 17. Oracle Partition Index

假设提前知道查询 predicate $p$。

我们直接拿：

$$
X_p
$$

单独建立一个 HNSW。

因为：

$$
|X_p|=sn
$$

则这个专属 HNSW 的搜索复杂度可以写成：

$$
O(\log(sn)+K)
$$

这就是论文中的：

**Oracle Partition Index**

它很理想，但不现实。

因为 predicate 可能非常多，甚至查询前根本不知道。

不可能为每一种：

```text
price < 500
year > 2020
keyword contains AI
regex(...)
```

都单独建立 HNSW。

---

# 18. ACORN 的核心目标

ACORN 想做到：

$$
\boxed{
\text{不真的为每个 predicate 建 HNSW，
却让一次查询表现得像是在 }HNSW(X_p)\text{ 上搜索}
}
$$

核心概念：

$$
\boxed{\text{Predicate Subgraph}}
$$

即：

$$
G(X_p)
$$

由所有满足 predicate 的节点诱导出的子图。

ACORN 试图让任意 predicate subgraph 都尽量具备类似 HNSW 的：

- hierarchy；
- degree；
- connectivity；
- navigability。

---

# 19. ACORN-$\gamma$ 为什么要扩大邻居数

普通 HNSW 一个节点大约有：

$$
M
$$

个邻居。

如果 predicate selectivity 是 $s$，那么过滤后合法邻居期望数量约为：

$$
Ms
$$

例如：

$$
M=32,\quad s=0.1
$$

则：

$$
Ms=3.2
$$

也就是说原来 32 个邻居，过滤后平均只剩约 3 个。

图可能变得很稀疏，甚至断开。

ACORN 的做法是把邻居候选扩展到：

$$
M\gamma
$$

过滤后合法邻居期望变成：

$$
M\gamma s
$$

如果希望过滤后仍有大约 $M$ 个邻居：

$$
M\gamma s\approx M
$$

消掉 $M$：

$$
\gamma s\approx1
$$

于是：

$$
\boxed{\gamma\approx\frac{1}{s}}
$$

论文推荐的一种简单配置是：

$$
\gamma=\frac{1}{s_{\min}}
$$

其中 $s_{\min}$ 是系统希望由 ACORN 处理的最低 selectivity。

---

# 20. ACORN-$\gamma$ 的构建过程

假设：

$$
L=4
$$

新节点：

$$
l(v)=2
$$

ACORN 仍然继承 HNSW 的大框架。

```text
Level 4：greedy navigation
Level 3：greedy navigation
Level 2：开始真正寻找邻居并建边
Level 1：继续建边
Level 0：继续建边
```

所以：

$$
L\rightarrow l(v)+1
$$

仍然主要负责定位。

真正变化发生在：

$$
l(v)\rightarrow0
$$

---

# 21. ACORN 构建时到底看 $M$ 还是 $M\gamma$

这是一个非常容易混淆、也非常重要的问题。

答案分成两个层次。
## 21.1 图遍历时

当 ACORN construction search 访问一个已经存在的旧节点 $u$ 时：

即使 $u$ 实际最多存了：

$$
M\gamma
$$

个邻居，

为了控制构建代价，遍历时只使用它邻居表中的前：

$$
M
$$

个节点继续导航。

因此：

$$
\boxed{
\text{construction traversal 每访问一个旧节点，主要展开前 }M\text{ 个邻居}
}
$$

---

## 21.2 为新节点收集 candidate edges 时

ACORN 的目标是为新节点收集最多：

$$
M\gamma
$$

个 approximate nearest neighbors 作为 candidate edges。

所以：

$$
\boxed{
M=\text{单个旧节点用于导航的展开规模}
}
$$

而：

$$
\boxed{
M\gamma=\text{最终希望为新节点收集的 candidate edge 规模}
}
$$

不要把它理解成：

```text
一次直接看 Mγ 个旧邻居
```

更像：

```text
每访问一个旧节点看前 M 个
↓
不断扩大搜索范围
↓
最终累计收集最多 Mγ 个 candidate neighbors
```

---

# 22. $ef_c$ 与 $M\gamma$ 的关系

一个重要误区是：

```text
先得到 efc 个候选
↓
再从里面选 Mγ 个
```

这个理解不准确。

原因很简单：

可能出现：

$$
M\gamma>ef_c
$$

例如：

$$
M=32,\quad \gamma=12
$$

则：

$$
M\gamma=384
$$

而 $ef_c$ 可能只有 40。

因此不可能简单理解为：

```text
从 40 个候选里选 384 个
```

更准确地说：

- $ef_c$：底层 construction search 的搜索宽度参数；
- $M\gamma$：ACORN 希望获得的 candidate edge 数量级。

---

# 23. 为什么 ACORN 不能继续直接使用 HNSW 的 RNG pruning

这是 ACORN 的关键创新之一。

假设 HNSW 中：

```text
v ----- a ----- b
 \_____________/
```

因为：

$$
d(a,b)<d(v,b)
$$

HNSW 可能删除：

$$
v-b
$$

因为它认为：

$$
v\rightarrow a\rightarrow b
$$

仍然能访问到 $b$。

普通 ANN 中这通常没问题。

但 Hybrid Search 中，如果：

$$
p(v)=True
$$

$$
p(b)=True
$$

而：

$$
p(a)=False
$$

过滤后：

```text
v ✓          b ✓
     a ×
```

原本依赖的：

$$
v\rightarrow a\rightarrow b
$$

路径失效。

因此 HNSW 的 RNG pruning 可能错误删除：

$$
v-b
$$

这条对某个未来 predicate 非常重要的边。

根本原因：

$$
\boxed{
\text{HNSW pruning 只考虑几何关系，不知道未来 predicate}
}
$$

---

# 24. 为什么不能让 pruning 直接考虑 predicate

如果 predicate 是固定的，理论上可以。

但 ACORN 要支持 arbitrary / unbounded predicates。

构建时不知道未来查询会是：

```text
price < 500
year > 2020
keyword contains AI
regex(...)
```

所以构建时无法知道某个中间节点未来是否会被过滤掉。

因此 ACORN 需要：

$$
\boxed{\text{predicate-agnostic pruning}}
$$

---

# 25. $M_\beta$ compression

ACORN-$\gamma$ 建得更密，最多产生：

$$
M\gamma
$$

个 candidate edges。

如果全部存储，内存会明显增加。

因此论文引入：

$$
M_\beta
$$

其中：

$$
0\le M_\beta\le M\gamma
$$

核心思想：

$$
\boxed{
\text{近邻直接保存，较远邻居如果可以通过 two-hop 恢复，就删掉直接边}
}
$$

---

# 26. $M_\beta$ compression 的具体流程

假设：

$$
M=3
$$

$$
\gamma=2
$$

所以：

$$
M\gamma=6
$$

候选边按距离排序：

$$
[a,b,c,d,e,f]
$$

设：

$$
M_\beta=2
$$

---

## 26.1 第一步：直接保留前 $M_\beta$ 个

直接保存：

$$
a,b
$$

即：

```text
N(v) = [a,b]
```

这两个最近邻不参与激进压缩。

---

## 26.2 第二步：初始化 two-hop 覆盖集合

定义：

$$
H=\varnothing
$$

这里的 $H$ 表示：

```text
当前已经能够通过保留邻居的邻接表，在 two-hop expansion 中重新发现的节点集合
```

---

## 26.3 检查候选 c

如果：

$$
c\notin H
$$

则：

- 保留 $v-c$；
- 把 $c$ 的邻居加入 $H$。

假设：

$$
N(c)=\{d,e,x\}
$$

那么：

$$
H=\{d,e,x\}
$$

---

## 26.4 检查候选 d

因为：

$$
d\in H
$$

说明以后可以通过：

```text
v
↓
找到 c
↓
展开 N(c)
↓
重新发现 d
```

所以直接边：

$$
v-d
$$

可以 prune。

---

## 26.5 检查候选 e

同理：

$$
e\in H
$$

所以：

$$
v-e
$$

也可以 prune。

---

## 26.6 检查候选 f

如果：

$$
f\notin H
$$

则保留：

$$
v-f
$$

并把：

$$
N(f)
$$

加入 $H$。

最终可能：

```text
原候选：
a b c d e f

最终直接存：
a b c f
```

其中：

```text
d e
```

不直接保存，但可以在查询时通过 two-hop expansion 找回来。

---

# 27. ACORN compression 与 HNSW RNG pruning 的根本区别

HNSW RNG：

$$
\boxed{
\text{因为“以后可以沿中间节点走过去”，所以删边}
}
$$

ACORN compression：

$$
\boxed{
\text{因为“以后可以通过邻居表展开重新发现”，所以删边}
}
$$

差别非常重要。

---

# 28. 如果中间节点不满足 predicate，ACORN 为什么还能恢复

假设：

```text
v ✓
|
c ×
|
d ✓
```

并且直接边：

$$
v-d
$$

已经被压掉。

普通 HNSW 如果依赖：

$$
v\rightarrow c\rightarrow d
$$

那么 $c$ 被过滤后，路径就断了。

但 ACORN 不是要求搜索真的走进 $c$。

而是在访问 $v$ 时：

```text
读取 v 的邻居
↓
发现 c
↓
直接读取 c 的邻居表
↓
发现 d
↓
再执行 predicate(d)
```

也就是说：

$$
c
$$

可以只作为“邻接表中介”，不必成为 predicate subgraph 中实际被遍历的节点。

因此即使：

$$
p(c)=False
$$

仍然可以通过 two-hop expansion 找到：

$$
d
$$

这就是 ACORN compression 能保持 predicate-agnostic 的关键。

---

# 29. $M_\beta$ 的取值意味着什么

$M_\beta$ 大：

```text
更多近邻直接保存
↓
two-hop expansion 少
↓
查询更快
↓
内存更大
```

$M_\beta$ 小：

```text
更多边被压缩
↓
内存更省
↓
查询时需要更多 two-hop expansion
```

极端情况：

$$
M_\beta=M\gamma
$$

意味着几乎没有压缩，所有候选边都直接保留。

---

# 30. ACORN-$\gamma$ 搜索

ACORN 的搜索主体仍然很像 HNSW。

主要数据结构：

- $T$：visited set；
- $C$：candidate set；
- $W$：当前较好的结果。

核心区别在：

$$
GET\_NEIGHBORS(c,l,p_q)
$$

普通 HNSW：

```text
直接读取 c 的邻居
```

ACORN：

```text
读取 c 的邻居
↓
必要时做 two-hop expansion
↓
predicate filtering
↓
只把满足 predicate 的节点提供给真正的距离搜索
```

---

# 31. ACORN-$\gamma$ 不压缩时的 GET_NEIGHBORS

节点 $v$ 最多有：

$$
M\gamma
$$

个邻居。

搜索时：

1. 扫描这些邻居；
2. 判断：

$$
p(u)=True?
$$

3. 只保留合法节点；
4. 最终取前：

$$
M
$$

个用于距离搜索。

即：

$$
N_p(v)=\{u\in N(v)\mid p(u)=True\}
$$

然后：

$$
N_p(v)[1:M]
$$

参与下一步 ANN 搜索。

---

# 32. ACORN-$\gamma$ 压缩后的 GET_NEIGHBORS

如果使用 $M_\beta$ compression：

- 前 $M_\beta$ 个邻居直接检查；
- 对后半部分保留邻居执行 neighbor-of-neighbor expansion；
- 先把可能被压缩掉的节点恢复出来；
- 再应用 predicate；
- 最后截断到 $M$。

可以理解为：

```text
1-hop
+
部分 2-hop
↓
predicate filter
↓
取前 M
```

---

# 33. ACORN-1

ACORN-1 更偏向减少构建成本。

论文中它可以看作：

$$
\gamma=1
$$

的特殊构建方式。

它不在 construction 阶段真正建立 $M\gamma$ 的稠密图，而是在查询时进行更完整的 neighbor expansion：

```text
访问 v
↓
收集 1-hop 邻居
+
2-hop 邻居
↓
predicate filtering
↓
截断为 M
```

因此：

| 项目 | ACORN-$\gamma$ | ACORN-1 |
|---|---|---|
| 构建 | 更密 | 接近普通 HNSW |
| 索引大小 | 更大 | 更小 |
| TTI | 更高 | 更低 |
| 查询 | 较快 | 需要更多在线 expansion |
| 适合 | 查询密集 | 构建资源受限 |

---

# 34. ACORN 为什么还需要 pre-filter fallback

如果 selectivity 极低：

$$
s\ll1
$$

那么即使图很密，合法节点之间仍然可能难以保持良好连接。

如果硬要求：

$$
\gamma\approx\frac{1}{s}
$$

则 $s$ 极小时：

$$
\gamma
$$

会非常大，构建代价和内存不可接受。

因此论文提出：

$$
s_{\min}=\frac{1}{\gamma}
$$

若：

$$
s>\frac{1}{\gamma}
$$

优先搜索 ACORN。

若：

$$
s\le\frac{1}{\gamma}
$$

可以退回 pre-filtering。

因为此时 $sn$ 很小，直接过滤并扫描合法向量反而可能很划算。

---

# 35. Oracle HNSW 的复杂度为什么是 $O(\log(sn)+K)$

普通 HNSW：

$$
O(\log n+K)
$$

而 Oracle Partition Index 只建立在：

$$
X_p
$$

上。

因为：

$$
|X_p|=sn
$$

所以把普通 HNSW 中的 $n$ 替换为：

$$
sn
$$

得到：

$$
\boxed{
O(\log(sn)+K)
}
$$

它可以理解为：

$$
\underbrace{O(\log|X_p|)}_{\text{在合法节点专属 HNSW 中导航}}
+
\underbrace{O(K)}_{\text{返回 K 个结果}}
$$

---

# 36. ACORN 的构建复杂度

论文分析 ACORN-$\gamma$ 的预期构建复杂度约为：

$$
O(n\gamma\log n\log\gamma)
$$

相较 HNSW：

$$
O(n\log n)
$$

ACORN 额外增加的代价主要来自：

- 更大的 candidate edge 数量；
- 更大的搜索范围；
- 维护更大的候选结构。

因此：

$$
\gamma\uparrow
$$

往往意味着：

```text
predicate subgraph 更强
但
TTI 更高
内存更大
```

---

# 37. ACORN 搜索复杂度

论文给出的 ACORN-$\gamma$ 预期搜索复杂度形式是：

$$
O\left(
(d+\gamma)\log(sn)+\log(1/s)
\right)
$$

可以理解成两阶段：

## 阶段 1：找到 predicate subgraph 的入口

从 ACORN 全局 entry point 开始。

在还没有进入合法 predicate 节点区域时，需要逐层过滤。

大约产生：

$$
O(\log(1/s))
$$

的额外代价。

## 阶段 2：在 predicate subgraph 上搜索

predicate subgraph 大约有：

$$
sn
$$

个节点。

搜索类似于在：

$$
HNSW(X_p)
$$

上工作。

向量距离计算产生和维度 $d$ 相关的代价，邻居 predicate 检查产生和 $\gamma$ 相关的额外代价。

因此形成：

$$
O((d+\gamma)\log(sn))
$$

---

# 38. 一组完整参数示例

假设：

$$
M=32
$$

系统希望支持最低：

$$
s_{\min}=\frac{1}{12}
$$

那么：

$$
\gamma=12
$$

因此候选边规模：

$$
M\gamma=32\times12=384
$$

如果查询 selectivity：

$$
s=\frac{1}{12}
$$

则过滤后期望合法邻居数量：

$$
M\gamma s
=
32\times12\times\frac{1}{12}
=
32
$$

刚好约等于：

$$
M
$$

这就是 ACORN-$\gamma$ 最重要的数学直觉。

---

# 39. 实现 HNSW 时需要的数据结构

一个常见的实现可以包含：

```cpp
struct Node {
    int id;
    std::vector<float> vector;
    int max_level;

    // neighbors[level] = 当前层邻居 ID 列表
    std::vector<std::vector<int>> neighbors;
};
```

索引本身：

```cpp
struct HNSWIndex {
    int M;
    int efConstruction;
    int maxLevel;
    int entryPoint;

    std::vector<Node> nodes;
};
```

搜索通常需要：

```cpp
visited set
candidate min-heap
result max-heap
```

其中：

- candidate min-heap：最靠近 query 的候选优先展开；
- result max-heap：堆顶保存当前结果中最差的节点，便于淘汰。

---

# 40. HNSW SEARCH-LAYER 概念伪代码

下面是理解版伪代码，不是论文源码：

```cpp
search_layer(query, entry, ef, level):
    visited = {}
    candidates = min_heap()
    results = max_heap()

    d0 = distance(query, entry)

    candidates.push(entry, d0)
    results.push(entry, d0)
    visited.insert(entry)

    while candidates not empty:

        c = candidates.top_min()
        worst = results.top_max()

        if distance(c, query) > distance(worst, query)
           and results.size() >= ef:
            break

        candidates.pop()

        for neighbor in neighbors[c][level]:

            if neighbor already visited:
                continue

            visited.insert(neighbor)

            d = distance(query, neighbor)

            if results.size() < ef
               or d < distance(results.top_max(), query):

                candidates.push(neighbor, d)
                results.push(neighbor, d)

                if results.size() > ef:
                    results.pop_max()

    return results
```

---

# 41. HNSW 插入节点的概念伪代码

```cpp
insert(v):

    l = random_level()

    if index empty:
        entryPoint = v
        maxLevel = l
        return

    ep = entryPoint

    // 第一阶段：只导航
    for level = maxLevel down to l + 1:
        ep = greedy_search(v, ep, level)

    // 第二阶段：开始建边
    for level = min(l, maxLevel) down to 0:

        candidates =
            search_layer(v, ep, efConstruction, level)

        neighbors =
            rng_prune(candidates, M)

        connect(v, neighbors, level)

        ep = best candidates

    if l > maxLevel:
        entryPoint = v
        maxLevel = l
```

---

# 42. ACORN-$\gamma$ 构建概念伪代码

重点展示相较 HNSW 的变化：

```cpp
insert_acorn(v):

    l = random_level()
    ep = global_entry_point

    // 高层仍然用于定位
    for level = maxLevel down to l + 1:
        ep = greedy_navigation_using_first_M_neighbors(v, ep, level)

    // 从 l(v) 开始真正建边
    for level = min(l, maxLevel) down to 0:

        candidates =
            expanded_construction_search(
                v,
                ep,
                efConstruction,
                target_candidate_edges = M * gamma,
                traversal_degree = M
            )

        if level == 0:
            final_neighbors =
                acorn_compress(
                    candidates,
                    M_beta,
                    M * gamma
                )
        else:
            final_neighbors =
                candidates_up_to_M_gamma

        connect(v, final_neighbors, level)

        ep = suitable_entry_for_next_level
```

注意：

```text
traversal_degree = M
```

和：

```text
target_candidate_edges = M * gamma
```

是两个不同概念。

---

# 43. ACORN compression 概念伪代码

```cpp
compress(candidates, M_beta, M_gamma):

    // candidates 已按与 v 的距离排序

    selected = []
    H = set()

    // 1. 前 M_beta 个直接保存
    for i in [0, M_beta):
        selected.push(candidates[i])

    // 2. 处理剩余候选
    for c in candidates[M_beta:]:

        if c in H:
            // c 可以通过 two-hop 恢复
            continue

        selected.push(c)

        // 以后通过 c 的邻居表可以恢复这些节点
        for x in neighbors[c]:
            H.insert(x)

        if stopping_condition_reached:
            break

    return selected
```

其中：

$$
H
$$

是动态 two-hop 覆盖集合。

---

# 44. ACORN 搜索概念伪代码

```cpp
acorn_search_layer(query, predicate, entry, ef, level):

    visited = {entry}
    candidates = min_heap(entry)
    results = max_heap(entry)

    while candidates not empty:

        c = nearest candidate
        worst = furthest current result

        if c is worse than worst
           and results.size() >= ef:
            break

        neighborhood =
            get_neighbors(c, level, predicate)

        for v in first M nodes of neighborhood:

            if v already visited:
                continue

            visited.insert(v)

            if v improves current result
               or results.size() < ef:

                candidates.push(v)
                results.push(v)

                if results.size() > ef:
                    remove furthest result

    return results
```

---

# 45. ACORN 的 GET_NEIGHBORS

## 不压缩版本

```cpp
get_neighbors(v, level, predicate):

    valid = []

    for u in N(v):
        if predicate(u):
            valid.push(u)

    return first M valid neighbors
```

---

## compression 版本

概念上：

```cpp
get_neighbors(v, level, predicate):

    expanded = []

    // 前 M_beta 个直接邻居
    for u in N(v)[0:M_beta]:
        expanded.push(u)

    // 后半部分执行 two-hop expansion
    for u in N(v)[M_beta:]:
        expanded.push(u)

        for x in N(u):
            expanded.push(x)

    valid = filter(expanded, predicate)

    return first M valid nodes
```

真实实现会做去重、访问控制、顺序管理以及性能优化。

---

# 46. 用户学习过程中最有价值的问题与答案

下面集中整理前面学习过程中几个非常容易混淆的问题。

---

## Q1：$X_p$ 是在 KNN 筛选出来的向量里，还是全部数据库里？

答案：

$$
\boxed{
X_p\text{ 是整个数据库中所有满足 predicate 的向量集合}
}
$$

不是 KNN 的候选子集。

---

## Q2：Hybrid Search 是先筛属性，再筛距离吗？

从数学目标看：

$$
\boxed{
\text{先满足属性约束，再在合法节点中按距离找 Top-K}
}
$$

但 ACORN 的实际执行不是先完整枚举 $X_p$。

它是在图搜索过程中把：

```text
predicate checking
+
distance search
```

交织在一起执行。

---

## Q3：$s$、$n$、$K$ 分别是什么？

$$
n=\text{数据库总量}
$$

$$
s=\frac{|X_p|}{n}
$$

$$
K=\text{最终返回的近邻个数}
$$

因此：

$$
sn=|X_p|
$$

---

## Q4：HNSW 插入新节点时，每一层都搜索 $ef_c$ 并建边吗？

不是。

假设：

$$
L=4,\quad l(v)=2
$$

则：

```text
Level 4：只导航
Level 3：只导航
Level 2：开始找候选并建边
Level 1：找候选并建边
Level 0：找候选并建边
```

所以：

$$
L\rightarrow l(v)+1
$$

只定位。

而：

$$
l(v)\rightarrow0
$$

才真正参与新节点建边。

---

## Q5：greedy search 会不会检查 entry point 本身与 query 的距离？

会。

入口点本身就是当前搜索基准。

先有：

$$
d(q,e)
$$

再检查邻居是否能改善：

$$
d(q,u)<d(q,e)
$$

如果改善，就继续移动。

---

## Q6：为什么普通 HNSW 复杂度写成 $O(\log n+K)$？

论文为了分析，把 HNSW 搜索看成：

```text
多层导航：O(log n)
+
返回 K 个结果：O(K)
```

所以：

$$
O(\log n+K)
$$

这是分析上的期望/理想化表达。

---

## Q7：为什么 Oracle Partition 是 $O(\log(sn)+K)$？

因为：

$$
|X_p|=sn
$$

专门在 $X_p$ 上建立 HNSW，相当于把普通 HNSW 中：

$$
n
$$

替换成：

$$
sn
$$

因此：

$$
O(\log(sn)+K)
$$

---

## Q8：ACORN 是不是和 HNSW 一样先随机层高，然后高层找入口？

是。

ACORN 没有推翻 HNSW 的 hierarchy。

仍然：

```text
随机 l(v)
↓
高层 greedy navigation
↓
到 l(v) 后开始真正处理新节点邻居
```

主要改变的是：

```text
邻居候选扩展
+
predicate-agnostic pruning/compression
+
search 阶段的 predicate-aware neighbor lookup
```

---

## Q9：ACORN 是先得到 $ef_c$ 个候选，再从里面选 $M\gamma$ 个吗？

不应这样理解。

因为完全可能：

$$
M\gamma>ef_c
$$

例如：

$$
M=32,\quad\gamma=12,\quad ef_c=40
$$

则：

$$
M\gamma=384>40
$$

所以：

$$
ef_c
$$

和：

$$
M\gamma
$$

不是简单的“先后包含关系”。

---

## Q10：ACORN construction 从全局最高层到 $l(v)+1$，到底看 $M$ 个还是 $M\gamma$ 个邻居？

在构建阶段做图遍历时：

$$
\boxed{
\text{每访问一个旧节点，主要查看它前 }M\text{ 个邻居}
}
$$

即使该旧节点实际可能存了：

$$
M\gamma
$$

条边。

原因是：

- 前 $M$ 条边被认为足够维持 navigability；
- 如果每次都检查 $M\gamma$，构建时间会显著增加。

---

## Q11：那 $M\gamma$ 到底在哪里体现？

体现在：

$$
\boxed{
\text{ACORN 为新节点希望收集的 candidate edges 数量}
}
$$

不是每访问一个旧节点就展开 $M\gamma$ 个邻居。

---

## Q12：为什么 ACORN 不能直接使用 HNSW 的 RNG pruning？

因为 HNSW 删除边时依赖：

```text
v -> a -> b
```

这条替代路径。

但 predicate 可能让：

```text
a ×
```

失效。

结果：

```text
v ✓      b ✓
```

在 predicate subgraph 中断开。

---

## Q13：ACORN compression 为什么更安全？

因为它删除边的依据不是：

```text
以后一定能走过中间节点
```

而是：

```text
以后访问当前节点时，可以通过 two-hop neighbor expansion 把目标节点重新发现出来
```

因此中间节点不一定要满足 predicate。

---

## Q14：$M_\beta$ 到底是什么？

$$
M_\beta
$$

控制多少个最近邻直接保留。

前：

$$
M_\beta
$$

个 candidate edges：

```text
无条件直接保存
```

后面的：

```text
如果已被 two-hop 覆盖，则可以 prune
```

因此它控制：

$$
\boxed{
\text{内存}
\leftrightarrow
\text{查询时 two-hop expansion 成本}
}
$$

之间的权衡。

---

# 47. 学习 ACORN 时最容易犯的错误

## 错误 1：把 $X_p$ 理解成 ANN 搜索后的候选集合

正确：

$$
X_p
$$

来自整个数据库的 predicate 定义。

---

## 错误 2：认为 ACORN 先把所有合法节点完整筛出来

正确：

ACORN 主要在图搜索过程中动态进行 predicate filtering。

---

## 错误 3：认为 $ef_c$ 就是节点最终边数

正确：

$$
ef_c=\text{construction search breadth}
$$

而最终边数由具体索引和 pruning 决定。

---

## 错误 4：认为 ACORN 每访问一个节点都要计算 $M\gamma$ 个邻居距离

正确：

construction traversal 中主要使用前：

$$
M
$$

个邻居维持导航。

---

## 错误 5：认为 two-hop expansion 必须真的走进中间节点

正确：

中间节点可以只是用于读取邻接表。

即使中间节点：

$$
p(c)=False
$$

仍然可以利用：

$$
N(c)
$$

恢复满足 predicate 的二跳邻居。

---

## 错误 6：认为 ACORN-$\gamma$ 只是“把 M 改成 Mγ”

不够准确。

完整创新包括：

1. predicate-agnostic neighbor expansion；
2. predicate-agnostic compression；
3. search 时 predicate filtering；
4. two-hop recovery；
5. selectivity 较低时回退 pre-filtering。

---

# 48. 从 HNSW 到 ACORN 的核心逻辑链

建议把下面这条链背熟：

```text
HNSW
↓
普通 ANN 很快
↓
加入 predicate
↓
原图中的大量邻居被过滤
↓
predicate subgraph degree 降低
↓
图可能难以导航甚至断开
↓
ACORN 先把邻居候选扩大到 Mγ
↓
希望过滤后仍剩约 M 个合法邻居
↓
但 Mγ 太占内存
↓
引入 M_beta compression
↓
近邻直接存，远邻通过 two-hop 恢复
↓
搜索时先扩展邻居，再 predicate filter
↓
尽量模拟 HNSW(X_p)
```

数学核心：

$$
M\gamma s\approx M
$$

因此：

$$
\gamma\approx\frac{1}{s}
$$

工程核心：

$$
\boxed{
\text{Dense graph}
+
\text{Predicate filtering}
+
\text{Two-hop recovery}
}
$$

---

# 49. 建议的复现顺序

如果准备真正实现 ACORN，建议按下面顺序做。

### 第一步：普通 HNSW

先实现或跑通：

- random level；
- insertion；
- greedy navigation；
- SEARCH-LAYER；
- $M$；
- $ef_c$；
- $ef_s$；
- Recall@K；
- QPS。

### 第二步：HNSW + post-filter

实现：

```text
ANN search
↓
predicate filter
```

观察 $s$ 下降时性能恶化。

### 第三步：ACORN-1

先不改 construction。

查询时：

```text
1-hop
+
2-hop
↓
predicate filter
↓
取前 M
```

这是理解 predicate subgraph traversal 的最简单入口。

### 第四步：ACORN-$\gamma$ without compression

构建时扩大 candidate edge 数：

$$
M\rightarrow M\gamma
$$

先验证搜索性能。

### 第五步：实现 $M_\beta$ compression

加入：

- nearest-$M_\beta$ direct preservation；
- two-hop coverage set $H$；
- recoverable candidate pruning。

### 第六步：做消融实验

重点改变：

1. selectivity $s$；
2. $\gamma$；
3. $M_\beta$；
4. query correlation；
5. dataset size；
6. vector dimension。

记录：

- Recall@K；
- QPS；
- distance computations；
- index size；
- TTI。

---

# 50. 读论文时应该重点对应的章节

建议阅读顺序：

1. **Section 2.1 - Hierarchical Navigable Small Worlds**
   - HNSW construction
   - HNSW search
   - $M$、$ef_c$、$ef_s$

2. **Section 3 - Problem Definition and Challenges**
   - $X_p$
   - selectivity $s$
   - pre-filter
   - post-filter
   - query correlation

3. **Section 4 - Theoretical Ideal Hybrid Search**
   - Oracle Partition Index
   - $O(\log(sn)+K)$

4. **Section 5 - ACORN Overview**
   - predicate subgraph

5. **Section 5.1 - ACORN-$\gamma$ Search**
   - Algorithm 2
   - GET-NEIGHBORS
   - filtering
   - two-hop expansion

6. **Section 5.2 - ACORN-$\gamma$ Construction**
   - $M\gamma$
   - $\gamma=1/s_{\min}$
   - $M_\beta$
   - compression
   - 为什么 HNSW RNG pruning 不适合 hybrid search

7. **Section 5.3 - ACORN-1**
   - search-time expansion

8. **Section 6 - Discussion**
   - index size
   - construction complexity
   - search complexity

9. **Section 7 - Evaluation**
   - Recall-QPS
   - TTI
   - index size
   - selectivity / correlation / scale 实验

---

# 51. 最终记忆版

如果只允许记 10 句话，可以记下面这些：

1. HNSW 是多层图式 ANN。
2. 高层负责快速定位，Level 0 负责精细搜索。
3. $M$ 控制图连接度，$ef_c$ 控制建图搜索宽度，$ef_s$ 控制查询搜索宽度。
4. Hybrid Search 的目标是从 $X_p$ 中找 Top-K，而 $|X_p|=sn$。
5. Pre-filter 扫描 $X_p$，复杂度约 $O(sn+K)$。
6. Oracle HNSW 若只建立在 $X_p$ 上，复杂度约 $O(\log(sn)+K)$。
7. ACORN 想让任意 predicate subgraph 尽量像一个 HNSW。
8. ACORN-$\gamma$ 把邻居候选从 $M$ 扩展到 $M\gamma$，使过滤后期望仍保留约 $M$ 个合法邻居。
9. HNSW RNG pruning 依赖“中间节点可走”，而 predicate 可能把中间节点过滤掉。
10. ACORN 用 $M_\beta$ compression + two-hop recovery，在节省空间的同时尽量保留未来任意 predicate 下的可导航性。

最核心的两个公式：

$$
|X_p|=sn
$$

以及：

$$
M\gamma s\approx M
$$

因此：

$$
\gamma\approx\frac{1}{s}
$$

---

# 52. 一句话理解整篇 ACORN

$$
\boxed{
\text{ACORN 通过构造更稠密、可压缩、可二跳恢复的 HNSW 类图，
使任意查询 predicate 过滤后的子图仍尽量保持可导航，
从而近似在 }X_p\text{ 上单独建立 HNSW 的搜索效果。}
}
$$