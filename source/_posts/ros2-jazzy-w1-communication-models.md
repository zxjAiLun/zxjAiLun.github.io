---
title: ROS2 Jazzy 面试 demo 集：W1 - Topic/Service/Action 通信模型
date: 2026-03-21
categories:
  - ROS2
  - 工程实践
tags:
  - ROS2
  - Jazzy
  - 面试
  - demo
  - Topic
  - Service
  - Action
---

## 概述

W1 解决的是"能把 Topic / Service / Action 的语义差异讲清楚"这个问题。这是所有 ROS2 工程题的基础——通信模型选错了，后面全是坑。

---

## 前置要求

- ROS2：**Jazzy**
- 构建工具：`colcon`

环境加载（每个新终端都要做）：

```bash
source /opt/ros/jazzy/setup.bash
colcon build
source install/setup.bash
```

---

## 核心概念：三种通信模型的语义差异

### Topic（发布-订阅）

**定位：** 数据流单向分发，不保证请求-响应配对。

Publisher 把消息发布到某个 topic，Subscriber 订阅同一个 topic 收到消息。DDS 按 QoS 尽力送达，**不提供"谁发了→谁该收"的协议语义**。如果业务需要配对，需要在 payload 里带 request_id，在订阅端维护状态。

**适用场景：**
- 传感器数据流（激光雷达、摄像头）
- 状态广播（里程计、坐标变换）
- 任何"持续产生、不关心谁来取"的数据

**不适用：**
- 需要明确"这次调用有没有返回"的 RPC 场景
- 强一致的一次性请求

### Service（服务）

**定位：** 一次请求、一次响应，调用方等待返回结果。

Client 发起调用，Server 处理后返回 response。同步等待 `call()` 会阻塞直到收到结果或超时；异步等待 `call_async()` + future callback 可避免阻塞。

**为什么"回调里同步等待"有风险：**
- 回调在 executor 线程里执行
- 如果回调里同步等待另一个 service 的结果，而那个结果又需要同一 executor 线程来响应
- → 形成死锁

**适用场景：**
- 查参数、查状态、一次性计算
- 调用时长 < 几百毫秒的快速操作

### Action（动作）

**定位：** 带进度反馈的长任务，可取消。

Action 有 Goal-Feedback-Result 三段式语义。客户端发 goal 后可以边等反馈边做别的事，还能主动取消。Service 做不到这一点——它要么等结果要么超时，没有中间状态。

**适用场景：**
- 导航（实时反馈位置、可以取消）
- 机械臂轨迹执行（持续反馈完成度）
- 任何 > 1 秒的长任务

---

## Demo 运行方法

所有 demo 对应同一个业务：计算 `1..n` 的和。

### Topic Demo（开 3 个终端）

```bash
# 终端 A - worker 计算节点
ros2 run w1_demo_pkg sum_topic_worker

# 终端 B - result listener
ros2 run w1_demo_pkg sum_topic_result_listener

# 终端 C - publisher 发起请求
ros2 run w1_demo_pkg sum_topic_publisher
```

### Service Demo（开 2 个终端）

```bash
# 终端 A - server
ros2 run w1_demo_pkg sum_service_server

# 终端 B - client（默认参数 3+4）
ros2 run w1_demo_pkg sum_service_client
```

### Action Demo（开 2 个终端）

```bash
# 终端 A - action server
ros2 run w1_demo_pkg sum_action_server

# 终端 B - action client
ros2 run w1_demo_pkg sum_action_client
```

---

## 面试口述模板（可直接背）

**三分钟版：**
> "同一功能我做过三种通信实现：Topic 更像数据流，DDS 只负责把消息送达，不保证请求-响应配对；Service 是标准的请求-响应语义，调用方必须等这次结果；Action 适合长任务，有 Goal-Feedback-Result 三段式，客户端可以边等反馈边做别的事，还能主动取消。"

**30 秒精华版：**
> "Topic 是发布-订阅，数据流，不保证配对；Service 是一次性 RPC，等结果；Action 是带进度的长任务，可取消。选哪个看任务是否超过秒级、是否需要进度反馈。"

**追问："为什么长任务不用 Service？"**
> "Service 同步等结果会阻塞回调链路，长任务会饿死同线程其他回调甚至死锁。Action 有异步反馈机制，更适合。"

---

## 常见坑

| 症状 | 原因 | 解决方案 |
|------|------|----------|
| `ros2 run` 找不到包 | 没 source install/setup.bash | 每个终端都要 source |
| Topic 收得到但总觉得"不对" | 误用 Topic 做 RPC，没有配对语义 | 业务层带 request_id，或改用 Service |
| Service 调用超时 | 长任务同步等待，阻塞链路 | 改 `call_async` + MultiThreadedExecutor |

---

## 验证标准

- 能用 30 秒说清 Topic/Service/Action 核心差异
- 能解释为什么"长任务用 Action 而不是 Service"
- 能用一条 `ros2` 命令证明某种通信在工作

---

## 下一步

[W2：死锁复现与修复](./ros2-jazzy-w2-deadlock-fix.md)
