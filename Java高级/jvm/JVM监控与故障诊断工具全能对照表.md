这份文档系统性整理了 JVM 生态中常用的监控、诊断、调优工具，涵盖 JDK 自带命令行、图形化工具，以及主流第三方诊断工具，包含每个工具的作用、常用指令、核心参数、用法示例、工具差异，可作为日常开发、运维、故障排查的速查手册。

---

## 一、工具总览对比表

快速了解各工具的定位、性能影响与适用场景：

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|工具名称|工具类型|核心功能|性能影响|适用场景|JDK 版本要求|推荐度|
|jps|命令行|列出 Java 进程与 PID|无|定位目标 Java 进程，所有工具的入口|1.5+|⭐⭐⭐⭐⭐|
|jstat|命令行|实时 GC / 内存 / 类加载监控|极低|线上实时监控 GC 行为、内存变化|1.5+|⭐⭐⭐⭐⭐|
|jinfo|命令行|查看 / 动态修改 JVM 参数|低|查看运行时配置、动态调整参数|1.6+|⭐⭐⭐⭐|
|jmap|命令行|生成堆快照、对象统计|中（dump 会触发 STW）|内存泄漏分析、堆内存查看|1.6+|⭐⭐⭐（推荐用 jcmd 替代）|
|jhat|命令行|堆快照 HTTP 分析服务|中|离线简单分析堆快照|1.6+|⭐⭐（推荐用 MAT 替代）|
|jstack|命令行|生成线程快照|低|死锁、高 CPU、线程阻塞分析|1.5+|⭐⭐⭐⭐⭐|
|jcmd|命令行|多功能整合诊断工具|低|替代旧工具，JFR 操作、Native 内存分析|1.7+|⭐⭐⭐⭐⭐|
|jhsdb|命令行|离线 core dump 分析|无（离线）|JVM 崩溃后离线分析 core 文件|9+|⭐⭐⭐|
|jconsole|图形化|基础 JVM 监控|低|本地 / 远程基础监控|1.5+|⭐⭐⭐|
|jvisualvm|图形化|全功能监控分析|中|开发 / 测试环境深度分析|1.6+（9 + 需单独下载）|⭐⭐⭐⭐|
|JMC/JFR|图形化 / 记录|低开销事件记录|极低（<1%）|生产环境长期监控、深度诊断|7u40+（11 + 开源）|⭐⭐⭐⭐⭐|
|Arthas|动态诊断工具|线上动态追踪诊断|低|线上问题排查、方法调用分析|全版本|⭐⭐⭐⭐⭐|
|Eclipse MAT|图形化|堆快照深度分析|无（离线）|内存泄漏、大对象定位|全版本|⭐⭐⭐⭐⭐|
|async-profiler|采样分析工具|低开销 CPU / 内存采样|极低|热点方法、火焰图生成|全版本|⭐⭐⭐⭐|
|GCViewer|图形化|GC 日志可视化分析|无（离线）|GC 日志深度分析|全版本|⭐⭐⭐⭐|

---

## 二、基础命令行工具详解

### 1. jps：Java 进程状态工具

**作用**：快速列出当前用户的所有 Java 进程，获取进程 ID（PID）和主类信息，是所有 JVM 工具的入口，类似 Unix 的`ps`命令，但仅聚焦 Java 进程。

**常用指令**：

```bash
jps [options] [hostid]
```

**核心参数**：

|   |   |
|---|---|
|参数|说明|
|`-l`|输出主类的全限定名或 Jar 包的完整路径|
|`-v`|输出传递给 JVM 的启动参数|
|`-m`|输出传递给 main 方法的参数|
|`-q`|仅输出进程 PID，省略其他所有信息|

**用法示例**：

```bash
# 查看所有Java进程的完整信息
jps -lvm
# 输出示例：12345 com.example.Application -Xmx2G -Xms2G
```

**注意事项**：仅能查看当前用户有权限访问的进程，远程监控需要提前启动 jstatd 服务。

---

### 2. jstat：JVM 统计监控工具

**作用**：轻量级实时监控工具，以固定间隔采样 JVM 的运行状态，尤其是 GC、内存、类加载、JIT 编译等指标，对性能影响极小，是线上监控的首选工具。

**常用指令**：

```bash
jstat -<option> <pid> [interval] [count]
```

- `option`：监控维度
    
- `interval`：采样间隔（毫秒）
    
- `count`：采样次数，省略则持续输出
    

**核心监控参数**：

