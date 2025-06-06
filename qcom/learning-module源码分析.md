# Learning-module 源码分析

## 核心架构

Learning Module采用了**三元组架构**（Meter-Algorithm-Action）：

### 1. **Meter（测量器）**

- **作用**：收集系统和应用的性能数据
- **实现**：`Meter`基类和具体实现如`ReferenceMeter`
- **数据收集**：通过异步触发器收集时间戳、内存状态等数据
- **线程模型**：每个Meter在独立线程中运行，使用`ThreadHelper`管理

### 2. **Algorithm（算法）**

- **作用**：分析收集到的数据，进行机器学习和模式识别
- **实现**：`Algorithm`基类，具体如`ReferenceAlgo`
- **学习过程**：分析应用启动间隔时间等模式
- **状态管理**：通过`FeatureState`跟踪每个应用的学习状态

### 3. **Action（动作）**

- **作用**：根据算法输出应用性能优化策略
- **实现**：`Action`基类，具体如`ReferenceAction`
- **优化策略**：可以调用perflock、调整CPU频率等

## 核心组件详解

### **LearningModule（核心管理器）**

```cpp
// 主要职责：
- 初始化整个框架
- 管理Feature生命周期
- 处理来自perf-hal的异步触发器
- 调度空闲时算法执行
```

### **Feature（功能特性）**

```cpp
// 每个Feature包含：
- MetaMeter: 管理多个Meter
- Algorithm: 数据分析算法
- Action: 性能优化动作
- FeatureState: 应用状态管理
```

### **数据库系统（LMDB）**

- **输入数据库**：存储Meter收集的原始数据
- **输出数据库**：存储Algorithm分析结果
- **基于SQLite**：持久化存储，支持事务

### **异步触发机制**

```cpp
// 触发器类型：
- VENDOR_HINT_FIRST_LAUNCH_BOOST: 应用首次启动
- VENDOR_HINT_FIRST_DRAW: 应用首次绘制
- VENDOR_HINT_DISPLAY_OFF: 屏幕关闭（触发算法）
```

## 工作流程

1. **初始化阶段**：
   - 解析XML配置文件
   - 动态加载Feature库
   - 初始化数据库和线程
2. **数据收集阶段**：
   - 接收perf-hal的异步触发器
   - 启动相应的Meter收集数据
   - 数据存储到输入数据库
3. **学习阶段**：
   - 屏幕关闭时触发算法
   - 分析数据库中的历史数据
   - 生成优化策略存储到输出数据库
4. **优化应用阶段**：
   - 根据输出数据库的策略
   - 在适当时机应用性能优化



