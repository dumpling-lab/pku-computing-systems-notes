# Lec12_Propositional Logic and SAT(Part 2)_Notes

> AI 整理。

------

## 零、这一讲要解决什么问题？

上一讲介绍了命题逻辑的语法、语义、解释（interpretation）、可满足性（satisfiability）等基本概念。这一讲进一步回答两个核心问题：

1. **如何把任意布尔公式变成 SAT solver 更喜欢的形式？**
   - 先介绍三种 normal form：NNF、DNF、CNF。
   - 重点是 CNF，因为现代 SAT solver 基本都以 CNF 作为输入形式。
   - 直接转换 CNF/DNF 可能指数爆炸，所以引入 **Tseitin Transformation**，通过增加辅助变量，在保持“可满足性”等价的前提下线性规模转换成 CNF。

2. **SAT solver 如何判断 CNF 是否可满足？**
   - 朴素方法：真值表、ROBDD、语义推导。
   - 经典回溯算法：DPLL。
   - 现代主流算法：CDCL，即 Conflict-Driven Clause Learning。
   - 特殊高效情形：2-SAT 可以用强连通分量 SCC 在线性时间解决。

------

## 一、基本术语回顾

### 1.1 命题变量、文字、子句、公式

命题逻辑中最基本的对象是布尔变量，例如：

$$
x_1, x_2, x_3, \ldots
$$

每个变量只能取两个值：真或假，常写作：

$$
\mathrm{True}/\mathrm{False}, \quad 1/0
$$

**文字（literal）** 是一个变量或它的否定：

$$
\ell ::= x \quad \text{or} \quad \neg x
$$

例如：

- $x_1$ 是正文字；
- $\neg x_2$ 是负文字。

后面讲 CNF 和 DNF 时会频繁出现“子句（clause）”这个词。需要注意：不同教材有时会把 DNF 中的“合取项”也叫 clause，但在 SAT solver 语境里，**clause 通常特指 CNF 里的一个析取子句**，也就是若干 literal 的 OR。

### 1.2 解释、满足、可满足

一个 **interpretation（解释/赋值）** 就是给每个变量指定 True 或 False。

例如对变量 $x,y,z$，一个解释可以是：

$$
I(x)=1,\quad I(y)=0,\quad I(z)=1
$$

若某个解释 $I$ 使公式 $F$ 的值为真，就说 $I$ **satisfies** $F$，记作：

$$
I \models F
$$

若存在至少一个解释使 $F$ 为真，则 $F$ 是 **SAT / satisfiable（可满足）**。若不存在任何解释使它为真，则 $F$ 是 **UNSAT / unsatisfiable（不可满足）**。

------

## 二、Normal Forms：范式的整体概念

### 2.1 什么是 normal form？

一个公式 $G$ 是公式 $F$ 的 normal form，如果：

1. $G$ 与 $F$ 在语义上等价；
2. $G$ 满足某种固定的语法限制。

语义等价指的是：对任意解释 $I$，两者真值完全相同：

$$
F \equiv G
\quad \Longleftrightarrow \quad
\forall I,\ I(F)=I(G)
$$

这一讲主要介绍三种范式：

- NNF：Negation Normal Form，否定范式；
- DNF：Disjunctive Normal Form，析取范式；
- CNF：Conjunctive Normal Form，合取范式。

它们之间的关系可以粗略理解为：

- NNF 是最宽松的，只限制否定号的位置；
- DNF 是“外面 OR，里面 AND”；
- CNF 是“外面 AND，里面 OR”；
- DNF 和 CNF 都天然属于 NNF，因为它们的否定只会出现在 literal 上。

------

## 三、NNF：Negation Normal Form（否定范式）

### 3.1 NNF 的语法限制

一个公式是 NNF，需要满足两个条件：

1. 只使用下面三种逻辑联结词：

$$
\neg,\quad \land,\quad \lor
$$

也就是说，公式里不能直接出现：

$$
\to,\quad \leftrightarrow
$$

2. 否定号 $\neg$ 只能直接作用在变量上，也就是只能出现在 literal 中。

例如：

$$
(x \lor \neg y) \land z
$$

是 NNF，因为 $\neg$ 只作用在 $y$ 上。

但是：

$$
\neg(x \land y)
$$

不是 NNF，因为 $\neg$ 作用在了整个 $(x \land y)$ 上。

再比如：

$$
\neg\neg x
$$

也不是标准 NNF，因为有双重否定，还没有化到 literal 层面。

### 3.2 转换到 NNF 的步骤

把任意公式转换成 NNF，一般分两步。

第一步：消去蕴含和等价。

常用等价式：

$$
F \to G \equiv \neg F \lor G
$$

$$
F \leftrightarrow G \equiv (F \to G) \land (G \to F)
$$

因此也可以写成：

$$
F \leftrightarrow G
\equiv
(\neg F \lor G) \land (\neg G \lor F)
$$

第二步：把否定号往里推，直到否定号只作用在变量上。

使用 De Morgan 定律：

$$
\neg(F \land G) \equiv \neg F \lor \neg G
$$

$$
\neg(F \lor G) \equiv \neg F \land \neg G
$$

以及双重否定：

$$
\neg\neg F \equiv F
$$

### 3.3 NNF 转换例子

例如把下面公式转成 NNF：

$$
\neg\big((p \to q) \land \neg r\big)
$$

先消去 $\to$：

$$
p \to q \equiv \neg p \lor q
$$

所以原公式变成：

$$
\neg\big((\neg p \lor q) \land \neg r\big)
$$

再用 De Morgan 定律：

$$
\neg(\neg p \lor q) \lor \neg\neg r
$$

继续往里推：

$$
(p \land \neg q) \lor r
$$

这个公式中否定号只作用在变量 $q$ 上，因此是 NNF。

### 3.4 记忆方法

NNF 的核心不是“外面长什么样”，而是：

> **所有否定号都必须压到变量旁边。**

所以判断 NNF 时只看两点：

- 公式里有没有 $\to$、 $\leftrightarrow$；
- 有没有 $\neg$ 作用在括号、大公式或另一个 $\neg$ 上。

------

## 四、DNF：Disjunctive Normal Form（析取范式）

### 4.1 DNF 的形式

DNF 是若干个“合取项”的析取。标准形式可以写成：

$$
F = C_1 \lor C_2 \lor \cdots \lor C_m
$$

