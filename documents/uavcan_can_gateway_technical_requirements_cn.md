# 基于 libcanard + SocketCAN 的 UavcanCanGateway 软件技术需求文档

## 1. 文档信息

| 项目 | 内容 |
|---|---|
| 模块名称 | UavcanCanGateway |
| 软件定位 | EROS HAL Bus-Gateway 系列库之一，为 HAL 组件及非 HAL 硬件通讯程序提供 UAVCAN/CAN 总线接入能力 |
| 通讯协议 | UAVCAN/Cyphal over CAN |
| CAN 驱动 | SocketCAN |
| UAVCAN 协议栈 | libcanard |
| 运行平台 | Linux |
| 目标语言 | C++20 |
| Code Style | Google C++ Style |
| 文档版本 | v1.0 |
| 日期 | 2026-05-11 |

---

## 2. 架构定位与核心设计原则

### 2.1 UavcanCanGateway 的架构定位

UavcanCanGateway **不是 CAN 总线的数据垄断者**，而是 UAVCAN 网络中的**协调者、身份代理、注册中心**。

CAN 是广播型总线，SocketCAN 内核驱动天然支持多进程同时监听同一接口。UavcanCanGateway 的设计尊重这一物理特性：HAL 组件可直接通过 libcanard + SocketCAN 收发数据，无需强制经过 Gateway 中转；Gateway 的核心价值在于**管理 UAVCAN 节点身份、统一分配端口、监控总线健康**，而非拦截所有数据流。

**Gateway 协调的职责**：
- UAVCAN NodeID 身份管理（心跳、GetInfo、PortList）
- Subject-ID / Service-ID 的集中注册与防冲突分配
- CAN 总线参数配置（波特率、模式、FD 开关）
- 总线健康监控（Bus-Off、错误计数、负载率）
- 外部节点发现、健康监测、OTA 管理
- 总线时间同步
- 跨协议转换（ROS2/DDS ↔ UAVCAN，供非 HAL 组件使用）

**Client Library 的两种接入模式**：

```text
模式 A：CAN 直连（推荐用于 HAL / 实时组件）
  HAL 组件 → libcanard + SocketCAN → CAN 总线
            ↓
         IPC（向 Gateway 注册端口、获取配置）

模式 B：IPC 中转（用于非 HAL / 无 CAN 权限组件）
  业务组件 → Client Library → IPC → Gateway → libcanard + SocketCAN → CAN 总线
```

---

## 3. 命名规范与架构定位

### 3.1 统一命名规范

**格式**：[上层协议] + [物理总线] + [Gateway/Bridge]

| 模块名 | 含义 | 适用场景 |
|---|---|---|
| **UavcanCanGateway** | UAVCAN 协议 over CAN 总线 网关 | 本模块，总线接入、路由、和机器人 OS 内部消息总线对接 |
| UavcanEthGateway | UAVCAN 以太网网关 | 后续扩展 |
| UavcanSerialGateway | UAVCAN 串口网关 | 后续扩展 |
| ModbusCanGateway | Modbus 转 CAN 网关 | 后续扩展 |
| DdsCanGateway | DDS 与 CAN 总线网关 | 后续扩展 |

**Bridge vs Gateway 选择原则**：
- **Bridge**：涉及跨协议转发、同异构总线协议桥接
- **Gateway**：作为总线接入汇聚、路由转发节点

### 3.2 核心架构原则

**UavcanCanGateway 是 UAVCAN 网络中的协调者，而非数据垄断者。**

**为什么需要 Gateway 集中协调**：

| 维度 | 说明 |
|---|---|
| UAVCAN 节点身份唯一性 | 机器人操作系统在 UAVCAN 网络中呈现为单一逻辑节点。Gateway 负责发送心跳、GetInfo、PortList，代表整个系统的 UAVCAN 身份 |
| 端口分配集中化 | Subject-ID / Service-ID 由 Gateway 统一注册分配，确保同 NodeID 下各应用端口不重叠，避免 Transfer-ID 冲突 |
| 总线状态统一决策 | Bus-Off 恢复、错误帧处理、总线负载监控、静默/监听模式切换，必须由单一决策点执行 |
| 系统抽象与解耦 | 非 HAL 组件通过 IPC 接入，不感知底层 CAN/UAVCAN 细节；切换底层总线无需改应用代码 |
| 故障隔离 | 网关独立进程，复杂逻辑集中处理；单个 App 崩溃不影响 CAN 硬件状态和其他 App 通信 |

---

## 4. 项目背景

机器人系统通常包含多个 MCU、驱动器、电池管理系统、传感器、IO 模块以及上位控制器。这些设备大量使用 CAN/CANFD 作为底层现场总线，通过 UAVCAN/Cyphal 协议进行通信。

**UavcanCanGateway** 属于 EROS HAL Bus-Gateway 系列库，该系列库为 HAL 组件以及不通过 HAL 与硬件直接通讯的程序提供总线接入支撑。Bus-Gateway 的典型实现包含一个 **Gateway 模块**（服务端，负责总线硬件管理、协议栈运行、消息路由转发）和一个 **Gateway Client Library**（客户端库，供应用程序调用以收发总线消息）。

**UavcanCanGateway** 作为该系列中面向 UAVCAN/CAN 的具体实现，承担 UAVCAN 网络中的**协调者、身份代理、注册中心**职责，是 UAVCAN/CAN 网络接入机器人主控的核心边界模块。

核心职责：

- **UAVCAN 节点身份代理**：代表机器人 OS 发送心跳、GetInfo、PortList，维护单一 NodeID 身份
- **端口注册中心**：集中管理 Subject-ID / Service-ID 的分配与注册，防止同 NodeID 下各应用端口冲突
- **CAN 总线配置管理**：波特率、模式、FD 开关等全局参数的统一配置
- **总线健康监控**：Bus-Off 恢复、错误帧处理、负载率监控
- **外部节点管理**：自动发现、心跳监测、掉线检测、OTA
- **跨协议转换**：将 UAVCAN 消息转换为 ROS2/DDS 消息，供非 HAL 组件消费
- **多 CAN 口管理**：支持多路 CAN 总线并发
- **高频数据协调**：为 HAL 组件提供 CAN 直连通道，为非 HAL 组件提供 IPC 低延迟通道
- **提供日志、诊断、监控能力**：工业级可观测性
- **提供高可靠、高实时性的总线通讯能力**

本项目基于：

- SocketCAN（CAN 硬件抽象）
- libcanard（UAVCAN 协议栈）
- Linux epoll/eventfd/timerfd
- 多线程 Reactor 架构
- 机器人 OS 消息总线（ROS2/DDS/Cyber 等）

---

## 5. 总体目标

### 4.1 功能目标

支持：

- CAN2.0/CANFD 物理层收发
- UAVCAN v1 / Cyphal 协议编解码
- 多 CAN 口管理（>= 16 路）
- UAVCAN 节点自动发现与枚举
- 发布/订阅（Pub/Sub）协议适配
- RPC 服务代理（请求/响应）
- 时间同步
- 参数服务
- 固件升级
- 日志采集
- 诊断信息
- 总线统计与监控
- 高频数据专用通道（IMU 等 1kHz 数据分流）

### 4.2 性能目标

| 指标 | 目标 |
|---|---|
| 单总线吞吐 | >= 8000 frames/s |
| 最大 CAN 通道数 | >= 16 |
| 单进程节点数 | >= 512 |
| Topic 延迟（普通消息） | < 5ms |
| 高频数据延迟（专用通道） | < 1ms |
| CPU 占用 | < 20%（4 总线典型负载） |
| 内存占用 | < 256MB |

### 4.3 可靠性目标

- 总线异常自动恢复
- 节点失联检测（心跳超时）
- 支持 watchdog
- 支持热插拔
- 支持进程级故障隔离
- 单节点故障不影响整总线
- 支持异常日志追踪

---

## 6. 系统架构

### 5.1 总体架构

