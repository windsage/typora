# mp-ctl模块源码分析

## 1. 编译产物分析

根据源码结构和Android.bp配置，主要编译产物包括：

```
libqti-perfd.so     # 性能调优动态库
perfd               # 性能守护进程
libqti-perf.so      # 性能接口库
```

**作用说明：**

- `libqti-perfd.so`: 核心性能调优库，包含所有性能控制逻辑
- `perfd`: 系统性能守护进程，负责处理性能锁请求
- `libqti-perf.so`: 对外接口库，供应用层调用

## 2. mp-ctl模块作用分析

### 2.1 核心功能架构

```mermaid
graph TB
    A[PerfHAL] --> B[mp-ctl Module]
    B --> C[ActiveRequestList]
    B --> D[ResourceQueues] 
    B --> E[OptsHandler]
    B --> F[Target]
    
    C --> G[请求管理]
    D --> H[资源队列管理]
    E --> I[性能优化操作]
    F --> J[目标设备适配]
    
    B --> K[HintExtHandler]
    K --> L[扩展提示处理]
    
    subgraph "核心组件"
        G
        H
        I
        J
        L
    end
```

### 2.2 主要模块功能

| 模块         | 主要功能                 | 关键类                           |
| ------------ | ------------------------ | -------------------------------- |
| **请求管理** | 管理性能锁请求的生命周期 | `ActiveRequestList`, `Request`   |
| **资源调度** | 管理系统资源的分配和释放 | `ResourceQueues`, `ResourceInfo` |
| **性能操作** | 执行具体的性能优化操作   | `OptsHandler`, `ResetHandler`    |
| **设备适配** | 针对不同SOC的适配        | `Target`, `TargetConfig`         |
| **扩展处理** | 处理特殊性能提示         | `HintExtHandler`                 |

### 2.3 性能锁机制

```mermaid
stateDiagram-v2
    [*] --> Requested: perf_lock_acq()
    Requested --> Active: 验证并添加到队列
    Active --> Pending: 更高优先级请求
    Pending --> Active: 高优先级释放
    Active --> Released: perf_lock_rel()
    Released --> [*]
    
    Active --> Timeout: 定时器到期
    Timeout --> Released
```

## 3. PerfHAL到mp-ctl调用流程

### 3.1 整体调用架构



```mermaid
sequenceDiagram
    participant App as 应用层
    participant HAL as PerfHAL
    participant MP as mp-ctl
    participant Kernel as 内核
    
    App->>HAL: powerHint()/perfLockAcquire()
    HAL->>MP: perfmodule_submit_request()
    MP->>MP: internal_perf_lock_acq_verification()
    MP->>MP: AddAndApply()
    MP->>Kernel: 写入sysfs节点
    Kernel-->>MP: 应用性能参数
    MP-->>HAL: 返回handle
    HAL-->>App: 返回结果
```

### 3.2 请求处理详细流程

```mermaid
flowchart TD
    A[perfmodule_submit_request] --> B{请求类型判断}
    
    B -->|ACQUIRE| C[internal_perf_lock_acq_verification]
    B -->|RELEASE| D[internal_perf_lock_rel]
    B -->|HINT| E[HintExtHandler处理]
    
    C --> F[创建Request对象]
    F --> G[验证请求参数]
    G --> H[生成唯一handle]
    H --> I[添加到ActiveRequestList]
    I --> J[ResourceQueues.AddAndApply]
    
    J --> K[OptsHandler.ApplyOpt]
    K --> L[写入sysfs节点]
    L --> M[启动定时器]
    
    D --> N[从ActiveRequestList移除]
    N --> O[ResourceQueues.RemoveAndApply]
    O --> P[OptsHandler.ResetOpt]
    P --> Q[恢复默认值]
    
    E --> R[预处理]
    R --> S[获取配置]
    S --> T[后处理]
```

### 3.3 核心数据流

```mermaid
graph LR
    A[perfmodule_submit_request] --> B[mpctl_msg_t]
    B --> C[Request对象]
    C --> D[ResourceInfo数组]
    D --> E[ResourceQueues]
    E --> F[OptsHandler]
    F --> G[sysfs节点]
    
    H[ActiveRequestList] --> I[handle管理]
    J[HintExtHandler] --> K[特殊处理逻辑]
    
    style A fill:#e1f5fe
    style G fill:#c8e6c9
    style I fill:#fff3e0
    style K fill:#f3e5f5
```

### 3.4 关键接口说明

#### 主要入口函数

```cpp
// 模块初始化
int32_t perfmodule_init(void);

// 请求提交
int perfmodule_submit_request(mpctl_msg_t *msg);

// 模块退出  
void perfmodule_exit(void);
```

#### 核心处理流程

```cpp
// 性能锁获取验证
Request* internal_perf_lock_acq_verification(
    int32_t &handle, uint32_t duration, int32_t list[], 
    uint32_t num_args, pid_t client_pid, pid_t client_tid, 
    bool renew, bool isHint, enum client_t client
);

// 性能锁应用
int32_t internal_perf_lock_acq_apply(int32_t handle, Request *req);

// 性能锁释放
int32_t internal_perf_lock_rel(int32_t handle);
```

### 3.5 资源管理机制

mp-ctl通过以下机制管理系统资源：

