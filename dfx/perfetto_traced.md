# traced_probes/traced/perfetto  三者关系



## 关系概览

```
[perfetto client (e.g. shell command / app)]
        |
        v
+------------------+
|     perfetto     | ← 用户空间 CLI 工具（或libperfetto）
+------------------+
        |
        v
+------------------+                +-------------------+
|     traced       | <------------> |   traced_probes   |
+------------------+                +-------------------+
       ↑                                ↑
     收集数据                           内核+原生探针数据源
```



##  perfetto（客户端工具 / 库）

- Android 平台上的 CLI 工具（如 `perfetto` 命令），或嵌入到应用中的客户端库（libperfetto）。

- 主要职责：
  - 发起 trace session
  - 配置 trace config（比如收集哪些数据源：CPU 调度、内存、binder、Java 堆等）
  - 启动 / 停止 trace
  - 接收 trace 数据（或写入文件）



## traced（**中央控制进程**）

- 系统服务守护进程，负责控制 trace session 生命周期。
- 管理所有 trace producers 和 consumers。
- 负责：
  - 和客户端（如 `perfetto`）建立连接
  - 转发配置给 `traced_probes`（或其他 producer）
  - 聚合所有 trace 数据片段
  - 写入 ring buffer 或 trace 文件
- 监听 socket：`/dev/socket/traced_consumer`, `/dev/socket/traced_producer`



## traced_probes（**探针数据采集器**）

- 是 `traced` 管理的一个 **producer**，负责与系统内核或原生模块对接，采集各种底层事件。
- 数据源包括：
  - ftrace（内核 tracepoint，比如调度、IRQ、binder、CPU freq 等）
  - atrace
  - system stats（如 meminfo、vmstat）
  - Perf events (有时)
- 数据通过共享内存传回 `traced`。



## Trace 一次 CPU 调度

1. `perfetto` CLI 发起一次 trace config，请求收集 `sched/sched_switch`。
2. 它将配置通过 socket 发给 `traced`。
3. `traced` 分发配置给 `traced_probes`。
4. `traced_probes` 启用对应的 ftrace 事件并采集数据。
5. 采集的数据写入共享内存。
6. `traced` 聚合这些数据并写入 trace buffer。
7. `perfetto` 结束 session，trace 数据被 flush 成 `.perfetto-trace` 文件。



## 相关 socket 通信路径

```
/dev/socket/traced_consumer
/dev/socket/traced_producer
```



## 总结理解一句话：

> `perfetto` 是控制台或应用层发起者，`traced` 是核心协调者，`traced_probes` 是具体的数据采集工人。



`traced_probes`、`traced` 和 `perfetto` 是 **Perfetto** 性能分析工具套件中的核心组件，它们协同工作以收集、管理和存储系统级的性能数据（如 CPU、内存、I/O、内核事件等）。以下是它们的详细关系和功能解析：

---

