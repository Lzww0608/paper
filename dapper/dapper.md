# Dapper 

## 1. Dapper 是什么

Dapper 是 Google 在生产环境中长期运行的一套**分布式跟踪（distributed tracing）基础设施**。它要解决的问题不是“单机程序慢在哪里”，而是“一个用户请求穿过前端、RPC、中间层、后端、共享存储、网络之后，整条调用链到底发生了什么”。典型场景是 Web Search / Universal Search：一个查询会扇出到大量服务和机器，任何一个子系统的波动都可能放大成尾延迟，但仅凭单点日志或总体延迟指标，很难回答“到底谁慢、为什么慢、慢在链路的哪一段”。

Dapper 的价值不只在于抓一条 trace。更重要的是，它把 tracing 变成了一个平台：统一的数据模型、统一的传播机制、统一的采样、统一的采集和集中存储，再加上查询 API 和交互式 UI，让开发、测试、运维、性能分析、依赖分析、网络归因等工作都能建立在同一份时序事实之上。

---

## 2. 设计初衷：Google 需要怎样的 tracing 系统

Google 的互联网服务是由大量不同团队、不同语言、不同模块共同构成的大规模分布式系统，而工程师排查性能问题时通常缺的不是监控图表，而是**跨进程、跨机器、跨服务的一致视角**。

围绕这个背景，Dapper 有两个底层要求：

1. **无处不在（ubiquitous deployment）**：只要系统里有一小块没有被纳入 tracing，整条链路就可能断裂，trace 的解释力会显著下降。
2. **持续开启（continuous monitoring）**：很多线上异常很难复现，所以 tracing 不能是“出问题后再临时打开”的调试开关，而应是默认在线的生产能力。

从这两个要求出发，Dapper 明确提出了三个核心设计目标：

- **低开销（Low overhead）**：对业务服务的性能影响必须小到可以长期接受；否则服务 owner 会把 tracing 关掉。
- **应用级透明（Application-level transparency）**：不能要求每个业务开发者都主动配合埋点，否则系统会因为遗漏、误用、代码变更而变脆弱。
- **可扩展（Scalability）**：必须能覆盖 Google 当时以及未来若干年的服务规模。

此外，论文还提出一个很重要但容易被忽略的目标：**trace 数据要尽快可分析，理想情况下在 1 分钟内到达分析面**，因为新鲜数据直接决定了线上异常处置的速度。

---

## 3. 设计理念：Dapper 为什么“成了”

### 3.1 最小侵入，而不是全栈手工埋点

Dapper 不是把 tracing 责任层层下放给业务团队，而是把核心埋点收敛到**线程、控制流、RPC**这些几乎所有服务都会经过的公共库上。论文明确指出，正是因为 Google 内部有较强的基础设施同构性，Dapper 才能把 instrumentation 限定在一小组通用库中，并对应用开发者做到“近乎零干预”。

这背后体现的是一个非常重要的设计判断：**在大规模组织中，tracing 的首要问题不是功能够不够强，而是覆盖率能不能长期稳定地接近 100%**。Dapper 牺牲了一部分“想看哪里就看哪里”的自由度，换取大规模系统级可用性。

### 3.2 先把“因果关系”建出来

Dapper 的核心不是收集更多日志，而是恢复一个请求在分布式系统中的**因果树（causal tree）**。一旦 trace / span / parent-child 关系被正确建立，延迟、网络、跨服务依赖、异常上下文、资源归因等分析都能基于这棵树做二次计算。

这也是为什么 Dapper 的数据模型看起来很克制：

- trace 代表一次完整请求；
- span 代表一个基本工作单元；
- annotation 代表补充语义；
- trace id / span id / parent id 用来重建结构。

它先保证“骨架”正确，再允许业务用 annotations 往上加肉。

### 3.3 采样不是补丁，而是第一性原理

Dapper 没有把采样看成后加的存储优化，而是把它当作 tracing 能否长期在线的前提。论文甚至明确说，对高吞吐、低延迟服务而言，**sampling 是低开销的必要条件**；更进一步，Google 的经验是“每几千个请求采一个样本”依然足以支撑大部分常见分析。

这是一种非常工程化的取舍：**不是追求全量观察，而是追求低失真地观察系统行为模式**。对高频事件，全量不是必须；对低频系统，再自适应提高采样率。

### 3.4 Tracing 必须是平台，而不是单点工具

Dapper 论文中一个很有分量的结论是：Dapper 一开始只是 tracing tool，后来演化成了 monitoring platform。原因很简单：如果 trace 数据只服务于某一个 UI，那么它的价值会很快封顶；但如果有统一数据仓库、查询 API、MapReduce 批量分析接口和 Web UI，那么很多意料之外的工具都会从同一份 trace 数据里长出来。