```text
+------------------------------------------------+
|              HAL Components                    |
|    电机驱动 / IMU 采集 / 编码器读取 / 伺服控制   |
+------------------------+-----------------------+
                         |
          +--------------+--------------+
          |                             |
          v                             v
+------------------------------------------------+
|  模式 A：CAN 直连              |  模式 B：IPC 中转        |
|  libcanard + SocketCAN        |  Client Library → IPC    |
|  （高频传感数据最小延迟路径）    |  （非 HAL / 无权限组件）  |
+------------------------+-----------------------+
                         |                             |
                         v                             v
+------------------------+-----------------------+
|           机器人 OS 内部消息总线（可选）          |
|         ROS2 / DDS / Cyber / RPC               |
+------------------------+-----------------------+
                         |
                         v
+------------------------------------------------+
|         UavcanCanGateway (Gateway 模块)        |
|  UAVCAN 协调者 + 身份代理 + 注册中心             |
|                                                |
| +--------------------------------------------+ |
| | PortRegistry（端口注册中心）                 | |
| | - Subject-ID / Service-ID 分配与注册         | |
| | - 端口冲突检测                               | |
| | - PortList 统一发布                          | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | NodeManager（节点身份管理）                  | |
| | - 心跳发送 / GetInfo 响应                    | |
| | - 外部节点自动发现 / 心跳监测 / 状态缓存      | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | BusManager（总线管理）                       | |
| | - CAN 接口配置（波特率/模式/FD）             | |
| | - Bus-Off 恢复 / 错误帧处理                  | |
| | - 总线负载监控                               | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | MessageRouter（跨协议消息路由）              | |
| | - 静态话题映射表（OS ↔ UAVCAN）              | |
| | - 动态路由转发                               | |
| | - 数据转换适配                               | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | UpgradeManager（OTA 管理）                   | |
| | - 固件升级 / Bootloader 升级                 | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | LibcanardAdapter（UAVCAN 协议栈封装）        | |
| | - libcanard 实例（Gateway 自身及 IPC 路径）  | |
| | - 内存池管理                                 | |
| +--------------------------------------------+ |
|                                                |
| +--------------------------------------------+ |
| | SocketCANDriver（CAN 硬件驱动）              | |
| | - 多 CAN 口管理                              | |
| | - 共享硬件访问（SocketCAN 允许多进程监听）   | |
| +--------------------------------------------+ |
+------------------------------------------------+
                         |
                         v
+------------------------------------------------+
|               CAN / CANFD BUS                  |
|    电机 / IMU / 电调 / 传感器 / BMS / IO      |
+------------------------------------------------+
```

### 5.2 数据流架构

**模式 A：CAN 直连（HAL 组件 ↔ CAN 总线）**

```text
发送路径（HAL → CAN）：
  HAL 组件（已注册端口）
    → libcanard 编码（应用自行维护 TID）
      → SocketCAN 发送
        → CAN 总线

接收路径（CAN → HAL）：
  CAN 总线帧
    → SocketCAN 接收（内核广播到所有监听进程）
      → libcanard 解码
        → HAL 组件直接消费
```

**模式 B：IPC 中转（非 HAL 组件 ↔ CAN 总线）**

```text
发送路径（App → CAN）：
  App 通过 Client Library 发布消息
    → IPC → UavcanCanGateway 接收
      → MessageRouter 查映射表 / 协议封装
        → LibcanardAdapter 编码
          → TxScheduler 优先级队列
            → SocketCANDriver 发送
              → CAN 总线

接收路径（CAN → App）：
  CAN 总线帧
    → SocketCANDriver 接收
      → RxDispatcher 分发
        → TransferAssembler 组包
          → LibcanardAdapter 解码
            → MessageRouter 路由决策 / 协议解包
              → IPC → Client Library → App
```

---

## 7. 核心顶层能力

### 6.1 UAVCAN 节点身份代理

- 网关代表机器人 OS 作为 UAVCAN 网络中的单一逻辑节点，占用固定 Node-ID
- 负责发送 UAVCAN 心跳（Heartbeat）、响应 GetInfo 请求、发布 PortList
- 支持 Node-ID 静态配置 / 动态自动分配
- HAL 组件通过 CAN 直连收发数据时，不发送系统级广播，身份由 Gateway 统一代理

### 6.2 端口注册与管理

- HAL 组件及非 HAL 组件向 Gateway 注册需要使用的 Subject-ID / Service-ID
- Gateway 负责冲突检测与分配，确保同一 NodeID 下各应用端口不重叠
- Gateway 汇总所有已注册端口，统一生成并发送 `uavcan.node.PortList`
- 应用启动前必须完成端口注册，运行时动态增删端口需向 Gateway 申请

### 6.3 双向协议透传转发（IPC 路径）

- 非 HAL 组件通过 IPC 接入：机器人 OS 内部消息 → Gateway 封装为 UAVCAN 帧 → CAN 总线
- CAN 总线收到 UAVCAN 帧 → Gateway 解包 → IPC → 非 HAL 组件
- 应用通过 Client Library 接入，不感知底层 CAN/UAVCAN 细节

### 6.4 多设备总线汇聚与监控

- 统一发现 CAN 总线上所有 UAVCAN 外设，维护节点状态机
- 外部节点心跳监测、掉线检测、健康状态上报
- 总线负载率、错误计数、Bus-Off 状态统一监控
- 节点发现、心跳丢失、健康状态变更等产生系统事件，供机器人 OS 事件总线消费

---

## 8. UAVCAN 协议栈层必备功能

### 7.1 UAVCAN 标准帧编解码

- CAN 2.0A/B 帧与 UAVCAN 传输帧互转
- 支持 UAVCAN 基础数据类型、服务（请求/响应）
- 支持 DSDL 数据类型解析与绑定
- 预加载 UAVCAN DSDL 定义，自动匹配消息类型、校验数据结构合法性

### 7.2 发布/订阅（Pub/Sub）协议适配

- 映射机器人 OS 话题 ↔ UAVCAN 广播消息（Subject-ID 映射）
- 订阅远端 UAVCAN 节点消息，转发到本地 OS 话题
- 支持动态订阅/取消订阅、Topic filtering、port-id routing

### 7.3 RPC 服务代理

- 机器人 OS 内部服务调用 → 转为 UAVCAN Service Request
- 接收 UAVCAN Service Response → 回传给本地 OS 服务端
- 支持超时、重试、并发调用管理

### 7.4 UAVCAN 节点自动枚举与在线发现

- 自动扫描 CAN 总线上在线 UAVCAN 节点
- 上报节点 ID、设备类型、固件版本、健康状态、工作模式给机器人 OS
- 节点状态机：UNKNOWN → DISCOVERED → ONLINE → OFFLINE

---

## 9. CAN 总线链路层功能

### 8.1 CAN 硬件适配抽象

- 兼容 CAN2.0A / CAN2.0B
- 支持标准帧、扩展帧
- 适配不同 CAN 控制器/外设（SocketCAN、硬件 CAN、虚拟 CAN vcan）
- **共享访问**：SocketCAN 内核驱动天然支持多进程同时监听同一接口；Gateway 不垄断 CAN 硬件，HAL 组件可直接 libcanard + SocketCAN 收发

### 8.2 CAN 总线参数配置

- 波特率配置：125k/250k/500k/1M 可配置
- 工作模式：正常模式、静默监听模式、回环测试模式
- CANFD 支持：数据段波特率、MTU 配置

### 8.3 CAN 收发队列管理

- 接收缓冲队列、发送优先级队列
- 防溢出、队列水位阈值控制
- 批量发送优化

### 8.4 总线监听与旁路抓包

- 可选旁路监听所有 CAN 报文，不干扰正常业务
- 原始 CAN 帧录制、回放（调试用）
- 报文注入：从 OS 侧下发自定义 CAN/UAVCAN 帧，用于测试外设

---

## 10. 消息映射与路由核心功能（最重要）

### 9.1 静态话题映射表

配置文件定义：

```yaml
mappings:
  - os_topic: "/uavcan/servo/setpoint"
    uavcan_subject_id: 1001
    data_type: "reg.udral.physics.electricity"
    direction: "os_to_can"

  - os_topic: "/uavcan/imu/raw"
    uavcan_subject_id: 2001
    data_type: "uavcan.si.unit.acceleration"
    direction: "can_to_os"
    high_freq: true  # 标记为高频，走专用通道
```

### 9.2 动态路由转发