### 1. **Perfetto 概述**
Perfetto 是 Google 开发的下一代性能诊断工具，用于替代传统的 `systrace`。它支持跨平台（Android、Linux、ChromeOS）的高效性能数据采集和分析，提供：
- **低开销**：通过内核接口（如 `ftrace`）和用户空间探针收集数据。
- **灵活配置**：通过 JSON 或 Protobuf 配置文件定义数据源。
- **可视化分析**：通过 [Perfetto UI](https://ui.perfetto.dev) 或 `traceconv` 工具解析二进制 trace 文件。

---

### 2. **核心进程与关系**
#### **(1) `traced_probes`**
- **角色**：**数据收集器**（Data Sources）。
- **功能**：
  - 通过内核接口（如 `ftrace`、`/proc`、`/sys`）采集硬件和系统级事件。
  - 管理用户空间探针（如 Android 的 HAL 层、自定义打点）。
  - 支持动态启停数据源（如 CPU 频率、进程调度、内存统计）。
- **依赖**：需要 `traced` 服务协调数据流。

#### **(2) `traced`**
- **角色**：**中央守护进程**（Tracing Service）。
- **功能**：
  - 控制整个 tracing 会话的生命周期（开始/停止/配置）。
  - 接收来自 `traced_probes` 或其他数据源（如 Chrome、自定义应用）的数据。
  - 将数据流聚合到共享内存缓冲区，最终写入 trace 文件（`.perfetto-trace`）。
- **关键协议**：通过 `protobuf` 与客户端（如 `perfetto` 命令行工具）通信。

#### **(3) `perfetto`**
- **角色**：**命令行客户端**（Frontend）。
- **功能**：
  - 解析用户配置（如 `config.pbtxt`），向 `traced` 发送控制命令。
  - 生成 trace 文件，或直接流式传输到分析工具。
  - 支持本地录制（Android/Linux）或远程设备抓取。

---

### 3. **协作流程**
1. **初始化**：
   - 用户通过 `perfetto` 命令启动 tracing，指定配置（如 `duration: 10s, buffers: 32MB`）。
   - `perfetto` 将配置发送给 `traced` 服务。

2. **数据采集**：
   - `traced` 通知 `traced_probes` 激活匹配的数据源（如启用 `ftrace` 的 `sched_switch` 事件）。
   - `traced_probes` 将采集的数据通过 IPC 发送到 `traced` 的共享内存缓冲区。

3. **结束与存储**：
   - 到达预设条件（如超时）后，`traced` 停止会话，将缓冲区数据写入 `.perfetto-trace` 文件。
   - 用户通过 Perfetto UI 或 `traceconv` 分析文件。

---

### 4. **关键技术与优势**
- **高效 IPC**：使用共享内存减少数据拷贝开销。
- **模块化设计**：数据源（如 `traced_probes`）可独立更新或扩展。
- **安全隔离**：`traced_probes` 以最小权限运行（如 `nobody` 用户）。

---

### 5. **调试与排查**
若进程异常（如高 CPU 占用），可通过以下命令检查：
```bash
# 查看进程状态
ps aux | grep -E 'traced|traced_probes'
# 查看日志（Android）
adb logcat | grep -i perfetto
# 强制停止所有 tracing
adb shell stop perfetto
```

---

### 总结
- **`traced_probes`**：底层数据采集的“工人”。
- **`traced`**：协调数据流的“调度中心”。
- **`perfetto`**：用户交互的“指挥官”。

三者共同构成 Perfetto 的流水线，适用于从低功耗设备到大型分布式系统的性能分析场景。





`atrace` 和 `perfetto` 虽然都与 `traced` 交互，但它们的通信机制和架构设计存在显著差异。以下是详细对比和流程解析：

---

### **1. `atrace` 的工作流程**
#### **(1) 传统 `atrace`（基于 systrace）**
- **角色**：Android 专用的**高层封装工具**，主要用于采集 **ftrace 数据**（如 CPU 调度、Binder 调用等）。
- **通信路径**：
  ```mermaid
  graph LR
    A[atrace] -->|命令行参数| B[ftrace]
    B -->|直接写入| C[内核环形缓冲区]
    C -->|读取| D[traced_probes]
    D -->|IPC| E[traced]
  ```
- **关键点**：
  - **直接操作 ftrace**：`atrace` 通过写入 `/sys/kernel/debug/tracing/trace_marker` 或 `trace_options` 文件控制 ftrace，**不依赖 `traced`**。
  - **数据流向**：内核 ftrace 数据由 `traced_probes` 读取后，再通过共享内存传递给 `traced`（如果 Perfetto 会话正在运行）。
  - **独立性**：`atrace` 可以独立运行（如 `atrace -c`），无需 `traced` 服务。

#### **(2) `atrace` 与 Perfetto 的兼容模式**
- **现代 Android 版本**（如 Android 10+）中，`atrace` 可通过 `--perfetto` 参数将数据导入 Perfetto：
  ```bash
  atrace --perfetto --categories=sched,gfx -t 5s -o trace.pb
  ```
  - 此时 `atrace` 作为 **Perfetto 的数据源**，通过 `traced` 协调：
    ```mermaid
    graph LR
      A[atrace] -->|--perfetto| B[traced]
      B -->|配置| C[traced_probes]
      C -->|采集 ftrace| D[内核]
      D -->|数据| C --> B --> E[.perfetto-trace]
    ```

---

### **2. `perfetto` 的工作流程**
#### **(1) 原生 Perfetto 模式**
- **角色**：跨平台的**统一追踪框架**，支持多数据源（ftrace、heap profiling、自定义用户空间探针等）。
- **通信路径**：
  ```mermaid
  graph LR
    A[perfetto CLI] -->|protobuf| B[traced]
    B -->|IPC| C[traced_probes]
    C -->|ftrace/sysfs/proc| D[内核]
    C -->|其他数据源| E[用户空间]
    B --> F[.perfetto-trace]
  ```
- **关键点**：
  - **集中式管理**：`traced` 是核心枢纽，所有数据源（包括 `traced_probes`）需通过它注册和传输数据。
  - **灵活配置**：通过 `.pbtxt` 文件定义数据源（如 `ftrace`、`android_log`、`heap_profile`）。

---

### **3. 核心差异总结**
| **特性**          | `atrace` (传统模式)   | `atrace --perfetto`      | `perfetto`          |
| ----------------- | --------------------- | ------------------------ | ------------------- |
| **通信对象**      | 直接操作 ftrace       | 通过 `traced` 协调       | 通过 `traced` 协调  |
| **依赖 `traced`** | 否                    | 是                       | 是                  |
| **数据范围**      | 仅 ftrace 事件        | ftrace + 可选其他源      | 多平台多数据源      |
| **输出格式**      | 文本/systrace 格式    | Perfetto 二进制格式      | Perfetto 二进制格式 |
| **典型用途**      | 快速 Android 调度分析 | Android 与 Perfetto 集成 | 跨平台综合性能分析  |

---

### **4. 底层机制详解**
#### **(1) `atrace` 如何绕过 `traced`？**
- **直接内核接口**：`atrace` 通过 `/sys/kernel/debug/tracing/` 下的文件直接启停 ftrace：
  ```bash
  echo 1 > /sys/kernel/debug/tracing/events/sched/enable
  cat /sys/kernel/debug/tracing/trace_pipe > trace.txt
  ```
- **`traced_probes` 的被动角色**：即使 `atrace` 独立运行，`traced_probes` 仍可能读取 ftrace 缓冲区（如果其他会话需要），但两者无强依赖。

#### **(2) Perfetto 的统一模型**
- **数据源抽象**：`traced_probes` 是 Perfetto 的**默认数据收集器**，但其他数据源（如 Chrome、Android HAL）可通过 `Producer` API 直接连接 `traced`。
- **并行会话**：`traced` 可同时管理多个 tracing 会话，动态分配资源。

---

### **5. 实际场景示例**
#### **场景 1：仅使用 `atrace`**
```bash
# 独立采集 ftrace，不依赖 traced
adb shell atrace -c sched,gfx -t 5 -o trace.txt
```
- **流程**：`atrace` → 内核 ftrace → 直接输出文本文件。

#### **场景 2：`atrace` 集成到 Perfetto**
```bash
# 通过 traced 采集，生成 Perfetto 格式
adb shell atrace --perfetto -c sched,gfx -t 5 -o trace.pb
```
- **流程**：`atrace` → `traced` → `traced_probes` → 内核 → `traced` → `trace.pb`。

#### **场景 3：原生 `perfetto`**
```bash
# 综合采集 ftrace + 内存 + 自定义事件
adb shell perfetto -c /data/misc/perfetto-config.pbtxt -o trace.pb
```
- **流程**：`perfetto` CLI → `traced` → 多数据源（包括 `traced_probes`） → `trace.pb`。

---

### **6. 总结**
- **`atrace` 传统模式**：直接操作内核 ftrace，独立于 `traced`，适合快速 Android 专用分析。
- **`atrace --perfetto`**：作为 Perfetto 生态的兼容层，通过 `traced` 统一管理。
- **原生 `perfetto`**：全功能框架，通过 `traced` 集中调度所有数据源（包括 `traced_probes`）。

两者选择取决于需求：**快速诊断 Android 调度问题用 `atrace`，复杂跨平台分析用 `perfetto`**。
