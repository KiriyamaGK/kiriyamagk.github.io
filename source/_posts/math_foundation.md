---
title: 四个空间、直和与正交补、伪逆
date: 2026-05-03
tags: [数学]
categories: 理论基础
mathjax: true
---

# 四个空间（行空间、列空间、零空间、左零空间）
## 定义
假设$A\in\mathbb{R}^{m\times n}$ 
- **列空间（Column Space）**：  
  $$
  R(A) = \{ Ax \mid x \in \mathbb{R}^n \} \subseteq \mathbb{R}^m,dim=r(A)
  $$

- **行空间（Row Space）**：  
  $$
  R(A^T) = \{ A^T y \mid y \in \mathbb{R}^m \} \subseteq \mathbb{R}^n,dim=r(A)
  $$

- **零空间（Null Space）**：  
  $$
  N(A) = \{ x \in \mathbb{R}^n \mid Ax = 0 \},dim=n-r(A)
  $$

- **左零空间（Left Null Space）**：  
  $$
  N(A^T) = \{ y \in \mathbb{R}^m \mid A^T y = 0 \},dim=m-r(A)
  $$
  注：$dim(R(A^T))=r(A)$是由于$R(A^T)$事实上就是$A$的行向量张成的空间

## 性质：成对正交补
- $R(A)和N(A^T)$(零空间和行空间)是正交补
- $N(A)和R(A^T)$(左零空间和列空间)是正交补

# 直和与正交补
## 直和
### 定义
设 $U,W$ 是 $V$的子空间，若 $U \cap W=\{0\}$，则 $U+W$ 为直和，记为 $U\oplus W$，其中 $U+W:=\{u+w \mid u\in U,w \in W \}$

### 充要条件
- 交为$\{0\}$
- 0向量表示唯一
- 任意向量表示唯一
- $dim(U) +dim(W)=dim(U+W)$

特别地：$\mathbb{R}^n=U\oplus W \iff \begin{cases} dim(U)+dim(W)=n \ 或 \ U+W=\mathbb{R}^n \\ U \cap W=\{0\} \end{cases}$

## 正交补
### 定义
$S$ 是 $\mathbb{R}^n$ 的一个子空间，$S^\perp :=\{v \in \mathbb{R}^n \mid v \cdot s=0,s \in S\}$

### 充要条件（$W=U^\perp$）
- $U=W^\perp$
- $U \perp W$ 且 $dim(U)+dim(W)=n$ （或$U+W=\mathbb{R}^n$）

### 性质
- $\mathbb{R}^n=S\oplus S^\perp$
- $(S^\perp)^\perp=S$
- $dim(S)+dim(S^\perp)=n$
- $\forall x \in \mathbb{R}^n$ 可唯一分解成 $x=s+t,s \in S,t \in S^ \perp$（必然有$s \cdot t=0$）; 特别地，$x$ 可唯一分解成 $x_n+x_r$，其中 $x_n \in N(S),x_r \in R(S^T)$

<!-- $$
\underbrace{U + W}_{\text{所有 } u+w \text{ 的组合}} = \{ u + w \mid u \in U, w \in W \}
$$ -->

# 伪逆
## 定义
设$A \in \mathbb{R}^{m \times n}$，一定存在唯一的 $A^+ \in \mathbb{R}^{n \times m}$，使得：$\begin{cases} AA^+A=A \\ A^+AA^+=A^+
\\ AA^+对称 \\ A^+A对称\end{cases}$，则称$A^+$为$A$的伪逆

## 性质

- 1. $A$ 列满秩 $\implies A^+ = (A^T A)^{-1} A^T$

- 2. $A$ 行满秩 $\implies A^+ = A^T (A A^T)^{-1}$

- 3. 存在且唯一性

- 4. $(A^+)^+ = A$，$(A^T)^+ = (A^+)^T$

- 5. $\text{rank}(A^+) = \text{rank}(A)$

- 6. $AA^+$ 是到 $R(A)$ 的正交投影，$A^+A$ 是到 $R(A^T)$ 的正交投影，进一步有$A^+$是到 $R(A^T)$ 的映射

- 7. $x^\star = A^+ b$ 为 $Ax = b$ 的模长最小的最小二乘解，且 $x^\star \in R(A^T)$

- 8. $\forall x \in \mathbb{R}^n$ 可唯一分解为
  $$
  x = \underbrace{A^+ A x}_{\in R(A^T)，由性质6} + \underbrace{(I - A^+ A)x}_{\in N(A)，左乘A可证}
  $$

### 性质证明
#### 6证明（只证 $A^+A$ 是到 $R(A^T)$ 的正交投影）：

- 先证值域保持：

$A^+Ax=(A^+A)^Tx=A^T(A^{+})^{T}x \in R(A^T)$

- 再证正交投影：

$\forall x \in R(A^T): A^+Ax=A^+AA^Tu \ (\exists u \in \mathbb{R}^m)$

由 $A^+$ 的定义和对称性，上式 $=(A^+A)^TA^Tu=A^T(A^T)^+A^Tu=A^Tu=x$

进一步，$A^+x=(A^+A)(A^+x)\in R(A^T)$

#### 7证明

- 先证$x^\star$是最小二乘解:

$x$是最小二乘解 $\iff A^TAx=A^Tb$

左边 $=A^TAx^ \star=A^TAA^+b=A^T(AA^+)^Tb=A^T(A^T)^+A^Tb=A^Tb=$ 右边

- 再证明 $x^\star$ 范数最小：

根据线性方程组解的结构定理，最小二乘解集为 $x_0+N(A^TA)$，其中 $x_0$ 是特征方程的特解

由于 $x^\star$ 是特解，而$N(A^TA)=N(A)$，故解集可以写成 $S=x^\star+N(A)$

由性质6，$x^\star = A^+b \in R(A^T)$，进一步由正交补的性质：$\forall y \in N(A): x^\star \perp y$

则 $x^\star+y \in S$，且$\|x^\star+y\|^2=\|x^\star\|^2+\|y\|^2 \ge \|x^\star\|^2$

- 最后，根据性质6，$x^\star \in R(A^T)$