- 按消息类型、NodeID 做定向转发
- 支持过滤：只转发指定设备、指定消息类型
- 支持黑名单/白名单机制

### 9.3 数据转换适配

- 单位换算、量程映射、数据截断/补齐
- 字节序、大小端统一适配
- DSDL 类型到 OS 消息类型的自动转换

### 9.4 流量限流与降噪

- 高频 UAVCAN 消息降频转发到 OS（如 1kHz IMU 降频到 100Hz 给普通应用）
- 限流、丢包策略可配置
- 关键控制指令优先队列保障

---

## 11. 高频数据接入策略

### 10.1 分层分流设计

CAN 是广播型总线，SocketCAN 支持多进程同时监听。UavcanCanGateway 的高频数据策略尊重这一物理特性：

```text
最低延迟路径（HAL 组件）：
  CAN → SocketCAN → libcanard → HAL 组件（直接消费，零中转）

中等延迟路径（非 HAL 实时组件）：
  CAN → Gateway → IPC → Client Library → 实时组件

普通延迟路径（控制/诊断/可视化）：
  CAN → Gateway → 机器人 OS 消息总线 → 普通应用（可配合降频）
```

### 10.2 HAL 组件直接接入

对于 IMU、编码器等高频传感器数据：
- **HAL 组件直接 libcanard + SocketCAN 接收**：内核将 CAN 帧广播到所有监听进程，HAL 组件直接解码消费，不经过任何 IPC 或 Gateway 中转
- **延迟最优**：路径最短，无序列化、无进程切换、无队列排队
- **前提约束**：HAL 组件必须向 Gateway 注册所使用的 Subject-ID，不发送心跳/PortList 等系统广播

### 10.3 非 HAL 组件高频接入

对于无法直接访问 CAN 的非 HAL 组件：
- Gateway 通过 IPC 分发高频数据
- 零拷贝传输，无序列化开销
- 同时支持降频后发布到 ROS2/DDS 消息总线，供普通应用消费

---

## 12. 状态管理、故障与安全

### 11.1 网关自身状态机

```text
INIT
  -> STARTING
      -> RUNNING
          -> DEGRADED（部分总线故障）
              -> RECOVERING
                  -> RUNNING

RUNNING
  -> STOPPING
      -> STOPPED
```

状态流转要求：

| 状态 | 说明 |
|---|---|
| INIT | 初始化配置、加载 DSDL、创建内存池 |
| STARTING | 启动 CAN 接口、注册 UAVCAN 节点、建立订阅 |
| RUNNING | 正常运行，全功能可用 |
| DEGRADED | 部分 CAN 总线故障或节点大面积离线，核心功能降级运行 |
| RECOVERING | 自动恢复中（CAN 接口重启、节点重新枚举） |
| STOPPING | 优雅关闭中，完成发送队列、通知节点离线 |
| STOPPED | 完全停止 |

### 11.2 CAN 总线故障检测

- 总线离线、总线关闭（Bus-Off）
- 错误计数超限（TX/RX Error Counter）
- 节点掉线、心跳超时检测（默认 1s 心跳，3s 超时，连续丢失 3 次判定离线）
- 总线负载率监控，超阈值告警

### 11.3 容错与降级

- 单节点故障不影响整总线
- 总线异常自动重启 CAN 接口、自动重连
- 丢包、重复帧去重（序号校验、时间戳去重）
- 丢失告警上报机器人 OS
- RX OOM：丢弃低优先级消息
- TX OOM：限流

---

## 13. 配置、日志、运维

### 12.1 可动态配置

- NodeID、波特率、映射表、转发规则
- 支持启动加载配置、运行时热配置（Reload）
- 配置方式：YAML / JSON / 环境变量

### 12.2 日志与监控

- 原始 CAN 日志、UAVCAN 协议日志、转发日志
- 统计：收发帧数、错误数、丢包率、总线负载率、队列深度
- 多级日志：TRACE / DEBUG / INFO / WARN / ERROR / FATAL
- 异步写盘、RingBuffer、日志压缩

### 12.3 运维接口

- 提供 OS 内部服务：查询总线状态、节点列表、错误统计
- 支持远程复位网关、重启 CAN 链路
- Metrics 导出：Prometheus / JSON Export / REST API

```text
关键 Metrics：
  uavcan_can_rx_frames_total
  uavcan_can_tx_frames_total
  uavcan_can_rx_errors_total
  uavcan_nodes_online
  uavcan_bus_load
  uavcan_gateway_state
```

---

## 14. 调试与仿真功能

### 13.1 虚拟 UAVCAN 节点模拟

- 可模拟外设发帧，不用接真实硬件调试
- 支持模拟多种设备类型（电机、IMU、传感器）

### 13.2 报文注入

- 从 OS 侧下发自定义 CAN/UAVCAN 帧，用于测试外设
- 支持单帧注入、周期性注入、批量注入

### 13.3 离线回放

- 录制 CAN 报文，离线回放复现问题
- 支持时间戳精确回放、倍速回放

---

## 15. 多总线架构设计

### 15.1 设计目标

- 多 CAN 口并发（>= 16 路）
- 总线隔离（故障不扩散）
- 动态上线/下线
- 不同波特率、不同 MTU

### 15.2 线程模型

Gateway 内部线程模型如下：

```text
+-------------------+
| Main Thread       |  基于 cytoskeleton::itc::message_queue::Looper
|                   |  - 生命周期管理、配置热加载
|                   |  - 端口注册处理、Client IPC 控制消息
|                   |  - 心跳定时发送（PostDelayed）
|                   |  - 总线状态机转换
+-------------------+

+-------------------+
| BusThread can0    |  独立 epoll、独立 TX/RX queue、独立统计
+-------------------+

+-------------------+
| BusThread can1    |
+-------------------+

+-------------------+
| BusThread canN    |
+-------------------+

+-------------------+
| WorkerThread Pool |  消息处理、协议编解码
+-------------------+

+-------------------+
| MonitorThread     |  监控统计、Metrics 上报
+-------------------+
```

**Main Thread 基于 cytoskeleton message_queue**：

Gateway 的 Main Thread 采用 `cytoskeleton::itc::message_queue::Looper` 作为消息循环基础设施：
- 所有非实时控制逻辑（配置变更、端口注册、Client 连接管理、心跳定时）通过 `Post` / `PostDelayed` 投递到 Main Thread 串行处理
- BusThread 接收到的 CAN 帧若需 Gateway 处理（如节点发现、IPC 分发），通过 `Post` 投递到 Main Thread，避免 BusThread 阻塞
- 同步查询（如 Client 请求获取当前节点列表）通过 `Invoke` 在 Main Thread 执行并等待返回
- 日志输出使用 **spdlog 异步后端**，不占用独立线程

**高频数据处理**：
- HAL 组件通过 CAN 直连直接消费高频数据，不经过 Gateway
- IPC 模式的高频数据由 Gateway 直接分发，无需独立处理线程

**日志处理**：
- 统一使用 spdlog 异步日志（`spdlog::async_logger` + `spdlog::details::thread_pool`），Gateway 内部不维护独立的日志写入线程

每个 BusThread：

- 独立 epoll
- 独立 TX queue / RX queue
- 独立 libcanard 实例（或共享但分区）
- 独立统计
- 避免跨总线锁竞争

---

## 16. SocketCAN 驱动层设计

### 16.1 功能要求

- CAN RAW Socket
- CANFD
- 非阻塞 IO
- epoll 集成
- 自动重连
- Bus-Off 恢复
- 错误帧处理

### 16.2 初始化流程

```text
socket(PF_CAN, SOCK_RAW, CAN_RAW)
    -> bind(canX)
    -> setsockopt(CAN_RAW_FD_FRAMES)
    -> O_NONBLOCK
    -> epoll_ctl
```

### 16.3 数据接收流程

```text
epoll_wait
    -> recvmsg
        -> can_frame/canfd_frame
            -> RxDispatcher
```

### 16.4 数据发送流程

```text
TxQueue
    -> write/sendmsg
        -> SocketCAN
```

### 16.5 错误处理