其中每个 $C_i$ 是若干 literal 的 AND：

$$
C_i = \ell_{i1} \land \ell_{i2} \land \cdots \land \ell_{ik_i}
$$

所以整体形状是：

$$
F =
(\ell_{11}\land\ell_{12}\land\cdots)
\lor
(\ell_{21}\land\ell_{22}\land\cdots)
\lor
\cdots
$$

直观记忆：

> **DNF = OR of ANDs = 外层是 OR，内层是 AND。**

例如：

$$
(x \land \neg y) \lor (z \land w) \lor (\neg x)
$$

就是 DNF。

### 4.2 DNF 一定是 NNF

DNF 里每个否定号都只会作用在变量上，而且只使用 $\neg,\land,\lor$。因此 DNF 一定满足 NNF 的要求。

但是 NNF 不一定是 DNF。例如：

$$
x \land (y \lor z)
$$

是 NNF，但不是 DNF，因为里面出现了 AND 包着一个 OR，需要展开后才是 DNF。

### 4.3 转换到 DNF

把公式转换到 DNF 的流程是：

1. 先转换成 NNF；
2. 用分配律把 $\lor$ 推到外层。

关键分配律是：

$$
A \land (B \lor C)
\equiv
(A \land B) \lor (A \land C)
$$

以及：

$$
(A \lor B) \land C
\equiv
(A \land C) \lor (B \land C)
$$

### 4.4 DNF 转换例子

例如：

$$
x \land (y \lor \neg z)
$$

它已经是 NNF，但还不是 DNF。用分配律展开：

$$
x \land (y \lor \neg z)
\equiv
(x \land y) \lor (x \land \neg z)
$$

右边就是 DNF。

再比如：

$$
(a \lor b) \land (c \lor d)
$$

展开成：

$$
(a\land c) \lor (a\land d) \lor (b\land c) \lor (b\land d)
$$

这说明 DNF 本质上很像“列出所有能让公式为真的情况”。

### 4.5 DNF 的可满足性为什么容易判断？

若公式是：

$$
F = C_1 \lor C_2 \lor \cdots \lor C_m
$$

那么只要其中某一个 $C_i$ 可满足，整个 $F$ 就可满足。

而每个 $C_i$ 是若干 literal 的 AND：

$$
C_i = \ell_1 \land \ell_2 \land \cdots \land \ell_k
$$

要让它为真，所有 literal 都必须为真。因此只需要检查它内部是否存在冲突：

$$
x \quad \text{和} \quad \neg x
$$

是否同时出现。

若同时出现，则该合取项不可满足。例如：

$$
x \land \neg x \land y
$$

不可能为真。

若没有冲突，则它可满足。例如：

$$
x \land \neg y \land z
$$

可由赋值 $x=1,y=0,z=1$ 满足。

所以 DNF 的 SAT 判定非常简单：

> **逐项检查，只要有一个合取项内部没有变量冲突，整个公式就是 SAT。**

### 4.6 为什么不用 DNF 直接解决 SAT？

虽然 DNF 的可满足性很好判断，但把普通公式转成 DNF 可能导致指数级膨胀。

典型例子：

$$
F=(a_1\lor b_1)\land(a_2\lor b_2)\land\cdots\land(a_n\lor b_n)
$$

转换成 DNF 时，每个括号都要选择一个 literal，最终会出现：

$$
2^n
$$

个合取项。

例如 $n=3$：

$$
(a_1\lor b_1)(a_2\lor b_2)(a_3\lor b_3)
$$

展开后已经有 8 项。一般情况下，这和枚举真值表差不多糟糕。

因此：

> DNF 对判定 satisfiability 很方便，但转换到 DNF 本身可能指数爆炸，所以实际 SAT solver 不这么做。

------

## 五、CNF：Conjunctive Normal Form（合取范式）

### 5.1 CNF 的形式

CNF 是若干个“析取子句”的合取。标准形式可以写成：

$$
F = C_1 \land C_2 \land \cdots \land C_m
$$

其中每个 $C_i$ 是若干 literal 的 OR：

$$
C_i = \ell_{i1} \lor \ell_{i2} \lor \cdots \lor \ell_{ik_i}
$$

所以整体形状是：

$$
F =
(\ell_{11}\lor\ell_{12}\lor\cdots)
\land
(\ell_{21}\lor\ell_{22}\lor\cdots)
\land
\cdots
$$

直观记忆：

> **CNF = AND of ORs = 外层是 AND，内层是 OR。**

例如：

$$
(x \lor \neg y \lor z) \land (\neg x \lor w) \land (y)
$$

就是 CNF。

其中：

- $(x \lor \neg y \lor z)$ 是一个 clause；
- $(\neg x \lor w)$ 是一个 clause；
- $(y)$ 是一个 unit clause，即只有一个 literal 的子句。

### 5.2 CNF 也一定是 NNF

CNF 只使用 $\neg,\land,\lor$，并且 $\neg$ 只作用在变量上，所以 CNF 也是 NNF。

### 5.3 转换到 CNF

流程与 DNF 类似：

1. 先转换到 NNF；
2. 用分配律把 $\land$ 推到外层。

关键分配律是：

$$
A \lor (B \land C)
\equiv
(A \lor B) \land (A \lor C)
$$

以及：

$$
(A \land B) \lor C
\equiv
(A \lor C) \land (B \lor C)
$$

### 5.4 CNF 转换例子

例如：

$$
x \lor (y \land \neg z)
$$

已经是 NNF，但不是 CNF。用分配律：

$$
x \lor (y \land \neg z)
\equiv
(x \lor y) \land (x \lor \neg z)
$$

右边就是 CNF。

再比如：

$$
(a \land b) \lor (c \land d)
$$

先对右侧分配：

$$
[(a\land b)\lor c] \land [(a\land b)\lor d]
$$

继续分配：

$$
(a\lor c)\land(b\lor c)\land(a\lor d)\land(b\lor d)
$$

### 5.5 CNF 的 satisfiability 为什么不容易？

DNF 是 OR of ANDs，只要找到一个不冲突的合取项即可。

CNF 是 AND of ORs，必须同时满足每一个 clause。一个变量的取值可能会影响很多 clause，局部满足某个子句不代表整体可满足。

例如：

$$
(x\lor y)\land(\neg x\lor y)\land(x\lor \neg y)\land(\neg x\lor \neg y)
$$