这也是 Dapper 真正“超出论文本身”的地方。Google 在文中展示的场景已经不止是看一条调用链，而是包括依赖推断、网络流量归因、共享存储分析、测试与上线回归等。

### 3.5 安全与隐私默认保守

Dapper 明确不记录 RPC payload，只记录 RPC method name；如需更丰富语义，由应用开发者通过 opt-in 的 annotations 提供。这是一个非常保守但合理的选择：tracing 的首要目标是定位系统行为，而不是复制业务数据；在大公司内部，未经约束的 payload 采集会迅速踩到隐私与权限红线。

---

## 4. 数据模型：Trace、Span、Annotation 到底怎么表达一次请求

Dapper 把一次请求建模为一棵 trace tree：

- **Trace**：一次端到端请求的整体；
- **Span**：一个基本工作单元，常见情况下对应一次 RPC；
- **Parent/Child**：span 之间的因果关系；
- **Root span**：没有 parent id 的 span；
- **Annotation**：附着在 span 上的时间戳记录或应用补充信息。

论文给出的定义很经典：span 既是 trace tree 里的一个节点，也是一个“带时间戳记录的日志单元”，其中记录了 span 的开始/结束时间、RPC timing 信息，以及零个或多个应用级注解。

Dapper 为重建整棵树记录：

- `trace id`
- `span id`
- `parent id`
- `span name`

这些 id 都是**概率唯一的 64 位整数**。

有两个实现细节很关键：

1. **一个 RPC span 往往跨两个主机**。同一个 span 里可能同时包含 client 和 server 两侧的 annotation，因此 “two-host spans” 是最常见情况。 
2. **必须处理时钟偏移（clock skew）**。因为 client/server 时间来自不同机器，Dapper 在分析时利用“client send 一定发生在 server recv 之前”等因果约束，为 server 侧时间提供上下界，而不是盲目信任物理时钟完全对齐。 

这两个点说明 Dapper 的 tracing 不是“函数调用栈可视化”，而是**分布式因果结构的工程化重建**。 

---

## 5. 实现架构：Dapper 是如何跑起来的

### 5.1 Runtime instrumentation：把 trace context 跟着执行流走

Dapper 通过对少量公共库做 instrumentation 来跟踪控制路径，核心做法包括： 

- 当线程正在处理一条被跟踪的控制路径时，Dapper 把 trace context 挂到 **thread-local storage**；
- 当计算被延后或变成异步回调时，公共控制流库会把创建者的 trace context 一起保存，并在回调执行时重新关联到执行线程；
- Google 的大部分跨进程通信都建立在统一 RPC 框架上，Dapper 就在这里定义 span，并把 span id / trace id 从 client 传到 server。 

这套设计的本质，是把“上下文传播（context propagation）”做成基础设施能力，而不是业务代码责任。Dapper 也因此能够**透明地穿过异步控制流**，而不仅仅是同步 RPC 栈。 

### 5.2 可选 annotations：自动骨架 + 人工补语义

仅靠公共库 instrumentation，Dapper 已经能对复杂分布式系统生成有用 trace；但它仍然提供轻量 API，让业务代码补充应用语义。例如 cache hit/miss、输入输出大小、逻辑阶段、表名、用户定义键值等都可以挂到 span 上。 

论文特别强调两个约束：

- annotation API 必须足够轻；
- 单个 span 的 annotation 总量要有上限，防止“把 tracing 当日志系统用”导致成本失控。 

这形成了一个非常实用的分层模型：

- **自动 instrumentation** 负责结构完整；
- **手工 annotations** 负责业务可解释性。 

### 5.3 采样：运行时采样 + 采集侧二次采样

Dapper 的采样并不是一层，而是两层： 

#### 运行时采样

最初的生产版 Dapper 对所有进程使用统一采样概率，平均大约是 **1/1024**。这对高流量在线服务已经足够，因为重要模式即使只采千分之一，也会反复出现。 

但对低流量系统，这样的固定概率可能过低，所以 Dapper 又引入了**自适应采样**思路：不是指定固定概率，而是指定“单位时间目标样本数”。这样低流量系统自动提高采样率，高流量系统自动降低采样率。 

#### 采集侧二次采样

运行时采样主要为降低业务侧开销；但中心仓库仍然要控制写入量。于是 Dapper 在 collection system 里又做了一层基于 `trace id` 哈希的二次采样：对于同一个 trace，要么整条保留，要么整条丢弃，而不是只丢部分 span。 