| 错误类型 | 处理方式 |
|---|---|
| Bus-Off | 自动恢复（退出 Bus-Off 后重新初始化） |
| TX Timeout | 重试，超次数告警 |
| RX Overflow | 统计 + 告警 + 丢弃低优先级帧 |
| Socket Error | 自动重建 socket |
| Interface Down | 周期检测恢复 |

---

## 17. libcanard 适配层设计

### 17.1 功能要求

封装：

- CanardInstance（Gateway 自身及 IPC 路径的 libcanard 实例；HAL 组件直连时可拥有独立实例）
- Memory allocator（固定内存池）
- Transfer receive
- Transfer transmit
- Subscription 管理
- Service client/server

### 17.2 内存管理

采用固定内存池：

```text
+----------------------+
| Fixed Memory Pool    |
+----------------------+
        |
        +--> RX transfer
        +--> TX queue
        +--> Fragment buffer
```

要求：

- 禁止运行期频繁 malloc/free
- 避免内存碎片
- 支持 OOM 检测
- 支持水位统计

### 17.3 Transfer 接收

流程：

```text
CAN Frame
    -> canardRxAccept
        -> TransferAssembler
            -> Topic/Service Dispatch
```

### 17.4 Transfer 发送

流程：

```text
Application Message
    -> canardTxPush
        -> TxQueue
            -> SocketCAN
```

---

## 18. NodeManager 设计

### 18.1 功能

负责：

- 节点发现（自动枚举 CAN 总线上所有 UAVCAN 节点）
- 节点状态维护
- 心跳检测（网关自身心跳发送 + 外设节点心跳监测）
- 节点上下线管理
- 节点能力缓存（设备类型、固件版本、健康状态）
- 固件版本检查与 OTA 触发

### 18.2 节点发现机制

Gateway 通过以下方式主动发现和识别 UAVCAN 节点：

**1. 被动监听（Passive Discovery）**
- 监听总线上所有 UAVCAN 心跳帧（`uavcan.node.Heartbeat`，Subject-ID 7509）
- 提取 Source NodeID，记录心跳时间戳
- 新 NodeID 首次出现时，状态设为 `DISCOVERED`

**2. 主动查询（Active Enumeration）**
- Gateway 定期向总线广播 `uavcan.node.GetInfo` 服务请求（Service-ID 430）
- 收到响应后提取节点信息：
  - 节点名称（`uavcan.node.GetInfo.Response.name`）
  - 软件版本（`software_version.major / .minor`）
  - 硬件版本（`hardware_version.major / .minor`）
  - 证书哈希（`certificate_of_authenticity`）
  - 端口列表（后续通过 PortList Subject 7509 获取）

**3. 固件版本比对**
- 节点进入 `DISCOVERED` 状态后，Gateway 立即查询 `/opt/cosmos/lib/firmware/` 中该节点对应的最新固件版本
- 若当前固件版本低于仓库最新版本，自动创建 OTA 升级任务
- 若版本已最新，直接进入 `CONFIGURATING` 状态

### 18.3 节点状态机

```text
UNKNOWN
   |
   v
DISCOVERED          <- 首次收到心跳 / GetInfo 响应
   |
   +---> [DAG 依赖检查] --前置依赖未满足-->
   |                                      |
   |                              WAITING_FOR_DEPENDENCY
   |                                      |
   |   [前置依赖完成] -------------------->+
   |                                      |
   +---> [版本检查] --需要更新--> OTA 流程
   |                                 |
   |   PENDING                       |
   |        |                        |
   |   DOWNLOADING                   |
   |        |                        |
   |   VERIFYING                     |
   |        |                        |
   |      READY                      |
   |        |                        |
   |   ACTIVATING                    |
   |        |                        |
   |   REBOOTING                     |
   |        |                        |
   |   CONFIRMING                    |
   |        |                        |
   |   UPGRADE_FIRMWARE_SUCCESS      |
   |        |                        |
   +--------+------------------------+
            v
      CONFIGURATING     <- OTA 完成或无需更新，加载节点运行参数配置
            |
            v
         ACTIVE         <- 配置完成，允许客户端通信
            |
            +----> OFFLINE（心跳超时）
```

**状态说明**：

| 状态 | 说明 |
|---|---|
| `UNKNOWN` | 节点从未在总线上出现 |
| `DISCOVERED` | 首次发现节点，正在进行版本检查和身份识别 |
| `WAITING_FOR_DEPENDENCY` | DAG 前置依赖节点尚未完成升级，暂停本节点的 OTA 流程 |
| `PENDING` | OTA 任务已创建，固件已定位，等待调度 |
| `DOWNLOADING` | 正在向节点传输固件 |
| `VERIFYING` | 节点对接收到的固件进行校验 |
| `READY` | 固件校验通过，等待激活 |
| `ACTIVATING` | 节点正在切换活动镜像 |
| `REBOOTING` | 节点已重启，Gateway 监测心跳恢复 |
| `CONFIRMING` | 心跳恢复，版本号确认中 |
| `UPGRADE_FIRMWARE_SUCCESS` | 固件升级成功完成 |
| `CONFIGURATING` | 固件已就绪，正在加载节点运行参数配置 |
| `ACTIVE` | 节点完全就绪，允许上层应用通信 |
| `OFFLINE` | 心跳超时，节点失联 |

**通信权限控制**：

Gateway 通过分层机制控制对非 `ACTIVE` 节点的通信：

| 接入模式 | 拦截机制 | 说明 |
|---|---|---|
| **IPC 模式** | Gateway 物理拦截 | 所有数据经过 Gateway，Gateway 在转发前检查目标节点状态，非 `ACTIVE` 直接丢弃并返回 `ClientError::kNodeNotActive` |
| **CAN 直连模式** | 端口注册拦截 + PortList 机制 | Gateway 在端口注册阶段检查目标节点状态；非 `ACTIVE` 时拒绝注册该节点相关端口。已注册端口的 Client 需定期查询 PortList 获取节点可用性，自行决定是否发送 |

**端口注册拦截**：
- Client 调用 `RegisterPublisher(target_node_id, subject_id)` 或 `RegisterServiceClient(target_node_id, service_id)` 时，Gateway 检查 `target_node_id` 状态
- 若目标节点不在 `ACTIVE` 状态，注册被拒绝，返回 `ClientError::kNodeNotActive`
- Client 需等待节点进入 `ACTIVE` 后重新注册，或订阅系统事件获知节点就绪

**PortList 机制**：
- Gateway 发布的 `uavcan.node.PortList` 仅包含 `ACTIVE` 节点的端口信息
- Client 通过读取 PortList 即可判断哪些节点当前可用，避免向未就绪节点发送数据
- 节点状态变更时，Gateway 立即更新并广播 PortList

**注意**：CAN 直连模式下，已持有有效端口注册的 Client 在目标节点掉线后仍可继续发送。Gateway 通过心跳超时检测将节点标记为 `OFFLINE`，同时移除该节点在 PortList 中的条目，Client 应在下一轮 PortList 更新后自行停止发送。

### 18.4 节点信息

| 字段 | 描述 |
|---|---|
| Node-ID | 节点 ID |
| HW Version | 硬件版本 |
| SW Version | 软件版本 |
| Vendor | 厂商 |
| Heartbeat | 心跳时间 |
| Health | 健康状态 |
| Mode | 工作模式 |

### 18.5 心跳检测

默认：

- 网关自身心跳周期：1s
- 外设心跳超时：3s
- Offline 判定：连续丢失 3 次

---

## 19. 固件升级设计（OTA）

### 19.1 设计参考

本 OTA 机制参考 Linux 内核固件加载子系统（`request_firmware` / `firmware_class`）的设计哲学：

- **固件仓库集中管理**：类似 `/lib/firmware/`，Gateway 维护集中式固件仓库，按设备类型和版本分层存储
- **异步加载模型**：升级请求非阻塞，Gateway 后台完成固件传输、验证、激活，调用方通过状态查询或事件获取结果
- **验证后再激活**：固件写入目标节点后必须完成 CRC/签名验证，验证通过才允许切换活动镜像
- **双镜像与回滚**：目标节点保留旧版本镜像，新版本激活失败时自动或手动回滚
- **缓存复用**：同一固件文件在 Gateway 本地缓存，批量升级同一型号节点时无需重复上传

### 19.2 固件仓库（Firmware Repository）

