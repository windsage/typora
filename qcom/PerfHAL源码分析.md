# 1. PerfHAL 组件部署结构

当前模块代码路径为`vendor/qcom/proprietary/perf-core/perf-hal`

## 1.1 Binary 文件

```bash
# AIDL HAL服务
/vendor/bin/hw/vendor.qti.hardware.perf2-service

# Socket HAL服务
/vendor/bin/hw/vendor.qti.hardware.perf-hal-service

# 客户端库
/system/lib64/libqti-perfd-client.so
/system/lib64/libqti-perfd-client_system.so
/vendor/lib64/libqti-perfd.so
```

## 1.2 配置文件

```bash
# 服务启动配置
/vendor/etc/init/vendor.qti.hardware.perf2-hal-service.rc
/vendor/etc/init/vendor.qti.hardware.perf-hal-service.rc

# XML配置文件
/vendor/etc/perf/targetconfig.xml
/vendor/etc/perf/perfconfigstore.xml  
/vendor/etc/perf/perfboostsconfig.xml

# 接口描述文件
/vendor/etc/vintf/manifest/vendor.qti.hardware.perf2.xml
```

## 1.3 启动时机分析

### AIDL服务启动

```bash
# vendor.qti.hardware.perf2-hal-service.rc
service vendor.qti.hardware.perf2-service /vendor/bin/hw/vendor.qti.hardware.perf2-service
    class hal
    user system
    group system
    capabilities SYS_NICE
    task_profiles ProcessCapacityHigh HighPerformance
```

### Socket服务启动

```bash
# vendor.qti.hardware.perf-hal-service.rc  
service vendor.qti.hardware.perf-hal-service /vendor/bin/hw/vendor.qti.hardware.perf-hal-service
    class hal
    user system
    group system
    socket perfservice stream 0666 system system
```

### AIDL vs Socket服务区别

##### AIDL服务 (`service.cpp` + `Perf.cpp`)

- **用途**：标准Android HAL服务，面向应用框架层
- **客户端**：通过Binder IPC调用，主要是系统服务和应用
- **特点**：完整的HAL接口，支持所有perfHint、perfLock等API
- **场景**：正常Android系统中的性能优化请求

##### Socket服务 (`service_sock.cpp` + `Perf_sock.cpp`)

- **用途**：轻量级本地通信，面向native客户端

- **客户端**：通过Unix Domain Socket连接的native进程

- **特点**：精简的消息协议，减少开销

- **场景**：嵌入式或资源受限环境、系统启动早期阶段

  

# 2. 系统架构图 

```mermaid
graph TB
    subgraph "Application Layer 应用层"
        APP[Applications 应用程序]
        BF[BoostFramework]
        PS[PerfService]
    end
    
    subgraph "JNI/Client Layer JNI/客户端层"
        JNI[JNI Layer]
        CLIENT[mp-ctl-client]
    end

    subgraph "PerfHAL Module"
        subgraph "Service Interface 服务接口层"
            PERF["Perf.cpp/h<br/>AIDL接口实现"]
            PERF_SOCK["Perf_sock.cpp/h<br/>Socket接口实现"]
            PERF_STUB["Perf_Stub.cpp<br/>存根实现"]
            SERVICE["service.cpp<br/>AIDL服务启动"]
            SERVICE_SOCK["service_sock.cpp<br/>Socket服务启动"]
        end

        subgraph "Configuration Management 配置管理层"
            TC["TargetConfig.cpp/h<br/>目标平台配置管理<br/>• CPU集群信息<br/>• 核心数配置<br/>• 频率设置<br/>• SoC ID映射"]
            DC["DivergentConfig.cpp/h<br/>动态硬件配置检测<br/>• 运行时读取集群信息<br/>• 显示/GPU子系统检测<br/>• 物理集群ID映射"]
            PC["PerfConfig.cpp/h<br/>性能配置存储<br/>• XML配置解析<br/>• 属性值存储<br/>• 目标特定配置"]
        end

        subgraph "Core Processing 核心处理层"
            GL["PerfGlueLayer.cpp/h<br/>胶水层管理器<br/>• 模块注册管理<br/>• 请求分发<br/>• 线程池处理<br/>• 生命周期管理"]
            
            subgraph "Performance Modules 性能模块"
                MPCTL["libqti-perfd.so<br/>主要性能控制模块"]
                LM["liblearningmodule.so<br/>学习模块"]
                MEMPERF["libmemperfd.so<br/>内存性能模块"]
                OTHER[其他性能模块...]
            end
        end

        subgraph "Utility Layer 工具层"
            VIG["VendorIGlue.cpp/h<br/>厂商接口胶水层"]
        end
    end

    subgraph "System Resources 系统资源"
        subgraph "SysFS Nodes 系统文件节点"
            CPUFREQ["/sys/devices/system/cpu/cpufreq/<br/>CPU频率控制"]
            CAPACITY["/sys/devices/system/cpu/cpu*/cpu_capacity<br/>CPU容量信息"]
            CLUSTER["/sys/devices/system/cpu/cpu*/topology/cluster_id<br/>集群ID信息"]
            SUBSYS["/sys/devices/soc0/<br/>子系统可用性检查"]
        end

        subgraph "Configuration Files 配置文件"
            TARGET_XML["targetconfig.xml<br/>目标配置文件"]
            PERF_XML["perfconfigstore.xml<br/>性能配置存储"]
        end

        subgraph "System Properties 系统属性"
            PROPS["ro.product.name<br/>ro.board.first_api_level<br/>vendor.debug.trace.perf<br/>vendor.disable.perf.hal"]
        end
    end

    %% 连接关系
    APP --> BF
    BF --> PS
    PS --> JNI
    JNI --> CLIENT

    CLIENT --> PERF
    CLIENT --> PERF_SOCK

    SERVICE --> PERF
    SERVICE_SOCK --> PERF_SOCK

    PERF --> GL
    PERF_SOCK --> GL

    GL --> MPCTL
    GL --> LM
    GL --> MEMPERF
    GL --> OTHER

    PERF --> TC
    PERF --> DC
    PERF --> PC

    TC --> TARGET_XML
    PC --> PERF_XML
    DC --> CPUFREQ
    DC --> CAPACITY
    DC --> CLUSTER
    DC --> SUBSYS

    TC --> PROPS

    VIG --> GL

    %% API流程标注
    PERF -.->|"perfLockAcquire<br/>perfHint<br/>perfEvent<br/>perfGetProp"| GL
    GL -.->|"SubmitRequest<br/>SyncRequest"| MPCTL
    GL -.->|异步处理| LM
    GL -.->|条件加载| MEMPERF

    %% 样式
    classDef appLayer fill:#e1f5fe
    classDef serviceLayer fill:#f3e5f5
    classDef configLayer fill:#e8f5e8
    classDef coreLayer fill:#fff3e0
    classDef moduleLayer fill:#fce4ec
    classDef sysLayer fill:#f0f4c3

    class APP,BF,PS appLayer
    class PERF,PERF_SOCK,PERF_STUB,SERVICE,SERVICE_SOCK serviceLayer
    class TC,DC,PC configLayer
    class GL coreLayer
    class MPCTL,LM,MEMPERF,OTHER moduleLayer
    class CPUFREQ,CAPACITY,CLUSTER,SUBSYS,TARGET_XML,PERF_XML,PROPS sysLayer

```



# 3. 调用链路分析

## 3.1 客户端调用入口

### mp-ctl-client 调用链

```cpp
// mp-ctl-client/client.cpp
int perf_lock_acq(int handle, int duration, int list[], int numArgs) {
    // 1. 检查HAL类型 (AIDL vs HIDL)
    if (isHalTypeAidl()) {
        if (getPerfAidlAndLinkToDeath() == false) {
            return rc;
        }
    } else {
        if (getPerfServiceAndLinkToDeath() == false) {
            return rc;
        }
    }
    
    // 2. 准备参数
    std::vector<int32_t> boostsList;
    std::vector<int32_t> paramList;
    
    paramList.push_back(gettid());  // 客户端TID
    paramList.push_back(getpid());  // 客户端PID
    boostsList.assign(list, (list + numArgs));
    
    // 3. 调用HAL接口
    shared_lock<shared_timed_mutex> read_lock(gPerf_lock);
    if (gPerfAidl != NULL) {
        gPerfAidl->perfLockAcquire(handle, duration, boostsList, paramList, &intReturn);
        rc = intReturn;
    }
    
    return rc;
}
```

