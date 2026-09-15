# Lec14_First-order Logic and SMT(Part 2)_Notes

> AI 整理。

------

## 一、一阶理论

### 1.本讲总体框架

本讲接在 lec13 之后。lec13 主要讲了：

- FOL（一阶逻辑）为什么比命题逻辑表达能力更强；
- 一阶逻辑的语法：变量、常量、函数、谓词、逻辑连接词、量词；
- 一阶逻辑的语义：domain、structure、interpretation、assignment；
- FOL 中的 satisfiable、valid、unsat、refutable；
- 一般 FOL 不可判定，但某些一阶理论或片段可判定。

lec14 进入 SMT 的关键部分：**一阶理论（First-Order Theory）**。

本讲可以按下面几条线索理解：

1. **一阶理论是什么**：
   
   一阶理论 = 签名（signature） + 公理（axioms）。

2. **为什么需要理论**：
   
   普通 FOL 里，函数符号和谓词符号的含义可以任意解释；而 SMT 需要让某些符号有固定语义，例如 `=` 表示真正的相等，`+` 表示整数加法，`select/store` 表示数组读写，`xor` 表示位向量按位异或。

3. **什么是 satisfiability modulo theory**：
   
   判断公式是否可满足时，不再考虑所有任意结构，而只考虑满足该理论公理的结构。

4. **常见理论**：
   
   - $T_E$：Equality / Equality with Uninterpreted Functions，等式理论 / 未解释函数等式理论；
   - $T_{PA}$：Peano Arithmetic，皮亚诺算术；
   - $T_{\mathbb{N}}$：Presburger Arithmetic，自然数上的加法理论；
   - $T_{\mathbb{Z}}$：Linear Integer Arithmetic，线性整数算术；
   - $T_A$：Array theory，数组理论；
   - $T_{BV}$：Bit-vector theory，位向量理论。

5. **重点算法**：
   
   量词自由等式理论（QF Equality / QF EUF）中的 satisfiability 判断，核心是：
   
   - 用等式合并等价类；
   - 用 congruence（一致性）推出更多等式；
   - 最后检查不等式是否与等价类冲突。

6. **实际案例**：
   
   RTL Repair：把 RTL 代码修复问题编码成 SMT，通过 SMT 求解器找出满足 testbench 的修复方案。

------

### 2.一阶理论的动机

#### 2.1 FOL 中符号的含义本来是任意的

在普通一阶逻辑中，语法只规定哪些公式“长得合法”。例如有函数符号 $f$、谓词符号 $P$，可以写：

$$
P(f(x), y)
$$

但是 $f$ 到底是什么意思？$P$ 到底是什么意思？这取决于 structure / interpretation。

也就是说，在 FOL 语义里：

- domain $D$ 可以任意选择；
- 常量符号可以解释成 $D$ 中任意元素；
- 函数符号可以解释成任意函数；
- 谓词符号可以解释成任意布尔关系。

这在纯逻辑理论上很灵活，但在程序验证、硬件验证中不够用。

例如，看到公式：

$$
x + y = y + x
$$

如果 `+` 只是一个普通二元函数符号，那么它不一定满足交换律。普通 FOL 不会自动知道 `+` 是整数加法。

同理，看到：

$$
select(store(A, i, d), i) = d
$$

普通 FOL 也不会自动知道这是数组“写完再读同一位置”的语义。

因此 SMT 引入 **theory（理论）**，用高层次方式固定符号含义。

------

### 3.First-Order Theory：定义

#### 3.1 理论 = 签名 + 公理

一阶理论 $T$ 由两部分组成：

$$
T = \text{Signature} + \text{Axioms}
$$

更正式地写：

$$
T = (\Sigma_T, A_T)
$$

其中：

- $\Sigma_T$ 是理论的 **signature（签名）**；
- $A_T$ 是理论的 **axioms（公理集合）**。

#### 3.2 Signature：决定语法

签名 $\Sigma_T$ 包含理论中允许使用的 **非逻辑符号**：

- constant symbols：常量符号；
- function symbols：函数符号；
- predicate symbols：谓词符号。

它决定哪些公式是合法的。

例如，Peano arithmetic 的签名里如果只有：

$$
\Sigma_{PA}: \{0, 1, \ldots, +, \times, =\}
$$

那么公式：

$$
f(x) = g(x)
$$

不是 $\Sigma_{PA}$-formula，因为 $f$ 和 $g$ 不在签名里。

#### 3.3 Axioms：决定语义

公理 $A_T$ 是一组没有自由变量的公式，用来限制哪些 structure 是合法的。

直观理解：

- signature 规定“能写什么”；
- axioms 规定“符号必须满足什么性质”。

例如，在等式理论里，`=` 不能被随便解释。它必须满足：

- 自反性：每个元素等于自己；
- 对称性：如果 $x=y$，则 $y=x$；
- 传递性：如果 $x=y$ 且 $y=z$，则 $x=z$；
- 一致性：相等的参数代入同一个函数后结果仍相等。

------

## 二、等式理论与理论模型

### 1.Theory of Equality：等式理论

#### 1.1 等式理论的签名

等式理论，也常叫 **Equality with Uninterpreted Functions（EUF）**，其签名包含：

$$
\Sigma_T : \{a,b,c,\ldots, f,g,h,\ldots, P,Q,R,\ldots, =\}
$$

其中：

- $a,b,c,\ldots$ 是常量符号；
- $f,g,h,\ldots$ 是函数符号；
- $P,Q,R,\ldots$ 是谓词符号；
- $=$ 是等式谓词。

这里的函数符号叫 **uninterpreted function（未解释函数）**，意思是：

- 函数具体做什么不固定；
- 但函数必须满足“相等输入给出相等输出”的一致性。

例如，$f$ 不一定是平方函数、加法函数或某种具体函数，但如果 $x=y$，那么一定要有：

$$
f(x)=f(y)
$$

这就是 EUF 的核心。

------

### 2.等式理论的基本公理

#### 2.1 Reflexivity：自反性

$$
\forall x.\ x=x
$$

含义：任何元素都等于自身。

#### 2.2 Symmetry：对称性

$$
\forall x,y.\ x=y \rightarrow y=x
$$

含义：如果 $x$ 等于 $y$，那么 $y$ 也等于 $x$。

#### 2.3 Transitivity：传递性

$$
\forall x,y,z.\ (x=y \land y=z) \rightarrow x=z
$$

含义：如果 $x$ 等于 $y$，$y$ 等于 $z$，那么 $x$ 等于 $z$。

这三个性质说明 `=` 至少是一个 **equivalence relation（等价关系）**。

------

### 3.公理如何限制合法结构

#### 3.1 structure 不再任意

本节约定：先把 `=` 作为待解释的谓词符号，再用等式理论公理限制其解释。因此，本节讨论的等价类划分应放在这一课程语境下理解。