1. **请求队列化**: 使用ResourceQueues管理资源请求的优先级
2. **动态调度**: 根据请求优先级动态调整资源分配
3. **定时管理**: 通过定时器自动释放临时性能锁
4. **状态恢复**: 通过ResetHandler恢复系统默认状态



## 4. 优先级机制的核心实现

### 4.1 优先级判断逻辑

在 `ResourceQueues.cpp` 中的 `AddAndApply` 方法是优先级处理的核心：

```cpp
bool ResourceQueue::AddAndApply(Request *req) {
    // ...
    for (i = 0; i < req->GetNumLocks(); i++) {
        ResourceInfo *res = req->GetResource(i);
        Resource &resObj = res->GetResourceObject();
        uint32_t level = resObj.value;  // 这就是性能级别
        
        current = GetNode(resObj);
        
        // 关键：比较性能级别来决定优先级
        if (resource_queue_not_empty) {
            needAction = oh.CompareOpt(resObj.qindex, level, current->resource.level);
        }
        
        if (needAction == ADD_AND_UPDATE_REQUEST) {
            // 新请求级别更高，当前请求被挂起
            pended = (struct q_node *)calloc(1, sizeof(struct q_node));
            CopyQnode(pended, current);
            
            // 应用新的高优先级请求
            rc = oh.ApplyOpt(resObj);
            current->handle = req;
            current->resource = resObj;
            current->next = pended;  // 原请求进入挂起队列
        }
        else if (needAction == PEND_REQUEST) {
            // 新请求级别较低，加入挂起队列
            recent = (struct q_node *)calloc(1, sizeof(struct q_node));
            recent->handle = req;
            recent->resource = resObj;
            // 插入到合适的挂起位置
            iter->next = recent;
        }
    }
}
```

### 4.2 比较函数实现

在 `OptsHandler.cpp` 中定义了多种比较策略：

```cpp
// 数值越高优先级越高（性能级别）
int32_t OptsHandler::higher_is_better(uint32_t reqLevel, uint32_t curLevel) {
    if (reqLevel > curLevel) {
        return ADD_AND_UPDATE_REQUEST;  // 新请求优先级更高，替换当前
    } else if (reqLevel == curLevel) {
        return PEND_REQUEST;            // 相同级别，加入挂起队列
    } else {
        return PEND_REQUEST;            // 新请求优先级较低，挂起
    }
}

// 数值越低优先级越高（延迟、功耗）
int32_t OptsHandler::lower_is_better(uint32_t reqLevel, uint32_t curLevel) {
    if (reqLevel < curLevel) {
        return ADD_AND_UPDATE_REQUEST;  // 新请求优先级更高
    } else {
        return PEND_REQUEST;            // 挂起
    }
}
```

### 4.3 优先级队列管理

```mermaid
graph TD
    A[新的性能请求] --> B{与当前活跃请求比较级别}
    
    B -->|新请求级别更高| C[替换当前请求]
    B -->|新请求级别相同/更低| D[加入挂起队列]
    
    C --> E[当前请求进入挂起队列]
    C --> F[应用新请求到sysfs]
    
    D --> G[按优先级排序插入]
    
    H[当前请求释放] --> I{检查挂起队列}
    I -->|有挂起请求| J[取最高优先级请求]
    I -->|无挂起请求| K[恢复默认值]
    
    J --> L[应用挂起的请求]
```

## 5. 具体资源的优先级配置

在 `OptsHandler.cpp` 的 `InitializeOptsTable()` 中为不同资源配置比较函数：

```cpp
void OptsHandler::InitializeOptsTable() {
    // CPU频率相关 - 频率越高性能越好
    mOptsTable[CPUFREQ_START_INDEX + CPUFREQ_MIN_FREQ_OPCODE].mCompareOpts = higher_is_better;
    mOptsTable[CPUFREQ_START_INDEX + CPUFREQ_MAX_FREQ_OPCODE].mCompareOpts = higher_is_better;
    
    // 调度相关 - boost值越高性能越好  
    mOptsTable[SCHED_START_INDEX + SCHED_BOOST_OPCODE].mCompareOpts = higher_is_better;
    
    // 带宽相关 - 带宽越高性能越好
    mOptsTable[CPUBW_HWMON_START_INDEX + CPUBW_HWMON_MINFREQ_OPCODE].mCompareOpts = higher_is_better;
    
    // GPU相关 - 功耗级别越低性能越好（反向逻辑）
    mOptsTable[GPU_START_INDEX + GPU_POWER_LEVEL].mCompareOpts = lower_is_better;
}
```

## 6. 实际使用示例分析

### 6.1 CPU频率请求场景

```cpp
// 场景：多个应用同时请求CPU最小频率
// App A: 请求 1.2GHz
int args_a[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 1200000};
int handle_a = perf_lock_acq(5000, 0, args_a, 2);  // 5秒

// App B: 请求 1.8GHz (更高性能)
int args_b[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 1800000};  
int handle_b = perf_lock_acq(3000, 0, args_b, 2);  // 3秒

// App C: 请求 1.0GHz (较低性能)
int args_c[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 1000000};
int handle_c = perf_lock_acq(8000, 0, args_c, 2);  // 8秒
```

**执行流程：**

1. App A请求生效：CPU最小频率设为1.2GHz
2. App B请求到达：1.8GHz > 1.2GHz，App B生效，App A挂起
3. App C请求到达：1.0GHz < 1.8GHz，App C挂起
4. 3秒后App B超时：检查挂起队列，App A(1.2GHz) > App C(1.0GHz)，App A恢复生效
5. 5秒后App A超时：App C生效
6. 8秒后App C超时：恢复系统默认值

