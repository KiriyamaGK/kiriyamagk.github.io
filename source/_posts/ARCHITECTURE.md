---
title: Panthera-HT SDK 架构文档
date: 2026-05-01
tags: [笔记, 文档]
categories: 文档
---

# Panthera-HT SDK 架构文档

本文档描述 Panthera-HT SDK 的整体架构、类继承与构造顺序、各层职责及主要 API，便于理解从硬件通信到机械臂高层控制的完整调用链。

---

## 1. 仓库总览

```
Panthera-HT_SDK/
├── panthera_cpp/                 # C++ SDK（核心）
│   ├── motor_cpp/                # 底层：电机 / CAN / 串口通信
│   └── robot_cpp/                # 上层：机械臂控制、运动学、动力学
├── panthera_python/              # Python SDK（pybind11 绑定 C++）
├── docs/
│   └── ARCHITECTURE.md           # 本文档
└── README.md
```

| 模块 | 库名 | 核心类 | 职责 |
|------|------|--------|------|
| `motor_cpp` | `hightorque_motor` | `hightorque_robot::robot` | USB 串口 → CAN 板 → 电机，底层通信与电机对象管理 |
| `robot_cpp` | `panthera_robot` | `panthera::Panthera` | 在 `robot` 之上增加关节限位、轨迹插值、FK/IK、动力学 |
| `panthera_python` | `panthera` (pip) | `Panthera` (Python) | 与 C++ `Panthera` 等价的 Python 接口 |

**依赖关系：**

```
panthera_python  →  panthera_robot  →  hightorque_motor
                         ↓
                    Pinocchio (URDF / 动力学)
                    yaml-cpp (配置)
```

`robot_cpp/CMakeLists.txt` 通过 `add_subdirectory(../motor_cpp)` 将 `motor_cpp` 作为子工程编译，并链接 `hightorque_motor` 库。

---

## 2. 分层架构

```
应用层 (example / 用户代码)
├── 1_PosVel_control.cpp
├── 3_interpolation_control_nozeroVel.cpp
├── 5_teleop_control.cpp
└── ...
    │
    ▼
机械臂层 (panthera::Panthera)
├── initialize() / initializeMembers()
├── posVelMaxTorque / posVelTorqueKpKd
├── quinticInterpolation / septicInterpolationWithVelocity
├── forwardKinematics / inverseKinematics
├── getGravity / getDynamics / getFrictionCompensation
└── checkJointLimits / loadConfig / loadURDFModel
    │
    ▼
底层机器人层 (hightorque_robot::robot)
├── init_robot()
├── motor_send_cmd()
├── send_get_motor_state_cmd()
└── Motors[] 电机对象向量
    │
    ▼
硬件抽象层
├── serial_driver    串口驱动
├── canboard         CAN 转发板
├── canport          CAN 端口
└── motor            单电机控制
    │
    ▼
物理设备
├── USB 虚拟串口  /dev/ttyACM*
├── CAN 转发板
└── 关节电机 + 夹爪电机
```

**数据流（一次控制指令）：**

```
用户代码
  → Panthera::posVelMaxTorque(pos, vel, max_torque)
    → checkJointLimits(pos)          // 仅检查位置
    → Motors[i]->pos_vel_MAXtqe(...) // 写入各电机命令缓存
    → robot::motor_send_cmd()        // 经串口/CAN 下发
      → serial_driver → canboard → canport → motor → 物理电机
```

---

## 3. 类继承与构造顺序

### 3.1 继承关系

```cpp
// panthera_cpp/robot_cpp/include/panthera/Panthera.hpp
class Panthera : public hightorque_robot::robot
```

- `hightorque_robot::robot` 定义于 `panthera_cpp/motor_cpp/include/hardware/robot.hpp`
- `Panthera` 通过 `#include "hardware/robot.hpp"` 引入父类
- 父类 **public** 成员 `Motors`、`motor_send_cmd()` 等，子类可直接使用
- 父类 **private** 成员 `init_robot()`，子类**不能**直接调用，只能由父类构造函数间接触发

### 3.2 构造调用链

以 `Panthera robot("../robot_param/Follower.yaml")` 为例：

```
Panthera(config_path)
│
├─ ① 父类构造（初始化列表）
│     hightorque_robot::robot(config_path)
│       └─ robot::robot(config_path)
│            └─ init_robot(config_path)     ← 硬件初始化（private，仅此入口）
│                 ├─ parseRobotParams()      读 motor_param/*.yaml
│                 ├─ init_ser()              打开串口
│                 ├─ 创建 CANboards / CANPorts
│                 ├─ 创建 Motors[]           电机对象
│                 ├─ check_motor_connection_*()
│                 └─ send_get_motor_state_cmd()
│
├─ ② 子类成员初始化（初始化列表）
│     motor_count_=0, gripper_id_=0, model_loaded_=false, ...
│
└─ ③ 子类构造函数体
      initialize(config_path)                  ← Panthera 层初始化
        ├─ loadConfig(config_path)           关节限位、力矩、速度限幅、URDF 路径
        ├─ motor_count_ = Motors.size() - 1  使用父类已建好的 Motors
        ├─ gripper_id_ = Motors.size()
        └─ loadURDFModel()                   Pinocchio 加载 URDF
```