Gateway 维护本地固件仓库，路径示例：

```text
/opt/cosmos/lib/firmware/
├── manifest.yaml              # 仓库索引
├── vendor_a/
│   ├── motor_controller/
│   │   ├── hw_v1/
│   │   │   ├── v2.1.0.bin
│   │   │   ├── v2.1.0.bin.sig
│   │   │   └── v2.1.0.json    # 元数据
│   │   └── hw_v2/
│   └── imu_sensor/
└── vendor_b/
    └── bms_controller/
```

**固件元数据（`.json`）**：

| 字段 | 说明 |
|---|---|
| `version` | 语义化版本，如 `2.1.0` |
| `target_node_id` | 兼容的 UAVCAN NodeID 范围（可选） |
| `hardware_version` | 目标硬件版本 |
| `dsdl_signature` | DSDL 接口签名哈希，确保协议兼容 |
| `size` | 固件文件大小（字节） |
| `crc32` | 文件 CRC32 |
| `sha256` | 文件 SHA256 |
| `signature` | 固件签名（厂商私钥签名，Gateway 公钥验证） |
| `min_bootloader_version` | 最低 Bootloader 版本要求 |

**仓库索引（`manifest.yaml`）**：
- 记录所有可用固件条目的快速索引
- Gateway 启动时加载，支持运行时热更新

### 19.3 升级状态机

目标节点的一次升级会话经历以下状态：

```text
PENDING
  └── 节点发现后自动触发，固件已定位，等待调度
        |
        v
DOWNLOADING
  └── 通过 UAVCAN File 服务或自定义 Chunk 协议向目标节点传输固件
        |
        +---> CANCELLED（调用方取消 / 更高优先级任务抢占）
        |
        v
VERIFYING
  └── 目标节点对接收到的固件进行 CRC / 签名 / 版本校验
        |
        +---> FAILED_VERIFY（校验不通过，任务失败）
        |
        v
READY
  └── 校验通过，新固件已写入非活动镜像区，等待激活指令
        |
        v
ACTIVATING
  └── 目标节点切换活动镜像指针，标记新固件为下次启动项
        |
        v
REBOOTING
  └── 目标节点重启，Gateway 监测心跳恢复
        |
        +---> TIMEOUT（心跳未在预期时间内恢复）
        +---> ROLLBACK（心跳恢复但版本不匹配，或节点上报启动失败）
        |
        v
CONFIRMING
  └── 心跳恢复，GetInfo 响应中的版本号与目标版本一致
        |
        v
UPGRADE_FIRMWARE_SUCCESS
  └── 固件升级成功完成，节点进入可用状态
        |
        v
CONFIGURATING
  └── Gateway 向节点下发运行参数配置（如电机 PID、IMU 校准参数等）
        |
        +---> FAILED_CONFIG（配置下发失败，可重试）
        |
        v
ACTIVE
  └── 节点完全就绪，允许客户端通信
```

**正常流程（无需更新固件）**：

```text
DISCOVERED
   |
   +---> [版本检查] --版本已最新-->
                                     |
                                     v
                              CONFIGURATING
                                     |
                                     v
                                   ACTIVE
```

**回滚（Rollback）**：
- 从 `REBOOTING` 或 `CONFIRMING` 可进入 `ROLLBACK`
- Gateway 向目标节点发送回滚指令（如支持），或目标节点 Bootloader 自动检测到启动失败后回退
- 回滚成功后节点恢复旧版本运行，升级任务标记为 `FAILED_ROLLBACK`

**通信拦截**：
- 节点在 `DOWNLOADING` / `VERIFYING` / `READY` / `ACTIVATING` / `REBOOTING` / `CONFIRMING` / `CONFIGURATING` 状态时，Gateway **拒绝** Client Library 向该节点发起的 `Publish` 和 `CallService`
- 只有状态为 `ACTIVE` 时，Client 才能与该节点正常通信

### 19.4 固件传输协议

Gateway 与目标节点之间的固件传输基于 **UAVCAN File 服务**（`uavcan.file.Read` / `uavcan.file.Write`）或自定义 Chunk 传输。

**Chunk 传输设计（自定义协议，兼容低资源节点）**：

| 参数 | 说明 |
|---|---|
| Chunk Size | 256 字节（适配 UAVCAN CAN 帧 MTU） |
| Sliding Window | 4 个 Chunk（允许接收方乱序缓存） |
| ACK 机制 | 每收到一个 Window 发送累积 ACK |
| Retry | ACK 超时 500ms 重传，最多 3 次 |
| Timeout | 整个传输会话 30s 无进展判定失败 |
| Resume | 支持断点续传，从上次 ACK 的 Offset 继续 |

**传输流程**：

```text
Gateway                            Target Node
   |                                     |
   |--- EnterBootloader (ExecuteCommand) ->|
   |                                     |
   |<-- Bootloader Ready (Heartbeat) -----|
   |                                     |
   |--- FileWrite.Chunk(offset=0) ------->|
   |--- FileWrite.Chunk(offset=256) ----->|
   |--- FileWrite.Chunk(offset=512) ----->|
   |<-- ACK(offset=768) ------------------|
   |                                     |
   |--- ... (continue) ----------------->|
   |                                     |
   |--- Finalize (ExecuteCommand) ------->|
   |<-- Verify OK ------------------------|
   |                                     |
   |--- Activate (ExecuteCommand) -------->|
   |<-- Rebooting ------------------------|
   |                                     |
   |<-- Heartbeat (new version) ----------|
```

### 19.5 批量升级策略

**节点分组**：
- 按设备类型、硬件版本、当前固件版本自动分组
- 支持手动选择目标节点列表

**并行下载**：
- 多节点同时接收固件，充分利用 CAN 总线广播特性（若节点支持广播下载）
- 或采用轮询方式逐个节点传输

**顺序重启**：
- 同类节点避免同时重启，防止整批节点同时离线导致系统功能丧失
- Gateway 控制重启节奏，例如：每批次最多 1 个节点重启，确认恢复后再进行下一个

**依赖检查**：
- 升级前检查目标节点当前 Bootloader 版本是否满足要求
- 检查节点当前健康状态，禁止对 `OFFLINE` 或 `DEGRADED` 节点发起升级

### 19.6 固件升级依赖顺序（DAG）

机器人系统中不同节点的固件升级存在依赖关系，Gateway 通过 **DAG（有向无环图）** 表达和管理升级顺序。

**DAG 定义示例**：

```yaml
firmware_upgrade_dag:
  nodes:
    - id: power_board
      name: "电源控制板"
      requires_reboot: true
      reboot_scope: system          # 重启范围：system = 整机重启

    - id: motor_controller
      name: "电机控制器"
      requires_reboot: true
      reboot_scope: self            # 重启范围：self = 仅本节点重启

    - id: imu_sensor
      name: "IMU 传感器"
      requires_reboot: true
      reboot_scope: self

    - id: servo_driver
      name: "伺服驱动器"
      requires_reboot: false        # 不需要重启

  edges:
    - from: power_board
      to: motor_controller          # 电源板升级完成后，才能升级电机控制器
    - from: power_board
      to: servo_driver
    - from: motor_controller
      to: imu_sensor                # 电机控制器升级完成后，才能升级 IMU
```

**关键设计**：

| 字段 | 说明 |
|---|---|
| `requires_reboot` | 升级后是否需要重启 |
| `reboot_scope` | `self` = 仅本节点重启；`system` = 需整机重启 |

**电源控制板卡（`reboot_scope: system`）升级策略**：

当 DAG 中存在 `reboot_scope: system` 的节点时：

1. **前置检查**：
   - 检查所有依赖该节点的下游节点是否已完成升级或无需升级
   - 确保整机重启不会导致未完成升级的节点处于不一致状态

2. **整机重启协调**：
   ```text
   Gateway 通知机器人 OS 调度器：
     "电源板固件即将升级，需整机重启，请保存状态"
   
   机器人 OS 完成以下动作：
     - 暂停所有运动控制任务
     - 保存关节状态、位置、速度
     - 通知上层应用进入维护模式
   
   Gateway 执行：
     - 对所有 CAN 节点发送 EnterBootloader（若需同步升级）
     - 升级电源板固件
     - 发送整机重启指令（通过电源板或 OS 的 reboot 接口）
   
   整机重启后：
     - Gateway 重新枚举所有节点
     - 按 DAG 顺序确认各节点版本
     - 节点进入 CONFIGURATING → ACTIVE
     - 通知机器人 OS 恢复运行
   ```

