---
title: ROS2 Jazzy 面试 demo 集：W4 - Service 延迟抖动优化
date: 2026-03-24
categories:
  - ROS2
  - 工程实践
tags:
  - ROS2
  - Jazzy
  - 面试
  - demo
  - 延迟
  - 抖动
  - QoS
---

## 概述

W4 解决的是"能解释大消息高频流压上来后，为什么 service 响应会明显变慢或抖动，并给出可运行的优化方案"。对应面试里的工程性能题——要有数据（avg/p95），要有根因分析，要有优化证据。

---

## 前置要求

- ROS2：**Jazzy**
- 构建工具：`colcon`

```bash
source /opt/ros/jazzy/setup.bash
colcon build --packages-select w4_demo_pkg
source install/setup.bash
```

---

## 核心概念：数据面和控制面争抢同一执行资源

### 问题根因

在单线程 executor 下：
- 重负载的 topic 订阅回调（数据面）和 service 回调（控制面）**争抢同一个线程**
- 数据面回调执行时间长、频率高，把线程占了
- service 回调被迫排队，RTT（Round-Trip Time）变大

### 优化方案

**并发隔离 + QoS 分级：**

1. **MultiThreadedExecutor**：多线程处理回调，数据面和控制面在不同线程
2. **Callback Group 隔离**：数据面用 MutuallyExclusive，控制面用 Reentrant，避免相互阻塞
3. **QoS 分级**：对高频大消息 topic 用 `BestEffort + KeepLast(1)`，减少积压

---

## Demo 运行方法

### 问题版（bad）

```bash
ros2 run w4_demo_pkg contention_bad_demo
```

**现象：**
- 单线程执行器下，重负载订阅回调和 service 回调争抢同一线程
- 日志里的 `avg / p95` RTT 偏高且波动明显

### 优化版（fixed）

```bash
ros2 run w4_demo_pkg contention_fixed_demo
```

**现象：**
- 多线程执行器 + callback group 隔离后，service RTT 更稳定
- 对重负载 topic 采用 `BestEffort + KeepLast(1)`，减少积压

---

## 面试口述模板（可直接背）

**标准 3 句话：**
1. "我先构造了高频大消息负载，观测 service RTT 的 avg/p95 抖动。"
2. "问题根因是数据面和控制面回调抢同一执行资源，出现排队。"
3. "优化用并发隔离（MultiThreadedExecutor + callback group）和 QoS 分级，最终 RTT 稳定下来。"

**追问："p95 是什么意思？"**
> "95% 分位数，即 100 次请求里有 95 次的响应时间低于这个值。比平均值更能反映真实用户体验，因为大延迟会被单独统计出来。"

**追问："为什么数据面和控制面不能混跑？"**
> "数据面回调特点是量大、耗时长、频繁；控制面 service 特点是要求快、确定性高。混跑会让控制面被数据面饿死，导致 service RTT 抖动。工程上必须隔离。"

---

## 常见坑

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| service RTT p95 偏高 | 单线程 executor 下争抢 | 改 MultiThreadedExecutor |
| 加了多线程还是抖 | callback group 没隔离 | 数据面 MutuallyExclusive，控制面 Reentrant |
| topic 队列积压严重 | KeepLast 队列太大 | 改 KeepLast(1) + BestEffort |

---

## 验证标准

- 能读懂 RTT avg/p95 的含义并解释
- 能画出"数据面和控制面争抢"的示意图
- 能说出 callback group 两种类型的区别和使用场景

---

## 下一步

[W5：Launch 编排与就绪检查](./ros2-jazzy-w5-launch-readiness.md)