### PerfService 调用链

```java
// Performance.java
public int perfLockAcquire(int duration, int... list) {
    if (sIsPlatformOrPrivApp && !sIsUntrustedDomain) {
        // 直接调用native方法
        mHandle = native_perf_lock_acq(mHandle, duration, list);
    } else {
        // 通过Binder调用PerfService
        synchronized (mLock) {
            if (sPerfService != null) {
                mHandle = sPerfService.perfLockAcquire(duration, list.length, list);
            }
        }
    }
    return mHandle;
}

// JNI调用 (com_qualcomm_qti_Performance.cpp)
static jint com_qualcomm_qti_performance_native_perf_lock_acq(JNIEnv *env, jobject clazz, 
                                                             jint handle, jint duration, 
                                                             jintArray list) {
    jint listlen = env->GetArrayLength(list);
    jint buf[MAX_SIZE_ALLOWED];
    env->GetIntArrayRegion(list, 0, listlen, buf);
    
    ret = perf_lock_acq(handle, duration, buf, listlen);
    return ret;
}
```

## 3.2 HAL服务接收处理

### AIDL接口实现 (Perf.cpp)

```cpp
ScopedAStatus Perf::perfLockAcquire(int32_t in_pl_handle, int32_t in_duration, 
                                   const std::vector<int32_t>& in_boostsList, 
                                   const std::vector<int32_t>& in_reserved, 
                                   int32_t* _aidl_return) {
    *_aidl_return = -1;
    
    // 1. 检查服务状态
    if (!checkPerfStatus(__func__)) {
        return ndk::ScopedAStatus::ok();
    }
    
    // 2. 构造内部消息结构
    mpctl_msg_t pMsg;
    memset(&pMsg, 0, sizeof(mpctl_msg_t));
    
    pMsg.req_type = MPCTL_CMD_PERFLOCKACQ;  // 请求类型
    
    // 3. 解析客户端信息
    for (uint32_t i = 0; i < INDEX_END && i < in_reserved.size(); i++) {
        switch (i) {
            case INDEX_CLIENT_TID: 
                pMsg.client_tid = in_reserved[i];
                break;
            case INDEX_CLIENT_PID: 
                pMsg.client_pid = in_reserved[i];
                break;
        }
    }
    
    // 4. 拷贝boost参数列表
    uint32_t size = in_boostsList.size();
    if (size > MAX_ARGS_PER_REQUEST) {
        QLOGE(LOG_TAG, "Maximum number of arguments exceeded");
        return ndk::ScopedAStatus::ok();
    }
    
    std::copy(in_boostsList.begin(), in_boostsList.end(), pMsg.pl_args);
    pMsg.data = size;           // 参数个数
    pMsg.pl_time = in_duration; // 持续时间
    pMsg.pl_handle = in_pl_handle; // 性能锁句柄
    pMsg.version = MSG_VERSION;
    pMsg.size = sizeof(mpctl_msg_t);
    
    // 5. 处理续期标志
    if (in_pl_handle > 0) {
        pMsg.renewFlag = true;
    }
    
    // 6. 提交到GlueLayer处理
    *_aidl_return = mImpl.PerfGlueLayerSubmitRequest(&pMsg);
    return ndk::ScopedAStatus::ok();
}
```

### Hint处理实现

```cpp
ScopedAStatus Perf::perfHint(int32_t hint, const std::string& userDataStr, 
                            int32_t userData1, int32_t userData2, 
                            const std::vector<int32_t>& reserved, 
                            int32_t* _aidl_return) {
    *_aidl_return = -1;
    
    if (!checkPerfStatus(__func__)) {
        return ndk::ScopedAStatus::ok();
    }
    
    mpctl_msg_t pMsg;
    memset(&pMsg, 0, sizeof(mpctl_msg_t));
    
    pMsg.req_type = MPCTL_CMD_PERFLOCKHINTACQ;
    
    // 解析扩展参数
    for (uint32_t i = 0; i < INDEX_END && i < reserved.size(); i++) {
        switch (i) {
            case INDEX_CLIENT_TID: 
                pMsg.client_tid = reserved[i];
                break;
            case INDEX_CLIENT_PID: 
                pMsg.client_pid = reserved[i];
                break;
            case INDEX_REQUEST_OFFLOAD_FLAG: 
                pMsg.offloadFlag = reserved[i];  // 异步处理标志
                break;
        }
    }
    
    pMsg.hint_id = hint;        // Hint ID
    pMsg.pl_time = userData1;   // 持续时间
    pMsg.hint_type = userData2; // Hint类型
    pMsg.version = MSG_VERSION;
    pMsg.size = sizeof(mpctl_msg_t);
    
    // 拷贝包名
    strlcpy(pMsg.usrdata_str, userDataStr.c_str(), MAX_MSG_APP_NAME_LEN);
    pMsg.usrdata_str[MAX_MSG_APP_NAME_LEN - 1] = '\0';
    
    *_aidl_return = mImpl.PerfGlueLayerSubmitRequest(&pMsg);
    
    // 发送回调通知
    if (*_aidl_return != INIT_NOT_COMPLETED) {
        sendPerfCallbackOffload(hint, userDataStr, userData1, userData2, reserved);
    }
    
    return ndk::ScopedAStatus::ok();
}
```

## 3.3 PerfGlueLayer 请求分发

### 请求类型判断与分发

```cpp
int32_t PerfGlueLayer::PerfGlueLayerSubmitRequest(mpctl_msg_t *msg) {
    PerfThreadPool &ptp = PerfThreadPool::getPerfThreadPool();
    int32_t handle = -1, ret = -1;
    uint8_t cmd = msg->req_type;
    TargetConfig &tc = TargetConfig::getTargetConfig();
    
    // 1. 检查初始化状态
    if (tc.getInitCompleted() == false) {
        QLOGE(LOG_TAG, "Request not submitted as target init is not complete");
        return INIT_NOT_COMPLETED;
    }
    
    pthread_mutex_lock(&mMutex);
    
    // 2. 根据命令类型分发处理
    if ((cmd >= MPCTL_CMD_PERFLOCKACQ) && 
        (cmd <= MPCTL_CMD_PERFLOCK_RESTORE_GOV_PARAMS)) {
        
        // 2.1 直接性能锁请求 - 同步处理
        QLOGL(LOG_TAG, QLOG_L2, "Direct perflock request");
        if (mMpctlMod && !mMpctlMod->IsEmpty()) {
            if (mMpctlMod->GetLib().SubmitRequest) {
                handle = mMpctlMod->GetLib().SubmitRequest(msg);
            }
        }
    }
    else if (MPCTL_CMD_PERFLOCKHINTACQ == cmd || MPCTL_CMD_PERFEVENT == cmd) {
        
        // 2.2 Hint请求 - 需要多模块处理
        QLOGL(LOG_TAG, QLOG_L2, "Hint request");
        handle = INVALID_VAL;
        uint32_t hint = msg->hint_id;
        
        // 异步处理其他模块
        mpctl_msg_t *tmp_msg = (mpctl_msg_t*)calloc(1, sizeof(mpctl_msg_t));
        if (tmp_msg != NULL) {
            memcpy(tmp_msg, msg, sizeof(mpctl_msg_t));
            
            // 提交异步任务
            int32_t rc = ptp.placeTask([=] () mutable {
                for (uint8_t i = 0; i < MAX_MODULES; i++) {
                    if (&mModules[i] == mMpctlMod) {
                        continue; // 跳过mpctl模块，后面同步处理
                    }
                    
                    if (!mModules[i].IsEmpty() && 
                        mModules[i].IsThisEventRegistered(hint)) {
                        if (mModules[i].GetLib().SubmitRequest) {
                            int32_t handle_in = mModules[i].GetLib().SubmitRequest(tmp_msg);
                            QLOGL(LOG_TAG, QLOG_L2, "Request %x for %s returned Handle: %d", 
                                  tmp_msg->hint_id, mModules[i].GetLib().mLibFileName, handle_in);
                        }
                    }
                }
                
                if (tmp_msg != NULL) {
                    free(tmp_msg);
                }
            });
            
            // 如果线程池满了，动态扩展
            if (rc == FAILED) {
                uint32_t new_size = ptp.resize(1);
                QLOGE(LOG_TAG, "Request %x failed, adding thread. New pool size: %u", 
                      msg->hint_id, new_size);
            }
        }
        
        // 同步处理mpctl模块
        if (mMpctlMod && !mMpctlMod->IsEmpty() && 
            mMpctlMod->IsThisEventRegistered(hint)) {
            if (mMpctlMod->GetLib().SubmitRequest) {
                handle = ret = mMpctlMod->GetLib().SubmitRequest(msg);
                QLOGL(LOG_TAG, QLOG_L1, "Package: %s, hint: %x, libname: %s, Handle: %d",
                      msg->usrdata_str, hint, mMpctlMod->GetLib().mLibFileName, handle);
            }
        }
    }
    else if (MPCTL_CMD_PERFGETFEEDBACK == cmd) {
        
        // 2.3 反馈请求 - 同步处理所有模块
        QLOGL(LOG_TAG, QLOG_L2, "Feedback request");
        handle = INVALID_VAL;
        uint32_t hint = msg->hint_id;
        
        for (uint8_t i = 0; i < MAX_MODULES; i++) {
            if (&mModules[i] == mMpctlMod) {
                continue; // 最后处理mpctl
            }
            
            if (!mModules[i].IsEmpty() && 
                mModules[i].IsThisEventRegistered(hint)) {
                if (mModules[i].GetLib().SubmitRequest) {
                    ret = mModules[i].GetLib().SubmitRequest(msg);
                    if (&mModules[i] == mMpctlMod) {
                        handle = ret;
                    }
                }
            }
        }
    }
    
    // 3. 返回值处理 - mpctl的返回值优先
    if (handle == INVALID_VAL) {
        handle = ret;
    }
    
    pthread_mutex_unlock(&mMutex);
    return handle;
}
```

