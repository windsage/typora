# 基于FPSGO我能做什么？

## 场景识别与策略配置

### 启用场景提示（Scenario Hint） 

* 通过 PowerHAL 的 `MTKPOWER_HINT_GAME_MODE` 识别游戏场景，确保 FPSGO 在游戏运行时接管帧率控制。 

* 配置文件路径： 

```
     vendor/mediatek/proprietary/hardware/power/config/mtxxxx/scn_tbl/powerscntbl.xml  
```

* 示例配置： 

```
     <scn name="GAME_MODE" hint="MTKPOWER_HINT_GAME_MODE" />  
```



#### 区分场景优先级 

* 游戏场景优先级高于普通 UX 或后台任务，需通过 PowerHAL 仲裁策略确保游戏线程获得足够的算力资源。 

* 避免非关键线程（如后台下载）因系统频率天花板（Per-cluster CPU freq ceiling）被压制，可通过 `SeparateCap` 分组优化。



## 核心参数调优

### 预期帧率（Expected FPS）

* 目标：设定合理的目标帧率（如 60FPS），避免过高帧率导致功耗激增。 

* 参数配置： 

```
adb shell "echo 60 > /sys/kernel/fpsgo/fstb/notify_fstb_target_fps_by_pid"  
```

* 动态调整：结合温控模块动态降低帧率，防止过热降频。



### 闭环控制（Closed Loop）

* 作用：通过实时校正确保实际帧率接近目标值。 

* 关键参数： 

  * `gcc_enable`：启用闭环（值设为 `2`）。 

  * `expect_fps_margin`：校正偏移量（如 `-100` 表示目标帧率微调为 `59.9`）。 

  ```
  adb shell "echo 2 > /sys/module/mtk_fpsgo/parameters/gcc_enable"  
  adb shell "echo -100 > /sys/module/mtk_fpsgo/parameters/expect_fps_margin"  
  ```



### 救援策略（Rescue）

* 功能：在突发负载导致掉帧时动态提升算力。 

* 调优参数： 

  * `rescue_enable`：启用救援（值设为 `1`）。 

  * `rescue_perframe_f`：增强幅度（默认 `25`，可调至 `30-40`）。 

  * `qr_enable`：启用配额救援（优于传统 VSYNC 救援）。 

  ```
  adb shell "echo 1 > /sys/kernel/fpsgo/fbt/rescue_enable"  
  adb shell "echo 35 > /sys/module/mtk_fpsgo/parameters/rescue_perframe_f"  
  ```



### 关键线程分组与绑定（Grouping & Boost Affinity）

* 分组策略： 

  * 按负载排名（`group_by_lr=0`）或渲染线程（R）与最重非渲染线程（L）分组（`group_by_lr=1`）。 

  * 为不同组分配差异化算力（如 Heavy 组不打折，Others 组限制为 `separate_pct_other=80`）。 

* 核心绑定： 

  * 通过 `boost_affinity` 将关键线程绑定到大核（如 CPU7），减少调度延迟。 

  ```
  adb shell "echo 1 > /sys/kernel/fpsgo/fbt/group_by_lr"  
  adb shell "echo 80 > /sys/kernel/fpsgo/fbt/separate_pct_other"  
  adb shell "echo 1 > /sys/module/mtk_fpsgo/parameters/boost_affinity"  
  ```



### 进程级调优（By PID）

#### 按进程独立配置 

* 在多窗口场景下，为游戏进程单独设置参数，避免影响其他应用。 

* 示例：仅针对游戏 PID `1234` 启用救援： 

```
adb shell "echo 1234 1 > /sys/kernel/fpsgo/fbt/rescue_enable_by_pid"  
```



#### 优先级与 RT 调度 

* 为关键线程设置实时优先级（RT），减少抢占延迟： 

```
adb shell "echo 12 > /sys/kernel/fpsgo/fbt/RT_prio1"  # Heavy 组优先级  
adb shell "echo 1 > /sys/kernel/fpsgo/fbt/boost_VIP"   # 启用 VIP Boost  
```



## 调试与监控

### 实时帧率查看 

```
adb shell "cat /sys/kernel/fpsgo/fstb/fpsgo_status"  
```

* 输出字段：`currentFPS`（实际帧率）、`targetFPS`（目标帧率）、`dfps_celling`（屏幕刷新率）。



### Systrace 分析 

* 启用 FPSGO 的 Systrace 事件 `fpsgo_main_systrace`，追踪帧率波动与算力分配。



### 功耗与性能平衡验证 

* 使用联发科工具（如 Battery Historian）分析调优后的功耗曲线，确保无异常耗电。





## 注意事项

1. 避免使用遗留参数：如 `gcc_fps_margin` 和 `limit_cfreq` 已废弃，改用 `expect_fps_margin` 和 `SeparateCap`。 

2. 场景专家咨询：在非标准场景（如云游戏、AR/VR）中启用 FPSGO 前，需联发科技术支持。 

3) 平衡策略：激进调优可能导致功耗上升，需通过实际测试确定最佳参数组合。



通过以上策略，下游厂商可基于 FPSGO 灵活调整游戏的性能与功耗平衡，适配不同硬件平台与用户场景，最终提升游戏流畅度与用户体验。