每个子句单独都很容易满足，但四个放在一起就是 UNSAT。因为前两个子句要求在 $x$ 两种情况下都要 $y=1$，后两个又会强迫矛盾。

所以：

> **CNF 的局部子句很简单，但全局 satisfiability 是困难的。一般命题 SAT 可以通过等可满足转换归约到 CNF-SAT。**

### 5.6 那为什么 SAT solver 还喜欢 CNF？

需要考虑的问题是：

> 直接转 CNF 似乎也会爆炸，CNF 的 satisfiability 又不简单，那为什么 SAT solver 还要先转 CNF？

答案是：SAT solver 不要求转换后的公式与原公式在每个变量赋值下完全等价，而只要求它们 **等可满足**。为此可以引入额外变量表示子公式，这就是 Tseitin Transformation。

------

## 六、等可满足性：Equisatisfiability

### 6.1 等价 vs 等可满足

两个公式 $F$ 和 $G$ **等价（equivalent）**，要求对所有解释真值完全相同：

$$
\forall I,\ I(F)=I(G)
$$

两个公式 $F$ 和 $G$ **等可满足（equisatisfiable）**，只要求它们在“是否存在满足赋值”这件事上一致：

$$
F \text{ is SAT} \quad \Longleftrightarrow \quad G \text{ is SAT}
$$

也就是：

- $F$ 可满足，则 $G$ 可满足；
- $G$ 可满足，则 $F$ 可满足。

但它们不需要在每一个赋值下真值相同。

### 6.2 为什么等可满足就够了？

SAT solver 的目标通常不是求出公式在所有赋值下的真值，而是判断：

> 有没有一个赋值让公式为真？

因此只要新公式和旧公式同为 SAT 或同为 UNSAT，求解结果就可靠。

这给了我们很大的自由度：

- 可以引入新变量；
- 可以构造一个和原公式不完全等价的公式；
- 只要保证“存在满足赋值”这一性质不变即可。

### 6.3 简单例子

设原公式是：

$$
F=a\lor b
$$

引入新变量 $x$，构造：

$$
G=x\land(x\leftrightarrow(a\lor b))
$$

把两式都看作变量集合 $\{a,b,x\}$ 上的公式时，它们并不等价。例如 $a=1,b=0,x=0$ 使 $F=1,G=0$。但它们等可满足：

- 如果 $F$ 可满足，例如 $a=1$，那么令 $x=1$， $G$ 也可满足；
- 如果 $G$ 可满足，因为 $G$ 中有 $x$，且 $x\leftrightarrow(a\lor b)$，所以 $a\lor b$ 必须为真，原公式 $F$ 可满足。

这个思想就是 Tseitin 转换的基础。

------

## 七、Tseitin Transformation：线性规模转 CNF

### 7.1 核心思想

Tseitin Transformation 的核心是：

> **为每个子公式引入一个新变量，让这个变量表示该子公式的真值。**

例如原公式是：

$$
F=(a\land b)\lor \neg c
$$

可以引入：

$$
z_1 \leftrightarrow (a\land b)
$$

$$
z_2 \leftrightarrow \neg c
$$

$$
z_3 \leftrightarrow (z_1\lor z_2)
$$

最后强制根变量 $z_3$ 为真：

$$
z_3
$$

于是新公式是：

$$
z_3
\land
(z_1 \leftrightarrow (a\land b))
\land
(z_2 \leftrightarrow \neg c)
\land
(z_3 \leftrightarrow (z_1\lor z_2))
$$

新公式与原公式等可满足。原公式的每个满足赋值都可以扩展为新公式的满足赋值；新公式的满足赋值限制到原变量后，也满足原公式。

### 7.2 从电路角度理解 Tseitin

如果把布尔公式看成组合逻辑电路：

- 输入变量是 primary input；
- 每个子公式对应一个逻辑门；
- 每个新变量对应一个门的输出；
- 根变量对应整个电路的输出。

Tseitin 的本质就是：

> 给每个 gate 的输出命名，然后用 CNF 子句约束“输出变量”和“输入变量之间的逻辑关系”。

这也是 SAT 在电路验证中非常自然的原因。

### 7.3 Tseitin 为什么是线性的？

直接用分配律转换 CNF 时，一个子公式可能会被反复复制，导致指数爆炸。

Tseitin 不复制大子公式，而是用一个新变量代表它。对这里的一元、二元逻辑门，每个门只需要产生常数个 CNF 子句。

如果原公式/电路有 $m$ 个逻辑连接或门，则转换后：

- 新变量数量是 $O(m)$；
- 子句数量是 $O(m)$；
- 文字总数也是 $O(m)$。

所以转换规模是线性的。

### 7.4 常见逻辑门的 CNF 编码

下面用 $z$ 表示新变量， $a,b$ 表示输入变量或输入 literal。

#### 7.4.1 否定门

若：

$$
z \leftrightarrow \neg a
$$

等价于：

$$
(z \lor a) \land (\neg z \lor \neg a)
$$

解释：

- 如果 $z=1$，则 $a=0$；
- 如果 $z=0$，则 $a=1$。

#### 7.4.2 与门

若：

$$
z \leftrightarrow (a\land b)
$$

编码为：

$$
(\neg z \lor a)
\land
(\neg z \lor b)
\land
(z \lor \neg a \lor \neg b)
$$

含义分解：

- $z\to a$：如果输出为 1，则 $a$ 必须为 1；
- $z\to b$：如果输出为 1，则 $b$ 必须为 1；
- $(a\land b)\to z$：如果两个输入都为 1，则输出必须为 1。

#### 7.4.3 或门

若：

$$
z \leftrightarrow (a\lor b)
$$

编码为：

$$
(\neg z \lor a \lor b)
\land
(z \lor \neg a)
\land
(z \lor \neg b)
$$

含义分解：

- $z\to (a\lor b)$：输出为 1，则至少一个输入为 1；
- $a\to z$：如果 $a=1$，输出必须为 1；
- $b\to z$：如果 $b=1$，输出必须为 1。

#### 7.4.4 蕴含

若：

$$
z \leftrightarrow (a\to b)
$$

由于：

$$
a\to b \equiv \neg a\lor b
$$

可看成 $z\leftrightarrow(\neg a\lor b)$，编码为：

$$
(\neg z \lor \neg a \lor b)
\land
(z \lor a)
\land
(z \lor \neg b)
$$