## 3.4 性能模块动态加载

### 模块加载机制

```cpp
int32_t PerfGlueLayer::LoadPerfLib(const char *libname) {
    int32_t ret = -1;
    PerfModule tmp;
    
    pthread_mutex_lock(&mMutex);
    
    // 1. 检查是否已加载
    if (!mLoadedModules.empty() &&
        std::find(mLoadedModules.begin(), mLoadedModules.end(), libname) != 
        mLoadedModules.end()) {
        pthread_mutex_unlock(&mMutex);
        return ret;
    }
    
    // 2. 标记正在加载
    mLoadedModules.push_back(libname);
    pthread_mutex_unlock(&mMutex);
    
    // 3. 加载动态库
    int32_t libret = tmp.LoadPerfLib(libname);
    
    pthread_mutex_lock(&mMutex);
    if (libret < 0) {
        // 加载失败，清理标记
        mLoadedModules.erase(std::remove(mLoadedModules.begin(),
                                       mLoadedModules.end(), libname),
                           mLoadedModules.end());
        pthread_mutex_unlock(&mMutex);
        return ret;
    }
    
    // 4. 查找已注册的模块并关联
    for (uint8_t i = 0; i < MAX_MODULES; i++) {
        int32_t len = strlen(mModules[i].GetLib().mLibFileName);
        if (!mModules[i].IsEmpty() &&
            !strncmp(mModules[i].GetLib().mLibFileName, libname, len) &&
            !mModules[i].GetLib().is_opened) {
            
            // 关联函数指针
            mModules[i].GetLib().is_opened = tmp.GetLib().is_opened;
            mModules[i].GetLib().dlhandle = tmp.GetLib().dlhandle;
            mModules[i].GetLib().Init = tmp.GetLib().Init;
            mModules[i].GetLib().Exit = tmp.GetLib().Exit;
            mModules[i].GetLib().SubmitRequest = tmp.GetLib().SubmitRequest;
            mModules[i].GetLib().SyncRequest = tmp.GetLib().SyncRequest;
            
            ret = i;
            QLOGL(LOG_TAG, QLOG_L1, "LoadPerfLib loading %s at %u", libname, ret);
            break;
        }
    }
    
    pthread_mutex_unlock(&mMutex);
    return ret;
}

// 模块具体加载实现
int32_t PerfModule::LoadPerfLib(const char *libname) {
    const char *rc = NULL;
    int32_t ret = -1;
    
    if (!mLibHandle.is_opened && (NULL != libname)) {
        // 1. 打开动态库
        mLibHandle.dlhandle = dlopen(libname, RTLD_NOW | RTLD_LOCAL);
        if (mLibHandle.dlhandle == NULL) {
            QLOGE(LOG_TAG, "Failed to dlopen %s: %s", libname, dlerror());
            return ret;
        }
        
        // 2. 获取函数符号
        dlerror();
        
        *(void **)(&mLibHandle.Init) = dlsym(mLibHandle.dlhandle, "perfmodule_init");
        if ((rc = dlerror()) != NULL) {
            QLOGE(LOG_TAG, "Failed to get perfmodule_init");
            dlclose(mLibHandle.dlhandle);
            return ret;
        }
        
        *(void **)(&mLibHandle.Exit) = dlsym(mLibHandle.dlhandle, "perfmodule_exit");
        if ((rc = dlerror()) != NULL) {
            QLOGE(LOG_TAG, "Failed to get perfmodule_exit");
            dlclose(mLibHandle.dlhandle);
            return ret;
        }
        
        *(void **)(&mLibHandle.SubmitRequest) = dlsym(mLibHandle.dlhandle, 
                                                     "perfmodule_submit_request");
        if ((rc = dlerror()) != NULL) {
            QLOGE(LOG_TAG, "Failed to get perfmodule_submit_request");
            dlclose(mLibHandle.dlhandle);
            return ret;
        }
        
        *(void **)(&mLibHandle.SyncRequest) = dlsym(mLibHandle.dlhandle, 
                                                   "perfmodule_sync_request_ext");
        if ((rc = dlerror()) != NULL) {
            QLOGE(LOG_TAG, "Failed to get perfmodule_sync_request_ext");
        }
        
        // 3. 标记加载成功
        mLibHandle.is_opened = true;
        strlcpy(mLibHandle.mLibFileName, libname, MAX_FILE_NAME);
        ret = 0;
    }
    
    return ret;
}
```

## 3.5 HAL初始化流程

### 服务初始化

```cpp
void Perf::Init() {
    char trace_prop[PROPERTY_VALUE_MAX];
    
    QLOGV(LOG_TAG, "PERF2: inside Perf::Init()");
    
    // 1. 初始化日志系统
    if (PerfLogInit() < 0) {
        QLOGE(LOG_TAG, "PerfLogInit failed");
    }
    
    // 2. 初始化目标配置
    mTargetConfig.InitializeTargetConfig();
    QLOGL(LOG_TAG, QLOG_L1, "TargetConfig Init Complete");
    
    // 3. 初始化配置存储
    mPerfDataStore.ConfigStoreInit();
    
    // 4. 加载核心性能模块
    mImpl.LoadPerfLib("libqti-perfd.so");
    
    // 5. 根据配置加载学习模块
    string prop = "";
    perfGetProp("vendor.debug.enable.lm", "false", &prop);
    bool enableLM = (!prop.compare("false")) ? false : true;
    if (enableLM) {
        QLOGL(LOG_TAG, QLOG_L1, "LM enabled: Loading liblearningmodule.so");
        mImpl.LoadPerfLib("liblearningmodule.so");
    }
    
    // 6. 根据内存配置加载内存性能模块
    perfGetProp("vendor.debug.enable.memperfd", "false", &prop);
    bool enableMemperfd = (!prop.compare("false")) ? false : true;
    if (enableLM && enableMemperfd) {
        perfGetProp("vendor.enable.memperfd_MIN_RAM_in_KB", "1048576", &prop);
        uint32_t minRAMKB = stoi(prop.c_str(), NULL);
        perfGetProp("vendor.enable.memperfd_MAX_RAM_in_KB", "20971520", &prop);
        uint32_t maxRAMKB = stoi(prop.c_str(), NULL);
        uint32_t memTotal = mTargetConfig.getRamInKB();
        
        if (memTotal > minRAMKB && memTotal <= maxRAMKB) {
            mImpl.LoadPerfLib("libmemperfd.so");
        }
    }
    
    // 7. 初始化调试跟踪
    if (property_get(PROP_NAME, trace_prop, NULL) > 0) {
        if (trace_prop[0] == '1') {
            perf_debug_output = PERF_SYSTRACE = atoi(trace_prop);
        }
    }
    
    // 8. 初始化所有已加载的模块
    mImpl.PerfGlueLayerInit();
}
```