例如 domain：

$$
D=\{1,2,3\}
$$

如果把 `=` 解释成：

$$
\{(1,1),(2,2)\}
$$

那么 $3=3$ 为假，违反自反性，所以这个解释不是等式理论的合法 structure。

如果把 `=` 解释成一个满足自反、对称、传递的关系，它才可能成为等式理论的合法解释。

#### 3.2 等式关系可以看成等价类划分

满足自反、对称、传递的等式关系可以用 **equivalence classes（等价类）** 表示；要构成整个等式理论的模型，还须检查后面的函数和谓词一致性。

例如 domain：

$$
D=\{1,2,3,4,5\}
$$

一种等式解释可以对应划分：

$$
\{\{1,3\},\{2,4,5\}\}
$$

这表示：

- $1=3$ 为真；
- $2=4$、$2=5$、$4=5$ 为真；
- $1=2$、$3=5$ 等跨等价类关系为假。

等价类划分自动满足：

1. 自反性：每个元素都在某个等价类里，所以 $x=x$；
2. 对称性：如果 $x,y$ 同类，则 $y,x$ 也同类；
3. 传递性：如果 $x,y$ 同类，$y,z$ 同类，则 $x,z$ 同类。

------

### 4.Congruence：一致性公理

仅有自反、对称、传递还不够。等式理论还需要 **congruence（一致性）**。

直观说：

#### 4.1 Function Congruence：函数一致性

对于 $n$ 元函数 $f$：

$$
\forall x_1,\ldots,x_n,y_1,\ldots,y_n.
(x_1=y_1 \land \cdots \land x_n=y_n)
\rightarrow
f(x_1,\ldots,x_n)=f(y_1,\ldots,y_n)
$$

简写成向量形式：

$$
\forall \vec{x},\vec{y}.\ \vec{x}=\vec{y} \rightarrow f(\vec{x})=f(\vec{y})
$$

例子：如果 $a=b$，那么：

$$
f(a)=f(b)
$$

如果 $a=b$ 且 $c=d$，那么：

$$
g(a,c)=g(b,d)
$$

#### 4.2 Predicate Congruence：谓词一致性

对于 $n$ 元谓词 $P$：

$$
\forall x_1,\ldots,x_n,y_1,\ldots,y_n.
(x_1=y_1 \land \cdots \land x_n=y_n)
\rightarrow
(P(x_1,\ldots,x_n) \leftrightarrow P(y_1,\ldots,y_n))
$$

简写成：

$$
\forall \vec{x},\vec{y}.\ \vec{x}=\vec{y} \rightarrow (P(\vec{x}) \leftrightarrow P(\vec{y}))
$$

含义：相同参数下同一个谓词真假值必须相同。

------

### 5.T-model：理论模型

#### 5.1 定义

如果一个 structure $M$ 满足理论 $T$ 的所有公理，也就是：

$$
M \models A
\quad \text{for every } A\in A_T
$$

那么称 $M$ 是 $T$ 的一个模型，记作 **$T$-model**。

#### 5.2 例子：满足等式理论的模型

假设 domain：

$$
D=\{1,2,3\}
$$

等式解释成等价类：

$$
\{\{1,2\},\{3\}\}
$$

如果函数 $f$ 的解释满足：

- 当输入在同一个等价类时，输出也在同一个等价类；
- 不出现 $x=y$ 但 $f(x)\ne f(y)$ 的情况；

那么这个 structure 就可能是等式理论的模型。

如果出现：

$$
x=y \quad \text{but} \quad f(x)\ne f(y)
$$

则违反 function congruence，不是等式理论模型。

------

### 6.Satisfiability / Validity Modulo Theory

#### 6.1 $T$-satisfiable

一个公式 $F$ 是 **$T$-satisfiable**，意思是：

存在一个 $T$-model $M$ 和一个 assignment $s$，使得：

$$
M,s\models F
$$

也就是说，公式在某个满足理论公理的模型中为真。

#### 6.2 $T$-valid

一个公式 $F$ 是 **$T$-valid**，意思是：

对于所有 $T$-model $M$ 和所有 assignment $s$，都有：

$$
M,s\models F
$$

记作：

$$
T\models F
$$

注意：$T$-valid 应理解为“valid modulo theory $T$”，不是 satisfiable。

#### 6.3 和普通 FOL satisfiability 的区别

普通 FOL satisfiable：

- 可以使用任意 structure；
- 只要某个 structure 让公式为真就行。

$T$-satisfiable：

- 只允许使用满足理论公理的 structure；
- 不满足公理的 structure 被直接排除。

因此 SMT 的 “modulo theory” 可以理解为：

------

## 三、常见一阶理论

### 1.常见一阶理论总览

本讲主要介绍四类常用理论：

1. Equality with Uninterpreted Functions：等式与未解释函数；
2. Integers：整数相关理论；
3. Array：数组理论；
4. Bit-vector：位向量理论。

这些理论可以看成 SMT 中常见的“专用语义模块”。

程序验证 / 硬件验证里经常会混合使用这些理论，例如：

- 控制条件：布尔逻辑；
- 寄存器和信号：位向量；
- 数组或 memory：数组理论；
- 程序变量赋值关系：等式理论；
- 循环展开后的状态转移：等式 + `ite` + 位向量。

------

### 2.Equality with Uninterpreted Functions：EUF

#### 2.1 EUF 适合表达什么

EUF 适合表达：

- 函数调用结果之间的相等关系；
- 数据流中的赋值关系；
- 对具体函数内部语义不关心，只关心“同输入同输出”的情况。

例如把平方运算 $x*x$ 抽象成一个未解释函数：

$$
sq(x)
$$

此时不需要知道平方的具体算术性质，只需要知道：

$$
x=y \rightarrow sq(x)=sq(y)
$$

这可以极大减少求解难度。

#### 2.2 例子：函数等价性检查

**教学示例约定：**未初始化局部变量在符号模型中表示任意初值；这些代码不是可直接运行的 C/C++ 示例。LIA 示例按数学整数解释，位向量示例按固定宽度解释。

考虑两个程序：

```c
int fun1(int y) {
    int x, z;
    z = y;
    y = x;
    x = z;
    return x*x;
}

int fun2(int y) {
    return y*y;
}
```

如果把 $x*x$ 看成 $sq(x)$，可以写出“两个函数不同”的 SMT 公式：

$$
(z_1 = y_0 \land y_1 = x_0 \land x_1 = z_1 \land r_1 = sq(x_1))
\land (r_2 = sq(y_0))
\land \neg(r_1 = r_2)
$$

含义：