### 6.2 队列状态变化

```mermaid
timeline
    title 性能锁优先级处理时序
    
    t0 : App A(1.2GHz)请求
       : 当前生效: A(1.2GHz)
       : 挂起队列: 空
    
    t1 : App B(1.8GHz)请求  
       : 当前生效: B(1.8GHz)
       : 挂起队列: A(1.2GHz)
    
    t2 : App C(1.0GHz)请求
       : 当前生效: B(1.8GHz)  
       : 挂起队列: A(1.2GHz) -> C(1.0GHz)
    
    t3 : App B超时释放
       : 当前生效: A(1.2GHz)
       : 挂起队列: C(1.0GHz)
    
    t5 : App A超时释放  
       : 当前生效: C(1.0GHz)
       : 挂起队列: 空
    
    t8 : App C超时释放
       : 当前生效: 默认值
       : 挂起队列: 空
```

## 7. 用户使用注意事项

### 7.1 性能级别设计原则

```cpp
// ✅ 正确：根据实际性能需求设置合理值
int light_boost[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 1200000};    // 轻度提升
int heavy_boost[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 1800000};    // 重度提升

// ❌ 错误：过度使用最高性能级别
int always_max[] = {MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0, 0xFFFFFFFF}; // 可能导致功耗问题
```

### 7.2 优先级冲突处理策略

**建议用户：**

1. **分层设计性能需求**

   ```cpp
   #define PERF_LEVEL_LOW    800000   // 基础性能
   #define PERF_LEVEL_MID    1200000  // 中等性能  
   #define PERF_LEVEL_HIGH   1800000  // 高性能
   #define PERF_LEVEL_BOOST  2200000  // 极限性能
   ```

2. **合理设置持续时间**

   ```cpp
   // 短时高性能冲刺
   perf_lock_acq(2000, 0, high_perf_args, 2);  // 2秒
   
   // 长时稳定性能
   perf_lock_acq(30000, 0, mid_perf_args, 2);  // 30秒
   ```

3. **及时释放不需要的锁**

   ```cpp
   if (task_completed) {
       perf_lock_rel(handle);  // 主动释放
   }
   ```

### 7.3 调试优先级问题

可以通过以下方式查看当前活跃的性能锁：

```bash
# 查看活跃的性能锁请求
adb shell dumpsys vendor.qti.hardware.perf@2.0::IPerf/default

# 或通过调试接口
perfmodule_sync_request_ext(SYNC_CMD_DUMP_DEG_INFO, &debugInfo);
```

**总结：** mp-ctl的优先级机制通过数值比较确定性能级别的高低，高性能级别的请求会抢占当前资源，而低优先级请求会被挂起。用户需要根据实际需求合理设置性能级别，避免不必要的高性能请求导致功耗问题。



## 8. Config解析

### 8.1 系统初始化时的配置加载流程

```mermaid
sequenceDiagram
    participant Main as 主进程
    participant Target as Target
    participant Store as PerfDataStore
    participant Parser as XmlParser
    participant FS as 文件系统
    
    Note over Main: 系统启动，perfmodule_init()
    Main->>Target: InitializeTarget()
    Target->>Store: Init()
    Store->>Store: XmlParserInit()
    
    Note over Store: 开始解析映射配置
    Store->>Parser: new AppsListXmlParser()
    Store->>Parser: Register(映射配置回调)
    Store->>Parser: Parse(perfmapping.xml)
    Parser->>FS: 读取perfmapping.xml
    FS-->>Parser: XML内容
    Parser->>Store: BoostParamsMappingsCB()
    Store->>Store: 构建mBoostParamsMappings
    
    Note over Store: 解析通用资源配置
    Store->>Parser: Register(资源配置回调)
    Store->>Parser: Parse(commonresourceconfigs.xml)
    Parser->>FS: 读取commonresourceconfigs.xml
    FS-->>Parser: XML内容
    Parser->>Store: CommonResourcesCB()
    Store->>Store: 构建mResourceConfig
    
    Note over Store: 解析系统节点配置
    Store->>Parser: Register(系统节点回调)
    Store->>Parser: Parse(commonsysnodesconfigs.xml)
    Parser->>FS: 读取commonsysnodesconfigs.xml
    FS-->>Parser: XML内容
    Parser->>Store: CommonSysNodesCB()
    Store->>Store: 构建mSysNodesConfig
    
    Store->>Parser: delete xmlParser
    Store-->>Target: 映射配置加载完成
```

### 8.2 目标设备特定配置加载流程