这个设计非常关键，因为**局部丢 span 会破坏 trace 的结构完整性**。二次采样以 trace 为单位决策，保证了剩下的数据仍可用于因果分析。 

### 5.4 Trace collection pipeline：本地日志、daemon、Bigtable

论文给出的采集链路是一个三阶段管线： 

```mermaid
flowchart LR
    A 业务进程中的 Dapper runtime] --> B 本地 log files]
    B --> C Dapper daemon / collection infrastructure]
    C --> D Regional Dapper Bigtable repositories]
    D --> E DAPI / Web UI / MapReduce tools]
```

具体流程是：

1. span 数据先写入**本地日志文件**；
2. 再由各主机上的 **Dapper daemons** 和 collection infrastructure 拉取；
3. 最终写入多个区域性的 **Dapper Bigtable repositories**。 

存储布局也很有代表性：

- **一条 trace = Bigtable 里的一行（row）**；
- **每个 span = 一个列（column）**；
- Bigtable 的 sparse table 特性很适合“一个 trace 里 span 数量不固定”的场景。 

### 5.5 为什么是 out-of-band collection，而不是把 trace 随 RPC 一起回传

Dapper 明确采用**带外采集（out-of-band collection）**，而不是把 trace 数据塞到 RPC response header 里回传。论文给了两个理由： 

1. **避免影响业务网络动态**：大 trace 可能包含成千上万个 span，如果把 tracing 数据塞回业务响应，trace 本身就会改变网络行为，分析结果会被 tracing 系统污染。 
2. **支持非完美嵌套的执行模式**：有些中间件会先返回结果，再等待自身的后端收尾；如果要求“所有子调用严格嵌套在父调用内部”，就无法正确表达这类真实系统行为。 

这是 Dapper 很容易被忽视但极其重要的一点：**一个 tracing 系统如果改变了被观察系统的行为，它就会失去一部分可信度。** 

### 5.6 分析面：DAPI 与 Web UI

Dapper 不只存数据，还提供了完整分析平面：

#### DAPI（Depot API）

DAPI 提供三种访问路径： 

- **按 trace id 读取单条 trace**；
- **批量访问**：借助 MapReduce 并行处理数十亿条 trace；
- **索引访问**：按 service / host / timestamp 等模式快速查数据。 

论文还提到一个很工程化的现实：trace 索引的压缩后存储成本只比原始 trace 数据低 **26%**，所以索引设计必须非常克制，最终选择了按 **service name + host machine + timestamp** 的组合索引。 

#### Web UI

Dapper 的交互式 UI 支持：

- 先按服务、时间窗口、span name、成本指标筛出执行模式；
- 再查看某类模式的分布；
- 再点进某条具体 trace，沿全局时间线展开/折叠子树；
- 对每个 RPC span 区分 server processing time 与 network time。 

对“救火”场景，UI 还可以**直接和生产机上的 daemon 通信**，在几秒内拿到高延迟样本 trace，而不必等待中心仓库聚合完成。 

---

## 6. 性能与规模：Dapper 为什么能常驻生产

Dapper 的工程成功，很大程度上来自它对开销的克制。

### 6.1 Runtime 本身很小

论文披露：Dapper 核心 instrumentation 代码量不到 **1000 行 C++**、不到 **800 行 Java**，而 key-value annotations 再增加约 **500 行**。这表明 Dapper 的核心并不复杂；它复杂的地方主要在部署、覆盖率、采样、采集、存储和分析生态，而不是 runtime API 花样。 

### 6.2 覆盖率极高

Dapper daemon 在 Google 的基础机器镜像中，因而存在于**几乎每台服务器**。论文估计，依赖 Dapper-instrumented 公共库的程度已经高到“**几乎每个生产进程都支持 tracing**”。在成千上万的应用中，只有 **40 个 C++ 应用和 33 个 Java 应用**需要手工 trace propagation 作为 workaround。 

### 6.3 微观开销数据

论文给出了 runtime operation 的纳秒级开销（测试机为 2.2GHz x86）： 

| 操作 | 平均开销 |
|---|---:|
| Root span 创建与销毁 | 204 ns |
| Non-root span 创建与销毁 | 176 ns |
| 未被采样 span 上的一次 annotation 查询 | 9 ns |
| 已采样 span 上追加一个字符串 annotation | 40 ns |

本地磁盘写是 runtime 中最贵的操作，但由于采用**异步写 + 合并写**，可见开销被显著压低。 

### 6.4 采集侧开销

