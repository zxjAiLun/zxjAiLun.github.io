---
title: ROS2 Jazzy 面试 demo 集：W3 - QoS/DDS 行为
date: 2026-03-23
categories:
  - ROS2
  - 工程实践
tags:
  - ROS2
  - Jazzy
  - 面试
  - demo
  - QoS
  - DDS
  - TransientLocal
---

## 概述

W3 解决的是"能讲明白为什么看起来收不到消息、为什么晚加入拿不到初始状态"。QoS/DDS 是工程化加分项——能解释现象到原因到方案，证明你不仅会用还对底层有理解。

---

## 前置要求

- ROS2：**Jazzy**
- 构建工具：`colcon`

```bash
source /opt/ros/jazzy/setup.bash
colcon build --packages-select w3_demo_pkg
source install/setup.bash
```

---

## 核心概念：订阅端是 request，发布端是 offer

DDS 的 QoS 匹配规则：**request 比 offer 更严格时会不匹配**。

- 发布端 offer：发布方声明的 QoS 策略
- 订阅端 request：订阅方要求的 QoS 策略
- 订阅端 request ≤ 发布端 offer → 兼容
- 订阅端 request > 发布端 offer → 不兼容，通信建立不起来

### Demo 1：Reliable vs BestEffort 不兼容

```bash
ros2 run w3_demo_pkg qos_mismatch_demo
```

**现象：**
- 发布端 `BEST_EFFORT`（尽力而为，允许丢包）
- 订阅端 A `RELIABLE`（要求可靠，不允许丢包）→ **不兼容，持续收不到**
- 订阅端 B `BEST_EFFORT` → **兼容，持续收到**

**面试解释：**
> "订阅端要求可靠（RELIABLE），但发布端只保证尽力送达（BEST_EFFORT），request 比 offer 更严格，DDS 判定不兼容，通信直接建立不起来。所以有时候 topic 看得到，但就是收不到。"

### Demo 2：晚加入订阅者与 durability

```bash
ros2 run w3_demo_pkg durability_late_joiner_demo
```

**现象：**
- 发布端 `TRANSIENT_LOCAL` 且先发消息
- 2 秒后两个订阅者才加入（都属于晚加入）
- `TRANSIENT_LOCAL` 订阅者**立刻拿到历史快照**
- `VOLATILE` 订阅者只能等下一次新消息

**面试解释：**
> "`TransientLocal` 相当于在 DDS 层保留最后一条消息状态，适合地图、配置、初始状态这类'晚加入也要拿到最新值'的场景。而 Volatile 是默认行为，只关心之后的消息，历史消息直接丢弃。"

---

## 面试口述模板（可直接背）

**"为什么收不到消息？" 回答框架：**
> "先看 topic 的 QoS 是否兼容——订阅端是 request，发布端是 offer，request 比 offer 严格就会不匹配。然后看是否是晚加入场景，如果 publish 时订阅者还没启动，且是 Volatile durability，那历史消息直接丢了。"

**TransientLocal 场景：**
> "`TransientLocal` 适合初始化场景。比如机器人启动时，地图节点先发出地图元数据，sensor 融合节点后启动，用 TransientLocal 可以让后启动的节点立刻拿到最新地图，而不用等下一次发布。"

---

## 常见坑

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| topic 看得到但收不到 | QoS 不兼容（request > offer） | 用 `ros2 topic info /xxx -v` 查看双方 QoS |
| 晚加入后收不到初始状态 | durability 是 VOLATILE | 改成 TRANSIENT_LOCAL |
| 消息看起来丢了 | reliability 是 BEST_EFFORT | 改 RELIABLE 或确认是否真的允许丢 |

---

## 验证标准

- 能用 `ros2 topic info /xxx -v` 读取并解释双方 QoS
- 能解释"不兼容"和"晚加入收不到"两个场景的根因
- 能给出一个适合 TransientLocal 的真实工程场景

---

## 下一步

[W4：Service 延迟抖动优化](./ros2-jazzy-w4-service-latency-jitter.md)
