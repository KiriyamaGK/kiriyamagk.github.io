---
title: Pinocchio的数值IK
date: 2026-05-05
tags: [ik]
categories: robotics
mathjax: true
---

## 思路

给定当前位姿 $X_{cur} \in SE(3)$，目标位姿 $X_{tar} \in SE(3)$，定义误差变换：

$$
\mathbf{E} = X_{cur}^{-1} X_{tar} \in SE(3)
$$

对应的6维误差向量为 $\mathbf{e}(q) = \log(\mathbf{E}(q))^\vee \in \mathbb{R}^6$，定义$\mathbf{J}_{\text{error}} := \frac{\partial \mathbf{e}}{\partial q}$。

希望误差指数衰减，即满足微分方程 $\dot{\mathbf{e}} = -\mathbf{e}$，则有：

$$
\dot{\mathbf{e}} = \mathbf{J}_{\text{error}} \dot{q} = -\mathbf{e}
$$


得到一个最小二乘解 $\dot q$：

$$
\dot{q} = -\mathbf{J}_{\text{error}}^+ \mathbf{e}
$$

最后通过离散积分更新关节角度：

$$
q_{\text{new}} = q + \dot{q} \cdot \Delta t
$$

重复迭代直到 $\|\mathbf{e}\| < \epsilon$。

---

## 1. 体速度与关节速度的关系

在局部（工具）坐标系下，末端执行器的体速度 $\mathbf{V}_{body} \in \mathbb{R}^6$ 与关节速度的关系为：

$$
\mathbf{V}_{body} = \mathbf{J}_{body}(q) \cdot \dot{q}
$$

其中 $\mathbf{J}_{body}(q) \in \mathbb{R}^{6 \times n}$ 是体雅可比矩阵（Pinocchio 中 `ReferenceFrame.LOCAL`）。

---

## 2. 误差变化率与体速度的关系

体速度旋量公式：

$$
\mathbf{V}_{body}^\wedge = X_{cur}^{-1} \dot{X}_{cur}
$$

对 $\mathbf{E} = X_{cur}^{-1} X_{tar}$ 求导（$X_{tar}$ 常数）：

$$
\dot{\mathbf{E}} = -X_{cur}^{-1} \dot{X}_{cur} \mathbf{E} = - \mathbf{V}_{body}^\wedge \mathbf{E} \tag{1}
$$

由指数映射的**左扰动微分公式**：

$$
\dot{\mathbf{E}} = \big( \mathbf{J}_l(\mathbf{e}) \dot{\mathbf{e}} \big)^\wedge \mathbf{E} \tag{2}
$$

其中 $\mathbf{J}_l(\mathbf{e})$ 是左雅可比矩阵。

联立 (1)(2)，消去 $\mathbf{E}$：

$$
- \mathbf{V}_{body} = \mathbf{J}_l(\mathbf{e}) \dot{\mathbf{e}}
$$

因此：

$$
\dot{\mathbf{e}} = - \mathbf{J}_l(\mathbf{e})^{-1} \mathbf{V}_{body} \tag{3}
$$

---

## 3. 误差雅可比矩阵

将 $\mathbf{V}_{body} = \mathbf{J}_{body} \dot{q}$ 代入 (3)：

$$
\dot{\mathbf{e}} = - \mathbf{J}_l(\mathbf{e})^{-1} \mathbf{J}_{body} \dot{q}
$$

则得到了误差雅可比矩阵：

$$
\mathbf{J}_{\text{error}} := \frac{\partial \mathbf{e}}{\partial q} = - \mathbf{J}_l(\mathbf{e})^{-1} \mathbf{J}_{body}
$$


---

## 4. 阻尼最小二乘求解

根据先前指数收敛的假设，需要求解方程：

$$
\mathbf{J}_{\text{error}} \dot{q} = -\mathbf{e}
$$

使用阻尼最小二乘法：

$$
\dot{q} = - \mathbf{J}_{\text{error}}^T \left( \mathbf{J}_{\text{error}} \mathbf{J}_{\text{error}}^T + \lambda \mathbf{I}_6 \right)^{-1} \mathbf{e}
$$

---

## 5. 离散时间迭代

设时间步长 $\Delta t$，关节位置通过 Pinocchio 的几何积分更新：

$$
q_{\text{new}} = \text{integrate}(q, \dot{q} \cdot \Delta t)
$$

重复直到 $\|\mathbf{e}\| < \epsilon$。

---

## 代码解读

### 主循环

```python
i = 0
while True:
    # 前向运动学：计算当前末端位姿
    pin.forwardKinematics(model, data, q)
    
    # 计算误差变换：E = X_cur^{-1} * X_tar
    iMd = data.oMi[JOINT_ID].actInv(oMdes)
    
    # 提取6维误差向量（在关节局部坐标系中）
    err = pin.log(iMd).vector
    
    # 收敛检查
    if norm(err) < eps:
        success = True
        break
    if i >= IT_MAX:
        success = False
        break
    
    # 计算体雅可比（局部坐标系）
    J = pin.computeJointJacobian(model, data, q, JOINT_ID)
    
    # 转换为误差雅可比：J_error = -J_l(e)^{-1} * J_body
    J = -np.dot(pin.Jlog6(iMd.inverse()), J)
    
    # 阻尼最小二乘求解关节速度
    # 解 (J J^T + λI) x = -e，然后 v = J^T x
    v = -J.T.dot(np.linalg.solve(J.dot(J.T) + damp * np.eye(6), err))
    
    # 几何积分更新关节角度
    q = pin.integrate(model, q, v * DT)
    
    # 每10次迭代打印进度
    if not i % 10:
        print(f"{i}: error = {err.T}")
    i += 1
```