#### 7.4.5 等价

若：

$$
z \leftrightarrow (a\leftrightarrow b)
$$

也就是 $z=1$ 当且仅当 $a,b$ 相等。可编码为：

$$
(\neg z\lor \neg a\lor b)
\land
(\neg z\lor a\lor \neg b)
\land
(z\lor \neg a\lor \neg b)
\land
(z\lor a\lor b)
$$

这个编码可以从以下逻辑拆出来：

- 若 $z=1$，则 $a\to b$ 且 $b\to a$；
- 若 $a=b=1$，则 $z=1$；
- 若 $a=b=0$，则 $z=1$。

### 7.5 Tseitin 转换完整流程

对任意公式 $F$：

1. 遍历公式语法树或电路 DAG；
2. 对每个非输入子公式 $g$ 创建新变量 $z_g$；
3. 根据子公式类型加入 $z_g\leftrightarrow g$ 的 CNF 编码；
4. 对根公式变量 $z_F$ 加入 unit clause：

$$
(z_F)
$$

最后得到一个 CNF 公式 $T(F)$。

性质：

$$
F \text{ is SAT} \quad \Longleftrightarrow \quad T(F) \text{ is SAT}
$$

### 7.6 CNF 的文本表示

CNF 可以用普通文本存储，用整数表示变量：

- 正整数 $i$ 表示变量 $x_i$；
- 负整数 $-i$ 表示文字 $\neg x_i$。

例如一行：

```text
-1 2 3
```

表示子句：

$$
\neg x_1\lor x_2\lor x_3
$$

例如：

```text
-1 2 3
2 4
1 -4
```

表示：

$$
(\neg x_1\lor x_2\lor x_3)
\land
(x_2\lor x_4)
\land
(x_1\lor \neg x_4)
$$

------

## 八、SAT 求解算法概览

SAT solving algorithms 主要包括：

- 基本方法：truth table、BDD、semantic argument；
- 经典算法：DP/DPLL；
- 现代算法：CDCL；
- 特殊情形：2-SAT。

历史上大致有这些节点：

- 1960：Davis-Putnam 提出 DP；
- 1962：DPLL 出现，用 unit propagation 替代一般 resolution；
- 之后 SAT 被证明为 NP-complete；
- 1990s 之后 CDCL、VSIDS、restarts、learned clause 等技术推动 SAT solver 变得很强；
- SAT 又进一步影响了 SMT、BMC、QBF、EDA 验证等方向。

------

## 九、Basic SAT Solving：朴素 SAT 方法

### 9.1 Truth Table 真值表法

若公式有 $n$ 个变量，则共有：

$$
2^n
$$

种解释。真值表法枚举所有解释，计算公式的值。

如果最后一列存在 True，则公式 SAT；否则 UNSAT。

优点：

- 概念简单；
- 一定正确；
- 适合变量很少的情况。

缺点：

- 枚举 $2^n$ 个赋值。若公式长度为 $|F|$，逐个求值的时间复杂度为 $O(2^n|F|)$；
- 变量稍多就不可行。

### 9.2 BDD / ROBDD 方法

BDD 是 Binary Decision Diagram，二叉决策图。ROBDD 是 Reduced Ordered BDD，归约有序 BDD。

用 ROBDD 判断 SAT 的思路：

- 构造公式的 ROBDD；
- 如果图中能到达 1 叶子，公式 SAT；
- 如果整个 ROBDD 只剩 0 叶子，公式 UNSAT。

ROBDD 的优点：

- 对某些结构化公式非常高效；
- 如果两个公式在同一变量顺序下得到相同 ROBDD，则它们等价。

ROBDD 的缺点：

- 大小非常依赖变量顺序；
- 某些函数的 ROBDD 会指数爆炸。

这也解释了为什么 SAT solver 和 BDD 在形式验证中各有用途。

### 9.3 Semantic Argument 语义推导

Semantic argument 使用推导规则分析公式。

基本思想是：

1. 假设公式可满足；
2. 根据公式结构推出变量必须满足的条件；
3. 如果推出矛盾，则说明 UNSAT；
4. 如果能完整推出一个无矛盾赋值，则说明 SAT。

例如：

$$
F = x \land \neg x
$$

假设 $F$ SAT，那么必须有：

$$
x=1
$$

同时：

$$
\neg x=1 \Rightarrow x=0
$$

矛盾，所以 UNSAT。

这种思路有助于理解 DPLL/CDCL 中的推导与冲突分析。

------

## 十、Resolution：归结规则

### 10.1 为什么需要 resolution？

DPLL 的推导部分可以通过受限的 resolution 理解：

> Resolution 只适用于 CNF，所以公式需要先转成 CNF。

CNF 由很多 clause 的 AND 组成。Resolution 可以从已有 clause 推出新的 clause。

### 10.2 一般 resolution 规则

考虑两个子句：

$$
(A\lor x)
$$

和：

$$
(B\lor \neg x)
$$

其中 $A,B$ 可以是若干 literal 的析取。

Resolution 规则可以推出 resolvent：

$$
A\lor B
$$

也就是：

$$
(A\lor x),\ (B\lor \neg x)
\quad \Longrightarrow \quad
(A\lor B)
$$

直观理解：

- 如果 $x=0$，为了满足 $(A\lor x)$，必须满足 $A$；
- 如果 $x=1$，为了满足 $(B\lor \neg x)$，必须满足 $B$；
- 无论 $x$ 是 0 还是 1，都能推出 $A\lor B$。

所以 resolvent 是原两个子句的逻辑后果。

### 10.3 空子句意味着 UNSAT

如果通过 resolution 推出了空子句：

$$
\Box
$$

则说明公式不可满足。

空子句表示“没有任何 literal 可以使这个子句为真”。在 CNF 中，所有子句都必须满足，一旦出现空子句，整个公式必然 UNSAT。

例如有两个 unit clause：

$$
(x),\quad(\neg x)
$$

resolution 后得到空子句：

$$
\Box
$$

说明矛盾。

------

## 十一、Unit Resolution 与 BCP

### 11.1 一般 resolution 的问题

一般 resolution 不一定能缩小问题规模，甚至可能生成大量新子句。

DPLL 采用受限版本：**unit resolution**。

### 11.2 Unit clause

只有一个 literal 的 clause 叫 unit clause。

例如：

$$
(x)
$$

或：