无参构造 `Panthera()` 的区别：

| 步骤 | 无参构造 | 带路径构造 |
|------|----------|------------|
| 父类配置 | `robot("../robot_param/Follower.yaml")` | `robot(config_path)` |
| 子类初始化 | `initializeMembers()` | `initialize(config_path)` |

两者逻辑等价，均会：父类 `init_robot` → 子类 `loadConfig` + `loadURDFModel`。

### 3.3 两层初始化职责

| 层级 | 入口 | 配置文件 | 主要工作 |
|------|------|----------|----------|
| 父类 `robot` | `init_robot()` | `Follower.yaml` → `param_file` → `motor_param/*.yaml` | 串口、CAN 板、电机枚举、通信线程、电机连接检测 |
| 子类 `Panthera` | `initialize()` / `initializeMembers()` | 同一份 `Follower.yaml`（再次 `loadConfig`） | 关节/夹爪限位、最大力矩、速度限幅、URDF、Pinocchio 模型 |

> **注意：** 同一份 YAML 被读取两次——父类读 `robot.param_file` 指向的电机参数，子类读 `joint_limits`、`urdf` 等机械臂参数。`Follower.yaml` 通过 `param_file` 字段桥接到底层电机配置。

### 3.4 析构顺序

```
~Panthera()
  → ~robot()
       ├─ set_brake() / set_stop() / set_reset()   // 根据配置决定
       ├─ 关闭串口、join 接收线程
       └─ 释放 CAN / 电机资源
```

---

## 4. 配置文件体系

### 4.1 两层 YAML

**机械臂配置**（`robot_cpp/robot_param/Follower.yaml` 或 `Leader.yaml`）：

```yaml
robot:
  robot_name: "Panthera-HT"
  param_file: "motor_param/6dof_Panthera_params_follower.yaml"  # → 指向底层参数
  joint_limits: { lower: [...], upper: [...] }
  gripper_limits: { lower: 0.0, upper: 2.0 }
  max_torque: [...]        # 可选
  velocity_limits: [...]   # 可选，缺省则 Panthera 使用 ±1.0 rad/s

urdf:
  file_path: "../Panthera-HT_description/urdf/..."

kinematics:
  joint_names: ["joint1", ..., "joint6"]

control:
  default_velocity: 0.5
  default_max_torque: 10.0
```

**电机底层配置**（`robot_cpp/robot_param/motor_param/6dof_Panthera_params_*.yaml`）：

- CAN 板数量、串口类型、波特率
- 各 CAN 口挂载的电机 ID、型号、名称
- 电机超时、刹车策略等

### 4.2 主臂 / 从臂

| 文件 | 用途 |
|------|------|
| `Leader.yaml` + `6dof_Panthera_params_leader.yaml` | 主臂（遥操作主动端） |
| `Follower.yaml` + `6dof_Panthera_params_follower.yaml` | 从臂（遥操作被动端 / 默认配置） |

---

## 5. 硬件层类与方法 (`motor_cpp`)

### 5.1 类层次

```
robot
 ├── serial_driver[]     串口驱动
 ├── canboard[]          CAN 转发板（每块板管理若干 CAN 口）
 │    └── canport[]      CAN 端口
 │         └── motor[]   单个关节/夹爪电机
 └── Motors[]            所有电机的扁平列表（public）
```

### 5.2 `hightorque_robot::robot` 主要方法

| 分类 | 方法 | 说明 |
|------|------|------|
| 生命周期 | `robot()`, `robot(config_path)`, `~robot()` | 构造时调用 `init_robot` |
| 通信 | `motor_send_cmd()` | 将各电机缓存命令打包下发 |
| 状态 | `send_get_motor_state_cmd()` | 请求电机反馈 |
| 状态 | `send_get_motor_version_cmd()` | 请求固件版本 |
| 安全 | `set_stop()`, `set_brake()`, `set_reset()` | 急停 / 刹车 / 复位 |
| 零位 | `set_reset_zero()`, `set_motor_runzero()` | 回零相关 |
| 超时 | `set_timeout(t_ms)` | 通信超时 |
| 诊断 | `check_motor_connection_position()` | 检测电机连接 |
| LCM | `lcm_enable()`, `publishJointStates()` | 可选状态发布 |
| 固件 | `canboard_bootloader()` | CAN 板升级 |

### 5.3 `motor` 主要控制方法