|   |   |
|---|---|
|参数|说明|
|`-gcutil`|最常用，输出各内存区域的使用率（百分比），以及 GC 次数与耗时|
|`-gc`|输出各内存区域的容量、已使用量（KB）的详细信息|
|`-gccapacity`|输出各内存区域的初始、当前、最大容量，用于验证 JVM 内存参数|
|`-gccause`|在`-gcutil`基础上，附加显示最近一次 GC 的触发原因|
|`-class`|输出类加载、卸载的数量与占用空间，用于排查元空间泄漏|
|`-compiler`|输出 JIT 编译的统计信息|

**输出字段解读（以****`-gcutil`****为例）**：

```Plain
Timestamp   S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
100.0       0.00  66.67  45.50  68.92  94.29  91.90     10    0.100     3    0.450    0.550
```

|   |   |
|---|---|
|字段|说明|
|S0/S1|Survivor0/Survivor1 区使用率|
|E|Eden 区使用率|
|O|老年代使用率|
|M|元空间使用率|
|YGC/YGCT|Young GC 次数 / 总耗时（秒）|
|FGC/FGCT|Full GC 次数 / 总耗时（秒）|
|GCT|所有 GC 的总耗时|

**用法示例**：

```bash
# 每1秒采样一次GC使用率，共采样10次
jstat -gcutil 12345 1000 10

# 持续监控类加载情况，每2秒输出一次
jstat -class 12345 2000
```

**异常识别**：

- 老年代使用率`O`持续增长，Full GC 后无明显下降：大概率存在内存泄漏
    
- YGC 频率过高（每秒多次）：年轻代过小，或对象分配速率过快
    
- Full GC 频率过高：老年代过小，或内存泄漏
    

---

### 3. jinfo：JVM 配置信息工具

**作用**：查看运行中 JVM 的参数和系统属性，支持动态修改部分可写的 JVM 参数，无需重启应用。

**常用指令**：

```bash
jinfo [option] <pid>
```

**核心参数**：

|                        |                                          |                     |
| ---------------------- | ---------------------------------------- | ------------------- |
| 参数                     | 说明                                       |                     |
| `-flags`               | 查看所有 JVM 启动参数（包括默认值和显式设置的参数）             |                     |
| `-sysprops`            | 查看 JVM 的系统属性，等价于`System.getProperties()` |                     |
| `-flag <name>`         | 查看指定 JVM 参数的值                            |                     |
| `-flag [+              | -]`                                      | 动态开启 / 关闭布尔型 JVM 参数 |
| `-flag <name>=<value>` | 动态修改数值型 JVM 参数                           |                     |
|                        |                                          |                     |

**用法示例**：

```bash
# 查看所有JVM启动参数
jinfo -flags 12345

# 查看最大堆内存参数的值
jinfo -flag MaxHeapSize 12345

# 动态开启GC日志打印
jinfo -flag +PrintGC 12345
```

**注意事项**：并非所有 JVM 参数都支持动态修改，只有标记为`Writeable`的参数才支持，可通过`jcmd <pid> VM.flags -all`查看参数的可写性。

---

### 4. jmap：内存映射工具

**作用**：生成 Java 堆快照（Heap Dump），查看堆内存的配置与使用情况，统计堆中对象的数量与大小，用于定位内存泄漏、大对象。

**常用指令**：

```bash
jmap [option] <pid>
```

**核心参数**：

|   |   |
|---|---|
|参数|说明|
|`-heap`|查看堆内存的配置、各代的使用情况，以及使用的 GC 收集器|
|`-histo[:live]`|输出堆中对象的直方图，统计每个类的实例数、内存占用；加`:live`仅统计存活对象，会触发 Full GC|
|`-dump:format=b,file=<file>`|生成二进制格式的堆快照文件（.hprof），可用于后续 MAT 等工具分析|

**用法示例**：

```bash
# 查看堆内存的配置与使用情况
jmap -heap 12345

# 查看存活对象的统计信息，输出前20行
jmap -histo:live 12345 | head -20

# 生成堆快照文件
jmap -dump:format=b,file=/tmp/heap.hprof 12345
```

**注意事项**：

- 生成堆快照和`-histo:live`会触发 Full GC，导致 Stop-The-World，大堆情况下可能停顿数秒到数十秒，生产环境建议在低峰期执行
    
- JDK9 之后，jmap 的部分功能已被 jcmd 替代，推荐优先使用 jcmd 的对应命令
    

---

### 5. jhat：堆快照分析工具

**作用**：解析 jmap 生成的堆快照文件，启动一个内置的 HTTP 服务器，供用户在浏览器中查看堆中的对象信息、引用关系。

**常用指令**：

```bash
jhat [options] heap-dump-file
```