- $z_1=y_0$ 表示 `z = y`；
- $y_1=x_0$ 表示 `y = x`；
- $x_1=z_1$ 表示 `x = z`；
- $r_1=sq(x_1)$ 表示 `fun1` 的返回值；
- $r_2=sq(y_0)$ 表示 `fun2` 的返回值；
- $\neg(r_1=r_2)$ 表示希望找到一个反例，让返回值不同。

由于 $z_1=y_0$ 且 $x_1=z_1$，可推出：

$$
x_1=y_0
$$

由函数一致性可推出：

$$
sq(x_1)=sq(y_0)
$$

也就是：

$$
r_1=r_2
$$

这与 $\neg(r_1=r_2)$ 冲突，因此公式 UNSAT，说明两个函数等价。

------

### 3.整数相关理论

下面区分三种整数/自然数理论。

#### 3.1 Peano Arithmetic：$T_{PA}$

Peano arithmetic 通常在自然数上讨论，并允许：

- 加法 $+$；
- 乘法 $\times$；
- 等式 $=$。

签名为：

$$
\Sigma_{PA}: \{0,1,\ldots,+,\times,=\}
$$

它的表达能力强，但是不可判定。

也就是说，不存在一个通用算法可以对任意 $T_{PA}$ 公式总是停机并判断有效性/可满足性。

#### 3.2 Peano 算术里不等式的表达

如果签名中没有 $<,\le,>,\ge$，也可以用 $+$ 和量词表达。

在自然数上：

$$
x < y
$$

可以表达为：

$$
\exists w.\ w\ne 0 \land x+w=y
$$

也可以从另一个角度表达为：

$$
\forall w.\ \neg(y+w=x)
$$

因为在自然数里，如果 $x<y$，那么不存在自然数 $w$ 使得 $y+w=x$。

同理：

$$
x\le y
$$

可以表达为：

$$
\exists w.\ x+w=y
$$

这里 $w$ 可以为 0。

#### 3.3 Presburger Arithmetic：$T_{\mathbb{N}}$

Presburger arithmetic 是自然数上的加法理论：

- 允许 $+$；
- 不允许两个变量之间的乘法。

它比 Peano arithmetic 弱，但有一个重要优点：**可判定**。

也就是说，Presburger arithmetic 中任意公式的有效性/可满足性理论上可以由算法判定。

#### 3.4 Linear Integer Arithmetic：$T_{\mathbb{Z}}$

Theory of integers $T_{\mathbb{Z}}$ 是整数上的线性整数算术，也叫 LIA。

签名可理解为：

$$
\Sigma_{\mathbb{Z}}:
\{\ldots,-2,-1,0,1,2,\ldots\}
\cup
\{\ldots,-3\times,-2\times,2\times,3\times,\ldots\}
\cup
\{+,-,=,<\}
$$

它允许：

- 整数常量；
- 加法；
- 减法；
- 比较；
- 常数乘法，例如 $3x$、$-2x$。

但不允许一般的变量乘变量，例如：

$$
x\times y
$$

因为这属于非线性整数算术，难度会显著上升。

#### 3.5 LIA 与 Presburger 的关系


- $T_{\mathbb{Z}}$ 和 Presburger arithmetic 在表达能力上等价；
- 但 $T_{\mathbb{Z}}$ 提供了更方便的符号；
- 可以直接写减法、不等式、常数乘法。

#### 3.6 LIA 等价性检查例子

考虑：

```c
int fun1(int y) {
    int x, z;
    x = 2*y + 1;
    z = 2*x + 1;
    return y + z;
}

int fun2(int y) {
    return 5*y + 2;
}
```

用 $T_{\mathbb{Z}}$ 可以写“两个函数不同”的公式：

$$
(x_1 = 2y_0 + 1 \land z_1 = 2x_1 + 1 \land r_1 = y_0 + z_1)
\land (r_2 = 5y_0 + 2)
\land \neg(r_1 = r_2)
$$

人工化简：

$$
x_1=2y_0+1
$$

$$
z_1=2x_1+1=2(2y_0+1)+1=4y_0+3
$$

$$
r_1=y_0+z_1=y_0+4y_0+3=5y_0+3
$$

而：

$$
r_2=5y_0+2
$$

所以实际上：

$$
r_1\ne r_2
$$

对所有整数 $y_0$ 都成立。于是“两个函数不同”的公式是 SAT，说明两个函数确实不等价。

------

### 4.Array Theory：数组理论 $T_A$

#### 4.1 数组理论的签名

数组理论的主要符号：

$$
\Sigma_A : \{select, store, =\}
$$

其中：

- $select(A,i)$ 表示读取数组 $A$ 的第 $i$ 个位置，也就是 $A[i]$；
- $store(A,i,d)$ 表示把数组 $A$ 的第 $i$ 个位置写成 $d$ 后得到的新数组。

注意：

$$
store(A,i,d)
$$

不是“修改原数组”的命令式操作，而是一个函数表达式，返回写入后的数组。

#### 4.2 数组理论公理

##### 公理 1：相同下标读出来相同

$$
\forall A,i,j.\ i=j \rightarrow select(A,i)=select(A,j)
$$

这条可以看作 `select` 对下标相等的 congruence。

##### 公理 2：写后读同一位置

$$
\forall A,i,d.\ select(store(A,i,d),i)=d
$$

含义：把 $d$ 写入 $A[i]$ 后，再读 $i$ 位置，一定得到 $d$。

##### 公理 3：写后读不同位置

$$
\forall A,d,i,j.\ i\ne j \rightarrow select(store(A,i,d),j)=select(A,j)
$$

含义：写 $i$ 位置不会影响 $j$ 位置。

这两条就是经典的 array read-over-write axioms。

#### 4.3 数组等价性检查例子

考虑：

```c
int fun1(int y) {
    int x[2];
    x[0] = y;
    y = x[1];
    x[1] = x[0];
    return x[1]*x[1];
}

int fun2(int y) {
    return y*y;
}
```

把平方抽象成 $sq$，数组操作用 $select/store$ 表示。

可以写成：

$$
x_1 = store(x_0,0,y_0)
$$

$$
y_1 = select(x_1,1)
$$

$$
x_2 = store(x_1,1,select(x_1,0))
$$

$$
r_1 = sq(select(x_2,1))
$$

$$
r_2 = sq(y_0)
$$

再加上“不等价条件”：

$$
\neg(r_1=r_2)
$$

关键推理：

1. $x_1=store(x_0,0,y_0)$，所以

   $$
   select(x_1,0)=y_0
   $$

2. $x_2=store(x_1,1,select(x_1,0))$，所以

   $$
   select(x_2,1)=select(x_1,0)
   $$

3. 因此：

   $$
   select(x_2,1)=y_0
   $$

4. 由函数一致性：

   $$
   sq(select(x_2,1))=sq(y_0)
   $$

5. 即：

   $$
   r_1=r_2
   $$