| 方法 | 说明 |
|------|------|
| `position(float)` | 纯位置控制 |
| `velocity(float)` | 纯速度控制 |
| `torque(float)` | 纯力矩控制 |
| `pos_vel_MAXtqe(pos, vel, torque_max)` | 位置 + 速度 + 最大力矩（最常用底层模式） |
| `pos_vel_tqe_kp_kd(pos, vel, tqe, Kp, Kd)` | MIT 五参数模式 |
| `pos_vel_kp_kd(pos, vel, Kp, Kd)` | 位置速度 PD |
| `get_current_motor_state()` | 返回 `motor_back_t*`（位置、速度、力矩） |
| `stop()`, `brake()`, `reset()` | 单电机急停 |

---

## 6. 机械臂层类与方法 (`panthera::Panthera`)

头文件：`panthera_cpp/robot_cpp/include/panthera/Panthera.hpp`  
实现：`panthera_cpp/robot_cpp/src/panthera/Panthera.cpp`

### 6.1 状态获取

| 方法 | 返回 | 说明 |
|------|------|------|
| `getCurrentState()` | `vector<motor_back_t*>` | 各关节原始状态 |
| `getCurrentPos()` | `vector<double>` | 关节位置 (rad) |
| `getCurrentVel()` | `vector<double>` | 关节速度 (rad/s) |
| `getCurrentTorque()` | `vector<double>` | 关节力矩 (Nm) |
| `getCurrentPosGripper()` 等 | 标量 / 指针 | 夹爪状态 |

### 6.2 控制接口

| 方法 | 位置限位 | 速度限幅 | 说明 |
|------|----------|----------|------|
| `posVelMaxTorque(pos, vel, max_torque, is_wait, ...)` | ✅ 检查 | ❌ 不限幅 | 最常用，轨迹示例均用此接口 |
| `posVelTorqueKpKd(pos, vel, torque, kp, kd)` | ✅ | ❌ | MIT 阻抗 / 力控 |
| `jointVel(vel)` | ❌ | ✅ 限幅 | 纯速度控制 |
| `jointsSyncArrival(pos, duration, ...)` | ✅ | 隐式（由 duration 算平均速度） | 多关节同步到达 |
| `gripperControl` / `gripperControlMIT` | 夹爪限位 | — | 夹爪控制 |
| `gripperOpen()` / `gripperClose()` | — | — | 夹爪开合快捷接口 |

> **安全提示：** `posVelMaxTorque` 不限制速度。若轨迹插值产生超速，需在上层自行检查 `traj_vel`，或调用前限幅。`jointVel` 才会使用 `velocity_limits_`（配置缺省为 ±1.0 rad/s）。

### 6.3 轨迹规划（静态方法）

| 方法 | 边界条件 | 典型用途 |
|------|----------|----------|
| `quinticInterpolation` | 起终点速度、加速度为 0 | 点到点平滑运动 |
| `septicInterpolation` | 起终点速度、加速度、加加速度为 0 | 更平滑的点到点 |
| `septicInterpolationWithVelocity` | 可指定起终点速度 | 连续轨迹、中间点不停顿 |

示例 `3_interpolation_control_nozeroVel.cpp` 使用 `septicInterpolationWithVelocity`，以 50 Hz 循环插值并调用 `posVelMaxTorque`。

### 6.4 运动学与动力学

| 方法 | 说明 |
|------|------|
| `forwardKinematics(joint_angles)` | 正运动学，返回位置、旋转、变换矩阵 |
| `inverseKinematics(target_position, ...)` | 数值逆运动学（需 URDF 已加载） |
| `getGravity(q)` | 重力补偿力矩 G(q) |
| `getCoriolis(q, v)` / `getCoriolisVector(q, v)` | 科氏力 |
| `getMassMatrix(q)` | 质量矩阵 M(q) |
| `getInertiaTerms(q, a)` | M(q)·a |
| `getDynamics(q, v, a)` | τ = M·a + C·v + G |
| `getFrictionCompensation(vel, Fc, Fv)` | 库伦 + 粘性摩擦补偿 |
| `clipTorque(torque, max_torque)` | 力矩限幅工具 |

依赖 **Pinocchio** 解析 `Panthera-HT_description/urdf/*.urdf`。

### 6.5 私有初始化方法

| 方法 | 调用时机 | 作用 |
|------|----------|------|
| `initialize(config_path)` | 带参构造 | 子类完整初始化 |
| `initializeMembers()` | 无参构造 | 同上，使用默认 Follower.yaml |
| `loadConfig(config_path)` | 上述两者内部 | 读关节限位、力矩、速度限幅 |
| `loadURDFModel()` | 上述两者内部 | Pinocchio 加载 URDF |
| `checkJointLimits(pos)` | 控制前 | 位置安全检查 |

---