在负载测试下，Dapper daemon 的 CPU 占用从 **0.125% 到 0.267% 单核**不等；在生产环境中，论文给出的结论是：daemon 在采集时**从不超过一颗 CPU 核心的 0.3%**，并且内存占用很小。 

网络方面，仓库中的每个 span 平均只有 **426 bytes**；放在 Google 生产环境总体网络流量中，Dapper 采集只占**不到 0.01%**。 

### 6.5 对真实业务延迟的影响

论文用 web search cluster 测了不同采样率对平均延迟和吞吐的影响： 

| 采样频率 | 平均延迟变化 | 平均吞吐变化 |
|---|---:|---:|
| 1/1 | +16.3% | -1.48% |
| 1/2 | +9.40% | -0.73% |
| 1/4 | +6.38% | -0.30% |
| 1/8 | +4.12% | -0.23% |
| 1/16 | +2.12% | -0.08% |
| 1/1024 | -0.20% | -0.06% |

论文结论很明确：

- **如果不采样，全量 tracing 会对高吞吐在线服务造成可见延迟影响**；
- 当采样频率低于 **1/16** 时，惩罚已落入实验误差范围；
- 对高流量服务，**1/1024** 仍然能提供足够样本。 

### 6.6 数据量与时效性

Dapper 采集链路的**中位采集延迟小于 15 秒**；98 分位采集延迟有双峰现象，其中约 **75%** 的时候低于 **2 分钟**，但另有约 **25%** 的时候可能增长到数小时。 

在存储规模上，论文披露 Google 生产集群每天会产生**超过 1 TB 的 sampled trace data**。用户希望这些数据至少保留两周，因此 Dapper 的采样、写入吞吐和存储成本始终是系统设计的核心约束。 

---

## 7. 典型应用场景：Dapper 在 Google 里到底怎么用

Dapper 的强项不是“生成漂亮的调用图”，而是让不同角色都能基于同一份 trace 数据做决策。

### 7.1 开发期与重构期：性能、正确性、理解、测试

论文中最完整的案例是 Ads Review 团队。该团队在重写系统时，从原型阶段到上线和维护阶段都持续使用 Dapper。Dapper 帮助他们：

- **性能**：定位关键路径上的不必要串行请求；
- **正确性**：发现本应打到只读副本的查询错误地打到了主库；
- **系统理解**：统计跨 Bigtable、数据库、索引服务等下游依赖的总成本；
- **测试与 QA**：把 Dapper trace 用作新版本行为与性能回归检查的一部分。 

团队给出的结果非常激进：他们估计其服务的 latency numbers 用 Dapper 平台上的数据**改善了两个数量级（two orders of magnitude）**。 

### 7.2 长尾延迟分析（tail latency）

对于 Universal Search 这种多团队、多子系统参与的大型服务，经验再丰富的工程师也常常会猜错瓶颈位置。论文指出，Dapper 能为这类问题提供“事实”。团队基于 DAPI Trace objects 推断 hierarchical critical paths，用来分析尾延迟并排序优化优先级。 

论文列出的发现包括：

- 关键路径上的瞬时网络退化不会显著影响整体吞吐，但会明显拉高尾延迟；
- 很多昂贵查询模式来自服务之间的意外交互；
- 结合安全日志仓库与 Dapper trace id，可以为不同子系统构造慢查询样本集合。 

### 7.3 服务依赖分析

静态配置通常不足以刻画真实的动态依赖。Google 的 Service Dependencies 项目利用 Dapper 的 core instrumentation 和 annotations，自动推断 job 之间以及 job 对共享基础设施的依赖关系。论文举的例子是：Bigtable 操作会带上 table name annotation，因此团队可以自动推断到“命名资源粒度”的依赖。 

### 7.4 网络流量归因

论文特别强调一个“不是为它设计，但它居然很适合”的场景：**跨集群网络活动的应用层归因**。基于 Dapper，Google 很快构建了一个持续更新的控制台，用来展示最活跃的应用层 endpoint，并且能够把昂贵网络请求追溯到 causal trace root，而不只是看到两个对端机器。这个 dashboard 是在 Dapper API 之上**不到两周**搭出来的。 

### 7.5 分层和共享存储系统

在 App Engine → entity store → Bigtable → Chubby/GFS 这种多层共享基础设施里，只看底层资源很难回答“是谁在消耗资源”。Dapper 的 UI 可以跨客户端聚合共享服务上的 trace 性能信息，使共享服务 owner 能够按入站/出站网络负载或服务时间对用户排序，从而更快定位热点与争用源。 

### 7.6 线上救火（firefighting）