### 目标配置初始化

```cpp
void TargetConfig::InitializeTargetConfig() {
    int16_t socid = 0;
    char prop_val[PROPERTY_VALUE_MAX];
    
    QLOGV(LOG_TAG, "Inside InitializeTargetConfig");
    
    // 1. 读取硬件信息
    socid = readSocID();           // 读取SoC ID
    mRam = readRamSize();          // 读取内存大小
    readVariant();                 // 读取设备变体
    readResolution();              // 读取屏幕分辨率
    readKernelVersion();           // 读取内核版本
    
    // 2. 获取API级别
    if (property_get(API_LEVEL_PROP_NAME, prop_val, "31")) {
        mFirstAPILevel = atoi(prop_val);
    }
    
    // 3. 解析XML配置
    InitTargetConfigXML();
    
    // 4. 应用配置
    TargetConfigInit();
    
    QLOGL(LOG_TAG, QLOG_L1, "Init complete for: %s", getTargetName().c_str());
    
    // 5. 输出配置信息
    DumpAll();
}
```

# 4. 参数解析与处理机制

## 4.1 Boost参数格式

### 标准格式

```cpp
// boost参数格式: [resource_id, value, resource_id, value, ...]
// 例如: CPU频率锁定
int32_t boost_params[] = {
    0x40800000, 1500000,  // CPU0最小频率设为1.5GHz
    0x40800100, 1200000,  // CPU1最小频率设为1.2GHz  
    0x40804000, 2000000,  // CPU0最大频率设为2.0GHz
    0x40804100, 1800000   // CPU1最大频率设为1.8GHz
};
```

### 参数验证

```cpp
// Perf.cpp中的参数检查
uint32_t size = in_boostsList.size();
if (size > MAX_ARGS_PER_REQUEST) {
    QLOGE(LOG_TAG, "Maximum number of arguments allowed exceeded");
    return ndk::ScopedAStatus::ok();
}
```

## 4.2 Hint参数处理

### Hint ID映射

```cpp
// mp-ctl.h中定义的Hint类型
enum {
    VENDOR_HINT_SCROLL_BOOST = 0x00001080,        // 滚动优化
    VENDOR_HINT_FIRST_LAUNCH_BOOST = 0x00001081,  // 首次启动优化
    VENDOR_HINT_ANIM_BOOST = 0x00001083,          // 动画优化
    VENDOR_HINT_TOUCH_BOOST = 0x00001085,         // 触摸优化
    VENDOR_HINT_PERFORMANCE_MODE = 0x00001091,    // 性能模式
};
```

### Hint处理逻辑

```cpp
// PerfGlueLayer.cpp中的Hint分发
uint32_t hint = msg->hint_id;

// 检查模块是否注册了该Hint
bool PerfModule::IsThisEventRegistered(int32_t event) {
    bool ret = false;
    
    // 首先检查是否在范围内
    ret = (event >= mEventsLowerBound) && (event <= mEventsUpperBound);
    
    if (!ret) {
        // 检查具体事件列表
        for (uint8_t i = 0; i < mNumEvents; i++) {
            if (event == mEvents[i]) {
                ret = true;
                break;
            }
        }
    }
    
    return ret;
}
```

## 4.3 配置参数解析

### XML配置解析

```cpp
// TargetConfig.cpp中的配置解析
void TargetConfig::TargetConfigsCB(xmlNodePtr node, void *index) {
    if (!xmlStrcmp(node->name, BAD_CAST TARGET_CONFIGS_XML_ELEM_TARGETINFO_TAG)) {
        // 解析目标基本信息
        config->mNumCluster = ConvertNodeValueToInt(node, 
                              TARGET_CONFIGS_XML_ELEM_NUMCLUSTERS_TAG, 
                              config->mNumCluster);
        config->mTotalNumCores = ConvertNodeValueToInt(node,
                                 TARGET_CONFIGS_XML_ELEM_TOTALNUMCORES_TAG,
                                 config->mTotalNumCores);
        
        // 解析SoC ID列表
        if (xmlHasProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_SOCIDS_TAG)) {
            idPtr = (char *)xmlGetProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_SOCIDS_TAG);
            if (NULL != idPtr) {
                config->mNumSocids = ConvertToIntArray(idPtr, config->mSupportedSocids, 
                                                      MAX_SUPPORTED_SOCIDS);
                xmlFree(idPtr);
            }
        }
    }
    
    if (!xmlStrcmp(node->name, BAD_CAST TARGET_CONFIGS_XML_ELEM_CLUSTER_TAG)) {
        // 解析集群配置
        id = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_ID_TAG, id);
        core_per_cluster = ConvertNodeValueToInt(node, 
                          TARGET_CONFIGS_XML_ELEM_NUMCORES_TAG, 
                          core_per_cluster);
        frequency = ConvertNodeValueToInt(node,
                   TARGET_CONFIGS_XML_ELEM_MAXFREQUENCY_TAG,
                   frequency);
        
        // 解析集群类型
        if (xmlHasProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TYPE_TAG)) {
            idPtr = (char *)xmlGetProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TYPE_TAG);
            if (NULL != idPtr) {
                config->mClusterNameToIdMap[idPtr] = id;
                
                // 设置集群类型标志
                if (id == 0) {
                    if (strncmp("little", idPtr, strlen(idPtr)) == 0) {
                        config->mType = 1;  // little core在cluster 0
                    } else {
                        config->mType = 0;  // big core在cluster 0
                    }
                }
                xmlFree(idPtr);
            }
        }
        
        // 保存集群配置
        config->mCorepercluster[id] = core_per_cluster;
        config->mCpumaxfrequency[id] = frequency;
    }
}
```



# 5. Socket服务实现

## 5.1 Socket服务架构

```cpp
// Perf_sock.cpp - Socket服务实现
class Perf {
private:
    int32_t mSocketHandler;    // Socket句柄
    int32_t mListnerHandler;   // 监听句柄
    bool mSignalFlag;          // 信号标志
    pthread_t mPerfThread;     // 服务线程
    struct sockaddr_un mAddr;  // Unix域套接字地址

public:
    // Socket连接建立
    int32_t connectToSocket() {
        char mode[] = "0666";
        int perm = 0;
        
        // 1. 创建Unix域套接字
        mListnerHandler = socket(AF_UNIX, SOCK_STREAM, 0);
        if (mListnerHandler < 0) {
            QLOGE(LOG_TAG_HAL, "Unable to open socket: %s", SOCKET_NAME);
            return FAILED;
        }
        
        // 2. 绑定套接字地址
        mAddr.sun_family = AF_LOCAL;
        snprintf(mAddr.sun_path, UNIX_PATH_MAX, SOCKET_NAME);
        unlink(mAddr.sun_path);  // 清理旧的套接字文件
        
        int32_t rc = bind(mListnerHandler, (struct sockaddr*)&mAddr, 
                         sizeof(struct sockaddr_un));
        if (rc < 0) {
            QLOGE(LOG_TAG_HAL, "bind() failed %s", SOCKET_NAME);
            return FAILED;
        }
        
        // 3. 设置权限
        perm = strtol(mode, 0, 8);
        if (chmod(SOCKET_NAME, perm) < 0) {
            QLOGE(LOG_TAG_HAL, "Failed to set file perm %s", mode);
        }
        
        // 4. 开始监听
        if (listen(mListnerHandler, MAX_CONNECTIONS) < 0) {
            QLOGE(LOG_TAG_HAL, "Unable to listen on handle %d for socket %s", 
                  mListnerHandler, SOCKET_NAME);
            return FAILED;
        }
        
        return SUCCESS;
    }
};
```

## 5.2 客户端连接处理