3. **降级策略**：
   - 若电源板升级失败且无法回滚，禁止整机重启
   - Gateway 保持当前电源板运行，上报 `CRITICAL_OTA_FAILURE` 系统事件
   - 等待人工干预或远程运维指令

**DAG 调度与随机发现处理**：

节点发现顺序是随机的（如 IMU 可能先于电源板出现），但 DAG 定义了升级依赖关系。Gateway 通过以下策略协调两者：

**1. 节点发现时的依赖检查**

当节点 `N` 被发现并进入 `DISCOVERED` 状态时：

```text
1. Gateway 在 DAG 中查找节点 N 的所有前置依赖节点（DAG 中的直接上游）
2. 检查每个前置依赖节点是否满足以下任一条件：
   a. 当前状态为 ACTIVE（已完成升级）
   b. 固件版本已最新，无需升级
   c. 从未在总线上出现（UNKNOWN），但该依赖标记为 optional
3. 若所有前置依赖均满足：
     - 节点 N 进入 PENDING，开始版本检查和 OTA 流程
4. 若存在前置依赖未满足：
     - 节点 N 进入 WAITING_FOR_DEPENDENCY
     - 记录其等待的前置依赖列表
```

**2. 前置依赖完成时的级联触发**

当节点 `M` 完成升级并进入 `ACTIVE` 时：

```text
1. Gateway 查找所有以 M 为前置依赖的下游节点
2. 对每个下游节点 D：
   a. 从 D 的等待列表中移除 M
   b. 若 D 的等待列表为空：
      - D 从 WAITING_FOR_DEPENDENCY 移入 PENDING
      - 触发 D 的版本检查和 OTA 流程
```

**3. 整机重启节点的特殊处理**

对于 `reboot_scope: system` 的节点（如电源板）：

```text
1. 电源板进入 PENDING 后，Gateway 阻塞其所有下游节点（电机、伺服等）
   - 下游节点即使已满足前置依赖，也保持 WAITING_FOR_DEPENDENCY
   - 直到电源板完成升级并 ACTIVE
2. 电源板完成升级后，按级联触发流程逐个释放下游节点
3. 若电源板升级需要整机重启：
   a. 先完成电源板自身升级流程（DOWNLOADING → ACTIVE）
   b. 暂停所有下游节点的升级流程
   c. 执行整机重启协调流程
   d. 重启完成后，重新枚举所有节点
   e. 继续下游节点的升级
```

**4. 超时与降级**

- 若某个前置依赖节点在 `node_timeout` 时间内始终未出现（UNKNOWN），Gateway 根据 DAG 配置决定：
  - `optional: false`（默认）：下游节点永久保持 WAITING_FOR_DEPENDENCY，上报 `MISSING_DEPENDENCY` 系统事件
  - `optional: true`：将该前置依赖视为已满足，释放下游节点

**DAG 调度器**：

```cpp
class UpgradeDagScheduler {
 public:
  // 加载 DAG 定义
  bool LoadDag(const std::string& dag_yaml_path);

  // 节点被发现时调用，返回是否允许立即开始升级
  bool OnNodeDiscovered(uint8_t node_id, const NodeInfo& info);

  // 节点升级完成后调用，触发下游节点的级联释放
  void OnNodeUpgradeCompleted(uint8_t node_id);

  // 获取节点当前等待的前置依赖列表
  std::vector<uint8_t> GetPendingDependencies(uint8_t node_id);

  // 检查节点是否需要整机重启（reboot_scope == system）
  bool IsSystemRebootRequired(uint8_t node_id);

  // 获取给定节点升级前必须先完成的所有前置节点
  std::vector<uint8_t> GetPrerequisites(uint8_t node_id);
};
```

### 19.7 OTA API

```cpp
// 固件仓库管理
class FirmwareRepository {
 public:
  bool UploadFirmware(const std::string& path, const FirmwareMeta& meta);
  bool DeleteFirmware(const std::string& path);
  std::vector<FirmwareMeta> ListFirmwares(const std::string& vendor,
                                           const std::string& device_type);
  bool VerifyFirmware(const std::string& path);  // CRC + 签名验证
};

// 单节点升级任务
struct UpgradeTask {
  uint64_t task_id;
  uint8_t target_node_id;
  std::string firmware_path;
  UpgradeState state;
  float progress;  // 0.0 ~ 1.0
  std::string error_message;
};

class UpgradeManager {
 public:
  // 发起单节点升级
  uint64_t StartUpgrade(uint8_t node_id, const std::string& firmware_path);

  // 发起批量升级
  uint64_t StartBatchUpgrade(const std::vector<uint8_t>& node_ids,
                             const std::string& firmware_path);

  // 查询任务状态
  UpgradeTask GetTaskStatus(uint64_t task_id);

  // 暂停 / 恢复 / 取消
  bool PauseTask(uint64_t task_id);
  bool ResumeTask(uint64_t task_id);
  bool CancelTask(uint64_t task_id);

  // 主动回滚（新版本确认失败时）
  bool Rollback(uint8_t node_id);

  // 查询所有活跃任务
  std::vector<UpgradeTask> GetActiveTasks();
};
```

### 19.8 安全与权限

- **固件签名**：所有入库固件必须携带厂商数字签名，Gateway 加载时验证
- **权限控制**：OTA 操作限制为特定 OS 服务角色，普通应用无权限发起升级
- **审计日志**：记录每次升级的发起人、目标节点、固件版本、时间、结果
- **写保护**：活动镜像区在正常运行时锁定，仅 Bootloader 可修改

---

## 20. 时间同步设计

### 20.1 功能

支持：

- 网络时间同步（UAVCAN Time Synchronization）
- 单调时钟同步
- 时间偏移估计

### 20.2 时间源

优先级：

```text
PTP
  -> NTP
      -> Local Monotonic Clock
```

### 20.3 同步精度

目标：

| 模式 | 精度 |
|---|---|
| 本地总线 | < 1ms |
| 跨设备 | < 5ms |

---

## 21. 并发模型设计

### 21.1 总体模型

采用：

- Reactor
- Producer-Consumer
- Lock-Free Queue
- Thread Affinity

### 21.2 锁策略

原则：

- 优先无锁
- 最小化锁粒度
- 避免跨线程共享
- 使用 RCU/Atomic

---

## 22. 内存设计

### 22.1 内存模型

```text
+----------------------+
| Static Pool          |
+----------------------+

+----------------------+
| Transfer Pool        |
+----------------------+

+----------------------+
| TX Queue Pool        |
+----------------------+

+----------------------+
| HighFreq Buffer      |  零拷贝 IPC
+----------------------+
```

### 22.2 原则

- 固定内存池
- 避免碎片
- 避免实时路径 malloc
- 支持水位统计

---

## 23. 实时性设计

### 23.1 实时要求

| 模块 | 最大延迟 |
|---|---|
| CAN RX | < 1ms |
| Transfer Assemble | < 2ms |
| Topic Dispatch（普通消息） | < 5ms |
| HighFreq Pipeline（高频数据） | < 1ms |
| Heartbeat | < 100ms |

### 23.2 优化策略

采用：

- epoll
- eventfd
- timerfd
- CPU affinity（高频线程绑定独立核心）
- SPSC Queue
- 批量发送
- cache-friendly 数据结构
- 零拷贝 IPC（高频通道）

---

## 24. 安全设计

### 24.1 功能

支持：

- 节点认证
- 固件签名
- ACL（访问控制列表）
- Service 权限控制
- Replay 防护

### 24.2 ACL 示例

```yaml
acl:
  node_10:
    allow:
      - service.reboot
      - service.upgrade
```

---

## 25. API 设计

### 25.1 Gateway 模块 API

```cpp
class UavcanCanGateway {
 public:
  // 生命周期
  bool Initialize(const GatewayConfig& config);
  bool Start();
  void Stop();

  // 运维查询
  BusStatus GetBusStatus(const std::string& interface_name);
  std::vector<NodeInfo> GetOnlineNodes();
  GatewayStats GetStatistics();
};
```