**核心参数**：

|   |   |
|---|---|
|参数|说明|
|`-port <port>`|指定 HTTP 服务的端口，默认 7000|
|`-baseline <file>`|指定基准堆快照，用于对比两个不同时间点的堆差异，定位新增的对象|

**用法示例**：

```bash
# 解析堆快照，启动HTTP服务
jhat /tmp/heap.hprof
# 之后访问 http://localhost:7000 即可查看分析结果
```

**注意事项**：jhat 功能简单，分析大堆文件时性能很差，界面简陋，目前已基本被 Eclipse MAT 替代，仅适合临时简单分析。

---

### 6. jstack：线程堆栈工具

**作用**：生成 JVM 当前时刻的线程快照（Thread Dump），用于定位线程死锁、线程阻塞、高 CPU 占用、程序挂死等问题。

**常用指令**：

```bash
jstack [option] <pid>
```

**核心参数**：

|   |   |
|---|---|
|参数|说明|
|`-l`|额外显示锁的详细信息，包括 java.util.concurrent 的锁信息，是死锁分析的必备参数|
|`-f`|强制生成线程快照，用于进程无响应、正常命令无法 attach 的情况|

**用法示例**：

```bash
# 生成线程快照，输出到文件
jstack -l 12345 > /tmp/thread_dump.txt
```

**高 CPU 问题排查技巧**：

1. 使用`top -Hp <pid>`找到 CPU 占用最高的线程 ID（比如 12345）
    
2. 将线程 ID 转换为 16 进制：`printf "%x\n" 12345`，得到`0x3039`
    
3. 在 jstack 的输出中，查找`nid=0x3039`的线程，即可定位到该线程的执行栈，找到耗时代码
    

**死锁检测**：jstack 会自动检测死锁，在输出的最后会打印`Found one Java-level deadlock`的提示，同时展示死锁线程的锁持有情况。

---

## 三、多功能整合工具：jcmd

**作用**：JDK7 引入的新一代诊断工具，Oracle 官方推荐的统一诊断工具，整合了 jps、jstat、jmap、jstack、jinfo 的几乎所有功能，同时支持操作 JFR、Native 内存跟踪等高级功能，逐步替代了旧的分散工具。

**常用指令**：

```bash
jcmd <pid | main-class> <command> [options]
```

### 核心命令与旧工具替代关系

|   |   |   |
|---|---|---|
|jcmd 命令|替代的旧工具命令|说明|
|`jcmd`|`jps`|列出所有 Java 进程|
|`jcmd <pid> help`|-|查看该进程支持的所有诊断命令|
|`jcmd <pid> Thread.print`|`jstack <pid>`|生成线程快照|
|`jcmd <pid> GC.heap_dump filename=heap.hprof`|`jmap -dump:file=heap.hprof <pid>`|生成堆快照|
|`jcmd <pid> GC.class_histogram`|`jmap -histo <pid>`|查看对象直方图|
|`jcmd <pid> GC.heap_info`|`jmap -heap <pid>`|查看堆内存配置与使用|
|`jcmd <pid> VM.flags`|`jinfo -flags <pid>`|查看 JVM 启动参数|
|`jcmd <pid> VM.system_properties`|`jinfo -sysprops <pid>`|查看系统属性|
|`jcmd <pid> VM.set_flag <name> <value>`|`jinfo -flag <name> <pid>`|动态修改 JVM 参数|
|`jcmd <pid> JFR.start`|-|启动 JFR 记录|
|`jcmd <pid> VM.native_memory`|-|查看 Native 内存分配（NMT），分析堆外内存泄漏|

**用法示例**：

```bash
# 生成堆快照
jcmd 12345 GC.heap_dump /tmp/heap.hprof

# 启动60秒的JFR记录，保存到文件
jcmd 12345 JFR.start duration=60s filename=/tmp/recording.jfr

# 查看Native内存分配情况
jcmd 12345 VM.native_memory summary
```

**注意事项**：JDK9 之后，很多旧的 j * 工具（如 jhat、jmap 的部分功能）被标记为废弃，Oracle 推荐优先使用 jcmd 作为统一的命令行诊断工具。

---

## 四、图形化监控工具

### 1. jconsole：基础监控控制台

**作用**：JDK 自带的轻量级图形化监控工具，支持本地和远程连接，实时查看 JVM 的内存、线程、类加载、MBean 信息，无需额外配置。

**启动方式**：命令行直接输入`jconsole`，选择要连接的进程即可。

**界面展示**：

**核心功能**：

- 内存：实时查看堆 / 非堆内存使用趋势，手动触发 GC
    