```cpp
// service_sock.cpp - 主服务循环
void Perf::startListener() {
    mpctl_msg_t msg;
    
    while (!mSignalFlag) {
        memset(&msg, 0, sizeof(mpctl_msg_t));
        struct sockaddr addr;
        socklen_t alen = sizeof(addr);
        
        // 1. 等待客户端连接
        QLOGV(LOG_TAG_HAL, "waiting to accept");
        int32_t handler = accept(mListnerHandler, (struct sockaddr *)&addr, &alen);
        QLOGL(LOG_TAG_HAL, QLOG_L2, "accepted");
        
        if (handler > 0) {
            // 2. 设置连接句柄
            setSocketHandler(handler);
            
            // 3. 接收消息
            if (recvMsg(msg) == SUCCESS) {
                // 4. 处理API调用
                callAPI(msg);
            }
        } else {
            QLOGE(LOG_TAG_HAL, "Failed to accept incoming connection: %s", 
                  strerror(errno));
        }
    }
    
    QLOGL(LOG_TAG_HAL, QLOG_L2, "Stopping Listener Handler");
    close(mListnerHandler);
    mListnerHandler = -1;
}

// 消息接收
int32_t Perf::recvMsg(mpctl_msg_t &msg) {
    QLOGV(LOG_TAG_HAL, "waiting to recv msg %s", __func__);
    int32_t msgSize = sizeof(mpctl_msg_t);
    int32_t bytes_read = recv(mSocketHandler, &msg, msgSize, 0);
    
    QLOGL(LOG_TAG_HAL, QLOG_L2, "received msg %s", __func__);
    if (bytes_read != msgSize) {
        QLOGE(LOG_TAG_HAL, "Invalid msg received from client");
        return FAILED;
    }
    return SUCCESS;
}

// 消息发送
int32_t Perf::sendMsg(void *msg, int32_t type) {
    int32_t msgSize = 0;
    int32_t rc = FAILED;
    
    switch (type) {
        case PERF_LOCK_TYPE: {
            msgSize = sizeof(client_msg_int_t);
            break;
        }
        case PERF_GET_PROP_TYPE: {
            client_msg_str_t *tmp = (client_msg_str_t*)msg;
            if (tmp == NULL) {
                return rc;
            }
            QLOGL(LOG_TAG_HAL, QLOG_L2, "value %s", tmp->value);
            msgSize = sizeof(client_msg_str_t);
            break;
        }
        case PERF_SYNC_REQ_TYPE: {
            msgSize = sizeof(client_msg_sync_req_t);
            break;
        }
        default: {
            QLOGE(LOG_TAG_HAL, "Invalid Msg type Received %d", type);
            return rc;
        }
    }
    
    rc = send(mSocketHandler, msg, msgSize, 0);
    QLOGV(LOG_TAG_HAL, "sent msg");
    return rc;
}
```

## 5.3 API调用分发处理

```cpp
void Perf::callAPI(mpctl_msg_t &msg) {
    QLOGV(LOG_TAG_HAL, "API type: %u", msg.req_type);
    printMsg(msg);  // 调试输出消息内容
    
    switch (msg.req_type) {
        case PERF_LOCK_CMD: {
            perfLockCmd(msg);
            break;
        }
        case PERF_LOCK_RELEASE: {
            perfLockRelease(msg);
            break;
        }
        case PERF_HINT: {
            perfHint(msg);
            break;
        }
        case PERF_GET_PROP: {
            perfGetProp(msg);
            break;
        }
        case PERF_LOCK_ACQUIRE: {
            perfLockAcquire(msg);
            break;
        }
        case PERF_LOCK_ACQUIRE_RELEASE: {
            perfLockAcqAndRelease(msg);
            break;
        }
        case PERF_GET_FEEDBACK: {
            perfGetFeedback(msg);
            break;
        }
        case PERF_EVENT: {
            perfEvent(msg);
            break;
        }
        case PERF_HINT_ACQUIRE_RELEASE: {
            perfHintAcqRel(msg);
            break;
        }
        case PERF_HINT_RENEW: {
            perfHintRenew(msg);
            break;
        }
        case PERF_SYNC_REQ: {
            perfSyncRequest(msg);
            break;
        }
        default:
            QLOGE(LOG_TAG_HAL, "Invalid API Called %d", msg.req_type);
    }
    
    // 处理完成后关闭连接
    closeSocket(getSocketHandler());
}
```

## 5.4 Socket版本API实现

```cpp
// 性能锁获取
int32_t Perf::perfLockAcquire(mpctl_msg_t &pMsg) {
    int32_t retVal = -1;
    client_msg_int_t reply;
    memset(&reply, 0, sizeof(client_msg_int_t));
    reply.handle = retVal;
    
    if (!checkPerfStatus(__func__)) {
        sendMsg(&reply);
        return retVal;
    }
    
    pMsg.req_type = MPCTL_CMD_PERFLOCKACQ;
    retVal = mImpl.PerfGlueLayerSubmitRequest(&pMsg);
    reply.handle = retVal;
    sendMsg(&reply);
    
    return retVal;
}

// 性能提示
int32_t Perf::perfHint(mpctl_msg_t &pMsg) {
    int32_t retVal = -1;
    client_msg_int_t reply;
    memset(&reply, 0, sizeof(client_msg_int_t));
    reply.handle = retVal;
    
    if (!checkPerfStatus(__func__)) {
        sendMsg(&reply);
        return retVal;
    }
    
    pMsg.req_type = MPCTL_CMD_PERFLOCKHINTACQ;
    retVal = mImpl.PerfGlueLayerSubmitRequest(&pMsg);
    QLOGL(LOG_TAG_HAL, QLOG_L2, "Perf glueLayer submitRequest returned %d", retVal);
    reply.handle = retVal;
    sendMsg(&reply);
    return retVal;
}

// 属性获取
void Perf::perfGetProp(mpctl_msg_t &pMsg) {
    client_msg_str_t reply;
    char *retVal = NULL;
    char trace_buf[TRACE_BUF_SZ];
    memset(&reply, 0, sizeof(client_msg_str_t));
    
    if (!checkPerfStatus(__func__)) {
        retVal = NULL;
    } else {
        retVal = mPerfDataStore.GetProperty(pMsg.usrdata_str, reply.value, 
                                          sizeof(reply.value));
        
        if (retVal != NULL) {
            if (perf_debug_output) {
                snprintf(trace_buf, TRACE_BUF_SZ, 
                        "perfGetProp: Return Val from %s is %s", __func__, retVal);
                QLOGE(LOG_TAG_HAL, "%s", trace_buf);
            }
        }
    }
    
    if (retVal == NULL) {
        strlcpy(reply.value, pMsg.propDefVal, PROP_VAL_LENGTH);
        reply.value[PROP_VAL_LENGTH - 1] = '\0';
    }
    
    sendMsg(&reply, PERF_GET_PROP_TYPE);
}

// 同步请求
void Perf::perfSyncRequest(mpctl_msg_t &pMsg) {
    client_msg_sync_req_t reply;
    memset(&reply, 0, sizeof(client_msg_sync_req_t));
    
    if (!checkPerfStatus(__func__)) {
        sendMsg(&reply, PERF_SYNC_REQ_TYPE);
        return;
    }
    
    std::string requestInfo = mImpl.PerfGlueLayerSyncRequest(pMsg.hint_id);
    strlcpy(reply.value, requestInfo.c_str(), SYNC_REQ_VAL_LEN);
    sendMsg(&reply, PERF_SYNC_REQ_TYPE);
}
```



# 6. 配置文件解析

## 6.1 targetconfig.xml解析

### 6.1.1 TargetConfig解析回调函数

