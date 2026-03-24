---
title: ROS2 Jazzy 面试 demo 集：W5 - Launch 编排与就绪检查
date: 2026-03-25
categories:
  - ROS2
  - 工程实践
tags:
  - ROS2
  - Jazzy
  - 面试
  - demo
  - Launch
  - 就绪检查
---

## 概述

W5 对应面试里最常被追问的"启动编排问题"：多节点同时启动时，依赖节点可能因为 provider 还没 ready 而失败。正确做法是在 launch 层对"就绪条件"做门控。

---

## 前置要求

- ROS2：**Jazzy**
- 构建工具：`colcon`

```bash
source /opt/ros/jazzy/setup.bash
colcon build --packages-select w5_demo_pkg
source install/setup.bash
```

---

## 核心概念：启动时序错误 vs 就绪门控成功

### 问题根因

多节点启动时，如果依赖方在 provider 就绪前就发起调用：
- provider 还没创建对应 service/topic
- consumer 调用失败，直接报错退出
- launch 进程没有重试机制，直接失败

### 正确做法：launch 层等待就绪

在启动 consumer 之前，等待 provider 的 service/topic 出现，再继续。这叫**就绪门控（readiness gate）**。

---

## Demo 运行方法

### bad（不等就绪就启动，失败）

```bash
ros2 launch w5_demo_pkg bad_launch.py
```

**现象：**
- `w5_readiness_provider` 延迟 5 秒才创建 `/w5/ready`
- `w5_requires_ready_client` 在很短超时下尝试调用 `/w5/ready`
- 服务还没出现，client 直接报错退出

### good（等待就绪后再启动依赖节点）

```bash
ros2 launch w5_demo_pkg good_launch.py
```

**现象：**
- launch 进程里会等待 `/w5/ready` 服务出现
- 等到服务可用后，再启动依赖节点
- client 能成功调用并打印 `message=ready`

---

## 面试口述模板（可直接背）

**标准 3 句话：**
1. "我把依赖关系显式化：provider 需要 ready，consumer 才能启动。"
2. "bad 版复现了启动时序错误导致的失败（短超时直接失败）。"
3. "good 版通过 launch 层等待 service 出现，实现稳定的启动编排。"

**追问："还有哪些就绪条件可以用？"**
> "除了 service 出现，还可以等待 topic 发布者数量达标（`ros2 topic find`）、节点健康状态（`/health` service）、参数服务器值到位、甚至外部硬件握手。核心是把'依赖方'和'被依赖方'的就绪契约显式化。"

**追问："为什么不用 sleep 等待？"**
> "sleep 是硬等待，不知道 provider 到底什么时候好。sleep 短了还是失败，sleep 长了浪费时间。就绪门控是动态感知，provider 好了立刻启动，更健壮。"

---

## 常见坑

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| launch 后节点直接失败 | 依赖的 service 还没创建 | 加就绪门控等待 service 出现 |
| launch 超时退出 | 超时设置太短或 provider 启动慢 | 调整超时或改用就绪门控 |
| 节点启动顺序混乱 | 没显式声明依赖关系 | 用 `launch.actions.RegisterEventHandler` 显式编排 |

---

## 验证标准

- 能描述 bad launch 的失败链条（provider 还没好，consumer 就开始调用）
- 能解释为什么 sleep 等待不如就绪门控
- 能列举 2 种以上可用于就绪判断的条件

---

## 下一步

[W6：LifecycleNode 生命周期](./ros2-jazzy-w6-lifecycle-node.md)
