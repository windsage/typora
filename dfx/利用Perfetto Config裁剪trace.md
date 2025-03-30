# 利用Perfetto Config裁剪trace

利用perfetto config来减少trace大小，以达到减小trace、减少内存占用的目的。为了合理使用perfetto config选项、正确设置perfetto config option值，特写此文章。


## 1. 几个时间间隔

* `file_write_period_ms`：这是控制数据写入文件的时间间隔，单位是毫秒，较大的值会导致数据在内存中累积更长时间，减少了磁盘 I/O 操作的频率，但可能会导致在发生系统崩溃时丢失更多数据。

* `flush_period_ms`：这是控制数据从内存缓冲区刷新到磁盘的时间间隔，较小的值会导致更频繁的磁盘 I/O 操作，但可以减少数据在内存中的累积时间，提高数据的安全性。

* `incremental_state_config.clear_period_ms`：是控制增量状态清除的时间间隔，单位是毫秒。配置了这个参数后，Perfetto会在指定的时间间隔内清除增量状态。

* `drain_period_ms`:这是控制从内核缓冲区传输到用户空间缓冲区的时间间隔，单位是毫秒，较小的值会导致更频繁的数据传输，减少数据在内核缓冲区中的延迟，但可能会增加系统的负载。

* `buffer_size_kb`：这是控制内核环形缓冲区的大小，单位是千字节，较大的值可以存储更多的数据，减少数据溢出的可能性，但也会占用更多的内存资源。

* `duration_ms`:指定的是 Trace 数据收集的持续时间，单位为毫秒

## 2. buffer_size_kb
这里的buffer_size_kb值得是data_souce中的buffer_size_kb

```c++
 // If true, |buffer_size_kb| is interpreted as a lower bound, allowing the
  // implementation to choose a bigger buffer size.
  //
  // Most configs for perfetto v43+ should simply leave both fields unset.
  //
  // If you need a config compatible with a range of perfetto builds and you
  // used to set a non-default buffer_size_kb, consider setting both fields.
  // Example:
  //   buffer_size_kb: 4096
  //   buffer_size_lower_bound: true
  // On older builds, the per-cpu buffers will be exactly 4 MB.
  // On v43+, buffers will be at least 4 MB.
  // In both cases, neither is guaranteed if there are other concurrent
  // perfetto ftrace sessions, as the buffers cannot be resized without pausing
  // the recording in the kernel.
  // Introduced in: perfetto v43.
  optional bool buffer_size_lower_bound = 27;
```
buffer_size_kb的默认大小是4096
从8096修改成1024，在连续应用启动案例中并未发现异常。

###  平衡buffer_size_kb与drain_period_ms

#### 延长drain_period_ms的潜在风险

* **数据传输延迟增加**
  延长 `drain_period_ms` 会增加内核缓冲区数据传输到用户空间的时间间隔。在高负载场景下，如果内核缓冲区被快速填满，而 `drain_period` 较长，则可能导致数据溢出（数据覆盖旧数据）或丢失。

* **内存占用优化 vs 数据完整性**
  虽然延长 `drain_period` 可以减少传输操作的频率，降低系统开销，但如果内核缓冲区较小且数据生成速度较快，这种优化可能会牺牲 Trace 的完整性。

***

### 减小buffer_size_kb的潜在风险

* **内核缓冲区容量不足**
  减小 `buffer_size_kb` 会显著降低内核缓冲区的容量。在高负载场景下，如果 Trace 数据生成速度超过传输速度，较小的内核缓冲区容易被填满，导致新数据覆盖旧数据（Ring Buffer 的特性），从而丢失关键信息。

* **内存占用优化 vs 数据完整性**
  虽然减小 `buffer_size_kb` 可以降低内存占用，但在高负载场景下，这可能导致大量数据丢失，Trace 信息不完整。

***

####  高负载场景下的优化建议

为了在高负载场景下既减少内存占用又保证 Trace 的完整性，可以采取以下优化策略：

##### （1）动态调整buffer_size_kb和drain_period_ms

* **按需调整缓冲区大小**
  根据实际工作负载动态调整 `buffer_size_kb`。例如，在低负载时使用较小的缓冲区，在高负载时临时增大缓冲区大小。