```cpp
void TargetConfig::TargetConfigsCB(xmlNodePtr node, void *index) {
    char *idPtr = NULL;
    char *tmp_target = NULL;
    int8_t id = 0, core_per_cluster = 0, target_index = 0;
    long int frequency = INT_MAX;
    long int capped_max_freq = INT_MAX;
    bool cluster_type_present = false;

    TargetConfig &tc = TargetConfig::getTargetConfig();
    
    // 获取配置索引 (Config1, Config2等)
    target_index = *((int8_t*)index) - 1;
    if((target_index < 0) || (target_index >= MAX_CONFIGS_SUPPORTED_PER_PLATFORM)) {
        QLOGE(LOG_TAG, "Invalid config index value while parsing the XML file.");
        return;
    }

    // 创建或获取配置对象
    if ((int8_t)tc.mTargetConfigs.size() <= target_index) {
        auto tmp = new(std::nothrow) TargetConfigInfo;
        if (tmp != NULL)
            tc.mTargetConfigs.push_back(tmp);
    }
    TargetConfigInfo *config = tc.mTargetConfigs[target_index];

    // 解析TargetInfo标签
    if (!xmlStrcmp(node->name, BAD_CAST TARGET_CONFIGS_XML_ELEM_TARGETINFO_TAG)) {
        // 解析Target属性
        if (xmlHasProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TARGET_TAG)) {
            tmp_target = (char *)xmlGetProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TARGET_TAG);
            if(tmp_target != NULL) {
                config->mTargetName = string(tmp_target);
                xmlFree(tmp_target);
            }
        }

        // 解析数值属性 - 使用辅助函数确保类型安全
        config->mNumCluster = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_NUMCLUSTERS_TAG, config->mNumCluster);
        config->mTotalNumCores = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_TOTALNUMCORES_TAG, config->mTotalNumCores);
        config->mCoreCtlCpu = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_CORECTLCPU_TAG, config->mCoreCtlCpu);
        
        // 解析SocIds数组
        if (xmlHasProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_SOCIDS_TAG)) {
            idPtr = (char *)xmlGetProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_SOCIDS_TAG);
            if (NULL != idPtr) {
                config->mNumSocids = ConvertToIntArray(idPtr, config->mSupportedSocids, MAX_SUPPORTED_SOCIDS);
                xmlFree(idPtr);
            }
        }

        // 配置验证逻辑
        if ((config->mNumCluster <= 0) || (config->mNumCluster > MAX_CLUSTER)) {
            config->mUpdateTargetInfo = false;
            QLOGE(LOG_TAG, "Number of clusters are not mentioned correctly in targetconfig xml file");
        }
    }

    // 解析ClustersInfo标签
    if (!xmlStrcmp(node->name, BAD_CAST TARGET_CONFIGS_XML_ELEM_CLUSTER_TAG)) {
        id = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_ID_TAG, id);
        core_per_cluster = ConvertNodeValueToInt(node, TARGET_CONFIGS_XML_ELEM_NUMCORES_TAG, core_per_cluster);

        // 解析集群类型 (little/big/prime)
        if (xmlHasProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TYPE_TAG)) {
            idPtr = (char *)xmlGetProp(node, BAD_CAST TARGET_CONFIGS_XML_ELEM_TYPE_TAG);
            if (NULL != idPtr) {
                cluster_type_present = true;
                config->mClusterNameToIdMap[idPtr] = id;
                
                // 根据集群类型设置mType
                if (id == 0) {
                    if (strncmp("little", idPtr, strlen(idPtr)) == 0) {
                        config->mType = 1;  // little cluster在索引0
                    } else {
                        config->mType = 0;  // big cluster在索引0
                    }
                }
                xmlFree(idPtr);
            }
        }

        // 存储集群配置
        config->mCorepercluster[id] = core_per_cluster;
        config->mCpumaxfrequency[id] = frequency;
        config->mCalculatedCores += core_per_cluster;
    }
}
```

### 6.1.2 辅助函数 - 类型安全转换

```cpp
long int TargetConfig::ConvertNodeValueToInt(xmlNodePtr node, const char *tag, long int defaultvalue) {
    char *idPtr = NULL;
    
    if (xmlHasProp(node, BAD_CAST tag)) {
        idPtr = (char *)xmlGetProp(node, BAD_CAST tag);
        if (NULL != idPtr) {
            defaultvalue = strtol(idPtr, NULL, 0);  // 支持十进制和十六进制
            xmlFree(idPtr);
        }
    }
    return defaultvalue;
}

uint32_t TargetConfig::ConvertToIntArray(char *str, int16_t intArray[], uint32_t size) {
    uint32_t i = 0;
    char *pch = NULL;
    char *end = NULL;
    char *endPtr = NULL;

    if ((NULL == str) || (NULL == intArray)) {
        return i;
    }

    end = str + strlen(str);
    
    // 解析逗号分隔的SocId列表 "636,640,641,657,658"
    do {
        pch = strchr(str, ',');
        intArray[i] = strtol(str, &endPtr, 0);
        i++;
        str = pch;
        if (NULL != pch) {
            str++;
        }
    } while ((NULL != str) && (str < end) && (i < size));

    return i;
}
```

## 6.2 perfconfigstore.xml解析核心代码

### 6.2.1 PerfConfig解析回调函数

```cpp
void PerfConfigDataStore::PerfConfigStoreCB(xmlNodePtr node, void *) {
    char *mName = NULL, *mValue = NULL, *mTarget = NULL, *mKernel = NULL,
         *mResolution = NULL, *mRam = NULL, *mVariant = NULL, *mSkewType = NULL;
    
    PerfConfigDataStore &store = PerfConfigDataStore::getPerfDataStore();
    TargetConfig &tc = TargetConfig::getTargetConfig();
    
    // 获取当前系统信息用于匹配
    const char *target_name = tc.getTargetName().c_str();
    const char *kernelVersion = tc.getFullKernelVersion().c_str();
    const char *target_name_variant = tc.getVariant().c_str();
    uint32_t tc_resolution = tc.getResolution();
    uint32_t tc_ram = tc.getRamSize();
    
    // 匹配标志
    bool valid_target = true, valid_kernel = true, valid_ram = true,
         valid_resolution = true, valid_target_variant = true;

    if(!xmlStrcmp(node->name, BAD_CAST PERF_CONFIG_STORE_PROP)) {
        // 解析基本属性
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_NAME)) {
            mName = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_NAME);
        }
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_VALUE)) {
            mValue = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_VALUE);
        }

        // 目标匹配逻辑 - 支持多目标 "Target1,Target2"
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_TARGET)) {
            mTarget = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_TARGET);
            if (mTarget != NULL) {
                valid_target = false;
                char *pos = NULL;
                char *tname_token = strtok_r(mTarget, ",", &pos);
                while(tname_token != NULL) {
                    if((strlen(tname_token) == strlen(target_name)) && 
                       (!strncmp(target_name, tname_token, strlen(target_name)))) {
                        valid_target = true;
                        break;
                    }
                    tname_token = strtok_r(NULL, ",", &pos);
                }
            }
        }

        // 变体匹配 (32bit/64bit/go版本)
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_TARGET_VARIANT)) {
            mVariant = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_TARGET_VARIANT);
            if (mVariant != NULL) {
                if(((strlen(mVariant) == strlen(target_name_variant)) && 
                   (!strncmp(target_name_variant, mVariant, strlen(mVariant))))) {
                    valid_target_variant = true;
                } else {
                    valid_target_variant = false;
                }
            }
        }

        // 内核版本匹配
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_KERNEL)) {
            mKernel = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_KERNEL);
            if (mKernel != NULL) {
                if((strlen(mKernel) == strlen(kernelVersion)) && 
                   !strncmp(kernelVersion, mKernel, strlen(mKernel))) {
                    valid_kernel = true;
                } else {
                    valid_kernel = false;
                }
            }
        }

        // 分辨率匹配
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_RESOLUTION)) {
            mResolution = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_RESOLUTION);
            if (mResolution != NULL) {
                uint32_t res = store.ConvertToEnumResolutionType(mResolution);
                if (res == tc_resolution) {
                    valid_resolution = true;
                } else {
                    valid_resolution = false;
                }
            }
        }

        // RAM大小匹配
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_RAM)) {
            mRam = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_RAM);
            if (mRam != NULL) {
                uint32_t ram = atoi(mRam);
                if (ram == tc_ram) {
                    valid_ram = true;
                } else {
                    valid_ram = false;
                }
            }
        }

        // SkewType处理 - 硬件功能检查
        if(xmlHasProp(node, BAD_CAST PERF_CONFIG_STORE_SKEW_TYPE)) {
            mSkewType = (char *) xmlGetProp(node, BAD_CAST PERF_CONFIG_STORE_SKEW_TYPE);
            if (mSkewType != NULL) {
                bool skewRetVal = false;
                int32_t skewType = atoi(mSkewType);
                if(mFeature_knob_func) {
                    skewRetVal = mFeature_knob_func(skewType);  // 调用libskewknob.so
                }

                if (!skewRetVal) {
                    // 功能不支持，强制设置为false
                    if (mValue) {
                        xmlFree(mValue);
                        mValue = (char *) xmlMalloc(sizeof(FALSE_STR) * sizeof(char));
                        if (mValue) {
                            memset(mValue, 0, sizeof(FALSE_STR));
                            strlcpy(mValue, FALSE_STR, sizeof(FALSE_STR));
                        }
                    }
                }
            }
        }

        // 存储匹配的配置
        if (mName != NULL && mValue != NULL) {
            if ((valid_kernel && valid_target && valid_target_variant && 
                 valid_resolution && valid_ram)) {
                UpdatePerfConfig(mName, mValue);
            } else if (!mTarget && !mVariant && !mKernel && !mResolution && !mRam) {
                // 通用配置，无条件匹配
                try {
                    store.mPerfConfigStore.insert_or_assign(mName, mValue);
                } catch (std::exception &e) {
                    QLOGE(LOG_TAG, "Exception caught: %s in %s", e.what(), __func__);
                }
            }
        }

        // 清理内存
        if(mName) xmlFree(mName);
        if(mValue) xmlFree(mValue);
        if(mTarget) xmlFree(mTarget);
        // ... 其他指针清理
    }
}
```