### 25.2 ClientConfig 与 GatewayConfig

```cpp
struct ClientConfig {
  // 连接模式
  enum class TransportMode {
    kAuto,        // 自动检测：有 SocketCAN 权限则直连，否则走 IPC
    kDirectCan,   // 强制 CAN 直连（HAL 组件推荐）
    kIpc,         // 强制 IPC 代理（非 HAL 组件推荐）
  };
  TransportMode transport_mode = TransportMode::kAuto;

  // Gateway 连接参数（IPC 模式或端口注册时使用）
  std::string gateway_service_name;  // IPC 服务发现标识

  // CAN 接口参数（直连模式使用）
  std::string interface_name = "can0";

  // 本地缓存
  size_t tx_memory_pool_size = 64 * 1024;
  size_t rx_memory_pool_size = 64 * 1024;
};

struct GatewayConfig {
  // CAN 总线配置
  std::vector<CanInterfaceConfig> interfaces;

  // UAVCAN 节点身份
  uint8_t node_id = 0;  // 0 = 动态分配
  std::string node_name = "eros.uavcan.gateway";

  // 心跳参数
  std::chrono::milliseconds heartbeat_period = std::chrono::milliseconds(1000);

  // 外部节点监测
  std::chrono::milliseconds node_timeout = std::chrono::milliseconds(3000);

  // IPC 服务参数
  std::string ipc_service_name = "eros.uavcan.gateway";

  // 日志
  std::string log_level = "INFO";
};
```

### 25.3 Client Library API

Client Library 对外暴露**统一接口**，不区分直连或 IPC。底层传输方式由 `ClientConfig::transport_mode` 决定，调用方无感知。

```cpp
class UavcanCanGatewayClient {
 public:
  // 生命周期
  bool Initialize(const ClientConfig& config);
  bool Start();
  void Stop();

  // 端口注册与注销（启动前必须完成注册）
  bool RegisterPublisher(uint16_t subject_id, const std::string& data_type);
  bool RegisterSubscriber(uint16_t subject_id, const std::string& data_type);
  bool RegisterServiceServer(uint16_t service_id, const std::string& data_type);
  bool RegisterServiceClient(uint16_t service_id, const std::string& data_type);

  bool UnregisterPublisher(uint16_t subject_id);
  bool UnregisterSubscriber(uint16_t subject_id);
  bool UnregisterServiceServer(uint16_t service_id);
  bool UnregisterServiceClient(uint16_t service_id);

  // 原始字节接口
  bool Publish(uint16_t subject_id, const uint8_t* data, size_t size);
  bool Subscribe(uint16_t subject_id, MessageCallback callback);

  bool CallService(
      uint16_t service_id,
      uint8_t dest_node_id,
      const uint8_t* request_data,
      size_t request_size,
      ServiceResponseCallback callback,
      std::chrono::milliseconds timeout);

  // 模板接口（T 为 libcanard DSDL 编译生成的 C struct）
  template <typename T>
  bool Publish(uint16_t subject_id, const T& msg);

  template <typename T>
  bool Subscribe(uint16_t subject_id, std::function<void(const T&)> callback);

  template <typename TRequest, typename TResponse>
  bool CallService(
      uint16_t service_id,
      uint8_t dest_node_id,
      const TRequest& request,
      std::function<void(const TResponse&)> callback,
      std::chrono::milliseconds timeout);

  // 配置查询
  uint8_t GetLocalNodeId() const;
  BusConfig GetBusConfig(const std::string& interface_name) const;
};
```

**接口使用约定**：
- `Initialize` 时根据 `ClientConfig::transport_mode` 选择底层实现（直连或 IPC），调用方无感知
- `RegisterPublisher` / `RegisterSubscriber` 等端口注册通过 IPC 与 Gateway 交互，无论直连还是 IPC 模式都必须完成注册
- 模板接口内部自动调用对应的 DSDL 序列化 / 反序列化函数，用户无需手动处理字节流
- 直连模式下，数据收发直接通过 libcanard + SocketCAN；IPC 模式下，数据收发通过 IPC 由 Gateway 代理。两者接口完全一致

### 25.4 Client Library 功能详解

#### 25.4.1 端口注册流程

Client 在使用任何数据收发功能前，必须先向 Gateway 完成端口注册：

```text
1. Client 启动
2. 通过 IPC 连接 Gateway
3. 发送 RegisterPublisher / RegisterSubscriber / RegisterServiceClient 请求
4. Gateway 检查 Subject-ID / Service-ID 是否已被其他 Client 占用
5. Gateway 返回确认或拒绝（冲突时返回已占用信息）
6. Gateway 更新内部端口表，重新生成并发送 PortList
7. Client 收到确认后，进入正常运行状态
```

**运行时动态增删端口**：
- Client 可在运行期申请新增或释放端口（`UnregisterPublisher` / `UnregisterSubscriber` 等）
- Gateway 实时更新 PortList 并广播

#### 25.4.2 生命周期

```text
Initialize(config)
  ├── 根据 transport_mode 选择底层实现（直连或 IPC）
  ├── 连接 Gateway，完成端口注册
  └── 初始化本地资源（libcanard 实例 + SocketCAN，或 IPC 通道）

Start()
  ├── 启动接收路径（SocketCAN epoll 或 IPC 接收线程）
  └── 进入就绪状态

Publish(subject_id, data, size)
  ├── 校验 subject_id 已注册
  ├── 直连模式：组装 CanardTransferMetadata → canardTxPush → SocketCAN write
  └── IPC 模式：编码后通过 IPC 发送至 Gateway → Gateway 转发至 CAN

Subscribe(subject_id, callback)
  ├── 校验 subject_id 已注册
  └── 注册本地回调，收到匹配帧时触发

CallService(service_id, dest_node_id, request, callback, timeout)
  ├── 校验 service_id 已注册
  ├── 直连模式：直接发送 Service Request，等待 Response 后回调
  └── IPC 模式：通过 IPC 发送至 Gateway，Gateway 代理收发后回调

Stop()
  ├── 释放本地资源
  └── 通知 Gateway 注销所有端口
```

#### 25.4.3 模板接口说明

libcanard 配套工具（`nunavut` / `dsdlc`）可将 DSDL 定义编译为 C 语言结构体及序列化 / 反序列化函数。Client Library 在此基础上提供模板封装：

```cpp
// DSDL 编译生成的类型
struct uavcan_si_unit_acceleration_Vector3_1_0 {
  float value[3];
};
// 同时生成：uavcan_si_unit_acceleration_Vector3_1_0_serialize_
//           uavcan_si_unit_acceleration_Vector3_1_0_deserialize_

// 使用模板接口
client.RegisterPublisher(2001, "uavcan.si.unit.acceleration.Vector3");

uavcan_si_unit_acceleration_Vector3_1_0 imu_data{{1.0f, 2.0f, 3.0f}};
client.Publish(2001, imu_data);

client.Subscribe<uavcan_si_unit_acceleration_Vector3_1_0>(
    2001, [](const auto& msg) {
      // 直接收到反序列化后的结构体
    });
```

**模板接口约束**：
- 类型 `T` 必须由 DSDL 编译器生成，具备对应的 `_serialize_` / `_deserialize_` 函数
- 编译期通过 concept 或静态断言检查序列化函数存在性
- 模板接口内部自动计算序列化缓冲区大小，无需用户分配

#### 25.4.4 错误处理与回调约定

```cpp
enum class ClientError {
  kOk = 0,
  kNotInitialized,      // Initialize 未调用或失败
  kPortNotRegistered,   // 使用了未注册的 Subject/Service ID
  kPortConflict,        // 该端口已被其他 Client 注册
  kNodeNotActive,       // 目标节点未完成 OTA 和配置，不允许通信
  kGatewayUnreachable,  // Gateway 连接断开
  kBusOff,              // CAN 总线 Bus-Off（直连模式）
  kTxQueueFull,         // 发送队列溢出
  kInvalidPayload,      // 载荷大小超过 UAVCAN MTU
};
```

所有异步操作（Subscribe、CallService）通过回调返回结果；同步操作（Publish）返回 `ClientError`。