与 $\neg(r_1=r_2)$ 冲突，因此“不等价公式”UNSAT，说明两个函数等价。

------

### 5.Bit-vector Theory：位向量理论 $T_{BV}$

#### 5.1 位向量理论的 domain

位向量理论的 domain 是固定宽度的 bit vector。

例如 8-bit vector：

$$
00000000, 00000001, \ldots, 11111111
$$

它不同于数学整数：

- 位宽固定；
- 加法可能溢出；
- 运算按机器语义执行；
- 有有符号/无符号比较的区别。

#### 5.2 位向量操作类型

常见操作包括以下几类。

##### 1. String-like operations

类似字符串操作：

- `concat`：拼接；
- `extract`：截取某些 bit。

例如：

$$
concat(a,b)
$$

表示把两个 bit-vector 拼接起来。

$$
a[3:0]
$$

表示截取低 4 位。

##### 2. Logical operations

按位逻辑：

- bitwise not；
- bitwise or；
- bitwise and；
- bitwise xor。

例如：

$$
a \oplus b
$$

表示按位异或。

##### 3. Arithmetic operations

算术：

- add；
- subtract；
- multiply。

注意这些通常是模 $2^w$ 的机器整数运算，其中 $w$ 是位宽。

##### 4. Comparison operations

比较：

- $<$；
- $>$；
- signed less-than；
- unsigned less-than。

#### 5.3 位向量公式例子

一个 8-bit 向量例子如下：

$$
a[1:0] \ne b[1:0]
\land (a|b)=c
\land a=c-b
\land a<c
\land a[1]\oplus b[1]=0
$$

这里同时用到了：

- bit extraction：$a[1:0]$；
- bitwise or：$a|b$；
- arithmetic subtraction：$c-b$；
- comparison：$a<c$；
- xor：$a[1]\oplus b[1]$。

#### 5.4 XOR 交换例子

考虑抽象的 XOR swap：

```c
int fun1(int y) {
    int x;
    x = x ^ y;
    y = x ^ y;
    x = x ^ y;
    return x;
}

int fun2(int y) {
    return y;
}
```

如果使用 EUF，无法理解 XOR 的位级语义。因此需要 $T_{BV}$。

用位向量理论可以写：

$$
x_1 = x_0 \oplus y_0
$$

$$
y_1 = x_1 \oplus y_0
$$

$$
x_2 = x_1 \oplus y_1
$$

$$
r_1=x_2
$$

$$
r_2=y_0
$$

再加上“不等价”：

$$
\neg(r_2=r_1)
$$

位向量求解器可以使用 XOR 的理论语义推出：

$$
x_2=y_0
$$

所以 $r_1=r_2$，不等价公式 UNSAT。

------

### 6.常见理论的可判定性总结

总结如下。这里的 QFF 表示 **Quantifier-Free Fragment（量词自由片段）**。

| 理论 | 名称 | 带量词整体是否可判定 | QFF 是否可判定 |
|---|---|---:|---:|
| $T_E$ | Equality | No | Yes |
| $T_{PA}$ | Peano Arithmetic | No | No |
| $T_{\mathbb{N}}$ | Presburger Arithmetic | Yes | Yes |
| $T_{\mathbb{Z}}$ | Linear Integer Arithmetic | Yes | Yes |
| $T_{\mathbb{R}}$ | Real Arithmetic | Yes | Yes |
| $T_{\mathbb{Q}}$ | Linear Rationals | Yes | Yes |
| $T_A$ | Arrays | No | Yes |
| $T_{BV}$ | Bit-vectors | Yes | Yes |

需要重点记住：

1. **Peano arithmetic 不可判定**。
2. **Presburger arithmetic 可判定**，因为只有加法，没有变量乘变量。
3. **LIA 可判定**，SMT 求解器很常用。
4. **数组理论带量词整体不可判定，但量词自由片段可判定**。
5. **bit-vector 理论可判定**，因为固定宽度下状态空间有限；实际求解中常结合位级推理、bit-blasting 或专用算法。
6. **EUF 的量词自由片段可判定**，其核心算法就是后面讲的 congruence closure。

------

## 四、等式合取的可满足性

### 1.Case Study：量词自由等式理论 satisfiability

本部分是本讲最重要的算法内容。

#### 1.1 问题范围

我们考虑 **theory of equality** 的量词自由片段。

为简化问题，假设：

- 唯一谓词是 `=`；
- 其他谓词都可以改写为函数形式。

例如一个谓词 $P(x)$ 可以改写为：

$$
P(x)=\top \quad \text{iff} \quad f_P(x)=t
$$

其中 $t$ 是一个特殊值。

同理：

$$
P(x)=\bot \quad \text{iff} \quad f_P(x)\ne t
$$

本例还额外限制公式为等式和不等式 literal 的**合取**。去掉量词或改写谓词，并不会自动消除析取等布尔结构；后面的算法不直接处理任意量词自由公式。

------

### 2.等式理论判定问题形式

目标：判断下面形式的公式在等式理论中是否可满足。

$$
F:
 s_1=t_1 \land \cdots \land s_m=t_m
 \land s_{m+1}\ne t_{m+1}\land \cdots \land s_n\ne t_n
$$

其中 $s_i,t_i$ 都是 term。

可以把公式分成两部分：

1. 等式集合：

   $$
   E=\{s_1=t_1,\ldots,s_m=t_m\}
   $$

2. 不等式集合：

   $$
   N=\{s_{m+1}\ne t_{m+1},\ldots,s_n\ne t_n\}
   $$

问题：

- 根据等式 $E$ 和 congruence 公理，可以推出哪些 term 必须相等？
- 是否有某个不等式 $s_i\ne t_i$ 的两边被推出相等？

如果有冲突，则 UNSAT；否则 SAT。

------

### 3.基本算法思想

#### 3.1 等价类

等式 $s=t$ 表示 $s$ 和 $t$ 必须在同一个等价类里。

因此可以从每个 term 自己一个等价类开始，然后逐步处理等式。

例如初始时：

$$
\{a\},\{b\},\{f(a,b)\},\{f(f(a,b),b)\}
$$

如果有等式：

$$
f(a,b)=a
$$

就把 $f(a,b)$ 和 $a$ 的等价类合并。

#### 3.2 Congruence 会推出新等式

合并等价类后，还需要检查父节点是否因为参数等价而 congruent。

例如：

$$
f(a,b)=a
$$

那么 $f(a,b)$ 和 $a$ 等价。

考虑两个 term：

$$
f(f(a,b),b)
$$

和

$$
f(a,b)
$$

它们的函数符号都是 $f$，第二个参数都是 $b$，第一个参数 $f(a,b)$ 与 $a$ 等价，所以由函数一致性推出：

$$
f(f(a,b),b)=f(a,b)
$$

