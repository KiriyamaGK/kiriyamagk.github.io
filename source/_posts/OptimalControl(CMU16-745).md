---
title: Optimal Control(CMU 16-745) Notes
date: 2026-05-17
tags: [control]
categories: control
mathjax: true
---

## lecture 1

介绍了一些常见的系统，包括：连续系统状态方程、线性系统、平衡点等。笔记：https://zhuanlan.zhihu.com/p/629135263

### 平衡点

平衡点为状态变量导数 $\dot x$ 为零的点。对于一般系统 $\dot x=f(x,u)$，在平衡点处可以对系统进行线性化：$0= \frac {\partial f}{\partial x}\Delta x+ \frac {\partial f}{\partial u}\Delta u$，以此近似平衡点附近系统的行为。

### 平衡点的稳定性

充分必要（可能不是）条件：系统矩阵 $ \frac {\partial f}{\partial x}|_{balence}$ 的特征值实部非负。

### 一些例子

对单摆系统在平衡点 $\theta=0$ 处线性化可知，该点系统特征实部为0，为一个临界稳定的平衡点

## lecture 2

上一节得到了连续时间系统的状态方程，即得到了系统状态变化的表达式，但是计算机难以对其进行连续积分，因此需要借助数值/离散积分来进行处理。理想上，我们希望这种离散积分有高效、高精度的特点。笔记：https://zhuanlan.zhihu.com/p/629135862

### 离散时间状态方程/状态方程的离散

#### 问题定义

简便起见，先讨论显式的方程。

假设有了连续时间的状态方程 $\dot x=f(x,u)$,和当前状态 $x_k$，想通过定义一个离散方程 $f_{discrete}$ 得到下一时刻状态 $x_{k+1}=f_{discrete}(x_k,u_k) $。

例如最简单的前向欧拉积分：$$\frac {dx}{dt}=f(x_k,u_k) \approx \frac {x_{k+1}-x_k} {h}$$ $$ \Rightarrow x_{k+1}:=f_{discrete}(x_k,u_k) =x_k+hf(x_k,u_k) $$

#### 积分方法

显式积分方法有前向欧拉积分、龙格库塔；隐式方法有后向欧拉积分。各种积分的特点略。

### 控制量$u(t)$的离散

计算机算出一个控制量，随后利用保持器填补成连续时间信号，输入执行器。

零阶保持器：表达式为分段的常值控制。需要很多点来刻画连续的输入

一阶保持器：表达式为折线段