```mermaid
sequenceDiagram
    participant Target as Target
    participant Store as PerfDataStore
    participant Parser as XmlParser
    participant TC as TargetConfig
    participant FS as 文件系统
    
    Note over Target: TargetInit()完成后
    Target->>Store: TargetResourcesInit()
    Store->>Parser: new AppsListXmlParser()
    
    Note over Store: 解析目标设备资源配置
    Store->>Parser: Register(目标资源回调)
    Store->>Parser: Parse(targetresourceconfigs.xml)
    Parser->>FS: 读取targetresourceconfigs.xml
    FS-->>Parser: XML内容
    
    loop 每个配置节点
        Parser->>Store: TargetResourcesCB()
        Store->>TC: 获取内核版本/目标名称
        TC-->>Store: 当前设备信息
        Store->>Store: 验证配置适用性
        alt 配置匹配当前设备
            Store->>Store: UpdateResourceConfig()
        else 配置不匹配
            Store->>Store: 跳过此配置
        end
    end
    
    Note over Store: 解析性能提升配置
    Store->>Parser: Register(性能提升回调)
    Store->>Parser: Parse(perfboostsconfig.xml)
    Parser->>FS: 读取perfboostsconfig.xml
    FS-->>Parser: XML内容
    
    loop 每个性能配置
        Parser->>Store: BoostConfigsCB()
        Store->>Store: 解析HintID/Type/Resources
        Store->>Store: 验证目标设备/内核/分辨率
        alt 配置匹配
            Store->>Store: 存储到mBoostConfigs
        else 配置不匹配
            Store->>Store: 跳过此配置
        end
    end
    
    Store-->>Target: 设备特定配置加载完成
```

### 8.3 两个加载流程的区别

#### 8.3.1 加载时机不同

- **系统初始化**：`perfmodule_init()` → `Target::InitializeTarget()` → `PerfDataStore::Init()`
- **目标设备特定**：`Target::TargetInit()` 完成后 → `PerfDataStore::TargetResourcesInit()`

#### 8.3.2 **配置文件类型不同**

- 系统初始化：加载通用配置
  - `perfmapping.xml` (参数映射)
  - `commonresourceconfigs.xml` (通用资源)
  - `commonsysnodesconfigs.xml` (通用系统节点)
- 目标设备特定：加载设备相关配置
  - `targetresourceconfigs.xml` (设备特定资源)
  - `targetsysnodesconfigs.xml` (设备特定系统节点)
  - `perfboostsconfig.xml` (性能提升配置)
  - `powerhint.xml` (电源提示配置)

### 8.4 XML配置文件解析详细流程

```mermaid
sequenceDiagram
    participant Parser as XmlParser
    participant CB as 回调函数
    participant Store as PerfDataStore
    participant Conv as 转换工具
    
    Note over Parser: 解析perfboostsconfig.xml
    Parser->>CB: BoostConfigsCB(xmlNode)
    
    Note over CB: 解析单个Config节点
    CB->>CB: xmlGetProp("Id") 
    CB->>CB: xmlGetProp("Type")
    CB->>CB: xmlGetProp("Enable")
    CB->>CB: xmlGetProp("Timeout") 
    CB->>CB: xmlGetProp("Target")
    CB->>CB: xmlGetProp("Resources")
    
    Note over CB: 验证配置适用性
    CB->>Store: 获取当前目标设备名称
    Store-->>CB: "msmnile"
    CB->>CB: 比较Target字段
    
    alt Target匹配当前设备
        Note over CB: 解析Resources字符串
        CB->>Conv: ConvertToIntArray(resourcesPtr)
        Note over Conv: "0x40800000 1200000, 0x40C00000 1"
        Conv->>Conv: strtok_r(str, ", ")
        Conv->>Conv: strtol(token, NULL, 0)
        Conv-->>CB: [0x40800000, 1200000, 0x40C00000, 1]
        
        CB->>Store: new BoostConfigInfo()
        CB->>Store: mBoostConfigs[hintId][type].push_back()
        
        Note over CB: 处理IgnoreHints（如果存在）
        CB->>CB: xmlGetProp("IgnoreHints")
        alt 存在IgnoreHints
            CB->>Store: setIgnoreHints()
            Store->>Store: mContextHints[id][type] = hints
        end
    else Target不匹配
        CB->>CB: 跳过此配置节点
    end
```

### 8.5 运行时配置查询流程

```mermaid
sequenceDiagram
    participant App as 应用层
    participant Target as Target
    participant Store as PerfDataStore
    participant Config as 配置数据
    
    Note over App: 性能提升请求
    App->>Target: getBoostConfig(hintId=0x1081, type=1)
    Target->>Store: GetBoostConfig(0x1081, 1, mapArray, timeout)
    
    Note over Store: 查找配置映射
    Store->>Config: mBoostConfigs.find(0x1081)
    Config-->>Store: 找到HintID配置组
    Store->>Config: [0x1081].find(type=1)
    Config-->>Store: 找到Type配置列表
    
    Note over Store: 遍历匹配条件
    loop 配置列表中的每项
        Store->>Store: 检查分辨率匹配
        Store->>Store: 检查FPS匹配
        Store->>Store: 检查分化配置匹配
        
        alt 所有条件匹配
            Store->>Store: 复制mConfigTable到mapArray
            Store->>Store: 设置timeout值
            Store-->>Target: 返回配置数组大小
        else 条件不匹配
            Store->>Store: 继续下一项
        end
    end
    
    Target-->>App: 返回解析后的资源配置
    
    Note over App: 应用配置到系统
    App->>App: 根据返回的配置数组设置系统参数
```

### 8.6 配置数据结构构建流程