Dapper 对线上紧急问题并非万能，但对高延迟和超时服务很有帮助：UI 可以直接和 daemon 通信，在几秒内提取高延迟样本 trace，用于定位瓶颈位置。论文也坦承，对共享存储这类需要“快速聚合视图”的场景，如果 bulk analysis 还做不到事件发生后 10 分钟内完成，Dapper 的救火价值就会打折扣。 

---

## 8. 优势与局限：应该怎样评价 Dapper

### 8.1 主要优势

**第一，覆盖率和稳定性优先。** Dapper 通过把 instrumentation 收敛到公共基础库，实现了极高覆盖率与很低维护成本。这是它能长期在线、真正进入生产核心路径的前提。 

**第二，数据模型极简但足够强。** Trace/Span/Annotation 的抽象没有过度设计，但足以支撑延迟分析、关键路径分析、依赖推断、资源归因和异常上下文关联。 

**第三，性能工程做得扎实。** 从纳秒级 runtime 开销、daemon 的低 CPU 占用、按 trace 采样，到带外采集和分层存储，Dapper 从设计之初就在为“永远在线”做准备。 

**第四，平台化能力强。** DAPI、MapReduce 接口、交互式 UI、实时 trace 拉取，说明 Dapper 不是单一产品，而是一套生态底座。论文中很多成功场景都来自“Dapper 团队没预想到”的二次开发。 

### 8.2 主要局限

**第一，对环境同构性有依赖。** 论文明确承认，Dapper 之所以能做到高透明度，部分原因是 Google 的线程模型、控制流库和 RPC 框架高度统一。换到异构性更强的组织，复制这套透明度会更难。 

**第二，对非标准控制流需要补丁。** 使用非标准 control-flow primitive、未被 instrumentation 的通信库（如 raw TCP、SOAP RPC）时，Dapper 可能无法正确跟踪控制路径，需要手工 propagation 或额外接入。 

**第三，能指出“哪里慢”，但未必自动回答“为什么慢”。** 论文在 lessons learned 中明确提到：排队、负载、批处理、内核级事件等因素，可能使某个请求看起来“为一大块工作背锅”。因此 Dapper 常常是定位入口，而不是全部答案。 

**第四，对批处理/合并执行场景表达不够好。** 论文提到 coalescing effects：如果多个请求被合并后一起处理，单一 trace id 模型可能把这段工作错误归因到某一个请求上。对于离线批处理工作负载，也需要重新定义“trace 的根对象”是什么。 

**第五，隐私与诊断能力存在张力。** Dapper 不记录 payload，这有利于安全，但也意味着某些依赖 payload 模式的分析做不了，必须借助 opt-in annotations 或其他系统补充。 

---

## 9. 今天再看 Dapper：它留下了什么方法论

Dapper 不只是“Google 2010 年的一篇经典论文”，它实际上定义了现代 distributed tracing 的主干方法：**上下文传播 + span 树 + 采样 + 集中后端 + 可视化分析**。Google Cloud 在 2017 年的官方博客中明确写到，Dapper 论文启发了 Zipkin 等开源项目，Dapper-style tracing 已经成为行业标准的一部分；2018 年 Google 又表示其对外 Trace 产品建立在 Dapper 之上；到 2021 年，Google 官方进一步把 Dapper 论文和后来的 OpenCensus / OpenTelemetry 演进链条连接起来。    

如果把 Dapper 与今天的 OpenTelemetry 放在一起看，会发现很多思想是一脉相承的：

- **自动 instrumentation 与手工补充 instrumentation 并存**；
- **context propagation 是一等公民**；
- **统一语义数据模型比单点工具更重要**；
- **边采边传的低成本路径和中心化分析面必须分离**；
- **tracing 必须和规模、存储、隐私一起设计，而不是事后外挂**。   

换句话说，Dapper 最值得学习的，不是某个 Google 私有实现细节，而是它的工程哲学：

> **在大规模生产系统中，最有价值的 tracing 方案不是“理论上最完整”的方案，而是“覆盖率高、成本低、默认在线、还能支持二次分析”的方案。**    

---

## 10. 总结

Dapper 的设计之所以经典，不在于它把 distributed tracing 讲得多复杂，而在于它证明了三件事：

1. **Tracing 可以成为生产系统的常驻基础设施，而不只是临时调试工具。** 
2. **只要 instrumentation 点选得对，应用透明性和系统级覆盖率可以兼得。** 
3. **Trace 数据一旦沉淀成平台能力，价值会远超“看调用链”本身。** 

如果要用一句话概括 Dapper，可以这样说：

**Dapper 不是“把一次请求画出来”的工具，而是把分布式系统中的执行事实沉淀成统一可分析数据面的工程体系。**    