### 6.2.2 分辨率类型转换

```cpp
ValueMapResType PerfConfigDataStore::ConvertToEnumResolutionType(char *res) {
    ValueMapResType ret = MAP_RES_TYPE_ANY;

    if (NULL == res) {
        return ret;
    }

    switch(res[0]) {
    case '1':
        if (!strncmp(res, MAP_RES_TYPE_VAL_1080p, strlen(MAP_RES_TYPE_VAL_1080p))) {
            ret = MAP_RES_TYPE_1080P;
        }
        break;
    case '7':
        if (!strncmp(res, MAP_RES_TYPE_VAL_720p, strlen(MAP_RES_TYPE_VAL_720p))) {
            ret = MAP_RES_TYPE_720P;
        }
        break;
    case 'H':  // HD+ Resolution (720x1440)
        if (!strncmp(res, MAP_RES_TYPE_VAL_HD_PLUS, strlen(MAP_RES_TYPE_VAL_HD_PLUS))) {
            ret = MAP_RES_TYPE_HD_PLUS;
        }
        break;
    case '2':
        if (!strncmp(res, MAP_RES_TYPE_VAL_2560, strlen(MAP_RES_TYPE_VAL_2560))) {
            ret = MAP_RES_TYPE_2560;
        }
    }
    return ret;
}
```



## 6.3 XML示例解析结果

### 6.3.1 targetconfig.xml解析结果:

- **Target**: "volcano"
- **NumClusters**: 3 (little/big/prime)
- **TotalNumCores**: 8
- **SocIds**: [636,640,641,657,658]
- **集群配置**:
  - Cluster 0: 4个little核心
  - Cluster 1: 3个big核心
  - Cluster 2: 1个prime核心

### 6.3.2 perfconfigstore.xml解析结果:

- **通用配置**: vendor.debug.enable.lm = "true"
- **目标特定**: ro.vendor.perf.ss = "true" (仅volcano目标)
- **条件配置**: ro.vendor.qti.sys.fw.bservice_age = "5000" (RAM≤4GB时)



## 6.4 解析时序图

```mermaid
sequenceDiagram
    participant MAIN as main()
    participant PERF as Perf构造函数
    participant TC as TargetConfig
    participant DC as DivergentConfig
    participant PC as PerfConfigDataStore
    participant XML as XmlParser
    participant SYS as 系统文件/属性
Note over MAIN,SYS: PerfHAL服务启动阶段

MAIN->>PERF: 创建Perf实例
PERF->>PERF: 检查vendor.disable.perf.hal属性

alt 性能HAL已启用
    PERF->>PERF: Init()
    
    Note over PERF,SYS: TargetConfig初始化
    PERF->>TC: InitializeTargetConfig()
    
    TC->>SYS: readSocID()
    SYS-->>TC: 返回SoC ID
    
    TC->>SYS: readRamSize()
    SYS-->>TC: 读取/proc/meminfo
    SYS-->>TC: 返回RAM大小分类
    
    TC->>SYS: readVariant()
    SYS-->>TC: 读取ro.product.name属性
    
    TC->>SYS: readResolution()
    SYS-->>TC: 读取显示分辨率节点
    
    TC->>SYS: readKernelVersion()
    SYS-->>TC: 读取/proc/sys/kernel/osrelease
    
    TC->>SYS: 读取ro.board.first_api_level
    SYS-->>TC: 返回API Level
    
    Note over TC,XML: XML配置解析
    TC->>TC: InitTargetConfigXML()
    
    loop 遍历最多5个配置
        TC->>XML: 注册Config1-5解析器
        XML->>SYS: 解析targetconfig.xml
        
        Note over XML: 解析TargetInfo标签
        XML->>XML: 解析Target、SocIds、NumClusters等
        
        Note over XML: 解析ClustersInfo标签
        loop 每个集群配置
            XML->>XML: 解析Id、NumCores、Type、MaxFrequency等
        end
        
        XML->>TC: TargetConfigsCB回调
        TC->>TC: 验证配置有效性
        TC->>TC: 存储到mTargetConfigs向量
    end
    
    TC->>TC: TargetConfigInit()
    TC->>TC: getTargetConfigInfo(socId)
    TC->>TC: 根据SoC ID选择匹配配置
    
    Note over TC,DC: DivergentConfig初始化
    TC->>DC: DivergentConfig构造函数
    
    DC->>SYS: readNumClusters()
    SYS-->>DC: 扫描/sys/devices/system/cpu/cpufreq/目录
    SYS-->>DC: 返回policy目录数量
    
    DC->>SYS: readPerClusterInfo()
    loop 每个集群
        SYS-->>DC: 读取related_cpus文件
        DC->>DC: 解析核心数和范围
    end
    
    DC->>SYS: readCapacityPerCluster()
    loop 每个集群
        SYS-->>DC: 读取cpu_capacity文件
        DC->>DC: 存储容量信息
    end
    
    DC->>SYS: readPhysicalCluster()
    loop 每个集群
        SYS-->>DC: 读取cluster_id文件
        DC->>DC: 映射物理集群ID
    end
    
    DC->>DC: determineGovernorInstType()
    DC->>DC: GenerateDivergentNumber()
    
    DC->>SYS: isSubSystemEnabled("display")
    SYS-->>DC: 检查显示子系统状态
    
    DC->>SYS: isSubSystemEnabled("gpu")
    SYS-->>DC: 检查GPU子系统状态
    
    DC->>DC: DumpAll() - 输出调试信息
    
    Note over TC: 完成TargetConfig配置
    TC->>TC: 使用DivergentConfig数据更新配置
    TC->>TC: CheckDefaultDivergent()
    TC->>TC: 设置集群信息、核心数等
    TC->>TC: updateClusterNameToIdMap()
    TC->>TC: setInitCompleted(true)
    
    Note over PERF,PC: PerfConfigDataStore初始化
    PERF->>PC: ConfigStoreInit()
    
    PC->>PC: 加载libskewknob.so(可选)
    
    PC->>XML: 注册perfconfigstore.xml解析器
    XML->>SYS: 解析perfconfigstore.xml
    
    loop 每个Prop标签
        XML->>XML: 解析Name、Value、Target等属性
        XML->>TC: 获取当前目标信息进行匹配
        
        alt 目标匹配成功
            XML->>PC: PerfConfigStoreCB回调
            PC->>PC: 存储配置到mPerfConfigStore
        else 目标不匹配
            XML->>XML: 跳过该配置项
        end
    end
    
    PC->>PC: 设置mPerfConfigInit = true
    
    Note over PERF: 配置初始化完成
    PERF->>PERF: 加载性能模块库
    PERF->>PERF: 启用系统跟踪(可选)
    
else 性能HAL已禁用
    PERF->>PERF: 跳过初始化
end

Note over MAIN,SYS: 配置管理层初始化完成
```



# 7. 完整调用时序图