```mermaid
sequenceDiagram
    participant XML as XML文件
    participant CB as 解析回调
    participant Info as ConfigInfo对象
    participant Store as 数据存储
    
    Note over XML: perfmapping.xml解析
    XML->>CB: BoostParamsMappingsCB()
    CB->>CB: 解析MapType="freq"
    CB->>CB: 解析Mappings="800,1100,1300,1500,1650"
    CB->>Info: new ParamsMappingInfo()
    Info->>Info: mMapType=MAP_TYPE_FREQ
    Info->>Info: mMapTable=[800,1100,1300,1500,1650]
    CB->>Store: mBoostParamsMappings.push_back()
    
    Note over XML: commonresourceconfigs.xml解析
    XML->>CB: CommonResourcesCB()
    CB->>CB: 解析Major/Minor值
    CB->>CB: 计算qindex = Major起始索引 + Minor
    CB->>CB: 解析Node路径和Supported标志
    CB->>Info: new ResourceConfigInfo()
    Info->>Info: mResId=qindex
    Info->>Info: mNodePath=sysfs路径
    Info->>Info: mSupported=true/false
    CB->>Store: mResourceConfig.push_back()
    
    Note over XML: perfboostsconfig.xml解析
    XML->>CB: BoostConfigsCB()
    CB->>CB: 解析Id/Type/Enable/Timeout
    CB->>CB: 解析Resources配置字符串
    CB->>Info: new BoostConfigInfo()
    Info->>Info: mId=HintID
    Info->>Info: mType=HintType  
    Info->>Info: mConfigTable=资源配置数组
    Info->>Info: mTimeout=超时时间
    CB->>Store: mBoostConfigs[id][type].push_back()
```



## 9. BoostConfigReader核心功能

### 9.1 主要解析的配置文件

```cpp
// BoostConfigReader.cpp - 配置文件路径定义
#define PERF_MAPPING_XML (VENDOR_DIR"/perf/perfmapping.xml")           // 性能映射配置
#define PERF_BOOSTS_CONFIGS_XML (VENDOR_DIR"/perf/perfboostsconfig.xml") // 性能提升配置  
#define POWER_CONFIGS_XML (VENDOR_DIR"/powerhint.xml")                 // 电源提示配置
#define COMMONRESOURCE_CONFIGS_XML (VENDOR_DIR"/perf/commonresourceconfigs.xml") // 通用资源配置
#define TARGETRESOURCE_CONFIGS_XML (VENDOR_DIR"/perf/targetresourceconfigs.xml") // 目标设备资源配置
```

### 9.2 配置数据存储结构

```mermaid
graph TB
    A[PerfDataStore] --> B[BoostConfigInfo]
    A --> C[ParamsMappingInfo] 
    A --> D[ResourceConfigInfo]
    
    B --> E[性能提升配置<br/>HintID->Type->Config]
    C --> F[参数映射表<br/>频率/集群映射]
    D --> G[资源节点配置<br/>sysfs路径映射]
    
    subgraph "配置类型"
        E
        F  
        G
    end
```

### 9.3 关键配置解析逻辑

#### 性能提升配置解析

```cpp
// BoostConfigReader.cpp
void PerfDataStore::BoostConfigsCB(xmlNodePtr node, void *) {
    // 解析性能提升配置
    if(!xmlStrcmp(node->name, BAD_CAST BOOSTS_CONFIGS_XML_ELEM_CONFIG_TAG)) {
        idnum = strtol(idPtr, NULL, 0);    // Hint ID
        type = strtol(idPtr, NULL, 0);     // Hint Type  
        timeout = strtol(timeoutPtr, NULL, 0); // 超时时间
        
        // 解析资源配置字符串 "0x40800000 1200000, 0x40C00000 1"
        resourcesPtr = (char *) xmlGetProp(node, BAD_CAST BOOSTS_CONFIGS_XML_ELEM_RESOURCES_TAG);
        
        // 存储到配置映射表
        auto obj = BoostConfigInfo(idnum, type, en, timeout, fps, tname, res, resourcesPtr);
        store->mBoostConfigs[idnum][type].push_back(obj);
    }
}
```

#### 参数映射解析

```cpp
void PerfDataStore::BoostParamsMappingsCB(xmlNodePtr node, void *) {
    // 解析频率/集群映射表
    maptype = (char *) xmlGetProp(node, BAD_CAST PERF_BOOSTS_XML_MAPTYPE_TAG);     // "freq" or "cluster"
    mappings = (char *) xmlGetProp(node, BAD_CAST PERF_BOOSTS_XML_MAPPINGS_TAG);   // "800,1100,1300,1500,1650"
    
    msize = ConvertToIntArray(mappings, marray, MAX_MAP_TABLE_SIZE);
    auto tmp = new ParamsMappingInfo(mtype, tname, res, marray, msize);
    store->mBoostParamsMappings.push_back(tmp);
}
```

### 9.4 配置查询接口

```cpp
// 获取性能提升配置
uint32_t PerfDataStore::GetBoostConfig(int32_t hintId, int32_t type, 
                                       int32_t *mapArray, int32_t *timeout,
                                       const char *tName, uint32_t res, int32_t fps) {
    // 从mBoostConfigs中查找对应配置
    if (mBoostConfigs.find(hintId) != mBoostConfigs.end() && 
        mBoostConfigs[hintId].find(type) != mBoostConfigs[hintId].end()) {
        
        for (auto it = itbegin; it != itend; ++it) {
            if (匹配条件) {
                mapsize = (*it).mConfigsSize;
                for (uint32_t i=0; i<mapsize; i++) {
                    mapArray[i] = (*it).mConfigTable[i];  // 返回资源配置数组
                }
                *timeout = (*it).mTimeout;
                return mapsize;
            }
        }
    }
}
```