这就是等式理论判定比普通并查集多的一步：

#### 3.3 最后检查不等式

处理完所有可推出等式后，对于每个不等式：

$$
s\ne t
$$

检查：

$$
Rep(s)=Rep(t)
$$

如果两个 term 的代表元相同，说明理论公理推出 $s=t$，与 $s\ne t$ 冲突，所以 UNSAT。

如果所有不等式都不冲突，则 SAT。

------

### 4.Basic Idea 总结

公式：

$$
F:
 s_1=t_1 \land \cdots \land s_m=t_m
 \land s_{m+1}\ne t_{m+1}\land \cdots \land s_n\ne t_n
$$

算法两阶段：

1. 从等式 $s_i=t_i$ 出发，计算所有由自反、对称、传递、函数一致性推出的等价类。
2. 对每个不等式 $s_i\ne t_i$，检查 $s_i,t_i$ 是否已经在同一个等价类中。

结论：

- 若某个不等式两边同类，则公式 UNSAT；
- 否则公式 SAT。

------

### 5.例子 1：$f(a,b)=a \land f(f(a,b),b)\ne a$

#### 5.1 公式

$$
F: f(a,b)=a \land f(f(a,b),b)\ne a
$$

子项集合：

$$
\{a, b, f(a,b), f(f(a,b),b)\}
$$

初始等价类：

$$
\{a\},\{b\},\{f(a,b)\},\{f(f(a,b),b)\}
$$

#### 5.2 处理显式等式

处理：

$$
f(a,b)=a
$$

合并：

$$
\{a, f(a,b)\}
$$

现在等价类大致为：

$$
\{a, f(a,b)\},\{b\},\{f(f(a,b),b)\}
$$

#### 5.3 用 congruence 推出新等式

因为：

$$
f(a,b)=a
$$

所以：

$$
f(f(a,b),b)=f(a,b)
$$

合并后：

$$
\{a, f(a,b), f(f(a,b),b)\}
$$

#### 5.4 检查不等式

公式中还有：

$$
f(f(a,b),b)\ne a
$$

但根据等价类：

$$
f(f(a,b),b)=a
$$

冲突，所以：

$$
F \text{ is UNSAT}
$$

------

### 6.例子 2：$f^3(a)=a \land f^5(a)=a \land f(a)\ne a$

#### 6.1 公式

第二个例子的公式为：

$$
F:
 f(f(f(a)))=a
 \land f(f(f(f(f(a)))))=a
 \land f(a)\ne a
$$

为了简洁，记：

$$
f^1(a)=f(a)
$$

$$
f^2(a)=f(f(a))
$$

$$
f^3(a)=f(f(f(a)))
$$

以此类推。

于是公式为：

$$
f^3(a)=a \land f^5(a)=a \land f(a)\ne a
$$

子项集合：

$$
\{a,f(a),f^2(a),f^3(a),f^4(a),f^5(a)\}
$$

#### 6.2 初始等价类

初始时每个 term 单独一个类：

$$
\{a\},\{f(a)\},\{f^2(a)\},\{f^3(a)\},\{f^4(a)\},\{f^5(a)\}
$$

#### 6.3 处理 $f^3(a)=a$

合并：

$$
\{a,f^3(a)\}
$$

根据 congruence，对两边同时套一层 $f$：

$$
f(f^3(a))=f(a)
$$

即：

$$
f^4(a)=f(a)
$$

再对这条等式两边套一层 $f$：

$$
f^5(a)=f^2(a)
$$

因此得到：

$$
\{f(a),f^4(a)\}
$$

$$
\{f^2(a),f^5(a)\}
$$

#### 6.4 处理 $f^5(a)=a$

显式等式：

$$
f^5(a)=a
$$

由于当前：

$$
f^5(a) \sim f^2(a)
$$

且：

$$
a \sim f^3(a)
$$

合并后有：

$$
\{a,f^2(a),f^3(a),f^5(a)\}
$$

根据 congruence，如果：

$$
f^2(a)=a
$$

则：

$$
f^3(a)=f(a)
$$

而 $f^3(a)$ 已经和 $a$ 同类，所以推出：

$$
f(a)=a
$$

#### 6.5 检查不等式

公式中要求：

$$
f(a)\ne a
$$

但等式闭包推出：

$$
f(a)=a
$$

冲突，所以：

$$
F \text{ is UNSAT}
$$

------

## 五、一致性闭包算法实现

### 1.算法实现：term DAG

#### 1.1 为什么需要 DAG

公式里的 term 可能共享子表达式。例如：

$$
f(f(a,b),b)
$$

里面包含子项：

- $a$；
- $b$；
- $f(a,b)$；
- $f(f(a,b),b)$。

这些子项之间天然形成有向无环图（DAG）：

- 每个节点表示一个唯一子项；
- 函数节点指向它的参数节点；
- 常量节点没有参数边。

例如：

$$
f(f(a,b),b)
$$

可以看成：

- 节点 $a$；
- 节点 $b$；
- 节点 $f(a,b)$，参数是 $a,b$；
- 节点 $f(f(a,b),b)$，参数是 $f(a,b),b$。

#### 1.2 每个子项一个唯一 id

类似 ROBDD 中“唯一表”的思想，term DAG 中也通常给每个不同子项一个唯一 id。

例如：

| id | term |
|---:|---|
| 0 | $a$ |
| 1 | $b$ |
| 2 | $f(a,b)$ |
| 3 | $f(f(a,b),b)$ |

这样处理等价类、父节点、congruence 时都可以用整数编号，提高效率。

------

### 2.并查集维护等价类

#### 2.1 representative

每个等价类有一个代表元（representative）。

用并查集维护：

- `find[x]` 是通向代表元的父指针，不一定直接指向代表元；
- 如果 `find[x] == x`，则 $x$ 自己是代表元；
- 合并两个等价类时，让一个代表指向另一个代表。

数学上记作：

$$
Rep(x)
$$

表示 term $x$ 当前所在等价类的代表元。

#### 2.2 等式处理

处理一个等式：

$$
s=t
$$

就是执行：

$$
Union(s,t)
$$

也就是合并 $Rep(s)$ 和 $Rep(t)$ 所在的集合。

#### 2.3 路径压缩

为了提高效率，`find` 通常使用路径压缩：

```cpp
int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}
```

路径压缩减少重复查询的开销。这里未使用按秩或按大小合并，也未分析父节点列表的处理代价，不将整个一致性闭包算法的复杂度写成近似 $O(1)$。

------

### 3.parents list：为什么需要父节点表

#### 3.1 parent 的定义

如果 term $u$ 的参数里包含 term $v$，则称 $u$ 是 $v$ 的 parent。

例如：

$$
f(a,b)
$$

是 $a$ 和 $b$ 的 parent。

$$
f(f(a,b),b)
$$

