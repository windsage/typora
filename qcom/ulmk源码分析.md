我将详细分析Qualcomm定制版lmkd的主要修改点，按功能分类列出关键函数和实现逻辑。

## 功能分类和关键函数分析

### 1. **首选应用(Preferred Apps)功能**

#### 关键函数：

- `init_PreferredApps()` - 初始化首选应用功能
- `update_perf_props()` - 更新性能属性
- `create_handle_for_perf_iop()` - 创建性能库句柄
- `close_handle_for_perf_iop()` - 关闭性能库句柄

#### 插入位置：

- 在 `main()` 函数中调用 `update_perf_props()`
- 在 `mp_event_psi()` 中集成首选应用更新逻辑

#### 实现逻辑和时序图：

```mermaid
sequenceDiagram
    participant Main as 主进程
    participant PerfLib as 性能库
    participant LMKD as LMKD核心
    participant Apps as 应用进程

    Main->>PerfLib: create_handle_for_perf_iop()
    PerfLib-->>Main: 返回库句柄
    
    Main->>Main: init_PreferredApps()
    Main->>PerfLib: dlsym获取API函数
    PerfLib-->>Main: 返回函数指针
    
    loop 内存压力事件
        LMKD->>LMKD: mp_event_psi()
        LMKD->>LMKD: 检查pa_update_timeout_ms
        alt 超时需要更新
            LMKD->>PerfLib: perf_ux_engine_trigger() 或 perf_sync_request()
            PerfLib-->>LMKD: 返回首选应用列表
            LMKD->>LMKD: 更新preferred_apps字符串
        end
        
        LMKD->>LMKD: proc_get_heaviest()
        LMKD->>LMKD: 检查进程是否在首选列表中
        alt 是首选应用
            LMKD->>LMKD: 降低杀死优先级
        else 非首选应用
            LMKD->>Apps: 正常杀死流程
        end
    end
```

### 2. **MGLRU状态检测和优化**

#### 关键函数：

- `get_MGLRU_status()` - 获取MGLRU状态
- `fill_log_pgskip_stats()` - 填充页面跳过统计
- `calc_zone_watermarks()` - 计算zone水位线

#### 插入位置：

- 在 `update_perf_props()` 中调用
- 在 `mp_event_psi()` 中用于内存策略决策

#### 实现逻辑：

```mermaid
sequenceDiagram
    participant LMKD as LMKD
    participant SysFS as /sys文件系统
    participant Memory as 内存管理
    
    LMKD->>SysFS: 读取/sys/kernel/mm/lru_gen/enabled
    SysFS-->>LMKD: 返回MGLRU状态值
    
    LMKD->>LMKD: get_MGLRU_status()
    LMKD->>LMKD: 解析16进制状态值
    
    alt MGLRU启用 (status > 0)
        LMKD->>Memory: 调整pgskip策略
        LMKD->>LMKD: 设置Normal zone pgskip_delta = 0
        LMKD->>LMKD: 只考虑水位线突破的zone的文件缓存
    else MGLRU禁用
        LMKD->>LMKD: 使用传统LRU策略
    end
```

### 3. **增强的内存压力检测**

#### 关键函数：

- `should_consider_cache_free()` - 判断是否考虑缓存为可用内存
- `get_lowest_watermark()` - 获取最低水位线违反情况
- `check_cont_lmkd_events()` - 检查连续LMKD事件

#### 插入位置：

- 在 `mp_event_psi()` 主循环中
- 在 `mainloop()` 的事件处理中

#### 实现逻辑：

```mermaid
sequenceDiagram
    participant PSI as PSI监控
    participant LMKD as LMKD核心
    participant WaterMark as 水位线检查
    participant KillLogic as 杀进程逻辑
    
    PSI->>LMKD: 内存压力事件触发
    LMKD->>LMKD: check_cont_lmkd_events()
    
    alt 连续事件检测
        LMKD->>LMKD: 计算事件间隔时间
        alt 间隔 < 1.5 * psi_window_size
            LMKD->>LMKD: count_upgraded_event++
            alt count >= psi_cont_event_thresh
                LMKD->>LMKD: 升级到CRITICAL级别
            end
        else 间隔过长
            LMKD->>LMKD: count_upgraded_event = 0
        end
    end
    
    LMKD->>WaterMark: get_lowest_watermark()
    WaterMark->>WaterMark: should_consider_cache_free()
    
    alt 需要考虑缓存
        WaterMark->>WaterMark: 计算file_cache
        WaterMark->>WaterMark: nr_cached_pages = cache_percent * file_cache
    end
    
    WaterMark->>WaterMark: 检查min/low/high水位线
    WaterMark-->>LMKD: 返回违反的水位线级别
    
    LMKD->>KillLogic: 根据水位线级别决定杀进程策略
```