### 9.5 实际使用流程

```mermaid
sequenceDiagram
    participant App as 应用
    participant Target as Target模块
    participant Store as PerfDataStore
    participant XML as XML配置文件
    
    Note over XML: 系统启动时加载配置
    XML->>Store: 解析XML配置文件
    Store->>Store: 构建配置映射表
    
    Note over App: 运行时查询配置
    App->>Target: getBoostConfig(hintId, type)
    Target->>Store: GetBoostConfig()
    Store->>Store: 查找配置映射表
    Store-->>Target: 返回资源配置数组
    Target-->>App: 应用配置参数
```

### 9.6 配置文件示例结构

```xml
<!-- perfboostsconfig.xml 示例 -->
<BoostConfigs>
    <PerfBoost>
        <Config Id="0x1081" Type="1" Enable="true" Timeout="4000" Target="msmnile">
            <Resources>0x40800000 1200000, 0x40C00000 1</Resources>
        </Config>
    </PerfBoost>
</BoostConfigs>

<!-- perfmapping.xml 示例 -->  
<PerfBoosts>
    <BoostParamsMappings>
        <BoostAttributes MapType="freq" Target="msmnile" Resolution="1080p">
            <Mappings>800,1100,1300,1500,1650</Mappings>
        </BoostAttributes>
    </BoostParamsMappings>
</PerfBoosts>
```



## 10. MP-CTL加密云控方案设计

### 10.1 架构设计

```mermaid
graph TB
    A[配置文件加载请求] --> B[路径优先级检查]
    B --> C{/data/perf/存在?}
    C -->|是| D[读取/data/perf/文件]
    C -->|否| E[读取/vendor/etc/perf/文件]
    D --> F[检测文件是否加密]
    E --> F
    F --> G{文件已加密?}
    G -->|是| H[解密文件内容]
    G -->|否| I[直接使用原文件]
    H --> J[XML解析]
    I --> J
    J --> K[配置数据存储]
```

### 10.2 核心实现策略

1. **文件路径管理器**：统一管理配置文件路径选择
2. **加密检测与解密**：自动识别加密文件并解密
3. **透明集成**：对现有XML解析逻辑透明

### 10.3 详细实现方案

#### 10.3.1 新增配置文件管理类

**插入位置：** `BoostConfigReader.h` 和 `BoostConfigReader.cpp`

```cpp
// BoostConfigReader.h 新增
class ConfigFileManager {
private:
    static const char* DEBUG_CONFIG_DIR;     // "/data/perf/"
    static const char* VENDOR_CONFIG_DIR;    // "/vendor/etc/perf/"
    
    struct ConfigFileInfo {
        const char* filename;
        bool isEncrypted;
        std::string content;
    };
    
public:
    // 获取配置文件路径（支持优先级）
    static std::string getConfigFilePath(const char* filename);
    
    // 读取并解密配置文件
    static bool readAndDecryptConfig(const std::string& filepath, std::string& content);
    
    // 检测文件是否加密
    static bool isFileEncrypted(const std::string& filepath);
    
    // 解密文件内容
    static bool decryptFileContent(const std::string& encryptedContent, std::string& decryptedContent);
};
```

#### 10.3.2 修改现有宏定义

**修改位置：** `BoostConfigReader.cpp` 文件开头

```cpp
// 原来的固定路径定义 - 删除
// #define PERF_MAPPING_XML (VENDOR_DIR"/perf/perfmapping.xml")

// 新的动态路径获取
#define PERF_MAPPING_XML ConfigFileManager::getConfigFilePath("perfmapping.xml").c_str()
#define PERF_BOOSTS_CONFIGS_XML ConfigFileManager::getConfigFilePath("perfboostsconfig.xml").c_str()
#define POWER_CONFIGS_XML ConfigFileManager::getConfigFilePath("powerhint.xml").c_str()
#define COMMONRESOURCE_CONFIGS_XML ConfigFileManager::getConfigFilePath("commonresourceconfigs.xml").c_str()
#define TARGETRESOURCE_CONFIGS_XML ConfigFileManager::getConfigFilePath("targetresourceconfigs.xml").c_str()
```

#### 10.3.3 核心实现代码

**插入位置：** `BoostConfigReader.cpp`