```mermaid
sequenceDiagram
    participant App as Application
    participant BF as BoostFramework
    participant PS as PerfService
    participant JNI as JNI Layer
    participant Client as mp-ctl-client
    participant HAL as PerfHAL
    participant GL as GlueLayer
    participant Module as PerfModule

    Note over App,Module: 应用启动性能优化场景

    App->>BF: perfHint(LAUNCH_BOOST, "com.app", 2000, 1)
    BF->>PS: perfHint() via Binder
    PS->>JNI: native_perf_hint()
    JNI->>Client: perf_hint()
    
    Note over Client: 检查HAL类型 (AIDL/HIDL)
    Client->>Client: isHalTypeAidl()
    Client->>Client: getPerfAidlAndLinkToDeath()
    
    Note over Client: 准备参数
    Client->>Client: 构造paramList [tid, pid, offloadFlag]
    
    Client->>HAL: perfHint() via AIDL
    
    Note over HAL: 参数解析和验证
    HAL->>HAL: 解析reserved参数 (tid, pid, offloadFlag)
    HAL->>HAL: 构造mpctl_msg_t
    
    HAL->>GL: PerfGlueLayerSubmitRequest(msg)
    
    Note over GL: 请求类型判断
    GL->>GL: cmd == MPCTL_CMD_PERFLOCKHINTACQ
    
    Note over GL: 异步处理其他模块
    GL->>GL: PerfThreadPool.placeTask()
    
    loop 遍历所有模块
        GL->>Module: IsThisEventRegistered(hint)
        alt 模块支持该Hint
            GL->>Module: SubmitRequest(msg)
            Module-->>GL: 处理结果
        end
    end
    
    Note over GL: 同步处理mpctl模块
    GL->>Module: mMpctlMod.SubmitRequest(msg)
    Module-->>GL: 返回handle
    
    GL-->>HAL: 返回handle
    HAL-->>Client: 返回handle
    Client-->>JNI: 返回handle
    JNI-->>PS: 返回handle
    PS-->>BF: 返回handle
    BF-->>App: 返回handle
```



# 8. PerfHAL加密云控方案

## 8.1 整体方案设计

### 8.1.1 架构设计

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

### 8.1.2 核心实现策略

- **文件路径管理器**：统一管理配置文件路径选择
- **加密检测与解密**：自动识别加密文件并解密
- **透明集成**：对现有XML解析逻辑透明

## 8.2 详细实现方案

### 8.2.1 新增配置文件管理类

**插入位置**：PerfConfig.h 和 PerfConfig.cpp

```cpp
// PerfConfig.h 新增
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

### 8.2.2 修改现有宏定义

**修改位置**：PerfConfig.cpp 文件开头

```cpp
// 原来的固定路径定义 - 删除
// #define PERF_CONFIG_STORE_XML (VENDOR_DIR"/perf/perfconfigstore.xml")

// 新的动态路径获取
#define PERF_CONFIG_STORE_XML ConfigFileManager::getConfigFilePath("perfconfigstore.xml").c_str()
```

**修改位置**：TargetConfig.cpp 文件开头

```cpp
// 原来的固定路径定义 - 删除  
// #define TARGET_CONFIGS_XML (VENDOR_DIR"/perf/targetconfig.xml")

// 新的动态路径获取
#define TARGET_CONFIGS_XML ConfigFileManager::getConfigFilePath("targetconfig.xml").c_str()
```

### 8.2.3 核心实现代码

**插入位置**：PerfConfig.cpp

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
    const char* key = "QualcommPerfHAL2024";
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

### 8.2.4 修改XML解析器集成

**修改位置**：XmlParser.h 和 XmlParser.cpp

需要为 AppsListXmlParser 类添加支持内存XML解析的方法：

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
    
    xmlNodePtr rootElement = xmlDocGetRootElement(doc);
    if (rootElement == NULL) {
        QLOGE(LOG_TAG, "Failed to get root element");
        xmlFreeDoc(doc);
        return false;
    }
    
    // 复用现有的解析逻辑
    if (mCb != NULL) {
        ParseNode(rootElement->children);
    }
    
    xmlFreeDoc(doc);
    return true;
}
```

### 8.2.5 修改PerfConfigDataStore解析流程

**修改位置**：PerfConfig.cpp 中的 ConfigStoreInit() 方法

```cpp
void PerfConfigDataStore::ConfigStoreInit() {
    int8_t idnum;
    AppsListXmlParser *xmlParser = new(std::nothrow) AppsListXmlParser();
    if (NULL == xmlParser) {
        return;
    }
    
    // 解析配置文件 - 修改后
    std::string configContent;
    std::string configPath = ConfigFileManager::getConfigFilePath("perfconfigstore.xml");
    if (ConfigFileManager::readAndDecryptConfig(configPath, configContent)) {
        // 加载obfuscation库
        void *handle = dlopen(OBFUSCATION_LIB_NAME, RTLD_LAZY);
        if (NULL != handle) {
            mFeature_knob_func = (feature_knob_func_ptr)dlsym(handle, "IsFeatureEnabled");
        }
        
        const string xmlPerfConfigRoot(PERF_CONFIG_STORE_ROOT);
        const string xmlPerfConfigChild(PERF_CONFIG_STORE_CHILD);

        idnum = xmlParser->Register(xmlPerfConfigRoot, xmlPerfConfigChild, PerfConfigStoreCB, NULL);
        xmlParser->ParseFromMemory(configContent);  // 使用内存解析
        xmlParser->DeRegister(idnum);

        if (NULL != handle) {
            mFeature_knob_func = NULL;
            dlclose(handle);
        }
    } else {
        QLOGE(LOG_TAG, "Failed to read config file: %s", configPath.c_str());
    }

    delete xmlParser;
    mPerfConfigInit = true;
    return;
}
```

### 8.2.6 修改TargetConfig解析流程

**修改位置**：TargetConfig.cpp 中的 InitTargetConfigXML() 方法

```cpp
void TargetConfig::InitTargetConfigXML() {
    AppsListXmlParser *xmlParser = new(std::nothrow) AppsListXmlParser();
    if (NULL == xmlParser) {
        return;
    }
    
    // 读取并解密配置文件
    std::string configContent;
    std::string configPath = ConfigFileManager::getConfigFilePath("targetconfig.xml");
    if (!ConfigFileManager::readAndDecryptConfig(configPath, configContent)) {
        QLOGE(LOG_TAG, "Failed to read target config file: %s", configPath.c_str());
        delete xmlParser;
        return;
    }
    
    QLOGV(LOG_TAG, "InitTargetConfigXML start with file: %s", configPath.c_str());

    const string xmlTargetConfigRoot(TARGET_CONFIGS_XML_ROOT);
    uint8_t idnum, i;
    char TargetConfigXMLtag[NODE_MAX] = "";
    string xmlChildTargetConfig;

    // 解析多个配置
    for(i = 1; i <= MAX_CONFIGS_SUPPORTED_PER_PLATFORM; i++) {
        snprintf(TargetConfigXMLtag, NODE_MAX, TARGET_CONFIGS_XML_CHILD_CONFIG, i);
        xmlChildTargetConfig = TargetConfigXMLtag;
        idnum = xmlParser->Register(xmlTargetConfigRoot, xmlChildTargetConfig, TargetConfigsCB, &i);
        xmlParser->ParseFromMemory(configContent);  // 使用内存解析
        xmlParser->DeRegister(idnum);
    }
    
    delete xmlParser;
    QLOGV(LOG_TAG, "InitTargetConfigXML end");
    return;
}
```

## 8.3 实现时序图

```mermaid
sequenceDiagram
    participant Store as PerfDataStore
    participant MGR as ConfigFileManager
    participant FS as 文件系统
    participant DECRYPT as 解密模块
    participant Parser as XmlParser
    
    Note over Store: 配置加载开始
    Store->>MGR: getConfigFilePath("perfconfigstore.xml")
    MGR->>FS: access("/data/perf/perfconfigstore.xml")
    FS-->>MGR: 存在/不存在
    
    alt /data/perf/存在
        MGR-->>Store: "/data/perf/perfconfigstore.xml"
    else vendor路径
        MGR-->>Store: "/vendor/etc/perf/perfconfigstore.xml"
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