是 $f(a,b)$ 和 $b$ 的 parent。

#### 3.2 parent 用来触发 congruence

当我们合并两个 term 的等价类时，它们的 parent 可能因此变得 congruent。

例如合并：

$$
f(a,b)=a
$$

之后，考虑 parent：

- $f(f(a,b),b)$ 是 $f(a,b)$ 的 parent；
- $f(a,b)$ 是 $a$ 的 parent。

因为：

- 两个 parent 的函数符号都是 $f$；
- 第一个参数 $f(a,b)$ 和 $a$ 同类；
- 第二个参数都是 $b$；

所以这两个 parent congruent，于是要继续处理：

$$
f(f(a,b),b)=f(a,b)
$$

这就是 parent list 的作用：

#### 3.3 在代表元上维护 parents

为了方便，每个等价类的所有 parent 可以存放在该等价类代表元的 `parents` 列表里。

每个 term 需要两个字段：

1. `find`：并查集指针；
2. `parents`：如果自己是代表元，则保存该等价类成员的所有父节点；如果不是代表元，则可以为空。

当合并两个等价类时：

- 合并 representative；
- 把两个 parents list 合并；
- 检查两边 parents 中是否有 congruent pair。

------

### 4.Congruent 的判断

两个 term $p_i$ 和 $p_j$ congruent，当且仅当：

1. 它们的函数符号相同；
2. 参数个数相同；
3. 对每个对应参数，都属于同一个等价类。

如果：

$$
p_i = f(a_1,\ldots,a_k)
$$

$$
p_j = f(b_1,\ldots,b_k)
$$

并且：

$$
Rep(a_1)=Rep(b_1),\ldots,Rep(a_k)=Rep(b_k)
$$

那么：

$$
p_i \equiv_{cong} p_j
$$

于是根据 function congruence，需要继续合并：

$$
p_i=p_j
$$

------

### 5.核心伪代码

伪代码如下。

#### 5.1 `Rep(x)`

```text
fn Rep(x) {
  if (x.find != x)
    return Rep(x.find);
  return x;
}
```

含义：找到 $x$ 所在等价类的代表元。

实际实现中一般加路径压缩。

#### 5.2 `merge(s,t)`

```text
fn merge(s,t) {
  let r1 = Rep(s);
  let r2 = Rep(t);
  if (r1 == r2) return;
  r1.find = r2;
  r2.parents += r1.parents;
  r1.parents = null;
}
```

含义：把 $s$ 和 $t$ 的等价类合并，同时合并 parent list。

#### 5.3 `process_equality(s,t)`

```text
fn process_equality(s,t) {
  if (Rep(s) == Rep(t)) return;
  let p1 = copy(Rep(s).parents);
  let p2 = copy(Rep(t).parents);
  merge(s,t);
  for (pi,pj) in product(p1,p2) {
    if (Rep(pi) != Rep(pj) && congruent(pi,pj))
      process_equality(pi,pj);
  }
}
```

含义：

1. 找到两边等价类的 parent list；
2. 先合并两边等价类；
3. 枚举原来两边 parent 的组合；
4. 如果某两个 parent 因参数等价而 congruent，则递归处理这个新等式。

注意：

- `product(p1,p2)` 表示笛卡尔积；
- 同类时立即返回，避免重复合并和递归；父节点列表须在合并前复制，避免清空或追加原列表影响枚举；
- 实际代码中为了避免递归过深，可以用队列代替递归。

#### 5.4 `check(conjunction)`

```text
fn check(conjunction) {
  for si == ti in conjunction {
    process_equality(si, ti);
  }
  for si != ti in conjunction {
    if (Rep(si) == Rep(ti))
      return unsat;
  }
  return sat;
}
```

这正是“先处理等式闭包，再检查不等式冲突”的思想。

------

### 6.C++ 风格实现框架：QF EUF congruence closure

下面是 **C++17 实现框架（AI 辅助整理）**。输入必须是已规范化的 term DAG：每个语法上相同的子项共用一个 id，参数 id 有效，且无环。每个 term 有：

- `op`：函数符号或常量名；
- `args`：参数 term id；
父节点列表另外存放在 `parentList` 中，不能与并查集的 `parent` 混淆。

代码重点展示 congruence closure 的核心流程，不包含字符串解析器或 main。每个独立公式应新建一个 `CongruenceClosure` 对象；重复调用同一个对象会保留先前合并的信息。

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <queue>
#include <utility>
#include <algorithm>

using namespace std;

struct Term {
    string op;              // 常量可看成 0 元函数，例如 "a"；函数节点如 "f"
    vector<int> args;       // 参数 term 的 id
};

struct CongruenceClosure {
    vector<Term> terms;
    vector<int> parent;                 // 并查集 parent
    vector<vector<int>> parentList;     // 只有代表元处维护该等价类所有成员的父节点