#### 25.4.5 配置查询

Client 可通过统一接口查询以下运行时配置：
- `GetLocalNodeId()`：获取 Gateway 分配给本系统的 UAVCAN NodeID
- `GetBusConfig(interface)`：获取 CAN 接口的波特率、FD 参数
- `GetRegisteredPorts()`：获取本 Client 已注册的所有端口列表

---

## 26. 测试要求

### 26.1 单元测试

覆盖率目标：

| 模块 | 覆盖率 |
|---|---|
| 核心模块 | >= 90% |
| 普通模块 | >= 80% |

### 26.2 压力测试

支持：

- 高负载 CAN Flood
- 节点抖动
- 总线错误注入
- OOM 注入
- 长时间稳定性测试

### 26.3 长稳测试

目标：

- 连续运行 30 天
- 无内存泄漏
- 无线程泄漏
- 无 fd 泄漏

---

## 27. 部署设计

### 27.1 部署方式

支持：

- Systemd Service
- Docker
- Embedded Linux

### 27.2 systemd 示例

```ini
[Unit]
Description=UavcanCanGateway
After=network.target

[Service]
ExecStart=/usr/bin/uavcan_can_gateway --config /etc/uavcan/gateway.yaml
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 28. 目录结构建议

```text
uavcan_can_gateway/
├── apps/                          # 可执行文件入口（Gateway 主进程）
├── cmake/                         # CMake 模块
├── configs/                       # 配置文件模板
├── docs/                          # 文档
├── include/
│   ├── uavcan_can_gateway/        # Gateway 模块公共头文件
│   └── uavcan_can_gateway_client/ # Client Library 公共头文件
├── scripts/                       # 启动脚本、工具
├── src/
│   ├── gateway/                   # Gateway 模块（服务端，负责 CAN 硬件管理、协议栈运行、消息路由）
│   ├── client/                    # Gateway Client Library（客户端库，供应用程序集成）
│   ├── transport/                 # 传输层抽象
│   ├── socketcan/                 # SocketCAN 驱动
│   ├── protocol/                  # UAVCAN 协议适配（libcanard 封装）
│   ├── router/                    # 消息映射与路由
│   ├── high_freq/                 # 高频数据专用通道
│   ├── node/                      # 节点管理
│   ├── service/                   # RPC 服务代理（TBD，预留目录）
│   ├── upgrade/                   # 固件升级
│   ├── logging/                   # 日志系统
│   ├── diagnostics/               # 诊断与监控
│   └── utils/                     # 工具库
├── tests/
│   ├── unit/                      # 单元测试
│   ├── integration/               # 集成测试
│   └── stress/                    # 压力测试
└── third_party/
    ├── libcanard/                 # UAVCAN 协议栈
    └── ...
```

---

## 29. 第三方依赖

| 组件 | 用途 |
|---|---|
| libcanard | Cyphal/UAVCAN 协议栈 |
| linux-can | SocketCAN |
| yaml-cpp | YAML 配置解析 |
| spdlog | 日志库 |
| protobuf | 数据序列化（OS 消息总线交互） |
| gtest | 单元测试 |
| benchmark | 性能测试 |

---

## 30. 风险与挑战

### 30.1 技术风险

| 风险 | 描述 | 缓解措施 |
|---|---|---|
| 高负载丢帧 | 总线压力过大 | 流量限流、优先级队列、队列水位控制 |
| 高频数据延迟 | IMU 等数据走 IPC 中转引入抖动 | HAL 组件直接 libcanard + SocketCAN 消费；非 HAL 组件通过 IPC 零拷贝分发 |
| 内存碎片 | 长时间运行 | 固定内存池、禁止实时路径 malloc |
| 多线程竞争 | 锁争用 | 无锁队列、最小化锁粒度、RCU |
| 升级失败 | 固件损坏 | 双镜像升级、CRC/签名校验、回滚机制 |
| 总线异常 | Bus-Off、节点冲突 | 自动恢复、心跳监测、故障隔离 |
| 端口冲突 | 多 App 未注册直接使用同一 Subject-ID | Gateway 集中注册分配，运行时冲突检测与告警 |

---

## 31. 未来扩展方向（多总线架构铺路）

### 31.1 统一抽象层

后续所有总线网关共用同一套消息转发框架：

```text
+------------------------------------------------+
|              机器人 OS 消息总线                 |
+------------------------------------------------+
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v
+-------------+  +-------------+  +-------------+
|UavcanCanGw  |  |UavcanEthGw  |  |ModbusCanGw  |
+-------------+  +-------------+  +-------------+
       |                 |                 |
       v                 v                 v
+-------------+  +-------------+  +-------------+
|    CAN      |  |  Ethernet   |  |    CAN      |
+-------------+  +-------------+  +-------------+
```

上层接口完全一样，应用程序不需要改代码即可切换底层总线。

### 31.2 预留能力

- 多实例：多个 UavcanCanGateway 实例对应多路 CAN 总线
- Ethernet Transport：UavcanEthGateway
- TSN 支持
- DDS Bridge
- ROS2 Native Bridge
- WebSocket Gateway
- gRPC API
- Lua/Python 插件
- 规则引擎
- OTA 平台接入

---

## 32. 待确定事项（TBD）

以下功能模块当前阶段暂不详细定义，待后续版本根据实际需求补充：

### 32.1 TopicManager（发布/订阅适配）

负责 OS 内部消息总线话题与 UAVCAN Subject-ID 之间的映射、转换与路由。当前阶段：
- HAL 组件通过 CAN 直连直接收发 UAVCAN 帧，无需 Topic 映射
- 非 HAL 组件的 IPC 路径由 MessageRouter 统一处理，暂不需要独立 TopicManager

后续若需支持 ROS2/DDS 原生 Topic 与 UAVCAN Subject 的自动双向转换，再补充该模块设计。

### 32.2 ServiceManager（RPC 服务代理）

负责 UAVCAN Service 请求/响应与 OS 内部服务调用之间的代理转换。当前阶段：
- HAL 组件直接通过 libcanard 发起和接收 Service 调用
- 非 HAL 组件的 Service 需求可通过 IPC 透传由 Gateway 直接代理

后续若需支持复杂的服务超时重试、并发调用管理、服务发现等高级特性，再补充该模块设计。

---

## 33. 总结

**UavcanCanGateway 定位**：

EROS HAL Bus-Gateway 系列库中面向 UAVCAN/CAN 的实现，为 HAL 组件及非 HAL 硬件通讯程序提供总线接入支撑。其实现包含 **Gateway 模块**（服务端，负责 UAVCAN 节点身份代理、端口注册中心、总线管理、跨协议转换）和 **Gateway Client Library**（客户端库，支持 CAN 直连与 IPC 两种接入模式）。

**核心职责**：

UAVCAN 节点身份代理 + 端口注册与管理 + 外部节点发现与监控 + CAN 总线配置与健康管理 + 跨协议消息路由（ROS2/DDS ↔ UAVCAN）+ 固件 OTA + 配置日志运维。

**关键设计原则**：

1. **Gateway 是协调者，不是数据垄断者**：SocketCAN 天然支持多进程共享，HAL 组件可直接 libcanard + SocketCAN 收发高频数据
2. **端口分配集中化**：Subject-ID / Service-ID 由 Gateway 统一注册分配，确保同 NodeID 下不冲突
3. **分层接入，尊重总线特性**：HAL 组件走 CAN 直连（最低延迟），非 HAL 组件走 IPC 中转（统一抽象）
4. **统一命名范式**：`[Protocol][Bus]Gateway`，为多总线架构预留扩展空间
5. **工业级可靠性**：故障隔离、自动恢复、可观测、可运维

本设计基于：

- libcanard
- SocketCAN
- Reactor 架构
- 高性能内存池
- 多线程并发模型
- 零拷贝 IPC

实现：

- 高性能
- 高可靠
- 易扩展
- 工业级稳定性

的现场总线网关系统。

该系统可作为：

- 机器人底盘总线网关
- MCU 管理中心
- UAVCAN/Cyphal 网络核心
- ROS2/DDS FieldBus Bridge
- 工业 CAN Gateway

的统一基础设施。