### 4. **动态水位线增强(Watermark Boost)**

#### 关键函数：

- `wbf_effective` 全局变量管理
- 在 `mp_event_psi()` 中的动态调整逻辑

#### 插入位置：

- 在内存压力事件处理的开始和结束

#### 实现逻辑：

```mermaid
sequenceDiagram
    participant Event as 压力事件
    participant LMKD as LMKD
    participant WBF as Watermark Boost
    participant Kill as 杀进程
    
    Event->>LMKD: 触发内存压力事件
    LMKD->>WBF: 检查事件类型和优先级
    
    alt 高优先级事件
        WBF->>WBF: wbf_effective = wmark_boost_factor
    end
    
    LMKD->>LMKD: 执行杀进程逻辑
    
    alt 杀进程成功
        alt 严重杀进程原因
            WBF->>WBF: wbf_effective = min(wbf_effective + wbf_step, wmark_boost_factor)
        else 普通杀进程
            WBF->>WBF: wbf_effective = max(wbf_effective - wbf_step, 1)
        end
    end
    
    LMKD->>LMKD: 轮询窗口结束时
    WBF->>WBF: wbf_effective = wmark_boost_factor (重置)
```

### 5. **Native进程扫描和杀死**

#### 关键函数：

- `proc_get_script()` - 扫描和杀死native进程

#### 插入位置：

- 在 `find_and_kill_process()` 末尾作为备选方案

#### 实现逻辑：

```mermaid
sequenceDiagram
    participant LMKD as LMKD
    participant ProcFS as /proc文件系统
    participant Native as Native进程

    LMKD->>LMKD: find_and_kill_process()
    LMKD->>LMKD: 常规应用杀死失败

    alt userdebug构建 && min_score_adj == 0
        LMKD->>ProcFS: opendir("/proc")

        loop 遍历所有进程
            ProcFS-->>LMKD: 进程目录项
            LMKD->>ProcFS: 读取 /proc/pid/oom_score_adj
            ProcFS-->>LMKD: OOM调整分数

            alt oom_adj >= 0 && 非内核线程
                LMKD->>ProcFS: 读取 /proc/pid/statm 获取内存使用
                ProcFS-->>LMKD: 内存统计

                alt 内存使用 > 0
                    LMKD->>Native: kill(pid, SIGKILL)
                    LMKD-->>LMKD: 返回释放的页面数
                    note right of LMKD: 满足条件，提前中止遍历
                else 内存使用 <= 0
                    note right of LMKD: 继续遍历下一个进程
                end
            end
        end

        LMKD->>LMKD: 设置遍历延迟 (PSI_PROC_TRAVERSE_DELAY_MS)
    end

```





### 6. **增强的配置和属性管理**

#### 关键函数：

- `update_perf_props()` - 更新性能属性
- `printLMKDConfigs()` - 打印配置信息

#### 插入位置：

- 在 `update_props()` 函数末尾调用
- 在 `main()` 初始化时调用

#### 实现逻辑：

```mermaid
sequenceDiagram
    participant Main as 主进程
    participant VendorLib as Vendor库
    participant Props as 属性系统
    participant Config as 配置管理
    
    Main->>VendorLib: dlopen(PERFD_LIB)
    VendorLib-->>Main: 库句柄
    
    Main->>VendorLib: dlsym("perf_get_prop")
    VendorLib-->>Main: 函数指针
    
    loop 每个配置项
        Main->>VendorLib: perf_get_prop(property_name, default_value)
        VendorLib->>Props: 读取vendor属性
        Props-->>VendorLib: 属性值
        VendorLib-->>Main: 返回属性值
        
        Main->>Config: 更新对应的全局变量
    end
    
    Main->>Config: printLMKDConfigs()
    Config->>Config: 输出所有配置到日志
    
    Main->>VendorLib: dlclose()
```

这些修改展现了Qualcomm版本在以下方面的显著增强：

1. **智能化程度更高** - 通过首选应用和动态调整策略
2. **硬件适配性更强** - 支持MGLRU等新特性
3. **配置灵活性更大** - 大量可调参数和vendor集成
4. **监控能力更强** - 更详细的统计和日志
5. **性能优化更精细** - 多层次的优化策略

每个功能模块都经过精心设计，确保在复杂的商用环境中能够提供更好的内存管理性能。