* **动态调整传输周期**
  在高负载时缩短 `drain_period_ms`（例如从 2 秒缩短到 1 秒），以加快数据传输速度，避免内核缓冲区溢出。

##### （2）优化ftrace_events和atrace_categories

* **精简跟踪事件**
  如果某些 `ftrace_events` 或 `atrace_categories` 对您的分析不重要，可以移除它们以减少数据生成量。

* **优先跟踪关键事件**
  确保只跟踪对您的分析至关重要的事件（例如 `sched/sched_switch` 和 `power/cpu_frequency`），避免记录无关的数据。

##### （3）增加磁盘刷新频率

* **缩短 `flush_period_ms`**
  在高负载场景下，缩短 `flush_period_ms`（例如从 10 秒缩短到 5 秒），将内存中的数据更快地刷新到磁盘。虽然这会增加磁盘 I/O 负载，但可以减少内存占用并避免数据丢失。

* **结合 `file_write_period_ms` 使用**
  如果 `file_write_period_ms` 设置为较短的时间（例如 1 小时），可以确保数据定期写入文件，减少内存中的累积数据量。

##### （4）使用循环缓冲区策略

* **利用 Ring Buffer 的特性**
  如果必须使用较小的内核缓冲区，可以接受一定程度的数据覆盖（旧数据被新数据覆盖）。在这种情况下，应确保 Trace 数据的关键部分（例如最近的事件）被优先记录。

* **权衡取舍**
  在内存受限的情况下，可以选择牺牲早期数据的完整性，保留最近的数据（例如系统崩溃前的事件），以便更好地进行故障排查。

##### （5）监控和调整系统负载

* **实时监控内存和 CPU 使用情况**
  在运行 Trace 收集任务时，实时监控系统的内存和 CPU 使用情况。如果发现内存占用过高或 CPU 负载过重，可以动态调整配置参数（例如临时增大 `buffer_size_kb` 或缩短 `drain_period_ms`）。

* **避免过度优化**
  如果内存资源非常紧张，可以考虑使用外部存储（例如 SSD）来存储 Trace 数据，而不是完全依赖内存。


### drain_period_ms默认值分析

external/perfetto/src/traced/probes/ftrace/ftrace\_controller.cc

```c++
constexpr uint32_t kDefaultTickPeriodMs = 100;

uint32_t FtraceController::GetTickPeriodMs() {
  if (data_sources_.empty())
    return kDefaultTickPeriodMs;
  uint32_t kUnsetPeriod = std::numeric_limits<uint32_t>::max();
  uint32_t min_period_ms = kUnsetPeriod;
  bool using_poll = true;
  ForEachInstance([&](FtraceInstanceState* instance) {
    using_poll &= instance->buffer_watches_posted;
    for (FtraceDataSource* ds : instance->started_data_sources) {
      if (ds->config().has_drain_period_ms()) {
        min_period_ms = std::min(min_period_ms, ds->config().drain_period_ms());
      }
    }
  });

  // None of the active data sources requested an explicit tick period.
  // The historical default is 100ms, but if we know that all instances are also
  // using buffer watermark polling, we can raise it. We don't disable the tick
  // entirely as it spreads the read work more evenly, and ensures procfs
  // scrapes of seen TIDs are not too stale.
  if (min_period_ms == kUnsetPeriod) {
    return using_poll ? kPollBackingTickPeriodMs : kDefaultTickPeriodMs;
  }

  if (min_period_ms < kMinTickPeriodMs || min_period_ms > kMaxTickPeriodMs) {
    PERFETTO_LOG(
        "drain_period_ms was %u should be between %u and %u. "
        "Falling back onto a default.",
        min_period_ms, kMinTickPeriodMs, kMaxTickPeriodMs);
    return kDefaultTickPeriodMs;
  }
  return min_period_ms;
}
```
| 条件 | min_period_ms的值 | 返回值 |
| ---- | -------------------| -----------|
|data_sources_ 为空（无数据源）|	无意义|	直接返回 kDefaultTickPeriodMs（默认值）|
|所有数据源 均未设置 drain_period_ms|kUnsetPeriod（uint32_t 最大值）|根据 using_poll 返回 kPollBackingTickPeriodMs 或 kDefaultTickPeriodMs|
|至少一个数据源 设置了 drain_period_ms|所有设置值中的 最小值|若该值在 [kMinTickPeriodMs, kMaxTickPeriodMs] 范围内，返回该值；否则返回 kDefaultTickPeriodMs|