```cpp
// 静态成员定义
const char* ConfigFileManager::DEBUG_CONFIG_DIR = "/data/perf/";
const char* ConfigFileManager::VENDOR_CONFIG_DIR = "/vendor/etc/perf/";

std::string ConfigFileManager::getConfigFilePath(const char* filename) {
    std::string debugPath = std::string(DEBUG_CONFIG_DIR) + filename;
    std::string vendorPath = std::string(VENDOR_CONFIG_DIR) + filename;
    
    // 优先检查debug目录
    if (access(debugPath.c_str(), F_OK) == 0) {
        QLOGL(LOG_TAG, QLOG_L1, "Using debug config: %s", debugPath.c_str());
        return debugPath;
    }
    
    // 回退到vendor目录
    QLOGL(LOG_TAG, QLOG_L1, "Using vendor config: %s", vendorPath.c_str());
    return vendorPath;
}

bool ConfigFileManager::isFileEncrypted(const std::string& filepath) {
    FILE* file = fopen(filepath.c_str(), "rb");
    if (!file) return false;
    
    // 读取文件头部Magic Number判断是否加密
    char header[8];
    size_t read = fread(header, 1, 8, file);
    fclose(file);
    
    if (read >= 4) {
        // 检查加密标识 (例如: "ENC\x01")
        return (memcmp(header, "ENC\x01", 4) == 0);
    }
    return false;
}

bool ConfigFileManager::decryptFileContent(const std::string& encryptedContent, 
                                          std::string& decryptedContent) {
    // 实现你的解密算法
    // 这里以简单的XOR解密为例
    const char* key = "your_secret_key";
    size_t keyLen = strlen(key);
    
    decryptedContent.clear();
    decryptedContent.reserve(encryptedContent.length());
    
    for (size_t i = 4; i < encryptedContent.length(); ++i) { // 跳过4字节头部
        char decrypted = encryptedContent[i] ^ key[(i-4) % keyLen];
        decryptedContent.push_back(decrypted);
    }
    
    return true;
}

bool ConfigFileManager::readAndDecryptConfig(const std::string& filepath, 
                                           std::string& content) {
    // 读取文件内容
    std::ifstream file(filepath, std::ios::binary);
    if (!file.is_open()) {
        QLOGE(LOG_TAG, "Failed to open config file: %s", filepath.c_str());
        return false;
    }
    
    std::string fileContent((std::istreambuf_iterator<char>(file)),
                           std::istreambuf_iterator<char>());
    file.close();
    
    // 检查是否加密
    if (isFileEncrypted(filepath)) {
        QLOGL(LOG_TAG, QLOG_L1, "Decrypting config file: %s", filepath.c_str());
        return decryptFileContent(fileContent, content);
    } else {
        QLOGL(LOG_TAG, QLOG_L1, "Using plain config file: %s", filepath.c_str());
        content = fileContent;
        return true;
    }
}
```

#### 10.3.4 修改XML解析器集成

**修改位置：** `XmlParser.h` 和 `XmlParser.cpp`

需要为 `AppsListXmlParser` 类添加支持内存XML解析的方法：

```cpp
// XmlParser.h 新增方法声明
class AppsListXmlParser {
public:
    // 现有方法...
    
    // 新增：从内存内容解析XML
    bool ParseFromMemory(const std::string& xmlContent);
    
private:
    // 现有成员...
};

// XmlParser.cpp 实现
bool AppsListXmlParser::ParseFromMemory(const std::string& xmlContent) {
    xmlDocPtr doc = xmlParseMemory(xmlContent.c_str(), xmlContent.length());
    if (doc == NULL) {
        QLOGE(LOG_TAG, "Failed to parse XML from memory");
        return false;
    }
    
    // 复用现有的解析逻辑
    bool result = processXmlDoc(doc);
    xmlFreeDoc(doc);
    return result;
}
```

#### 10.3.5 修改PerfDataStore解析流程

**修改位置：** `BoostConfigReader.cpp` 中的 `XmlParserInit()` 和 `TargetResourcesInit()` 方法

```cpp
void PerfDataStore::XmlParserInit() {
    AppsListXmlParser *xmlParser = new(std::nothrow) AppsListXmlParser();
    if (NULL == xmlParser) return;

    // 解析映射配置 - 修改后
    std::string mappingContent;
    std::string mappingPath = ConfigFileManager::getConfigFilePath("perfmapping.xml");
    if (ConfigFileManager::readAndDecryptConfig(mappingPath, mappingContent)) {
        int32_t idnum = xmlParser->Register(xmlMappingsRoot, xmlChildMappings, 
                                          BoostParamsMappingsCB, NULL);
        xmlParser->ParseFromMemory(mappingContent);  // 使用内存解析
        xmlParser->DeRegister(idnum);
    }

    // 类似地修改其他配置文件的解析...
    
    delete xmlParser;
}

void PerfDataStore::TargetResourcesInit() {
    AppsListXmlParser *xmlParser = new(std::nothrow) AppsListXmlParser();
    if (NULL == xmlParser) return;
    
    // 解析性能提升配置 - 修改后
    std::string boostContent;
    std::string boostPath = ConfigFileManager::getConfigFilePath("perfboostsconfig.xml");
    if (ConfigFileManager::readAndDecryptConfig(boostPath, boostContent)) {
        int32_t idnum = xmlParser->Register(xmlConfigsRoot, xmlChildConfigs, 
                                          BoostConfigsCB, NULL);
        xmlParser->ParseFromMemory(boostContent);
        xmlParser->DeRegister(idnum);
    }
    
    // 类似地修改其他配置文件...
    
    delete xmlParser;
}
```

### 10.4 实现时序图

```mermaid
sequenceDiagram
    participant Store as PerfDataStore
    participant MGR as ConfigFileManager
    participant FS as 文件系统
    participant DECRYPT as 解密模块
    participant Parser as XmlParser
    
    Note over Store: 配置加载开始
    Store->>MGR: getConfigFilePath("perfmapping.xml")
    MGR->>FS: access("/data/perf/perfmapping.xml")
    FS-->>MGR: 存在/不存在
    
    alt /data/perf/存在
        MGR-->>Store: "/data/perf/perfmapping.xml"
    else vendor路径
        MGR-->>Store: "/vendor/etc/perf/perfmapping.xml"
    end
    
    Store->>MGR: readAndDecryptConfig(path)
    MGR->>FS: 读取文件内容
    FS-->>MGR: 文件二进制内容
    
    MGR->>MGR: isFileEncrypted()
    alt 文件已加密
        MGR->>DECRYPT: decryptFileContent()
        DECRYPT-->>MGR: 解密后的XML内容
    else 文件未加密
        MGR->>MGR: 直接使用原内容
    end
    
    MGR-->>Store: XML文本内容
    Store->>Parser: ParseFromMemory(xmlContent)
    Parser->>Parser: xmlParseMemory()
    Parser-->>Store: 解析完成
```

