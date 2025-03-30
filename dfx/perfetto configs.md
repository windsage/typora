# Perfetto Config指南

## config字段具体指南

[TraceConfig - Perfetto Tracing Docs](https://perfetto.dev/docs/reference/trace-config-proto)

```
// 设置缓冲区大小，缓冲区可以设置多个，索引从0开始，配合target_buffer 讲data_sources写入指定buffer
 buffers: {
   // buffer大小
   size_kb: 522240
   // buffer的填充策略，默认RING_BUFFER，可选discard，discard表示buffer满了后面就不要了，用于抓取初期现场。
   fill_policy: RING_BUFFER
 }
 buffers: {
   size_kb: 2048
   fill_policy: RING_BUFFER
 }
 data_sources: {
   config {
     // 列出所有安装的应用，包含应用版本信息
     name: "android.packages_list"
     target_buffer: 1
   }
 }
 data_sources: {
   config {
     name: "linux.sys_stats"
     sys_stats_config {
       // 以下三个cpu counters信息
       stat_period_ms: 1000
       stat_counters: STAT_CPU_TIMES
       stat_counters: STAT_FORK_COUNT
       // cpu频率和idle状态获取时间间隔
       cpufreq_period_ms: 1000
       // kernel内存信息，/proc/meminfo，可能不一定支持
       meminfo_period_ms: 2500
       meminfo_counters: MEMINFO_ACTIVE
       meminfo_counters: MEMINFO_ACTIVE_ANON
       meminfo_counters: MEMINFO_ACTIVE_FILE
       meminfo_counters: MEMINFO_ANON_PAGES
       meminfo_counters: MEMINFO_BUFFERS
       meminfo_counters: MEMINFO_CACHED
       meminfo_counters: MEMINFO_CMA_FREE
       meminfo_counters: MEMINFO_CMA_TOTAL
       meminfo_counters: MEMINFO_COMMIT_LIMIT
       meminfo_counters: MEMINFO_COMMITED_AS
       meminfo_counters: MEMINFO_DIRTY
       meminfo_counters: MEMINFO_GPU
       meminfo_counters: MEMINFO_INACTIVE
       meminfo_counters: MEMINFO_INACTIVE_ANON
       meminfo_counters: MEMINFO_INACTIVE_FILE
       meminfo_counters: MEMINFO_ION_HEAP
       meminfo_counters: MEMINFO_ION_HEAP_POOL
       meminfo_counters: MEMINFO_KERNEL_STACK
       meminfo_counters: MEMINFO_MAPPED
       meminfo_counters: MEMINFO_MEM_AVAILABLE
       meminfo_counters: MEMINFO_MEM_FREE
       meminfo_counters: MEMINFO_MEM_TOTAL
       meminfo_counters: MEMINFO_MISC
       meminfo_counters: MEMINFO_MLOCKED
       meminfo_counters: MEMINFO_PAGE_TABLES
       meminfo_counters: MEMINFO_SHMEM
       meminfo_counters: MEMINFO_SLAB
       meminfo_counters: MEMINFO_SLAB_RECLAIMABLE
       meminfo_counters: MEMINFO_SLAB_UNRECLAIMABLE
       meminfo_counters: MEMINFO_SWAP_CACHED
       meminfo_counters: MEMINFO_SWAP_FREE
       meminfo_counters: MEMINFO_SWAP_TOTAL
       meminfo_counters: MEMINFO_UNEVICTABLE
       meminfo_counters: MEMINFO_VMALLOC_CHUNK
       meminfo_counters: MEMINFO_VMALLOC_TOTAL
       meminfo_counters: MEMINFO_VMALLOC_USED
       meminfo_counters: MEMINFO_WRITEBACK
       meminfo_counters: MEMINFO_ZRAM
       vmstat_period_ms: 1000
       // 虚拟内存信息，很多时候不支持
       vmstat_period_ms: 1300
       vmstat_counters: VMSTAT_ALLOCSTALL
       vmstat_counters: VMSTAT_ALLOCSTALL_DEVICE
       vmstat_counters: VMSTAT_ALLOCSTALL_DMA
       vmstat_counters: VMSTAT_ALLOCSTALL_DMA32
       vmstat_counters: VMSTAT_ALLOCSTALL_MOVABLE
       vmstat_counters: VMSTAT_ALLOCSTALL_NORMAL
       vmstat_counters: VMSTAT_BALLOON_DEFLATE
       vmstat_counters: VMSTAT_BALLOON_INFLATE
       vmstat_counters: VMSTAT_BALLOON_MIGRATE
       vmstat_counters: VMSTAT_CMA_ALLOC_FAIL
       vmstat_counters: VMSTAT_CMA_ALLOC_SUCCESS
       vmstat_counters: VMSTAT_COMPACT_DAEMON_FREE_SCANNED
       vmstat_counters: VMSTAT_COMPACT_DAEMON_MIGRATE_SCANNED
       vmstat_counters: VMSTAT_COMPACT_DAEMON_WAKE
       vmstat_counters: VMSTAT_COMPACT_FAIL
       vmstat_counters: VMSTAT_COMPACT_FREE_SCANNED
       vmstat_counters: VMSTAT_COMPACT_ISOLATED
       vmstat_counters: VMSTAT_COMPACT_MIGRATE_SCANNED
       vmstat_counters: VMSTAT_COMPACT_STALL
       vmstat_counters: VMSTAT_COMPACT_SUCCESS
       vmstat_counters: VMSTAT_DROP_PAGECACHE
       vmstat_counters: VMSTAT_DROP_SLAB
       vmstat_counters: VMSTAT_KSWAPD_HIGH_WMARK_HIT_QUICKLY
       vmstat_counters: VMSTAT_KSWAPD_INODESTEAL
       vmstat_counters: VMSTAT_KSWAPD_LOW_WMARK_HIT_QUICKLY
       vmstat_counters: VMSTAT_NR_ACTIVE_ANON
       vmstat_counters: VMSTAT_NR_ACTIVE_FILE
       vmstat_counters: VMSTAT_NR_ALLOC_BATCH
       vmstat_counters: VMSTAT_NR_ANON_PAGES
       vmstat_counters: VMSTAT_NR_ANON_TRANSPARENT_HUGEPAGES
       vmstat_counters: VMSTAT_NR_BOUNCE
       vmstat_counters: VMSTAT_NR_DIRTIED
       vmstat_counters: VMSTAT_NR_DIRTY
       vmstat_counters: VMSTAT_NR_DIRTY_BACKGROUND_THRESHOLD
       vmstat_counters: VMSTAT_NR_DIRTY_THRESHOLD
       vmstat_counters: VMSTAT_NR_FASTRPC
       vmstat_counters: VMSTAT_NR_FILE_HUGEPAGES
       vmstat_counters: VMSTAT_NR_FILE_PAGES
       vmstat_counters: VMSTAT_NR_FILE_PMDMAPPED
       vmstat_counters: VMSTAT_NR_FOLL_PIN_ACQUIRED
       vmstat_counters: VMSTAT_NR_FOLL_PIN_RELEASED
       vmstat_counters: VMSTAT_NR_FREE_CMA
       vmstat_counters: VMSTAT_NR_FREE_PAGES
       vmstat_counters: VMSTAT_NR_GPU_HEAP
       vmstat_counters: VMSTAT_NR_INACTIVE_ANON
       vmstat_counters: VMSTAT_NR_INACTIVE_FILE
       vmstat_counters: VMSTAT_NR_INDIRECTLY_RECLAIMABLE
       vmstat_counters: VMSTAT_NR_ION_HEAP
       vmstat_counters: VMSTAT_NR_ION_HEAP_POOL
       vmstat_counters: VMSTAT_NR_ISOLATED_ANON
       vmstat_counters: VMSTAT_NR_ISOLATED_FILE
       vmstat_counters: VMSTAT_NR_KERNEL_MISC_RECLAIMABLE
       vmstat_counters: VMSTAT_NR_KERNEL_STACK
       vmstat_counters: VMSTAT_NR_MAPPED
       vmstat_counters: VMSTAT_NR_MLOCK
       vmstat_counters: VMSTAT_NR_OVERHEAD
       vmstat_counters: VMSTAT_NR_PAGE_TABLE_PAGES
       vmstat_counters: VMSTAT_NR_PAGES_SCANNED
       vmstat_counters: VMSTAT_NR_SEC_PAGE_TABLE_PAGES
       vmstat_counters: VMSTAT_NR_SHADOW_CALL_STACK
       vmstat_counters: VMSTAT_NR_SHADOW_CALL_STACK_BYTES
       vmstat_counters: VMSTAT_NR_SHMEM
       vmstat_counters: VMSTAT_NR_SHMEM_HUGEPAGES
       vmstat_counters: VMSTAT_NR_SHMEM_PMDMAPPED
       vmstat_counters: VMSTAT_NR_SLAB_RECLAIMABLE
       vmstat_counters: VMSTAT_NR_SLAB_UNRECLAIMABLE
       vmstat_counters: VMSTAT_NR_SWAPCACHE
       vmstat_counters: VMSTAT_NR_SWAPCACHED
       vmstat_counters: VMSTAT_NR_THROTTLED_WRITTEN
       vmstat_counters: VMSTAT_NR_UNEVICTABLE
       vmstat_counters: VMSTAT_NR_UNRECLAIMABLE_PAGES
       vmstat_counters: VMSTAT_NR_UNSTABLE
       vmstat_counters: VMSTAT_NR_VMSCAN_IMMEDIATE_RECLAIM
       vmstat_counters: VMSTAT_NR_VMSCAN_WRITE
       vmstat_counters: VMSTAT_NR_WRITEBACK
       vmstat_counters: VMSTAT_NR_WRITEBACK_TEMP
       vmstat_counters: VMSTAT_NR_WRITTEN
       vmstat_counters: VMSTAT_NR_ZONE_ACTIVE_ANON
       vmstat_counters: VMSTAT_NR_ZONE_ACTIVE_FILE
       vmstat_counters: VMSTAT_NR_ZONE_INACTIVE_ANON
       vmstat_counters: VMSTAT_NR_ZONE_INACTIVE_FILE
       vmstat_counters: VMSTAT_NR_ZONE_UNEVICTABLE
       vmstat_counters: VMSTAT_NR_ZONE_WRITE_PENDING
       vmstat_counters: VMSTAT_NR_ZSPAGES
       vmstat_counters: VMSTAT_OOM_KILL
       vmstat_counters: VMSTAT_PAGEOUTRUN
       vmstat_counters: VMSTAT_PGACTIVATE
       vmstat_counters: VMSTAT_PGALLOC_DEVICE
       vmstat_counters: VMSTAT_PGALLOC_DMA
       vmstat_counters: VMSTAT_PGALLOC_DMA32
       vmstat_counters: VMSTAT_PGALLOC_MOVABLE
       vmstat_counters: VMSTAT_PGALLOC_NORMAL
       vmstat_counters: VMSTAT_PGDEACTIVATE
       vmstat_counters: VMSTAT_PGDEMOTE_DIRECT
       vmstat_counters: VMSTAT_PGDEMOTE_KSWAPD
       vmstat_counters: VMSTAT_PGFAULT
       vmstat_counters: VMSTAT_PGFREE
       vmstat_counters: VMSTAT_PGINODESTEAL
       vmstat_counters: VMSTAT_PGLAZYFREE
       vmstat_counters: VMSTAT_PGLAZYFREED
       vmstat_counters: VMSTAT_PGMAJFAULT
       vmstat_counters: VMSTAT_PGMIGRATE_FAIL
       vmstat_counters: VMSTAT_PGMIGRATE_SUCCESS
       vmstat_counters: VMSTAT_PGPGIN
       vmstat_counters: VMSTAT_PGPGOUT
       vmstat_counters: VMSTAT_PGPGOUTCLEAN
       vmstat_counters: VMSTAT_PGREFILL
       vmstat_counters: VMSTAT_PGREFILL_DMA
       vmstat_counters: VMSTAT_PGREFILL_MOVABLE
       vmstat_counters: VMSTAT_PGREFILL_NORMAL
       vmstat_counters: VMSTAT_PGREUSE
       vmstat_counters: VMSTAT_PGROTATED
       vmstat_counters: VMSTAT_PGSCAN_ANON
       vmstat_counters: VMSTAT_PGSCAN_DIRECT
       vmstat_counters: VMSTAT_PGSCAN_DIRECT_DMA
       vmstat_counters: VMSTAT_PGSCAN_DIRECT_MOVABLE
       vmstat_counters: VMSTAT_PGSCAN_DIRECT_NORMAL
       vmstat_counters: VMSTAT_PGSCAN_DIRECT_THROTTLE
       vmstat_counters: VMSTAT_PGSCAN_FILE
       vmstat_counters: VMSTAT_PGSCAN_KSWAPD
       vmstat_counters: VMSTAT_PGSCAN_KSWAPD_DMA
       vmstat_counters: VMSTAT_PGSCAN_KSWAPD_MOVABLE
       vmstat_counters: VMSTAT_PGSCAN_KSWAPD_NORMAL
       vmstat_counters: VMSTAT_PGSKIP_DEVICE
       vmstat_counters: VMSTAT_PGSKIP_DMA
       vmstat_counters: VMSTAT_PGSKIP_DMA32
       vmstat_counters: VMSTAT_PGSKIP_MOVABLE
       vmstat_counters: VMSTAT_PGSKIP_NORMAL
       vmstat_counters: VMSTAT_PGSTEAL_ANON
       vmstat_counters: VMSTAT_PGSTEAL_DIRECT
       vmstat_counters: VMSTAT_PGSTEAL_DIRECT_DMA
       vmstat_counters: VMSTAT_PGSTEAL_DIRECT_MOVABLE
       vmstat_counters: VMSTAT_PGSTEAL_DIRECT_NORMAL
       vmstat_counters: VMSTAT_PGSTEAL_FILE
       vmstat_counters: VMSTAT_PGSTEAL_KSWAPD
       vmstat_counters: VMSTAT_PGSTEAL_KSWAPD_DMA
       vmstat_counters: VMSTAT_PGSTEAL_KSWAPD_MOVABLE
       vmstat_counters: VMSTAT_PGSTEAL_KSWAPD_NORMAL
       vmstat_counters: VMSTAT_PSWPIN
       vmstat_counters: VMSTAT_PSWPOUT
       vmstat_counters: VMSTAT_SLABS_SCANNED
       vmstat_counters: VMSTAT_SWAP_RA
       vmstat_counters: VMSTAT_SWAP_RA_HIT
       vmstat_counters: VMSTAT_THP_COLLAPSE_ALLOC
       vmstat_counters: VMSTAT_THP_COLLAPSE_ALLOC_FAILED
       vmstat_counters: VMSTAT_THP_DEFERRED_SPLIT_PAGE
       vmstat_counters: VMSTAT_THP_FAULT_ALLOC
       vmstat_counters: VMSTAT_THP_FAULT_FALLBACK
       vmstat_counters: VMSTAT_THP_FAULT_FALLBACK_CHARGE
       vmstat_counters: VMSTAT_THP_FILE_ALLOC
       vmstat_counters: VMSTAT_THP_FILE_FALLBACK
       vmstat_counters: VMSTAT_THP_FILE_FALLBACK_CHARGE
       vmstat_counters: VMSTAT_THP_FILE_MAPPED
       vmstat_counters: VMSTAT_THP_MIGRATION_FAIL
       vmstat_counters: VMSTAT_THP_MIGRATION_SPLIT
       vmstat_counters: VMSTAT_THP_MIGRATION_SUCCESS
       vmstat_counters: VMSTAT_THP_SCAN_EXCEED_NONE_PTE
       vmstat_counters: VMSTAT_THP_SCAN_EXCEED_SHARE_PTE
       vmstat_counters: VMSTAT_THP_SCAN_EXCEED_SWAP_PTE
       vmstat_counters: VMSTAT_THP_SPLIT_PAGE
       vmstat_counters: VMSTAT_THP_SPLIT_PAGE_FAILED
       vmstat_counters: VMSTAT_THP_SPLIT_PMD
       vmstat_counters: VMSTAT_THP_SWPOUT
       vmstat_counters: VMSTAT_THP_SWPOUT_FALLBACK
       vmstat_counters: VMSTAT_THP_ZERO_PAGE_ALLOC
       vmstat_counters: VMSTAT_THP_ZERO_PAGE_ALLOC_FAILED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_CLEARED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_CULLED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_MLOCKED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_MUNLOCKED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_RESCUED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_SCANNED
       vmstat_counters: VMSTAT_UNEVICTABLE_PGS_STRANDED
       vmstat_counters: VMSTAT_VMA_LOCK_ABORT
       vmstat_counters: VMSTAT_VMA_LOCK_MISS
       vmstat_counters: VMSTAT_VMA_LOCK_RETRY
       vmstat_counters: VMSTAT_VMA_LOCK_SUCCESS
       vmstat_counters: VMSTAT_WORKINGSET_ACTIVATE
       vmstat_counters: VMSTAT_WORKINGSET_ACTIVATE_ANON
       vmstat_counters: VMSTAT_WORKINGSET_ACTIVATE_FILE
       vmstat_counters: VMSTAT_WORKINGSET_NODERECLAIM
       vmstat_counters: VMSTAT_WORKINGSET_NODES
       vmstat_counters: VMSTAT_WORKINGSET_REFAULT
       vmstat_counters: VMSTAT_WORKINGSET_REFAULT_ANON
       vmstat_counters: VMSTAT_WORKINGSET_REFAULT_FILE
       vmstat_counters: VMSTAT_WORKINGSET_RESTORE
       vmstat_counters: VMSTAT_WORKINGSET_RESTORE_ANON
     }
   }
 }
 // 充电计数器和电池电源管理 IC 的瞬时功率
 data_sources: {
   config {
     name: "android.power"
     android_power_config {
       battery_poll_ms: 1000
       battery_counters: BATTERY_COUNTER_CAPACITY_PERCENT
       battery_counters: BATTERY_COUNTER_CHARGE
       battery_counters: BATTERY_COUNTER_CURRENT
       collect_power_rails: true
     }
   }
 }

 data_sources: {
   config {
     // native 内存dump
     name: "android.heapprofd"
     // 指定写入某buffer
     target_buffer: 0
     heapprofd_config {
       sampling_interval_bytes: 4096
       // dump 指定进程的native内存
       process_cmdline: "system_server"
       continuous_dump_config {
         // 第一次dump时间间隔
         dump_phase_ms: 500
         // 连续dump时间间隔
         dump_interval_ms: 200
       }
       shmem_size_bytes: 8388608
       // 要减慢应用的性能
       block_client: true
       all_heaps: true
     }
   }
 }
 data_sources: {
   config {
     // java堆dump
     name: "android.java_hprof"
     // 指定写入某buffer
     target_buffer: 0
     java_hprof_config {
       // 要dump的进程命
       process_cmdline: "com.android.vending"
       continuous_dump_config {
         // 第一次dump时间间隔
         dump_phase_ms: 60000
         // 连续dump时间间隔
         dump_interval_ms: 10000
       }
     }
   }
 }
 data_sources: {
   config {
     // 追踪每个进程的 RSS, swap and other /proc/status and oom_score_adj.
     name: "linux.process_stats"
     target_buffer: 1
     process_stats_config {
       proc_stats_poll_ms: 1000
     }
   }
 }
 data_sources: {
   config {
     name: "linux.ftrace"
     ftrace_config {
       // 以下是ftrace中追踪调度相关信息
       ftrace_events: "sched/sched_switch"
       ftrace_events: "power/suspend_resume"
       ftrace_events: "sched/sched_wakeup"
       ftrace_events: "sched/sched_wakeup_new"
       ftrace_events: "sched/sched_waking"
       ftrace_events: "sched/sched_process_exit"
       ftrace_events: "sched/sched_process_free"
       ftrace_events: "task/task_newtask"
       ftrace_events: "task/task_rename"
       // ftrace中cpu频率和idle状态信息
       ftrace_events: "power/cpu_frequency"
       ftrace_events: "power/cpu_idle"
       ftrace_events: "power/suspend_resume"
       // 追踪syscalls，需要userdebug或者eng版本
       ftrace_events: "raw_syscalls/sys_enter"
       ftrace_events: "raw_syscalls/sys_exit"
       // GPU频率
       ftrace_events: "power/gpu_frequency"
       // GPU内存信息 需要Android 12及以上
       ftrace_events: "gpu_mem/gpu_mem_total"
       // 追踪每个应用GPU使用，需要Android 14及以上
       ftrace_events: "power/gpu_work_period"
       // 电路板传感器的电压和频率变化
       ftrace_events: "regulator/regulator_set_voltage"
       ftrace_events: "regulator/regulator_set_voltage_complete"
       ftrace_events: "power/clock_enable"
       ftrace_events: "power/clock_disable"
       ftrace_events: "power/clock_set_rate"
       ftrace_events: "power/suspend_resume"
       // 追踪mm_event, rss_stat and ion, 需要android 10及以上
       ftrace_events: "mm_event/mm_event_record"
       ftrace_events: "kmem/rss_stat"
       ftrace_events: "ion/ion_stat"
       ftrace_events: "dmabuf_heap/dma_heap_stat"
       ftrace_events: "kmem/ion_heap_grow"
       ftrace_events: "kmem/ion_heap_shrink"
       ftrace_events: "sched/sched_process_exit"
       ftrace_events: "sched/sched_process_free"
       ftrace_events: "task/task_newtask"
       ftrace_events: "task/task_rename"
       // thermal 信息
       ftrace_events: "thermal/thermal_temperature"
       ftrace_events: "thermal/cdev_update"
       buffer_size_kb: 2048
       drain_period_ms: 250
       // lmkd事件
       ftrace_events: "lowmemorykiller/lowmemory_kill"
       ftrace_events: "oom/oom_score_adj_update"
       ftrace_events: "ftrace/print"
       atrace_apps: "lmkd"
       // 应用中的ATRACE_BEGIN相关信息
       ftrace_events: "ftrace/print"
       atrace_categories: "am"
       atrace_categories: "adb"
       atrace_categories: "aidl"
       atrace_categories: "dalvik"
       atrace_categories: "audio"
       atrace_categories: "binder_lock"
       atrace_categories: "binder_driver"
       atrace_categories: "bionic"
       atrace_categories: "camera"
       atrace_categories: "database"
       atrace_categories: "gfx"
       atrace_categories: "hal"
       atrace_categories: "input"
       atrace_categories: "network"
       atrace_categories: "nnapi"
       atrace_categories: "pm"
       atrace_categories: "power"
       atrace_categories: "rs"
       atrace_categories: "res"
       atrace_categories: "rro"
       atrace_categories: "sm"
       atrace_categories: "ss"
       atrace_categories: "vibrator"
       atrace_categories: "video"
       atrace_categories: "view"
       atrace_categories: "webview"
       atrace_categories: "wm"
       atrace_apps: "*"
       // ftrace 事件，包含了所有
       ftrace_events: "binder/*"
       ftrace_events: "block/*"
       ftrace_events: "clk/*"
       ftrace_events: "ext4/*"
       ftrace_events: "f2fs/*"
       ftrace_events: "fastrpc/*"
       ftrace_events: "i2c/*"
       ftrace_events: "irq/*"
       ftrace_events: "kmem/*"
       ftrace_events: "memory_bus/*"
       ftrace_events: "mmc/*"
       ftrace_events: "oom/*"
       ftrace_events: "power/*"
       ftrace_events: "regulator/*"
       ftrace_events: "sched/*"
       ftrace_events: "sync/*"
       ftrace_events: "task/*"
       ftrace_events: "vmscan/*"
       // 查看block_reason
       ftrace_events: "sched/sched_blocked_reason"
       // 查看block reason的function，需要userdebug或者eng版本
       symbolize_ksyms: true
     }
   }
 }
 data_sources: {
   config {
     // logcat 信息，会导致trace很大
     name: "android.log"
     android_log_config {
       log_ids: LID_EVENTS
       log_ids: LID_CRASH
       log_ids: LID_KERNEL
       log_ids: LID_DEFAULT
       log_ids: LID_RADIO
       log_ids: LID_SECURITY
       log_ids: LID_STATS
       log_ids: LID_SYSTEM
     }
   }
 }
 // 记录surface_flinger的预期/实际帧时间，需要android 12及以上版本。
 data_sources: {
   config {
     name: "android.surfaceflinger.frametimeline"
   }
 }
 // 应用的网络数据包，需要android 14及以上版本
 data_sources: {
   config {
     name: "android.network_packets"
     network_packet_trace_config {
       poll_ms: 250
     }
   }
 }
 // perf采样
 data_sources: {
   config {
     name: "linux.perf"
     perf_event_config {
       // 采样频率
       timebase {
         frequency: 60
         timestamp_clock: PERF_CLOCK_BOOTTIME
       }
       // 需要指定报名
       callstack_sampling {
         scope {
           target_cmdline: "com.android.vending"
           target_cmdline: "com.android.phone"
         }
       }
     }
   }
 }
 // trace抓取的时间
 duration_ms: 43200000
 // 刷新磁盘时间间隔
 flush_period_ms: 30000
 // 配合datasources中handles_incremental_state_clear 使用，给定的时间段内定期清除其增量状态
 incremental_state_config {
   clear_period_ms: 5000
 }
 // 抓大trace专用
 write_into_file: true
 // 可选，写入周期，默认100ms
 file_write_period_ms: 5000
 // 可选， 一旦文件最多写入 X 个字节，定期写入就会停止，即使duration_ms没有达到
 max_file_size_bytes: 10000000000
```
## 抓取命令
```
// android 12之前版本 
 adb push config.pbtx /data/local/tmp/config.pbtx 
 adb shell 'cat /data/local/tmp/config.pbtx | perfetto -c - -o /data/local/tmp/trace.perfetto-trace --detach=perf_debug'
 adb shell perfetto --attach=perf_debug --stop 
 // android 12及以后版本 
 adb push config.pbtx /data/misc/perfetto-configs/config.pbtx 
 adb shell perfetto --txt -c /data/misc/perfetto-configs/config.pbtx -o /data/misc/perfetto-traces/trace.perfetto-trace --detach=perf_debug 
 adb shell perfetto --attach=perf_debug --stop
```
## detach/attach

说明：--detach --attach 是相对的一组，如果不加--detach会出现如下的界面



中断perfetto需要用ctrl+z的方式，这样可能会导致写入trace失败。

1. detach的作用就是将perfetto抓取trace的过程转入后台执行，不会阻塞在前端。
2. detach/attach跟随的perf_debug可以理解为session_name，完全自定义，且支持同时抓取多份trace。

参考Google文档 [Running perfetto in detached mode - Perfetto Tracing Docs](https://perfetto.dev/docs/concepts/detached-mode)