> 默认值是100，说明是正常的，不影响性能




## 3. ftrace/print

不知道从哪儿抄的，需要验证

```shell
ftrace_config {
    ftrace_events: "print"
    filter: "msg ~ 'drm|gpu'"  # 仅保留含drm或gpu关键词的print
}
```



## 4. PrintFilter

The filter consists of multiple rules. As soon as a rule matches (the rules are processed in order), its `allow` field will be used as the outcome: if `allow` is true, the event will be included in the trace, otherwise it will be discarded. If an event does not match any rule, it will be allowed by default (a rule with an empty prefix and allow=false, disallows everything by default).

```c++
ftrace_config {
    ……
    print_filter {
      rules {
        atrace_msg {
          prefix : "add"
        }
        allow : false
      }
      rules {
        prefix : "uas_boost"
        allow : false
      }
    }
    buffer_size_kb: 1024
    drain_period_ms: 100
    disable_generic_events: false
}
```



```c++
 // Optional filter for "ftrace/print" events.
  //
  // The filter consists of multiple rules. As soon as a rule matches (the rules
  // are processed in order), its `allow` field will be used as the outcome: if
  // `allow` is true, the event will be included in the trace, otherwise it will
  // be discarded. If an event does not match any rule, it will be allowed by
  // default (a rule with an empty prefix and allow=false, disallows everything
  // by default).
  message PrintFilter {
    message Rule {
      // Matches an atrace message of the form:
      // <type>|pid|<prefix>...
      message AtraceMessage {
        optional string type = 1;
        optional string prefix = 2;
      }
      oneof match {
        // This rule matches if `prefix` matches exactly with the beginning of
        // the "ftrace/print" "buf" field.
        string prefix = 1;
        // This rule matches if the "buf" field contains an atrace-style print
        // message as specified in `atrace_msg`.
        AtraceMessage atrace_msg = 3;
      }
      optional bool allow = 2;
    }
    repeated Rule rules = 1;
  }
  optional PrintFilter print_filter = 22;
```

### 配置项用法

#### print_filter.rules的匹配机制