## 11. 自定义事件

### 11.1 定义新的Hint ID

**位置：** `VendorIPerf.h`

```cpp
VENDOR_HINT_DOWN_CONTROL = 0x00001093,
```

### 11.2 添加XML配置文件条目

**位置：** `perfboostsconfig.xml` 配置文件

```xml
<Config Id="0x1093" Type="0" Enable="true" Timeout="3000" Target="你的目标设备">
    <Resources>0x40800000 1200000, 0x40C00000 1</Resources>
</Config>
```

### 11.3 注册Hint扩展处理器（如需特殊处理）

**位置：** `TargetInit.cpp` 中的 `Target::InitializeTarget()`

```cpp
void Target::InitializeTarget() {
    // 现有的hint注册...
    
    // 添加新的hint处理器
    hintExt.Register(VENDOR_HINT_DOWN_CONTROL, 
                     DownControlAction::DownControlPreAction,    // 可选：预处理
                     DownControlAction::DownControlPostAction,   // 可选：后处理  
                     DownControlAction::DownControlHintExcluder); // 可选：排除器
}
```

### 11.4 实现Hint处理逻辑（如需特殊处理）

**位置：** `HintExtHandler.h` 和 `HintExtHandler.cpp`

#### 11.4.1 头文件声明

```cpp
// HintExtHandler.h
class DownControlAction {
public:
    static int32_t DownControlPreAction(mpctl_msg_t *pMsg);
    static int32_t DownControlPostAction(mpctl_msg_t *pMsg); 
    static int32_t DownControlHintExcluder(mpctl_msg_t *pMsg);
};
```

#### 11.4.2 实现文件

```cpp
// HintExtHandler.cpp
int32_t DownControlAction::DownControlPreAction(mpctl_msg_t *pMsg) {
    if (NULL == pMsg) {
        return FAILED;
    }
    
    // 你的预处理逻辑
    // 例如：修改参数、检查条件等
    QLOGL(LOG_TAG, QLOG_L1, "DownControl PreAction: hint_id=0x%x", pMsg->hint_id);
    
    return SUCCESS;
}

int32_t DownControlAction::DownControlPostAction(mpctl_msg_t *pMsg) {
    if (NULL == pMsg) {
        return FAILED; 
    }
    
    // 你的后处理逻辑
    // 例如：调整资源配置、设置特殊参数等
    uint16_t size = pMsg->data;
    
    for (uint16_t i = 0; i < size-1; i = i + 2) {
        // 处理资源配置对
        int32_t opcode = pMsg->pl_args[i];
        int32_t value = pMsg->pl_args[i+1];
        
        // 根据需要修改配置
        if (opcode == MPCTLV3_MIN_FREQ_CLUSTER_BIG_CORE_0) {
            pMsg->pl_args[i+1] = value * 0.8;  // 例如：降频处理
        }
    }
    
    return SUCCESS;
}

int32_t DownControlAction::DownControlHintExcluder(mpctl_msg_t *pMsg) {
    // 排除逻辑：返回SUCCESS表示排除此hint，返回FAILED表示允许执行
    if (某种条件) {
        return SUCCESS;  // 排除
    }
    return FAILED;       // 允许执行
}
```

## 5. 添加资源处理逻辑（如需新的OpCode）

**位置：** `OptsHandler.cpp` 中的 `InitializeOptsTable()`

如果你的hint使用了新的OpCode，需要注册对应的处理函数：

```cpp
void OptsHandler::InitializeOptsTable() {
    // 现有初始化...
    
    // 如果有新的资源OpCode需要特殊处理
    mOptsTable[YOUR_NEW_OPCODE_INDEX].mApplyOpts = your_apply_function;
    mOptsTable[YOUR_NEW_OPCODE_INDEX].mResetOpts = your_reset_function;
    mOptsTable[YOUR_NEW_OPCODE_INDEX].mCompareOpts = your_compare_function;
}
```

## 6. 调试和验证

### 6.1 日志验证

**位置：** `PerfController.cpp` 中的 `perfmodule_submit_request()`

```cpp
QLOGL(LOG_TAG, QLOG_L1, "Received hint_id=0x%" PRIx32 ", hint_type=%" PRId32, 
      pMsg->hint_id, pMsg->hint_type);
```

### 6.2 配置验证

检查你的XML配置是否被正确加载：

```bash
# 查看加载的配置
adb shell dumpsys vendor.qti.hardware.perf@2.0::IPerf/default
```

## 核心流程总结

```mermaid
graph TB
    A[应用调用VENDOR_HINT_DOWN_CONTROL] --> B[PerfController接收请求]
    B --> C[查找XML配置]
    C --> D[HintExtHandler预处理]
    D --> E[应用资源配置到OptsHandler]
    E --> F[写入sysfs节点]
    F --> G[HintExtHandler后处理]
    G --> H[返回结果]
```

