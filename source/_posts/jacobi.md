---
title: 左右雅可比的扰动公式（by deepseek）
date: 2026-05-05
tags: [李代数]
categories: robotics
mathjax: true
---

在机器人学、李群和李代数的微分几何中，**扰动公式**用于描述李群元素在微小变换下的变化，通常通过指数映射和对数映射来表达。  
这里以 **SO(3) 和 SE(3）** 为主要例子，但公式对一般李群李代数也成立。

---

## 1. 基础设定

- 设 $ G $ 为矩阵李群（如 $ SO(3), SE(3) $），  
- $ \mathfrak{g} $ 为其李代数，  
- $ \exp: \mathfrak{g} \to G $， $ \log: G \to \mathfrak{g} $ 是局部微分同胚。

给定一个群元素 $ R \in G $，其对应的李代数为  
$$
\boldsymbol{\phi}^\wedge = \log(R) \quad \text{（在某些表示下）}
$$

---

## 2. 右扰动 (Right perturbation)

**定义**：  
在 $ R $ 右侧乘一个小扰动 $ \exp(\delta\boldsymbol{\phi}^\wedge) $，即  
$$
R' = R \exp(\delta\boldsymbol{\phi}^\wedge)
$$  
其中 $ \delta\boldsymbol{\phi} $ 是**小量的李代数扰动向量**。

对扰动后取对数（一阶近似）：  
利用 Baker–Campbell–Hausdorff 公式：
$$
\log(R') \approx \log(R) + \mathbf{J}_r^{-1}(\boldsymbol{\phi}) \cdot \delta\boldsymbol{\phi}
$$
其中 
$$
\mathbf{J}_r(\boldsymbol{\phi}) = \frac{\sin(\|\boldsymbol{\phi}\|)}{\|\boldsymbol{\phi}\|} \mathbf{I} + \left(1 - \frac{\sin(\|\boldsymbol{\phi}\|)}{\|\boldsymbol{\phi}\|}\right) \frac{\boldsymbol{\phi}\boldsymbol{\phi}^T}{\|\boldsymbol{\phi}\|^2} + \frac{1 - \cos(\|\boldsymbol{\phi}\|)}{\|\boldsymbol{\phi}\|} \, \boldsymbol{\phi}^\wedge
$$  
是**右雅可比矩阵**（SO3 时，$\boldsymbol{\phi}^\wedge$ 为叉乘矩阵）。

小 $ \|\delta\boldsymbol{\phi}\| $ 时，  
$$
\mathbf{J}_r(\boldsymbol{\phi}) \approx \mathbf{I} + \frac12 \boldsymbol{\phi}^\wedge + \cdots
$$  
$$
\mathbf{J}_r^{-1}(\boldsymbol{\phi}) \approx \mathbf{I} - \frac12 \boldsymbol{\phi}^\wedge + \cdots
$$  
在 $ \boldsymbol{\phi} \approx 0 $ 时 $ \mathbf{J}_r \approx \mathbf{I} $。

---

## 3. 左扰动 (Left perturbation)

**定义**：  
在 $ R $ 左侧乘一个小扰动 $ \exp(\delta\boldsymbol{\phi}^\wedge) $，即  
$$
R' = \exp(\delta\boldsymbol{\phi}^\wedge) \, R
$$  
取对数：
$$
\log(R') \approx \log(R) + \mathbf{J}_l^{-1}(\boldsymbol{\phi}) \cdot \delta\boldsymbol{\phi}
$$
其中左雅可比为  
$$
\mathbf{J}_l(\boldsymbol{\phi}) = \mathbf{J}_r(-\boldsymbol{\phi})
$$
在 SO(3) 中，就是 $ \mathbf{J}_l(\boldsymbol{\phi}) = \mathbf{J}_r^T(\boldsymbol{\phi}) $。


## 4. 指数映射的微分公式（左/右扰动形式）

### 4.1 右扰动形式

对于时变李代数参数 $\boldsymbol{\xi}(t) \in \mathbb{R}^6$，定义 $\mathbf{Y}(t) = \exp(\boldsymbol{\xi}(t)^\wedge) \in SE(3)$，则：

$$
\boxed{\frac{d}{dt}\mathbf{Y}(t) = \mathbf{Y}(t) \left( \mathbf{J}_r(\boldsymbol{\xi}) \,\dot{\boldsymbol{\xi}} \right)^\wedge}
$$

其中 $\mathbf{J}_r(\boldsymbol{\xi})$ 是**右雅可比矩阵**，$\dot{\boldsymbol{\xi}}$ 是参数速度。

### 4.2 左扰动形式

$$
\boxed{\frac{d}{dt}\mathbf{Y}(t) = \left( \mathbf{J}_l(\boldsymbol{\xi}) \,\dot{\boldsymbol{\xi}} \right)^\wedge \mathbf{Y}(t)}
$$

其中 $\mathbf{J}_l(\boldsymbol{\xi})$ 是**左雅可比矩阵**，与右雅可比的关系为：

$$
\mathbf{J}_l(\boldsymbol{\xi}) = \mathbf{J}_r(-\boldsymbol{\xi})
$$

### 4.3 特殊情形（小角度/小位移）

当 $\|\boldsymbol{\xi}\| \to 0$ 时，$\mathbf{J}_l(\boldsymbol{\xi}) \approx \mathbf{I}$，$\mathbf{J}_r(\boldsymbol{\xi}) \approx \mathbf{I}$，微分公式退化为：

$$
\frac{d}{dt}\mathbf{Y}(t) \approx \dot{\boldsymbol{\xi}}^\wedge \quad (\text{左扰动}) \quad \text{或} \quad \frac{d}{dt}\mathbf{Y}(t) \approx \dot{\boldsymbol{\xi}}^\wedge \quad (\text{右扰动})
$$

此时左右形式不再区分。

### 4.4 与对数映射导数的关系

右扰动下，对数映射的导数为：
$$
\frac{\partial}{\partial \mathbf{R}} \log(\mathbf{R})^\vee = \mathbf{J}_r(\boldsymbol{\xi})^{-1}
$$

这正是 Pinocchio 中 `pin.Jlog6(R)` 返回的值。

