- 线程：查看所有线程的状态，自动检测死锁
    
- 类：查看类加载 / 卸载的统计趋势
    
- VM 摘要：查看 JVM 参数、系统属性
    
- MBean：查看和操作 JMX 的 MBean，支持动态修改配置
    

**适用场景**：开发 / 测试环境的快速基础监控，简单易用，适合快速查看 JVM 的整体状态。

---

### 2. jvisualvm：全功能可视化工具

**作用**：JDK 自带的全能可视化工具，是 jconsole 的超集，整合了多个命令行工具的功能，支持实时监控、线程分析、内存分析、CPU 抽样，还支持插件扩展，功能非常全面。

**启动方式**：JDK8 及之前版本输入`jvisualvm`即可启动；JDK9 + 之后，该工具不再内置，需要单独从[官网](https://visualvm.github.io/)下载。

**界面展示**：

**核心功能**：

- 实时监控 CPU、内存、线程、类加载的趋势
    
- 生成和分析堆快照、线程快照
    
- CPU / 内存抽样，定位热点方法、内存分配热点
    
- 插件扩展：比如 VisualGC 插件可以可视化展示 GC 区域的变化，MAT 插件可以集成堆快照分析
    
- 支持远程连接监控远程服务器的 JVM
    

**适用场景**：开发 / 测试环境的深度性能分析，功能全面，免费开源，是日常开发中非常实用的工具。

---

### 3. Java Mission Control & Java Flight Recorder

**作用**：JDK 自带的高性能监控套件，是生产环境深度诊断的神器：

- **Java Flight Recorder (JFR)**：JVM 内置的事件记录引擎，以极低的性能开销（通常 < 1%）持续记录 JVM 和应用的所有事件，包括 GC、内存分配、线程事件、方法调用、IO 等，可在生产环境长期开启。
    
- **Java Mission Control (JMC)**：用于分析 JFR 记录文件的图形化工具，提供非常详尽的分析视图，可回溯问题发生时的所有状态。
    

**核心优势**：传统的诊断工具往往是问题发生后才去采集数据，而 JFR 是提前持续记录，问题发生后可以直接回溯过去的历史数据，非常适合排查偶发的、难以复现的问题。

**常用操作**：

```bash
# 对运行中的进程启动60秒的JFR记录
jcmd 12345 JFR.start duration=60s filename=/tmp/recording.jfr
```

**适用场景**：生产环境的长期监控，偶发性能问题、疑难问题的深度诊断，是 Oracle 推荐的生产环境诊断工具。

---

## 五、第三方高级诊断工具

### 1. Arthas：阿里开源线上诊断神器

**作用**：Alibaba 开源的 Java 诊断工具，无需重启应用，无需提前埋点，即可动态 attach 到运行中的 Java 进程，实现方法调用追踪、反编译、热更新、监控等功能，是线上问题排查的利器。

**启动方式**：

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
# 然后选择要attach的Java进程即可
```

**常用命令**：

|   |   |
|---|---|
|命令|说明|
|`dashboard`|实时系统面板，查看线程、内存、GC 的整体状态|
|`thread`|线程状态分析：`thread -n 3`查看 CPU 占用最高的 3 个线程，`thread -b`一键检测死锁|
|`watch`|观察方法的入参、返回值、异常，比如`watch com.example.UserService getUser '{params, returnObj}' -x 3`|
|`trace`|追踪方法内部的调用路径，输出每个调用节点的耗时，定位慢调用|
|`jad`|反编译指定类，查看线上实际运行的代码，排查线上代码是否和预期一致|
|`heapdump`|生成堆快照，类似 jmap 的功能|
|`ognl`|执行 OGNL 表达式，查看 / 修改静态变量、调用静态方法，动态修改线上配置|
|`redefine`|热更新类文件，无需重启应用即可修复线上小问题|

**适用场景**：线上环境的问题排查，无需重启，无需提前准备，动态诊断线上的方法执行、参数、耗时等问题，极大提升线上问题排查的效率。

---

### 2. Eclipse MAT：内存分析工具

**作用**：开源的堆快照深度分析工具，是分析内存泄漏、OOM 问题的首选工具，功能比 jhat 强大得多，支持大堆文件的分析。

**界面展示**：

**核心功能**：

- **Leak Suspects**：自动检测可能的内存泄漏点，生成泄漏报告
    
- **Dominator Tree**：展示占用内存最多的对象，快速定位大对象
    
- **Path to GC Roots**：查看对象的 GC 引用链，定位对象无法被回收的原因
    
- **Histogram**：查看各个类的实例数量、内存占用统计
    
- 对比两个堆快照，定位对象的变化
    

**使用流程**：

1. 使用 jmap 或 jcmd 生成堆快照（.hprof 文件）
    
2. 用 MAT 打开该文件，运行泄漏检测报告
    
3. 根据报告分析内存泄漏的原因
    

**适用场景**：内存泄漏、OOM 问题的深度分析，是内存问题排查的标准工具。

---

### 3. async-profiler：低开销采样分析工具

**作用**：开源的低开销采样分析工具，支持 CPU、内存分配、锁等维度的采样，性能开销极低，可生成火焰图，快速定位热点方法、内存分配热点。

**常用指令**：

```bash
# 采样30秒CPU，生成火焰图
./profiler.sh -d 30 -f flamegraph.html 12345

# 采样60秒内存分配，生成内存分配火焰图
./profiler.sh -e alloc -d 60 -f alloc-flamegraph.html 12345
```

**适用场景**：生产环境的热点方法定位，性能瓶颈分析，火焰图可以非常直观的展示各个方法的 CPU 占用情况。

---

### 4. GCViewer：GC 日志分析工具

**作用**：开源的 GC 日志可视化分析工具，可解析 GC 日志，生成可视化的图表，统计 GC 的频率、停顿时间、吞吐量等指标，帮助分析 GC 的性能问题。

**使用方式**：下载 GCViewer 的 Jar 包，运行后打开 GC 日志文件即可。

**适用场景**：GC 日志的深度分析，可视化 GC 的行为，定位 GC 的性能问题。

---

## 六、故障排查场景速查表

遇到问题时，快速找到对应的工具和排查步骤：

|   |   |   |
|---|---|---|
|问题场景|推荐工具|排查步骤|
|找不到 Java 进程|jps|`jps -lvm` 列出所有 Java 进程，确认目标进程的 PID|
|GC 频繁、停顿时间长|jstat、GCViewer、JFR|1. `jstat -gcutil <pid> 1s` 实时监控 GC 指标 2. 开启 GC 日志，用 GCViewer 可视化分析 3. 用 JFR 记录 GC 事件，分析触发原因|
|内存泄漏、OOM|jcmd、MAT|1. `jstat -gcutil <pid>` 观察老年代使用率是否持续增长 2. `jcmd <pid> GC.heap_dump heap.hprof` 生成堆快照 3. 用 MAT 分析泄漏点，定位大对象和引用链|
|CPU 使用率过高|jstack、Arthas、async-profiler|1. `top -Hp <pid>` 找到高 CPU 线程 ID 2. 转换为 16 进制，`jstack <pid>` 找到对应线程栈 3. 或用 Arthas `thread -n 3` 直接查看最忙线程 4. 用 async-profiler 生成火焰图，定位热点方法|
|线程死锁、阻塞|jstack、Arthas|1. `jstack -l <pid>` 查看锁信息，自动检测死锁 2. 或用 Arthas `thread -b` 一键检测死锁|
|线上方法执行慢、参数异常|Arthas|1. `trace` 追踪方法内部调用耗时 2. `watch` 观察方法的入参和返回值|
|想查看线上实际运行的代码|Arthas|`jad com.example.Service` 反编译线上的类|
|动态修改 JVM 参数|jcmd、jinfo|`jcmd <pid> VM.set_flag PrintGC true` 动态开启 GC 日志|
|偶发的性能问题，需要回溯历史数据|JFR|提前开启 JFR 记录，问题发生后导出记录，用 JMC 分析|
|JVM 崩溃、进程挂死|jhsdb|用 jhsdb 分析 core dump 文件，离线查看崩溃时的堆、线程状态|
|堆外内存泄漏|jcmd|`jcmd <pid> VM.native_memory`查看 Native 内存分配，定位泄漏点|

---

## 七、生产环境使用注意事项

1. **STW 操作的风险**：`jmap -dump`、`jmap -histo:live`这类操作会触发 Full GC，导致 Stop-The-World，大堆情况下可能停顿数秒到数十秒，生产环境建议在低峰期执行，优先使用 jcmd 的对应命令。
    
2. **JFR 的安全性**：JFR 是生产环境安全的工具，性能开销极低（<1%），推荐在生产环境长期开启低配置的 JFR 记录，保留最近 24 小时的环形数据，以便问题发生时可以回溯。
    
3. **Arthas 的影响**：Arthas 是动态 attach 的工具，默认情况下对应用的影响极小，排查完成后记得退出，避免残留。
    
4. **权限问题**：所有 JVM 工具都需要和目标进程使用相同的用户权限，否则无法 attach 到进程。