$$
(\neg y)
$$

在 CNF 中，如果出现 unit clause，那么为了满足公式，这个 literal 必须为真。

例如 $(x)$ 出现时，必须令：

$$
x=1
$$

$(\neg y)$ 出现时，必须令：

$$
y=0
$$

### 11.3 Unit resolution 规则

若有：

$$
(x)
$$

和：

$$
(A\lor \neg x)
$$

因为 $x$ 必须为真，所以 $\neg x$ 为假。第二个子句要满足，就必须满足 $A$。因此可推出：

$$
A
$$

也就是把 $\neg x$ 从所有子句中删掉。

同时，所有包含 $x$ 的子句已经被满足，可以直接删掉。

### 11.4 BCP：Boolean Constraint Propagation

在 DPLL 中，不断应用 unit resolution 的过程叫：

> **Boolean Constraint Propagation，布尔约束传播，简称 BCP。**

BCP 操作可以总结为：

1. 找到 unit clause $(\ell)$；
2. 令 $\ell=1$；
3. 删除所有包含 $\ell$ 的子句，因为它们已满足；
4. 从所有包含 $\neg\ell$ 的子句中删去 $\neg\ell$；
5. 若某个子句被删空，则产生 conflict；
6. 重复直到没有 unit clause。

### 11.5 BCP 例子

设 CNF 为：

$$
(x)\land(\neg x\lor y)\land(\neg y\lor z)
$$

第一步由 $(x)$ 得到：

$$
x=1
$$

删除包含 $x$ 的子句，第二个子句 $(\neg x\lor y)$ 中 $\neg x$ 为假，于是变成：

$$
(y)
$$

再由 $(y)$ 得到：

$$
y=1
$$

第三个子句 $(\neg y\lor z)$ 变成：

$$
(z)
$$

最终推出：

$$
z=1
$$

这就是传播链。

------

## 十二、DPLL 算法

### 12.1 DPLL 的基本思想

DPLL 把 SAT 求解分成两部分：

1. **Search（搜索）**：选择一个未赋值变量，尝试赋值 True 或 False；
2. **Deduction（推导）**：用 BCP 等规则自动推出一些变量的必要取值。

它不是盲目枚举所有赋值，而是在每次决策后尽可能传播约束，尽早发现冲突。

### 12.2 DPLL 的递归框架

输入公式必须是 CNF。

基本伪代码如下：

```text
DPLL(F):
    F = BCP(F)

    if F has no clauses:
        return SAT

    if F contains an empty clause:
        return UNSAT

    choose an unassigned variable p

    if DPLL(F with p = true) == SAT:
        return SAT
    else:
        return DPLL(F with p = false)
```

其中：

- `F has no clauses` 表示所有子句都已经满足，所以 SAT；
- `F contains an empty clause` 表示某个子句无法满足，所以 UNSAT；
- `choose_var()` 的策略会显著影响效率。

### 12.3 Pure Literal Propagation：纯文字传播

DPLL 的一个优化是 Pure Literal Propagation，简称 PLP。

若某个变量 $x$ 在当前公式中只以正文字出现，从未出现 $\neg x$，那么令：

$$
x=1
$$

不会让任何子句变坏，只会满足所有包含 $x$ 的子句。

类似地，若 $x$ 只以负文字出现，则令：

$$
x=0
$$

只以一种极性出现的文字叫 pure literal（纯文字）。PLP 保持可满足性，但所选赋值不一定是所有满足解都必须采用的取值。

注意：pure literal 是相对于**当前简化后的公式**而言的。随着赋值和删子句的进行，一个变量可能后来才变成 pure。

### 12.4 带 PLP 的 DPLL

伪代码可以写成：

```text
DPLL(F):
    F = BCP(F)
    F = PLP(F)

    if F has no clauses:
        return SAT

    if F contains an empty clause:
        return UNSAT

    choose an unassigned variable p

    if DPLL(F with p = true) == SAT:
        return SAT
    else:
        return DPLL(F with p = false)
```

PLP 不是必须的，但能减少搜索空间。

### 12.5 DPLL 搜索树怎么理解？

DPLL 的搜索树中：

- 每一层代表一次手动决策；
- 左右分支代表对某个变量尝试不同取值；
- BCP 推出来的赋值不一定增加搜索树深度；
- 若某条路径出现 conflict，则回溯；
- 若所有 clause 都满足，则找到 SAT assignment。

例如：

$$
F=(a\lor b)\land(\neg a\lor c)\land(\neg c)
$$

先执行 BCP：由 $(\neg c)$ 得到 $c=0$，再由 $(\neg a\lor c)$ 得到 $a=0$，最后由 $(a\lor b)$ 得到 $b=1$。因此这个例子无需分支就得到 SAT。

若要观察回溯，可以考虑：

$$
F=(\neg p\lor q\lor r)\land(\neg q\lor r)\land(\neg q\lor\neg r)\land(p\lor\neg q\lor\neg r)
$$

初始没有单位子句和纯文字。选择 $q=1$ 后，分别产生 $(r)$ 与 $(\neg r)$，BCP 检出冲突。回溯并选择 $q=0$ 后，公式化为 $(\neg p\lor r)$，例如令 $p=0,r=1$ 即可满足。

### 12.6 DPLL 的局限

DPLL 的主要问题是：

1. **不会学习冲突原因**。
   - 如果某个局部赋值组合已经导致 conflict，DPLL 回溯后可能在别的分支再次碰到同样组合。

2. **通常只能按时间顺序回溯一层**。
   - 也就是 chronological backtracking。
   - 如果真正导致冲突的是更早的某个决策，DPLL 不能直接跳回那里。

这正是 CDCL 要解决的问题。

------

## 十三、CDCL：Conflict-Driven Clause Learning

### 13.1 CDCL 的核心思想

CDCL 是现代 SAT solver 的核心框架。它基于 DPLL，但增加两个关键机制：

1. **Clause Learning：从冲突中学习新子句**；
2. **Non-chronological Backtracking：非时序回溯，也叫 backjumping**。

直观地说，DPLL 遇到冲突后只是“这条路不通，退一步”。CDCL 会问：

> 到底是哪几个赋值共同导致了这个冲突？以后能不能直接禁止这个组合？

禁止这个组合的方法就是加入一个 learned clause。

### 13.2 Value、Decision Level、Reason

变量赋值需要记录以下属性。