## 7. 示例程序索引

路径：`panthera_cpp/robot_cpp/example/`

| 编号 | 文件 | 功能 |
|------|------|------|
| 0 | `0_robot_get_state.cpp` | 读取并打印关节状态 |
| 0 | `0_robot_set_zero.cpp` | 设置电机零位 |
| 1 | `1_PosVel_control.cpp` | 基础位置速度控制 |
| 1 | `1_PD_control.cpp` | PD 控制 |
| 1 | `1_Joints_Sync_Arrival_control.cpp` | 多关节同步到达 |
| 2 | `2_inv_PosVel_control.cpp` | 逆运动学位置控制 |
| 2 | `2_gravity_compensation_control.cpp` | 重力补偿 |
| 2 | `2_gravity_friction_compensation_control.cpp` | 重力 + 摩擦补偿 |
| 2 | `2_joint_impedance_control.cpp` | 关节阻抗（重力 + PD） |
| 2 | `2_joint_impedance_control_with_friction.cpp` | 含摩擦的阻抗控制 |
| 3 | `3_interpolation_control_zeroVel.cpp` | 七次插值，中间点零速 |
| 3 | `3_interpolation_control_nozeroVel.cpp` | 七次插值，中间点非零速（连续运动） |
| 3 | `3_sin_trajectory_control.cpp` | 正弦轨迹 |
| 3 | `3_gravity_compensation_with_fk.cpp` | 重力补偿 + 正运动学 |
| 4 | `4_impedance_trajectory_control_with_gra_pd.cpp` | 轨迹阻抗控制 |
| 5 | `5_record_trajectory.cpp` | 主从轨迹录制 |
| 5 | `5_replay_trajectory.cpp` | 轨迹回放 |
| 5 | `5_teleop_control.cpp` | 主从遥操作 |

`motor_cpp/example/` 提供更底层的电机级示例（单电机运动、回零、CAN 板升级等）。

---

## 8. Python SDK 关系

- 路径：`panthera_python/`
- 通过 **pybind11** 绑定 C++ `Panthera` 及底层 `robot` 接口
- API 命名采用 snake_case（如 `pos_vel_MAXtqe`），与 C++ camelCase（如 `posVelMaxTorque`）对应
- 功能集与 C++ `robot_cpp` 基本一致：控制、运动学、动力学、轨迹插值、主从遥操

安装与示例见 `panthera_python/README.md`。

---

## 9. 典型使用模式

### 9.1 最简控制

```cpp
#include "panthera/Panthera.hpp"

panthera::Panthera robot("../robot_param/Follower.yaml");

std::vector<double> pos(6, 0.0);
std::vector<double> vel(6, 0.3);
std::vector<double> max_torque(6, 5.0);

robot.posVelMaxTorque(pos, vel, max_torque, true);  // is_wait=true 等待到位
```

### 9.2 轨迹插值循环

```cpp
for (int step = 0; step < steps; ++step) {
    double t = step * dt;
    std::vector<double> traj_pos, traj_vel, traj_acc;
    panthera::Panthera::septicInterpolationWithVelocity(
        start_pos, end_pos, start_vel, end_vel, duration, t,
        traj_pos, traj_vel, traj_acc);
    robot.posVelMaxTorque(traj_pos, traj_vel, max_torque);
    // sleep_until 保持控制频率
}
```

### 9.3 继承父类能力

```cpp
robot.motor_send_cmd();              // 直接调用父类方法
robot.send_get_motor_state_cmd();
robot.set_stop();
size_t n = robot.Motors.size();      // 直接访问父类 public 成员
```

---

## 10. 关键源文件速查

| 文件 | 内容 |
|------|------|
| `motor_cpp/include/hardware/robot.hpp` | `hightorque_robot::robot` 类声明 |
| `motor_cpp/src/hardware/robot.cpp` | `init_robot`、串口/CAN 初始化 |
| `motor_cpp/include/hardware/motor.hpp` | 单电机控制接口 |
| `robot_cpp/include/panthera/Panthera.hpp` | `Panthera` 完整 API |
| `robot_cpp/src/panthera/Panthera.cpp` | 构造链、控制、插值、动力学实现 |
| `robot_cpp/robot_param/Follower.yaml` | 从臂机械臂配置 |
| `robot_cpp/robot_param/motor_param/*.yaml` | 电机/CAN 底层配置 |
| `robot_cpp/CMakeLists.txt` | 构建与依赖关系 |

---

## 11. 延伸阅读

- [根目录 README](../README.md) — 项目介绍与相关仓库
- [C++ SDK README](../panthera_cpp/robot_cpp/README.md) — 编译、示例、API 说明
- [motor_cpp README](../panthera_cpp/motor_cpp/README.md) — 底层电机 SDK
- [Python SDK README](../panthera_python/README.md) — Python 安装与用法