Perfetto 的 `print_filter` 允许对 **ftrace** 的 `"print"` 事件进行过滤。`print_filter` 由一组有序的规则（rules）组成，[每条规则按顺序进行匹配评估](https://android.googlesource.com/platform/external/perfetto/+/master/protos/perfetto/config/ftrace/ftrace_config.proto#:~:text=%2F%2F%20The%20filter%20consists%20of,message%20Rule)。当某个事件的消息内容（即 ftrace/print 事件的 `buf` 字段）匹配到列表中的第一条规则时，就会立即根据该规则的 `allow` [设置决定此事件是否被保留或丢弃](https://android.googlesource.com/platform/external/perfetto/+/master/protos/perfetto/config/ftrace/ftrace_config.proto#:~:text=%2F%2F%20The%20filter%20consists%20of,atrace%20message%20of%20the%20form)。具体来说：

* **按序匹配，优先第一命中**：规则从上到下依次检查，一旦有规则匹配，后续规则将不再考虑

* 。匹配到的规则的 `allow` 字段决定结果：`allow=true` 则允许该事件进入trace，`allow=false` 则过滤掉该事件。

* **默认允许未匹配事件**：如果没有任何规则匹配某个事件，那么该事件将**默认被记录**（即不被过滤掉）​。换言之，`print_filter` 默认是**非排他性的**，只有匹配到规则且该规则明确禁止时事件才会被丢弃。

需要注意的是，可以通过增加一条\*\*“兜底”规则**来改变默认行为。例如，添加一条前缀为空字符串且`allow=false`的规则可视作匹配所有内容，从而在没有更早的允许规则匹配时**禁止所有事件\*\*。这种空前缀规则放在规则列表末尾时，会导致默认行为变为丢弃未匹配的事件（将过滤列表转变为**白名单模式**），只有显式匹配了允许规则的事件才会记录。相反，如果不添加这样的兜底规则，`print_filter` 则表现为**黑名单模式**，仅过滤匹配了禁止规则的事件，其它都通过。



#### `prefix` 字段对匹配的影响

规则中的 `prefix` 用于直接匹配 **ftrace/print 消息**的起始文本。若规则定义了 `prefix`（且未使用 `atrace_msg`），则当且仅当事件消息的开头**精确匹配**该前缀字符串时，此规则才视为匹配。例如，`prefix: "USB"` 的规则将匹配所有消息以 `"USB"` 开头的 ftrace/print 事件（如 `"USB device connected"` 等）。前缀匹配是大小写敏感的精确匹配，且**只匹配消息开头**，并不匹配消息中间或结尾的子串。

特殊地，如果将 `prefix` 设置为空字符串（`""`），由于任何字符串的开头都可以看作空串，该规则会匹配**所有** ftrace/print 消息。这种用法通常用作**通配符规则**：配合 `allow=false` 可以充当“阻止所有”的默认规则，如上节所述。因此，空前缀规则通常放在列表末尾，用于捕获**所有未被前面规则匹配的事件**，从而改变默认通过的行为。

#### `atrace_msg` 规则的 atrace 消息匹配

`atrace_msg` 用于匹配 **Android Atrace 风格**的打印消息。Android 中用户空间通过 `ATRACE_BEGIN()/ATRACE_END()/ATRACE_INT()` 等宏产生的事件，会写入 ftrace 的 trace buffer，格式一般为：`<类型>|<PID>|<消息内容>`（有时counter还包含第四段数值）。Perfetto 将这些记录也作为 ftrace 的 `"print"` 事件来采集。通过 `atrace_msg` 规则，我们可以对这类特殊格式的消息进行解析匹配，其匹配逻辑如下：



**消息格式**：`atrace_msg` 假定目标消息符合 `类型|PID|内容...` 的结构。其中“类型”通常是一个标识符字符，例如：`B` 表示一次区间**开始** (Begin)，`E` 表示区间**结束** (End)，`C` 表示一个**计数器** (Counter) 更新

。`PID`是发出该消息的进程ID，`内容`则是用户提供的事件名称或信息（对Counter而言，`内容`后面还会有一个 `|值` 部分）。

**匹配规则字段**：`atrace_msg` 规则本身包含两个可选子字段：`type` 和 `prefix`。`type`用于指定要匹配的atrace消息类型（如 `"B"`, `"E"`, `"C"` 等），`prefix`用于指定atrace消息内容部分的前缀。规则会将每条 ftrace/print 消息尝试解析为上述格式，然后应用以下条件：

如果设置了 `type`，则消息的类型必须**完全等于**指定值才能匹配。例如，`type: "B"` 只匹配开始事件，`type: "C"` 只匹配计数器事件。

如果设置了 `prefix`，则消息内容（在 `PID` 之后的部分）必须以该前缀开头才能匹配。例如，`prefix: "Render"` 将匹配内容以 `"Render"` 开头的消息（如 `"RenderFrame"`、`"RenderInit"` 等）。

当同时指定 `type` 和 `prefix` 时，两个条件都需满足，即类型匹配且内容前缀匹配。如果只指定其中一项，另一项则不作为限制条件（相当于接受任意值）。

**匹配示例**：假设一条 ftrace 打印消息的 `buf` 内容是：`B|1234|CameraPreviewStarted`，它表示PID为1234的进程记录了一条名为“CameraPreviewStarted”的**区间开始**事件。对于规则 `atrace_msg { type: "B", prefix: "Camera" }`，解析后类型为`"B"`符合，内容部分 `"CameraPreviewStarted"` 以 `"Camera"` 开头，也符合，因此该规则会匹配此事件。再如，规则 `atrace_msg { type: "C", prefix: "Memory" }` 将匹配所有名称前缀为 "Memory" 的计数器更新事件（如 `"C|4321|MemoryUsage|1024"`）——因为内容 `"MemoryUsage|1024"` 以 `"Memory"` 开头。

**非 atrace 消息**：如果某条 ftrace/print 消息不含有上述 `|` 格式（例如纯粹的内核 trace\_printk 文本），那么任何 `atrace_msg` 规则都不会匹配它，因为解析格式不符。此时只能通过前述 `prefix` 规则来匹配这类普通打印消息。

####  `allow` 选项的作用和默认行为

每条 `print_filter` 规则都包含一个布尔字段 `allow`，用于指示当规则匹配时是否**允许**该事件进入最终trace。其作用可以总结如下：

当规则匹配时，如果 `allow = true`，则该 ftrace/print 事件会被**保留**（允许记录）​；如果 `allow = false`，则该事件会被**丢弃**，从trace中过滤掉。这相当于在规则层面设置“通过/阻止”开关。

多条规则可以组合实现复杂过滤策略。例如，可以先列出若干 `allow=false` 的规则来筛除不需要的噪音事件，再通过后续规则允许特定模式的事件；也可以相反，先允许特定事件，再在末尾放一个全拒绝的规则阻止其余所有事件。**需要注意规则顺序**：因为一旦某规则匹配，后续规则就不再查看，其它规则的 allow 设置也就无效了。因此，应将更具体的规则放在前面，通配的规则放在后面。

**默认行为**：正如前面所述，`print_filter` 对未匹配任何规则的事件默认是**允许**的。换句话说，如果想禁止某些事件，可以添加对应模式的 `allow=false` 规则；但如果想**仅**允许某些事件而禁止其他所有事件，则需要在允许规则之外，再添加一个 `allow=false` 的兜底规则，否则不匹配的事件仍会被记录。默认允许的设计可以防止由于规则遗漏而意外丢失重要事件，但同时要求用户明确定义想要排除哪些内容。

总结起来，`allow` 选项配合规则顺序，决定了哪些 ftrace 打印消息会出现在最终的 Perfetto trace 中。如果没有设置任何 `print_filter`，所有 ftrace/print 事件都会记录；一旦设置了过滤规则，就只有符合 **“先匹配到的允许规则”** 或 **“未被任何禁止规则命中”** 的事件才会留存。通过合理设置 `allow=true/false` 以及默认兜底规则，可以实现黑名单或白名单式的过滤策略。



#### 2.2.5 使用示例

### `print_filter` 过滤不同类型的 ftrace 打印

下面通过两个示例配置来说明如何使用 `print_filter` 规则实现不同的 ftrace 打印消息过滤需求。

**黑名单示例：过滤掉特定前缀的打印消息&#x20;**&#xA;假设我们想排除内核中的调试日志，例如所有以 `"DEBUG"` 开头的 ftrace 打印事件，而不过滤其他消息。可以使用仅含 `prefix` 的规则来实现：

```bash
print_filter {
  rules: [{
    prefix: "DEBUG",
    allow: false
  }]
}
```

这份配置包含一条规则：匹配所有消息前缀为 `"DEBUG"` 的打印事件，并将其丢弃（`allow=false`）。因为我们没有设置其他规则，未匹配到 `"DEBUG"` 前缀的事件将按照默认行为被**允许**记录

也就是说，除了被标记为DEBUG的打印，其它所有 ftrace/print 事件都会出现在trace中。

**白名单示例：只允许特定 atrace 类别的消息**
假设我们只关心应用/系统中的某模块相关的 Systrace (atrace) 信息，比如名称前缀为 `"Camera"` 的跟踪事件，而想排除其他所有 atrace 消息。我们可以结合 `atrace_msg` 和空前缀规则实现严格的允许列表：

```yaml
print_filter {
  rules: [
    {
      atrace_msg: { prefix: "Camera" },
      allow: true
    },
    {
      prefix: "",    # 空前缀匹配所有剩余未匹配事件
      allow: false
    }
  ]
}
```

第一条规则使用了 `atrace_msg`，指定 `prefix: "Camera"`（未限定类型，则匹配任何类型的 atrace 消息）​。这会**允许**所有名称以 "Camera" 开头的 atrace 打印事件进入trace，如 `"B|1234|CameraStart"` 或 `"C|1234|CameraFrames|30"` 等。第二条规则是空前缀且 `allow=false`，会匹配**所有**仍未被前面允许的打印事件，并将其丢弃。有了这条规则，任何不以 "Camera" 开头的打印消息（包括其它 atrace 分类的事件以及普通内核打印）都会被过滤掉，不出现在最终trace中。通过这种 **“允许Camera相关，禁止其他”** 的配置，实现了针对 atrace 消息的精确过滤。

以上示例展示了 `print_filter` 的两种典型用法：按前缀**剔除**不需要的信息，以及按前缀**保留**所需的信息并排除其余内容。借助 `print_filter`，我们可以灵活地控制 Perfetto trace 中 ftrace `"print"` 事件的记录，筛选出真正关心的内容。例如，除了上述示例，还可以利用 `atrace_msg.type` 来过滤特定类型的 atrace 事件——比如添加规则 `{ atrace_msg: { type: "C" }, allow: false }` 来**过滤掉所有计数器更新**，以减少Trace日志量。通过组合不同规则并善用 `allow` 布尔值，Perfetto 能满足各种细粒度的内核打印日志过滤需求。



## 5. compact_sched

/external/perfetto/protos/perfetto/config/ftrace/ftrace\_config.proto

```c++
// Configuration for compact encoding of scheduler events. When enabled (and
  // recording the relevant ftrace events), specific high-volume events are
  // encoded in a denser format than normal.
  message CompactSchedConfig {
    // If true, and sched_switch or sched_waking ftrace events are enabled,
    // record those events in the compact format.
    //
    // If the field is unset, the default is:
    // * perfetto v42.0+: enabled
    // * before: disabled
    optional bool enabled = 1;
  }
  optional CompactSchedConfig compact_sched = 12;
```

> 这个enable在perfetto 42+上默认开启。



## 6. disable_generic_events

Generates smaller trace files at the cost of extra CPU cycles when stopping the trace. Compression happens only after the end of the trace and does not improve the ring-buffer efficiency.

```c++
// If true, avoid enabling events that aren't statically known by
  // traced_probes. Otherwise, the default is to emit such events as
  // GenericFtraceEvent protos.
  // Prefer to keep this flag at its default. This was added for Android
  // tracing, where atrace categories and/or atrace HAL requested events can
  // expand to events that aren't of interest to the tracing user.
  // Introduced in: Android T.
  optional bool disable_generic_events = 16;
```

> 这个选项默认值是false，如果设置为true，则自定义的事件就无法跟踪。
>
> 在验证中发现设置为true以后trans_sched/dump_info无法被识别。



##  7. instance_name

```c++
  // If |instance_name| is not empty, then attempt to use that tracefs instance
  // for event recording. Normally, this means
  // `/sys/kernel/tracing/instances/$instance_name`.
  //
  // The name "hyp" is reserved.
  //
  // The instance must already exist, the tracing daemon *will not* create it
  // for you as it typically doesn't have such permissions.
  // Only a subset of features is guaranteed to work with non-default instances,
  // at the time of writing:
  //  * ftrace_events
  //  * buffer_size_kb
  optional string instance_name = 25;
```

> 这是可以替代`fans.ftrace`的一个属性，非常重要，问题是buffer\_size\_kb这个值如何控制？

```c++
        行 37526: 02-13 22:03:09.641 10899 10899 I perfetto:    perfetto_cmd.cc:1102 Connected to the Perfetto traced service, starting tracing
        行 37527: 02-13 22:03:09.642  1347  1347 I perfetto: ng_service_impl.cc:1125 Configured tracing session 6, #sources:2, duration:0 ms, #buffers:2, total buffer size:67584 KB, total sessions:1, uid:0 session name: ""
        行 37528: 02-13 22:03:09.642  1344  1344 I perfetto:  probes_producer.cc:132 Ftrace setup (target_buf=11)
        行 37537: 02-13 22:03:09.697  1344  1344 I perfetto:    ftrace_procfs.cc:441 disabled ftrace in /sys/kernel/tracing/instances/fans/
        行 37539: 02-13 22:03:09.710  1344  1344 E perfetto: ace_config_muxer.cc:656 Secondary ftrace instances do not support atrace_categories and atrace_apps options as they affect global state
        行 37540: 02-13 22:03:09.710  1344  1344 E perfetto:  probes_producer.cc:138 Failed to setup ftrace
        行 37541: 02-13 22:03:09.710  1344  1344 E perfetto:  probes_producer.cc:464 Failed to create data source 'linux.ftrace'
        行 37542: 02-13 22:03:09.711  1344  1344 E perfetto:  probes_producer.cc:480 Data source id=11 not found
        行 48084: 02-13 22:03:30.055  2351  2680 V PerfettoTrigger: Triggering /system/bin/trigger_perfetto com.android.telemetry.interaction-jank-monitor-8
```

instance\_name只能作用于ftrace\_events，如果有atrace\_categories等，就需要单独使用一个datasource

```c++
mkdir /sys/kernel/tracing/instances/trans
// /sys/kernel/tracing/instances/trans下修改如下几个关键节点读写权限
chmod 666 buffer_percent tracing_on buffer_size_kb trace_clock trace set_event 
chmod 222 trace_marker 
// 修改enable的权限
chmod 666 sched/sched_waking/enable sched/sched_switch/enable power/cpu_frequency/enable events/enable...
```

> denied  { write } for  comm="traced\_probes" name="enable" dev="tracefs" ino=164066 scontext=u:r:traced\_probes:s0 tcontext=u:object\_r:debugfs\_tracing\_instances:s0 tclass=file permissive=1
> 需要增加traced_probes 对debugfs_tracing_instrances的权限，这个已经合入主线。



## 8. compression_type

通过压缩减少 Trace 数据的体积，从而降低存储空间占用或提高传输效率。

### 8.1 可选的compression_type值

以下是 `perfetto` 支持的 `compression_type` 值及其特点：

#### 8.1.1 COMPRESSION_TYPE_NONE

* **描述**：不启用任何压缩。

* **特点**：

  * **优点**：速度快，无压缩开销。

  * **缺点**：文件体积较大。

* **适用场景**：对性能要求极高且存储空间充足的场景。

#### 8.1.2 COMPRESSION_TYPE_DEFLATE

* **描述**：使用 `zlib` 库实现的 `deflate` 压缩算法。

  ```c++
   // -# 压缩等级，默认3，最高19
   zstd -19 trace20250226114131.perfetto-trace -o trace20250226114131.perfetto-trace.zs
  ```

* **特点**：

  * **优点**：压缩比适中，压缩和解压速度较快。

  * **缺点**：相比无压缩会有一定的性能开销。

* **适用场景**：平衡压缩比和性能的场景。

#### 8.1.3 COMPRESSION_TYPE_SNAPPY

* **描述**：使用 `Snappy` 压缩算法。

* **特点**：

  * **优点**：压缩和解压速度非常快，适合实时处理。

  * **缺点**：压缩比相对较低。

* **适用场景**：对速度要求极高但对压缩比要求不高的场景。

#### 8.1.4 COMPRESSION_TYPE_ZSTD

* **描述**：使用 `Zstandard (zstd)` 压缩算法。

* **特点**：

  * **优点**：高压缩比，且支持多种压缩级别（从快速到极致压缩）。

  * **缺点**：在极高压缩级别下可能会有一定的性能开销。

* **适用场景**：需要高压缩比且对性能有一定容忍度的场景。

***

### 8.2 如何选择`compression_type`

* **基于性能需求**：

  * 如果需要最快的处理速度，选择 `COMPRESSION_TYPE_NONE` 或 `COMPRESSION_TYPE_SNAPPY`。

  * 如果需要更高的压缩比，选择 `COMPRESSION_TYPE_DEFLATE` 或 `COMPRESSION_TYPE_ZSTD`。

* **基于存储需求**：

  * 如果存储空间有限，优先选择 `COMPRESSION_TYPE_DEFLATE` 或 `COMPRESSION_TYPE_ZSTD`。

  * 如果存储空间充足且对速度要求极高，可以选择 `COMPRESSION_TYPE_NONE`。

* **综合平衡**：

  * 对大多数场景来说，`COMPRESSION_TYPE_DEFLATE` 是一个折中的选择，既能提供不错的压缩比，又能保持较快的速度。

> 选用COMPRESSION\_TYPE\_DEFLATE会导致在TNE打包时重复处理，经过zstd算法压缩过的文件再次压缩基本无效果