每个变量赋值要记录：

1. **Value**：变量取值是 0 还是 1；
2. **Decision level**：这个赋值发生在搜索树的哪一层；
3. **Reason**：这个赋值是手动决定的，还是由某个 clause 通过 BCP 推出来的。

例如：

- 在 level 1 手动决定 $a=0$，则 $a$ 的 reason 是 decision；
- 后来子句 $(a\lor b)$ 在 $a=0$ 后变成 unit clause $(b)$，于是推出 $b=1$，则 $b=1$ 的 reason 是 $(a\lor b)$。

这些赋值按发生顺序组成搜索的 trace。

### 13.3 Implication Graph：蕴含图

CDCL 用 implication graph 分析冲突。

注意这里的 implication graph 和 2-SAT 的 implication graph 不是同一个东西。

CDCL 的 implication graph 中：

- 节点表示一个被赋值的 literal，通常写作“变量取值 @ 决策层”；
- 手动决策变量是 decision node；
- BCP 推出的变量有 reason clause；
- 如果文字 $l$ 由子句 $C=(v_1\lor\cdots\lor v_k\lor l)$ 推出，则各个 $v_j$ 已为假。图中从为真的 $\neg v_j$ 节点连边到 $l$，记录它们共同导致的传播。单条边不表示该起点独自就能推出 $l$。

例如子句：

$$
(\neg a\lor \neg b\lor c)
$$

当：

$$
a=1,\quad b=1
$$

时，前两个 literal 为假，为了满足该子句，必须有：

$$
c=1
$$

于是 implication graph 中有边：

$$
a=1 \to c=1,
\quad
b=1 \to c=1
$$

### 13.4 Conflict Node

若某个 clause 中所有 literal 都被赋成 false，则产生 conflict。

例如 clause：

$$
(a\lor \neg b\lor c)
$$

若当前赋值为：

$$
a=0,\quad b=1,\quad c=0
$$

则三个 literal 都为假：

$$
a=0 \Rightarrow a \text{ false}
$$

$$
b=1 \Rightarrow \neg b \text{ false}
$$

$$
c=0 \Rightarrow c \text{ false}
$$

于是该 clause 被 falsify。implication graph 中会添加一个 conflict vertex，表示矛盾发生。

### 13.5 Conflict Cut 与 Reason Set

Conflict cut 用于划分冲突的原因和推导结果。

在 implication graph 中做一个 cut，把图分成两部分：

- 所有手动 decision 节点在 cut 的一侧；
- conflict vertex 在另一侧。

从 decision 侧跨到 conflict 侧的那些边，其起点对应的 literal 组成 reason set。

设 reason set 为：

$$
R=\{\ell_1,\ell_2,\ldots,\ell_k\}
$$

它表示：这些 literal 同时为真会导致冲突。

因此如果想避免同样冲突，就必须让其中至少一个 literal 为假。

### 13.6 Learned Clause 如何构造？

如果 reason set 是：

$$
R=\{\ell_1,\ell_2,\ldots,\ell_k\}
$$

说明：

$$
\ell_1\land\ell_2\land\cdots\land\ell_k
$$

这组赋值不能同时成立。

因此可以学习子句：

$$
\neg \ell_1\lor \neg \ell_2\lor\cdots\lor \neg \ell_k
$$

这个 clause 的意义是：

> 不允许 $\ell_1,\ell_2,\ldots,\ell_k$ 全部同时为真。

它会被加入原 CNF 中，帮助之后搜索避开同样的冲突。

### 13.7 一个小例子

假设当前冲突分析得到 reason set：

$$
R=\{a,\neg b,c\}
$$

说明：

$$
a=1,
\quad b=0,
\quad c=1
$$

三者同时出现会导致 conflict。

那么 learned clause 是：

$$
\neg a\lor b\lor \neg c
$$

加入该 clause 后，未来只要搜索再次试图同时令 $a=1,b=0,c=1$，这个 learned clause 会被完全赋假而产生冲突；若其中只有一个文字未赋值且其余都为假，则先通过 BCP 强制剩余文字为真。

### 13.8 Non-chronological Backtracking：非时序回溯

DPLL 遇到冲突一般退回上一层。但 CDCL 会根据 learned clause 直接跳回更早的层。

设 learned clause 为：

$$
C=(\ell_1\lor\ell_2\lor\cdots\lor\ell_k)
$$

每个 literal 所属变量都有一个 decision level。

如果 learned clause 中只有一个 literal 来自当前最高决策层，那么 CDCL 可以跳回到“第二高”的 decision level。回跳时撤销目标层以上的所有赋值，并更新当前决策层。此时 learned clause 只剩一个未赋值文字，BCP 将这个文字设为真，其对应变量的值与冲突时相反。若学习子句只有一个文字，则回到第 0 层。

这就是 backjumping。

直观理解：

- 冲突不是最后一步变量单独造成的，而是一组决策共同造成的；
- 学到 clause 后，不需要一步步退；
- 可以直接退到 learned clause 刚好能发挥作用的位置。

### 13.9 为什么要求 learned clause 在回溯后变成 unit？

如果回溯后 learned clause 是 unit，就能立刻通过 BCP 推出新事实。

例如 learned clause：

$$
(\neg a\lor \neg b\lor c)
$$

如果回溯后已有：

$$
a=1,
\quad b=1
$$

那么该子句变成：

$$
(c)
$$

于是强制：

$$
c=1
$$

这可以避免重新走向冲突。

如果 learned clause 回溯后不是 unit，就只能作为一个普通约束，不能立刻指导搜索。因此希望学习到回跳后能立即触发 BCP 的子句。

------

## 十四、UIP：Unique Implication Point

### 14.1 UIP 的定义

在 implication graph 中，UIP 是一个顶点，满足：

> 从最后一个 decision vertex 到 conflict vertex 的任意路径都必须经过它。

注意 UIP 不包括 conflict vertex 本身。

可以把 UIP 理解为这些路径的必经点。

### 14.2 UIP 的直觉

如果某个点是 UIP，说明在当前最后决策层中：

- 保持较低决策层赋值时，该 UIP 的当前取值会沿现有推导关系导致冲突；
- 它可以代表当前决策层导致冲突的一部分核心原因。

在保持较低决策层赋值的前提下：

- 在最后决策层，一个 UIP 可以“单独代表”通向冲突的所有路径；
- 非 UIP 的点往往需要和同层其他点一起才能导致冲突。

