---
title: ROS2 Jazzy 面试 demo 集：W2 - 死锁复现与修复
date: 2026-03-22
categories:
  - ROS2
  - 工程实践
tags:
  - ROS2
  - Jazzy
  - 面试
  - demo
  - 死锁
  - 并发
  - Executor
---

## 概述

W2 解决的是"能把回调里同步等待导致死锁的机制讲清楚，并知道怎么修"。这是面试追问高频区——光会说"不要在回调里同步等"不够，要能推导死锁链路。

---

## 前置要求

- ROS2：**Jazzy**
- 构建工具：`colcon`

```bash
source /opt/ros/jazzy/setup.bash
colcon build --packages-select w2_demo_pkg
source install/setup.bash
```

---

## 核心概念：死锁是怎么形成的

### 回调执行链路

ROS2 回调在 **Executor** 线程里执行：

1. 事件到达（service 请求、topic 消息等）
2. Executor 从等待队列取出事件
3. 在同一线程调用对应回调
4. 回调执行完才处理下一个事件

### 死锁的推导链路

假设回调 A 里同步等待 service B 的结果，而 service B 的响应需要回调 A 先执行完才能发送——这不可能发生，因为 A 已经在等 B，B 的回调得不到调度机会：

```
回调 A（持有线程）→ 同步等待 service B 的 response
                              ↑
                    service B 的回调在等待线程
```

这就是"我等你、你等我"——死锁。

### 修复组合拳

```
call_async + MultiThreadedExecutor + ReentrantCallbackGroup
```

- `call_async`：不阻塞当前线程，发起异步调用
- `MultiThreadedExecutor`：多线程处理回调，避免单线程争抢
- `ReentrantCallbackGroup`：允许同一回调组内回调嵌套执行

---

## Demo 运行方法

### 反例（故意触发死锁症状）

```bash
ros2 run w2_demo_pkg deadlock_bad_demo
```

**现象：**
- `sum_a` 回调进入后等待 `sum_b`
- 单线程执行器下，`sum_b` 回调拿不到调度机会
- 最终超时返回 `-1`（死锁症状标记）

### 修复版

```bash
ros2 run w2_demo_pkg deadlock_fixed_demo
```

**现象：**
- 同样是 `sum_a -> sum_b` 调用链
- 多线程 + 合理 callback group 让 `sum_b` 并发执行
- 最终正常返回 `21`

---

## 面试口述模板（可直接背）

**标准 3 句话：**
1. "我先复现了回调内同步等待导致的死锁症状，确保问题可观测。"
2. "修复用的是 `call_async + MultiThreadedExecutor + ReentrantCallbackGroup`，避免阻塞链路。"
3. "修复后从超时失败变为稳定返回，说明并发模型配置正确。"

**追问："为什么回调里不建议同步等待？"**
> "回调在 executor 线程里执行，如果同步等待另一个 service 结果，而那个结果需要同一线程的回调响应，就会形成'我等你、你等我'的死锁。工程里永远用 `call_async` 发起异步调用。"

---

## 常见坑

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| service 调用超时，但没明显报错 | 单线程 executor 下同步等待死锁 | 改 `call_async` + MultiThreadedExecutor |
| 加了多线程还是死锁 | callback group 是 MutuallyExclusive | 改 ReentrantCallbackGroup |
| 回调里用 `get()` 查 future 状态 | 同步等待，同一条线程 | 改 `future.add_done_callback()` 异步响应 |

---

## 验证标准

- 能画出死锁链路图（回调 A → 同步等 B，B 回调等 A）
- 能说出修复三要素及各自作用
- 能独立回答"为什么回调里不建议同步等待"

---

## 下一步

[W3：QoS/DDS 行为](./ros2-jazzy-w3-qos-dds-behavior.md)