    CongruenceClosure(const vector<Term>& inputTerms)
        : terms(inputTerms) {
        int n = (int)terms.size();
        parent.resize(n);
        parentList.assign(n, {});
        for (int i = 0; i < n; ++i) parent[i] = i;

        // 建立 parentList：如果 i 是某个 term 的参数，那么那个 term 是 i 的 parent
        for (int id = 0; id < n; ++id) {
            for (int arg : terms[id].args) {
                parentList[arg].push_back(id);
            }
        }
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    bool sameClass(int a, int b) {
        return find(a) == find(b);
    }

    // 判断两个 term 是否 congruent：函数符号相同，参数个数相同，对应参数在同一等价类
    bool congruent(int a, int b) {
        const Term& x = terms[a];
        const Term& y = terms[b];

        if (x.op != y.op) return false;
        if (x.args.size() != y.args.size()) return false;

        for (size_t i = 0; i < x.args.size(); ++i) {
            if (find(x.args[i]) != find(y.args[i])) return false;
        }
        return true;
    }

    // 合并两个等价类，并返回合并前两个代表元的 parentList
    pair<vector<int>, vector<int>> mergeClass(int a, int b) {
        int ra = find(a);
        int rb = find(b);

        if (ra == rb) return {{}, {}};

        // 简单起见，把 ra 合并到 rb；实际可以按大小合并
        vector<int> pa = parentList[ra];
        vector<int> pb = parentList[rb];

        parent[ra] = rb;
        parentList[rb].insert(parentList[rb].end(), parentList[ra].begin(), parentList[ra].end());
        parentList[ra].clear();

        return {pa, pb};
    }

    void processEquality(int s, int t) {
        queue<pair<int, int>> q;
        q.push({s, t});

        while (!q.empty()) {
            auto [a, b] = q.front();
            q.pop();

            if (find(a) == find(b)) continue;

            auto [pa, pb] = mergeClass(a, b);

            // 只检查合并前两边 parent list 的组合
            for (int x : pa) {
                for (int y : pb) {
                    if (find(x) != find(y) && congruent(x, y)) {
                        q.push({x, y});
                    }
                }
            }
        }
    }

    // equalities: 所有显式等式；disequalities: 所有显式不等式
    bool isSatisfiable(const vector<pair<int, int>>& equalities,
                       const vector<pair<int, int>>& disequalities) {
        for (auto [s, t] : equalities) {
            processEquality(s, t);
        }

        for (auto [s, t] : disequalities) {
            if (find(s) == find(t)) {
                return false; // UNSAT
            }
        }
        return true; // SAT
    }
};
```

#### 6.1 这段框架的关键点

1. 常量也可以看成 0 元函数。
2. `congruent(a,b)` 不是简单判断 $a,b$ 是否同类，而是判断两个函数表达式是否由于参数同类而应该合并。
3. `processEquality` 不只合并显式等式，还会因为 parent congruent 产生新等式。
4. 最后不等式只要两边代表元相同，就 UNSAT。

#### 6.2 可以优化的地方

可以进行以下优化。

##### 1. Path compression

并查集路径压缩，减少 `find` 查询开销。

##### 2. Linked list

合并 `parents` 时，如果每次复制 vector 可能开销较大，可以用链表或小集合合并技巧避免大量复制。

##### 3. HashMap 查找 congruent parents

朴素做法会枚举 parent list 的笛卡尔积，可能较慢。

更高效做法是把 term 的 congruence key 放入哈希表：

$$
(op, Rep(arg_1), Rep(arg_2), \ldots, Rep(arg_k))
$$

如果两个 term 的 key 相同，就说明它们 congruent，可以合并。

这和 ROBDD 中“用唯一表查找已经存在的节点”思想类似。

------

## 六、RTL Repair

### 1.RTL Repair 案例总览

#### 1.1 什么是 RTL Repair

RTL Repair 是一个基于语义的硬件设计代码修复工具。

目标：

给定：

- buggy RTL hardware design；
- failing test；

自动生成：

- repaired RTL hardware design。

传统方法可能是：

- 随机生成 patch；
- 反复仿真；
- 根据仿真结果搜索修复。

RTL-Repair 的思路是：

案例来源：“RTL-Repair: Fast Symbolic Repair of Hardware Design Code”（ASPLOS 2024）。

------

### 2.Bug Example：Mod-3 Counter

有缺陷的 RTL：

```verilog
module counter(
  input clock, input reset, input enable,
  output reg [1:0] count,
  output reg overflow
);
  always@(posedge clock) begin
    if (reset == 1'b1) begin
      count <= 2'b00;
      overflow <= 1'b0;
    end else if (enable) begin
      count <= count + 1;
    end
    if (count == 2'b10) begin
      overflow <= 1'b1;
    end
  end
endmodule
```

期望是 mod-3 counter，即 count 应该循环：

$$
0 \rightarrow 1 \rightarrow 2 \rightarrow 0 \rightarrow \cdots
$$

但原代码在 `count == 2'b10` 时只设置了：

```verilog
overflow <= 1'b1;
```

没有把 `count` reset 到 0。

因此可能出现测试失败：

$$
count@4: 3 \ne 0
$$

局部修复为：

```verilog
if (count == 2'b10) begin
  overflow <= 1'b1;
  count <= 2'b00;
end
```

------

**适用范围：**这段局部修复针对示例中的失败测试，不等于已经证明所有输入序列都正确。原代码的第二个 `if` 独立于复位分支，且条件读取本拍更新前的 `count`；同一时钟块内后面的赋值可覆盖前面的赋值。不能在建模时忽略这些优先级。

### 3.Repair Template：修复模板

#### 3.1 模板思想

修复模板把“可能修改的位置”和“可能的修改内容”参数化。

记号说明：

- $\phi_i$：布尔 guard，表示是否启用某个修复；
- $\alpha_i$：修复表达式，表示写入什么值。

例如 conditional overwrite template：

```verilog
if (ϕ1) begin
  if (ϕ2 ? enable : 1'b1) begin
    count <= α1;
  end
end
```

含义：

- 如果 $\phi_1$ 为真，则启用这个额外赋值；
- 内层条件可以由 $\phi_2$ 控制是否依赖 `enable`；
- $\alpha_1$ 是符号化的待求表达式或值。

SMT solver 的任务就是求出：

$$
\phi_i, \alpha_i
$$

使修复后的 RTL 满足 testbench constraints。

------

### 4.SMT Formulation：如何把 RTL 修复编码成 SMT

RTL Repair 的 SMT formulation 有三步：

1. Unrolling transition system：展开状态转移系统；
2. Encoding repair templates：编码修复模板；
3. Encoding testbench constraints：编码测试约束。

------

### 5.展开转移系统：Unrolling Transition System

考虑简单 RTL：

```verilog
always @(posedge clk)
  if (enable)
    count <= count + 1;
```

展开为 SMT 公式：

$$
count_1 = ite(enable_0, count_0 + 1, count_0)
$$

$$
count_2 = ite(enable_1, count_1 + 1, count_1)
$$

$$
count_3 = ite(enable_2, count_2 + 1, count_2)
$$

其中：

- $count_t$ 表示第 $t$ 个周期的 count；
- $enable_t$ 表示第 $t$ 个周期的 enable；
- $ite(c,a,b)$ 是 if-then-else：如果 $c$ 为真，值为 $a$，否则为 $b$。

这里涉及的理论：

- `=` 属于等式理论；
- `ite` 是 SMT-LIB 中常见内置表达式；
- `count + 1` 属于 bit-vector theory，因为 RTL 信号有固定位宽。

------

### 6.编码修复模板

如果加入修复模板：

```verilog
always @(posedge clk)
  if (φ)
    count <= α;
```

展开可写成：

$$
count_1 = ite(\phi, \alpha, count_0)
$$

$$
count_2 = ite(\phi, \alpha, count_1)
$$

$$
count_3 = ite(\phi, \alpha, count_2)
$$

**注意：**$\phi$ 和 $\alpha$ 不是每个周期重新选一遍，而是所有展开周期共享同一组修复变量。

这符合“代码修复”的含义：

- 修的是源代码中的一个位置；
- 同一个修复在每个周期都生效；
- 不能每个周期临时换一个修法。

------

### 7.编码 testbench constraints

测试约束包括两类。

#### 7.1 输入约束

例如 testbench 指定：

```text
reset@0: 1
reset@1: 0
```

编码为：

$$
reset_0 = 1
$$

$$
reset_1 = 0
$$

#### 7.2 期望输出约束

例如 testbench 指定：

```text
count@1: 0
count@2: 1
count@3: 2
count@4: 0
```

编码为：

$$
count_1=0
$$

$$
count_2=1
$$

$$
count_3=2
$$

$$
count_4=0
$$

SMT solver 需要寻找修复变量，使得展开后的转移公式和 testbench 输入/输出约束同时可满足。

------

### 8.SMT-LIB 形式示意

以下是带符号修复模板的 RTL 片段（不是已求得参数的完整修复程序）。下列希腊字母表示待求参数，代码用于说明模板：

```verilog
module counter_repair(
  // [...] I/O
  input φ0,
  input [1:0] α0
);

always@(posedge clock) begin
  // [...]
  if(count == 2'b10) begin
    overflow <= 1'b1;

    // symbolic repair template
    if(φ0)
      count <= α0;
  end
end
endmodule
```

上面的模板片段省略了部分原逻辑。若要编码上面的完整 count 更新，一拍的关系应同时保留复位、使能和最后一次覆盖赋值。以下是补齐这些条件的 **SMT-LIB 示例（AI 辅助整理）**，只建模 count，不包含 overflow：

```lisp
(declare-fun phi0 () Bool)
(declare-fun alpha0 () (_ BitVec 2))

; phi0、alpha0 在所有时钟周期共享
(define-fun next_count
  ((count (_ BitVec 2)) (reset Bool) (enable Bool)) (_ BitVec 2)
  (ite (and (= count #b10) phi0)
       alpha0
       (ite reset #b00
            (ite enable (bvadd count #b01) count))))

; 将一个输入和期望输出明确的测试展开为四拍
(declare-fun count@0 () (_ BitVec 2))
(assert (= count@0 #b00))
(define-fun count@1 () (_ BitVec 2)
  (next_count count@0 true true))
(define-fun count@2 () (_ BitVec 2)
  (next_count count@1 false true))
(define-fun count@3 () (_ BitVec 2)
  (next_count count@2 false true))
(define-fun count@4 () (_ BitVec 2)
  (next_count count@3 false true))

(assert (= count@1 #b00))
(assert (= count@2 #b01))
(assert (= count@3 #b10))
(assert (= count@4 #b00))
(check-sat)
(get-value (phi0 alpha0))
```

这里为展示完整可求解输入，明确选择四拍 enable 均为 true。它仅验证该测试，不代表对所有输入和全部输出完成验证。

求解器可能返回：

$$
\phi_0=1, \quad \alpha_0=0
$$

含义：启用这个修复模板，并把 count 写成 0。

对应代码就是：

```verilog
if (count == 2'b10) begin
  overflow <= 1'b1;
  count <= 2'b00;
end
```

------

### 9.Minimal Repair：最小修复

#### 9.1 为什么需要最小修复

一个测试失败可能有很多满足测试的修复方式，但通过有限测试不等于完整正确性证明。

例如可以：

- 改一处；
- 改两处；
- 改很多处但也能过 test。

通常我们希望修复尽量小，以避免过拟合或破坏原有功能。

#### 9.2 Max-SMT / Optimization SMT


一种简单做法是把优化目标变成 hard constraint，逐步放松直到 SAT。

例如有两个修复开关：

$$
\phi_0,\phi_1
$$

希望只启用一个修复，可以写：

```lisp
(assert (= #b01 (bvadd
  (ite φ1 #b01 #b00)
  (ite φ0 #b01 #b00))))
```

这表示启用的修复数为 1。

若已知不修改时测试失败，可以先尝试：

$
\#repair = 1
$

如果 UNSAT，再尝试：

$$
\#repair = 2
$$

对应 SMT-LIB：

```lisp
(assert (= #b10 (bvadd
  (ite φ1 #b01 #b00)
  (ite φ0 #b01 #b00))))
```

一般应从 $k=0$ 开始，逐一排除所有更小修复数后，才能将首次 SAT 的 $k$ 称为该模板、该测试约束下的最小修复数。每次尝试应替换上一轮的计数约束，不能把“等于 1”和“等于 2”同时保留。计数位宽也必须足以表示最大修复数，避免位向量加法溢出。

------

## 七、概念整理与注意事项

### 1.本讲核心概念对照

| 概念 | 作用 | 记忆方式 |
|---|---|---|
| Signature | 规定能写哪些非逻辑符号 | 决定语法是否合法 |
| Axioms | 规定符号必须满足的性质 | 限制合法结构 |
| $T$-model | 满足理论公理的 structure | 只看“合规解释” |
| $T$-satisfiable | 存在某个 $T$-model 让公式为真 | modulo theory 下可满足 |
| $T$-valid | 所有 $T$-model 都让公式为真 | modulo theory 下有效 |
| EUF | 未解释函数 + 等式 | 不关心函数具体含义，只用一致性 |
| Congruence | 相等参数代入同函数，结果相等 | EUF 的核心推理规则 |
| Congruence closure | 由等式不断闭包推出全部必然相等项 | QF EUF 判定算法核心 |
| Array theory | 用 `select/store` 表示读写 | 写后读同位得新值，读异位不变 |
| Bit-vector theory | 固定位宽机器语义 | 适合 RTL/底层程序验证 |
| RTL Repair | 把修复模板和测试约束编码成 SMT | 用 solver 找代码补丁 |

------

### 2.本讲容易混淆的点

#### 2.1 “未解释函数”不是“任意乱变的函数”

Uninterpreted function 没有具体算术语义，但仍然必须满足：

$$
x=y \rightarrow f(x)=f(y)
$$

同一个函数同样的输入必须得到同样的输出。

#### 2.2 等价关系和 congruence 不一样

等价关系只管：

- 自反；
- 对称；
- 传递。

Congruence 还要求：

$$
\vec{x}=\vec{y}\rightarrow f(\vec{x})=f(\vec{y})
$$

所以 congruence 是等式理论里对函数/谓词表达式的额外约束。

#### 2.3 并查集只能处理显式等式，不能自动处理 congruence

如果只对显式等式做 union，会漏掉：

$$
f(a,b)=a \Rightarrow f(f(a,b),b)=f(a,b)
$$

这种由函数一致性推出的隐式等式。

因此实现 EUF solver 时必须维护 parent list 或 congruence hash。

#### 2.4 $T$-satisfiable 不是普通 satisfiable

普通 satisfiable 可以选任意 structure。

$T$-satisfiable 只能选满足理论公理的 structure。

例如：

$$
f(a)=a \land f(f(a))\ne a
$$

在普通 FOL 中，如果没有等式理论约束，可能通过奇怪解释让它成立；但在等式理论中，若由 congruence 和传递推出冲突，就会 UNSAT。

#### 2.5 bit-vector 和 integer 不一样

整数理论中：

$$
255+1=256
$$

8-bit 位向量中：

$$
11111111 + 00000001 = 00000000
$$

因为溢出按模 $2^8$ 计算。

因此程序/硬件验证常用 bit-vector，而不是数学整数。

------