### 14.3 First UIP 与 Last UIP

通常会有多个 UIP。

- **First UIP**：离 conflict 最近的 UIP；
- **Last UIP**：最后一个 decision vertex 本身，离 conflict 最远。

现代 SAT solver 常用 **first-UIP cut**。

原因：

- first-UIP 是倾向于产生较短 learned clause 的启发式选择，不保证在所有可能的切割中最短；
- 得到的 learned clause 在相应回跳后成为单位子句，适合 backjumping；
- 实践中效果很好，例如 MiniSAT、zChaff 等 solver 都采用类似思想。

### 14.4 UIP cut 的性质

由 UIP 构造出的 conflict cut，有一个重要性质：

> reason set 中恰好包含一个来自当前最后决策层的 literal。

构造时，将 UIP 之后既可从 UIP 到达、又能到达冲突的节点放在冲突侧，UIP 留在原因侧。这样学到的 clause 在回跳到其余文字的最高决策层后成为 unit clause，从而触发 BCP。

这也是 UIP 与非时序回溯紧密相关的原因。

------

## 十五、CDCL 完整算法框架

CDCL 的伪代码如下：

```text
CDCL(F):
    assignment = empty
    decision_level = 0

    while true:
        BCP(F, assignment)

        if some clause is falsified:
            if decision_level == 0:
                return UNSAT

            learned_clause, backjump_level = conflict_analysis()
            add learned_clause to F
            backjump to backjump_level

        else:
            if all variables are assigned:
                return SAT

            decision_level += 1
            choose an unassigned literal l
            assign l = true as a decision
```

其中：

- BCP 负责传播；
- conflict_analysis 负责用 implication graph 学习新 clause；
- backjump 根据 learned clause 的 decision level 计算；
- choose literal 的策略也很重要。

### 15.1 CDCL 相比 DPLL 的优势

CDCL 的优势可以概括为：

1. **会学习**：每次冲突都会产生 learned clause，避免重复犯同样错误；
2. **会跳跃回溯**：不一定只退一层，可以直接退到冲突原因相关的层；
3. **可以重启**：restart 后保留 learned clauses，换一种搜索路径继续。

### 15.2 SAT solver 常用工程技术

相关优化包括：

- 高效 BCP 数据结构；
- adjacency list；
- lazy structures；
- decision heuristics；
- forgetting learned clauses；
- learned clause minimization；
- restarts。

其中最重要的几个是：

**Clause deletion**：learned clause 太多会拖慢 solver，所以需要删除低质量 learned clauses。

**Restart**：定期撤销当前决策及其传播赋值，但保留 learned clauses，相当于带经验地重新开始。

------

## 十六、2-SAT：SAT 的线性可解特殊情形

### 16.1 什么是 2-SAT？

2-SAT 是 SAT 的特殊情况：CNF 中每个 clause 包含两个 literal。单位子句 $(a)$ 可写为 $(a\lor a)$，从而一并处理；空子句则直接判为 UNSAT。

例如：

$$
(x\lor y)\land(\neg x\lor z)\land(\neg y\lor \neg z)
$$

是 2-SAT。

但：

$$
(x\lor y\lor z)
$$

不是 2-SAT，因为这个 clause 有三个 literal。

一般 SAT 是 NP-complete，但 2-SAT 可以在线性时间内解决。

### 16.2 2-SAT 的关键转化：子句变蕴含

一个二元子句：

$$
(a\lor b)
$$

等价于两个蕴含：

$$
\neg a \to b
$$

$$
\neg b \to a
$$

这是 2-SAT 的核心。

为什么？因为如果 $a$ 是假，为了让 $a\lor b$ 为真， $b$ 必须为真；反过来，如果 $b$ 是假， $a$ 必须为真。

### 16.3 Implication Graph

对每个变量 $x_i$，建立两个节点：

$$
x_i,
\quad
\neg x_i
$$

对于每个 clause $(a\lor b)$，加入两条有向边：

$$
\neg a \to b
$$

$$
\neg b \to a
$$

这个图叫 2-SAT 的 implication graph。

注意：

> 2-SAT 的 implication graph 是公式本身的图模型；CDCL 的 implication graph 是某次搜索过程中赋值传播的图。两者不是同一个概念。

### 16.4 可达性的含义

若 implication graph 中有路径：

$$
a \leadsto b
$$

表示：

$$
a \Rightarrow b
$$

也就是说，如果 $a$ 为真，则 $b$ 也必须为真。

若同时有：

$$
x \leadsto \neg x
$$

说明令 $x=1$ 会推出 $x=0$，因此 $x=1$ 不可行。

但这不一定说明公式 UNSAT，因为也许可以令 $x=0$。

真正不可满足的条件是：

$$
x \leadsto \neg x
\quad \text{and} \quad
\neg x \leadsto x
$$

这等价于 $x$ 和 $\neg x$ 在同一个强连通分量中。

### 16.5 2-SAT 的 UNSAT 判据

2-SAT 公式 UNSAT 当且仅当存在某个变量 $x$，使得：

$$
x
\quad \text{和} \quad
\neg x
$$

在 implication graph 的同一个 SCC 中。

原因：

- 如果 $x$ 和 $\neg x$ 在同一个 SCC，则 $x\Rightarrow \neg x$，且 $\neg x\Rightarrow x$；
- 无论给 $x$ 赋 0 还是 1，都会推出相反值；
- 因此没有合法赋值。

### 16.6 对称性性质

蕴含图具有以下对称性：

如果：

$$
a \leadsto b
$$

那么：

$$
\neg b \leadsto \neg a
$$

因为每条边都来自某个 2-SAT 子句 $(u\lor v)$，边 $\neg u\to v$ 必然伴随边 $\neg v\to u$。

推广到路径也成立：路径反向并取反后仍然存在。

这说明 implication graph 的结构具有“取反对偶”关系。

### 16.7 SCC 缩点与求解赋值

找到 SCC 后，可以把每个 SCC 缩成一个点，得到 DAG，叫 kernel graph 或 condensation graph。

先确认每个变量与其否定不在同一个 SCC 中，再构造解：

1. 找出出度为 0 的 SCC；
2. 将这个 SCC 中所有 literal 赋为 True；
3. 它们的对偶 SCC 赋为 False；
4. 删除这两个 SCC，重复。

直觉是：

- 出度为 0 的 SCC 赋 True 不会强迫图中其他 SCC 必须为 True；
- 根据对称性，它的对偶 SCC 入度为 0，可以安全赋 False。

常见编程实现中，也可以用 SCC 的拓扑序直接赋值。若用 Kosaraju 算法，常见写法是：

$$
x_i = \big(\mathrm{comp}(x_i) > \mathrm{comp}(\neg x_i)\big)
$$

但这个大于号方向依赖于具体 SCC 编号顺序，写代码时需要和算法实现对应，不能死记。

### 16.8 2-SAT 算法复杂度

设变量数为 $n$，子句数为 $m$。

implication graph 有：

$$
2n
$$

个点，每个子句产生 2 条边，所以有：

$$
2m
$$

条边。

Tarjan 或 Kosaraju 求 SCC 的复杂度为：

$$
O(n+m)
$$

因此 2-SAT 可以线性时间解决。

### 16.9 2-SAT C++ 模板

下面用 C++ 实现 SCC 求解过程。传入的变量编号须有效，约束通过 `add_clause` 添加，`add_implication` 只负责添加一条图边。返回 `false` 时没有有效赋值。递归 DFS 在很深的图上可能超出调用栈容量。变量编号用 $0\sim n-1$。`id(i, false)` 表示 $x_i$，`id(i, true)` 表示 $\neg x_i$。

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

struct TwoSAT {
    int n;
    vector<vector<int>> g, rg;
    vector<int> comp, order, vis;

    TwoSAT(int n_) : n(n_), g(2 * n_), rg(2 * n_), comp(2 * n_, -1), vis(2 * n_, 0) {}

    int id(int x, bool neg) {
        return 2 * x + (neg ? 1 : 0);
    }

    int neg_id(int v) {
        return v ^ 1;
    }

    void add_implication(int u, int v) {
        g[u].push_back(v);
        rg[v].push_back(u);
    }

    // 添加子句 (a or b)
    // a = x if a_neg=false, a = not x if a_neg=true
    void add_clause(int a, bool a_neg, int b, bool b_neg) {
        int A = id(a, a_neg);
        int B = id(b, b_neg);
        add_implication(neg_id(A), B);
        add_implication(neg_id(B), A);
    }

    void dfs1(int u) {
        vis[u] = 1;
        for (int v : g[u]) {
            if (!vis[v]) dfs1(v);
        }
        order.push_back(u);
    }

    void dfs2(int u, int c) {
        comp[u] = c;
        for (int v : rg[u]) {
            if (comp[v] == -1) dfs2(v, c);
        }
    }

    bool satisfiable(vector<int>& assignment) {
        fill(vis.begin(), vis.end(), 0);
        fill(comp.begin(), comp.end(), -1);
        order.clear();
        assignment.clear();
        for (int i = 0; i < 2 * n; ++i) {
            if (!vis[i]) dfs1(i);
        }

        reverse(order.begin(), order.end());

        int c = 0;
        for (int u : order) {
            if (comp[u] == -1) dfs2(u, c++);
        }

        assignment.assign(n, 0);
        for (int i = 0; i < n; ++i) {
            if (comp[2 * i] == comp[2 * i + 1]) {
                assignment.clear();
                return false;
            }
            assignment[i] = (comp[2 * i] > comp[2 * i + 1]);
        }
        return true;
    }
};
```

使用时，例如添加：

$$
(x_0 \lor \neg x_1)
$$

写成：

```cpp
solver.add_clause(0, false, 1, true);
```

------

## 十七、SAT 应用：组合电路等价性检查

### 17.1 什么是组合电路？

组合电路（combinational circuit）满足：

- 没有状态保持元件；
- 没有寄存器、锁存器等 memory；
- 没有反馈环；
- 输出完全由当前输入决定。

也就是说，它实现的是一个布尔函数：

$$
y = f(x_1,x_2,\ldots,x_n)
$$

### 17.2 问题：两个电路是否等价？

给定两个组合电路：

- Reference Design；
- Implementation Design。

它们输入相同，但内部结构可能不同。我们要检查：对任意输入，两者输出是否都相同。

若单输出，则要求：

$$
\forall x,
\quad
f(x)=g(x)
$$

若多输出，则要求每一位输出都相等。

### 17.3 方法一：用 ROBDD 比较

可以分别把两个电路函数构造成 ROBDD。

若在相同变量顺序下，两个函数的 ROBDD 完全相同，则两电路等价。

优点：

- ROBDD 是 canonical representation；
- 一旦构造出来，等价性比较很直接。

缺点：

- ROBDD 可能因为变量顺序不好而指数爆炸。

### 17.4 方法二：Miter Circuit + SAT

另一种方法是构造 miter circuit。

对单输出电路，构造：

$$
M(x)=f(x)\oplus g(x)
$$

然后问：

$$
M(x)=1
$$

是否可满足。

- 如果 $M$ SAT，说明存在一个输入让两电路输出不同，这个输入就是 counterexample；
- 如果 $M$ UNSAT，说明不存在任何输入让输出不同，所以两个电路等价。

因此：

$$
M \text{ is UNSAT}
\quad \Longleftrightarrow \quad
f \equiv g
$$

### 17.5 多输出 Miter

如果电路有多个输出：

$$
y_1,y_2,\ldots,y_k
$$

reference 输出为：

$$
r_1,r_2,\ldots,r_k
$$

implementation 输出为：

$$
i_1,i_2,\ldots,i_k
$$

则 miter 输出一般写为：

$$
M=(r_1\oplus i_1)
\lor
(r_2\oplus i_2)
\lor
\cdots
\lor
(r_k\oplus i_k)
$$

只要有任意一位输出不同， $M=1$。

所以：

$$
M \text{ UNSAT}
\quad \Longleftrightarrow \quad
\text{两个多输出电路完全等价}
$$

### 17.6 SAT 在等价性检查中的流程

完整流程是：

1. 把 reference design 和 implementation design 接到相同输入上；
2. 对对应输出做 XOR；
3. 把所有 XOR 结果 OR 起来得到 miter 输出；
4. 用 Tseitin Transformation 把 miter 电路编码成 CNF；
5. 强制 miter 输出为 1；
6. 调用 SAT solver。

结果解释：

- SAT：不等价，并且 solver 给出的 satisfying assignment 就是反例输入；
- UNSAT：等价。

------
