# DeathStarBench（ASPLOS 2019）论文解读：An Open-Source Benchmark Suite for Cloud and IoT Microservices

> 论文原题在不同版本中略有差异：arXiv 页面题为 **“An Open-Source Benchmark Suite for Cloud and IoT Microservices”**，ASPLOS 2019 论文版本题为 **“An Open-Source Benchmark Suite for Microservices and Their Hardware-Software Implications for Cloud & Edge Systems”**。本文统一称为 **DeathStarBench**。

## 0. 基本信息

| 项目 | 内容 |
|---|---|
| 论文 | *An Open-Source Benchmark Suite for Microservices and Their Hardware-Software Implications for Cloud & Edge Systems* |
| 会议 | ASPLOS 2019 |
| 作者 | Yu Gan, Yanqi Zhang, Dailun Cheng, Ankitha Shetty, Priyal Rathi, Nayan Katarki, Ariana Bruno, Justin Hu, Brian Ritchken, Brendon Jackson, Kelvin Hu, Meghna Pancholi, Yuan He, Brett Clancy, Chris Colen, Fukang Wen, Catherine Leung, Siyuan Wang, Leon Zaruvinsky, Mateo Espinosa, Rick Lin, Zhongling Liu, Jake Padilla, Christina Delimitrou |
| 开源项目 | DeathStarBench |
| 核心关键词 | microservices, cloud computing, edge systems, QoS, tail latency, cluster management, hardware acceleration, serverless, distributed tracing |

## 1. 一句话总结

这篇论文提出并开源了 **DeathStarBench**：一组由真实风格的端到端微服务应用组成的 benchmark suite，用于研究微服务架构对数据中心硬件、操作系统、网络栈、集群调度、编程框架和尾延迟的系统性影响。它的关键价值不只是“提供几个微服务 demo”，而是把微服务的核心复杂性——**服务依赖图、跨服务 RPC/REST 调用、语言/框架异构、端到端 QoS、级联瓶颈和 tail-at-scale 效应**——作为可复现实验对象暴露出来。

## 2. 论文背景：为什么需要 DeathStarBench

论文的出发点是：云服务正在从单体应用转向由数十到数百个 loosely-coupled microservices 组成的调用图。传统 benchmark 往往关注单层服务、两三层 web 服务、数据库、key-value store 或 batch workload，很难捕捉现代微服务系统中的几个关键问题：

1. **端到端延迟由依赖图决定**：一个用户请求会经过前端、缓存、数据库、推荐、认证、存储等多个服务；任何关键路径上的慢服务都会影响端到端 QoS。
2. **瓶颈会沿服务依赖传播**：一个后端服务轻微变慢，可能使上游服务排队、busy waiting 或饱和，形成 backpressure 和 cascading QoS violation。
3. **网络和 RPC 开销不再是边角成本**：微服务把原本进程内函数调用变成大量跨进程/跨机器 RPC 或 REST 调用，网络处理、kernel path、序列化、连接管理会成为重要开销。
4. **调度器不能只看单个容器 CPU 使用率**：微服务之间存在依赖；CPU 高的服务未必是真正 culprit，CPU 正常的下游服务也可能通过连接阻塞或排队拖垮上游。
5. **尾延迟问题被放大**：系统规模越大，服务图越复杂，请求 skew、慢服务器、网络抖动、错误配置对端到端 P99/P999 延迟的影响越严重。

因此，论文要解决的不是单个微服务性能测试问题，而是构造一个能够研究 **“微服务系统整体行为”** 的开放 benchmark。

## 3. DeathStarBench 的核心内容

### 3.1 Benchmark suite 的设计目标

论文提出的 DeathStarBench 遵循以下设计原则：

| 设计原则 | 含义 |
|---|---|
| Representativeness | 使用 NGINX、memcached、MongoDB、RabbitMQ、MySQL、Apache HTTP Server、Cylon、SockShop 等常见开源组件，模拟生产云服务常见软件栈。 |
| End-to-end operation | 不只测单个服务，而是覆盖从客户端请求进入、跨多个微服务处理、访问后端存储、再返回客户端的完整路径。 |
| Heterogeneity | 应用由 C/C++、Java、JavaScript、node.js、Python、Go、Scala、Ruby、PHP 等多种语言和框架组成，反映真实微服务的异构性。 |
| Modularity | 每个微服务尽量职责单一、松耦合，服务图接近真实组织结构和系统边界。 |
| Reconfigurability | 通过 RPC/HTTP API 使单个组件可以替换、重写或重新部署，便于研究不同框架、协议、调度策略和硬件平台。 |

### 3.2 论文版本中的主要服务

论文中的 suite 覆盖云端和边缘/IoT 场景，核心服务包括：

| 服务 | 场景 | 特点 |
|---|---|---|
| Social Network | 类 Twitter/Facebook 的社交网络 | 发帖、读 timeline、follow/unfollow、媒体上传、推荐、广告、搜索、用户关系图、缓存和持久化存储。 |
| Media Service / Movie Reviewing | 电影评论、评分、租赁、流媒体 | 登录、评论、评分、电影信息、推荐、视频流、NFS、memcached、MongoDB、MySQL。 |
| E-commerce Website | 电商网站 | 浏览商品、购物车、下单、支付、配送、发票、推荐、wishlist。 |
| Banking System | 银行业务系统 | 支付、信用卡、贷款、认证、银行信息、缓存和持久化存储。 |
| Swarm Cloud / Swarm Edge | 无人机群协调 | 研究 cloud-edge 划分：图像识别、避障、运动控制等计算在云端或边缘设备上执行。 |

论文表 1 显示，每个应用包含二十到四十多个 unique microservices，并且服务之间使用 RPC、REST 或 HTTP 通信。这个规模比传统单层/少数几层 benchmark 更接近真实微服务系统。

> 需要注意：当前 GitHub 仓库中的服务集合和论文当年的表述已有演进。当前仓库页面列出了 Social Network、Media Service、Hotel Reservation 等 released 服务，同时 E-commerce、Banking、Drone coordination 等条目仍显示为 in progress 或未完全 onboarded。写实验报告时应区分“ASPLOS 2019 论文版 suite”和“当前仓库实现”。

### 3.3 方法论：如何观测微服务系统

论文强调：微服务不能只靠客户端端到端延迟来定位问题。因为一个请求在服务图中会经过多个 RPC/REST 边，必须知道每个边、每个服务的 latency、queueing 和 resource state。

为此，作者实现了轻量级 distributed tracing：

- 在 RPC granularity 记录请求进入和离开每个 microservice 的时间。
- 将同一端到端请求内的多个 RPC 关联起来。
- 通过 Trace Collector 收集数据，并存储到集中式数据库。
- 论文报告 tracing 对端到端延迟的开销低于 0.1%，因此可用于性能剖析。

这一点后来成为 DeathStarBench 被 AIOps、root cause analysis、microservice resource management 等研究反复使用的重要原因：它天然适合生成服务图、调用链、延迟分解和故障传播数据。

## 4. 论文的核心实验与主要发现

### 4.1 硬件层：微服务更依赖可预测的单线程性能

论文发现，虽然单个微服务通常计算量不大，但每个服务 tier 的 latency budget 很小，因此端到端 QoS 对单线程性能、frequency scaling、cache 行为和硬件抖动非常敏感。

重要结论包括：

- 低功耗核心或频率较低的平台在低负载下可能能满足 QoS，但更早达到饱和。
- 对关键路径上的服务，predictably high single-thread performance 比单纯堆很多低功耗核心更重要。
- 低功耗机器仍可能适合放置 off critical path 或对频率不敏感的微服务。

这对数据中心硬件设计和调度有直接启示：微服务并不等于“每个服务都很轻，所以可以随便放到低性能核上”。关键路径位置必须被考虑。

### 4.2 OS 和网络层：RPC/REST 网络处理成为核心开销

论文对比了单层服务和端到端微服务应用中的网络处理开销。以 Social Network 为例，论文指出单层 NGINX、memcached、MongoDB 中网络处理只占较小部分，而在端到端 Social Network 微服务中，网络处理占总执行时间的 36.3%。

这说明：

- 微服务把原本应用内部的调用变成大量跨进程/跨机器通信。
- kernel networking、TCP packet processing、interrupt handling、RPC 序列化和 runtime library 开销显著上升。
- 高负载下 NIC queueing 和网络路径抖动会放大 tail latency。

论文还评估了 FPGA 形式的网络/RPC 加速：将 TCP/network processing offload 到 FPGA 后，网络处理 latency 相比 native TCP 提升 10–68x，端到端 tail latency 提升 43% 到最高 2.2x。这一实验不是说“所有云都应直接使用 FPGA”，而是说明微服务架构让网络栈从辅助成本变成主瓶颈，硬件/软件协同优化有实质空间。

### 4.3 集群管理层：依赖图导致 backpressure 和级联 QoS violation

论文最重要的系统发现之一是：微服务调度不能只看每个服务的局部 resource utilization。

典型例子是 nginx + memcached 两层结构：

- 情况 A：nginx 真正饱和，扩容 nginx 可以解决问题。
- 情况 B：memcached 下游服务造成连接阻塞或响应变慢，nginx 因等待下游返回而看起来“忙”，但真正 culprit 并不是 nginx。此时如果 autoscaler 只看 nginx CPU 或 latency 并扩容 nginx，可能不仅不能解决问题，还会放入更多请求，使系统更糟。

论文进一步指出，在 Social Network 这种复杂服务图中，误管理单个依赖关系就可能显著伤害 tail latency。例如论文在导言中报告，错误管理一个依赖可使 Social Network 的 tail latency 恶化 10.4x，并且恢复时间长于对应的单体服务。

这直接推动了后续大量研究：QoS-aware scheduling、dependency-aware resource management、root cause localization、AIOps fault diagnosis 等。

### 4.4 应用与编程框架层：瓶颈随负载和服务实现变化

论文使用 tracing 和 VTune 分析每个服务在不同负载下的 latency breakdown，发现：

- 低负载下，Social Network 和 Media Service 的延迟主要由前端 nginx 主导。
- 高负载下，瓶颈转移到后端数据库和管理数据库的微服务，例如 writeGraph 等。
- E-commerce 和 Banking 中一些高层语言服务（如 node.js、Go）以及订单、支付、认证等业务逻辑服务会成为主要延迟来源。
- 同一个微服务架构的瓶颈不是静态的，会随负载、请求类型、数据访问模式、语言实现和同步机制变化。

这说明静态 profiling 或一次性压测不足以指导微服务容量规划。系统需要动态 tracing、动态瓶颈定位和持续资源调整。

### 4.5 Serverless 层：弹性好，但状态传递和远端存储可能拖慢端到端延迟

论文还将一些微服务移植到 AWS Lambda 类 serverless framework 上，比较 EC2 容器部署和 Lambda 函数化部署。

主要结论：

- Serverless 对间歇性负载和快速弹性扩容有吸引力。
- 但如果依赖 S3 等远端持久化存储在函数之间传递中间状态，延迟会显著上升。
- 使用远端内存替代 S3 能降低一部分 overhead。
- 在日周期负载中，Lambda 能比 EC2 autoscaling 更快响应负载上升，但性能抖动和数据传递开销仍是问题。

论文的判断比较平衡：serverless 不是自动解决微服务问题的银弹；要发挥潜力，需要服务尽量 stateless，并使用高效的内存级状态传递机制。

### 4.6 Tail-at-scale：规模越大，慢服务和请求 skew 越致命

论文用 Social Network 的真实用户流量和 EC2 大规模部署研究 tail-at-scale：

- Social Network 有数百注册用户，平均 165 个 daily active users。
- 为扩展规模，作者在 EC2 上部署 40 到 200 个 c5.18xlarge 实例。
- 请求 skew 很常见：真实流量中约 5% 用户贡献超过 30% 请求。
- 当更少用户承担多数请求时，goodput 会快速下降；论文报告当少于 20% 用户贡献大多数请求时，goodput 几乎为 0。
- 慢服务器在复杂服务图中影响被放大：如果慢服务器承载了关键路径服务，端到端 QoS 会显著恶化。

这部分贡献在于把“tail latency”从单机/单服务层面推进到“微服务依赖图 + 大规模部署 + 真实用户流量”的层面。

## 5. 核心贡献总结

### 贡献 1：提出并开源了端到端微服务 benchmark suite

DeathStarBench 提供多个端到端服务，而不是孤立的单服务 benchmark。它让研究者能复现真实微服务系统的关键问题：调用图、依赖传播、跨语言组件、缓存和数据库交互、RPC/REST 通信、tail latency 和 QoS violation。

### 贡献 2：构造了跨系统栈的微服务分析框架

论文不是只评测应用吞吐量，而是从以下层面系统分析微服务：

- datacenter hardware；
- OS/kernel/networking；
- RPC/REST 和硬件加速；
- cluster management；
- serverless framework；
- edge/cloud 划分；
- tail-at-scale。

这种跨层分析使 DeathStarBench 不只是 benchmark，也成为研究微服务系统设计的一套实验方法。

### 贡献 3：揭示了网络/RPC 开销在微服务中的核心地位

论文量化了微服务中网络处理占比显著升高，并证明网络/RPC 加速能显著改善 tail latency。这个结论影响了后续关于 RPC acceleration、kernel bypass、SmartNIC/FPGA、service mesh 性能和低延迟网络栈的研究方向。

### 贡献 4：证明传统 autoscaling 和 cluster management 对微服务不够表达

论文表明：微服务中的服务依赖导致 backpressure 和 cascading QoS violation。单纯根据 CPU utilization 或单服务 latency 的扩容策略，可能定位错 culprit，甚至加剧问题。

这为后续 Sinan、Cloverleaf、Seer、Sage、Ursa 等 dependency-aware 或 ML-driven 微服务管理系统提供了实验动机和 benchmark 基础。

### 贡献 5：量化了微服务中的 tail-at-scale 效应

论文使用真实用户流量和 EC2 大规模部署，展示请求 skew、慢服务器、路由错误、关键路径服务饱和等因素如何在微服务图中被放大。这使“微服务可靠性/可预测性”成为一个可实验验证的问题，而不只是工程经验。

## 6. 核心创新点

### 6.1 从 component benchmark 转向 service-graph benchmark

传统 benchmark 多数测试单个组件：web server、cache、database、search、ML inference 等。DeathStarBench 的创新在于把 benchmark 单位变成完整服务图，即 **microservice dependency graph**。

这使它可以研究以下传统 benchmark 难以捕获的问题：

- 跨服务 queueing；
- fan-out/fan-in；
- 请求关键路径；
- downstream bottleneck 造成 upstream saturation；
- 数据库/缓存共享导致的跨请求干扰；
- 服务图复杂度对 tail latency 的影响。

### 6.2 将异构语言和真实中间件纳入 benchmark

论文没有把所有服务写成统一语言的 toy service，而是使用多语言、多框架、多中间件组合。这个选择提高了 benchmark 的复杂度，但也使其更接近真实云服务。

### 6.3 内置 distributed tracing 思维

DeathStarBench 不是简单压测脚本集合，而是强调 per-RPC/per-service tracing。这个设计让它天然适合后续 AIOps、RCA、故障注入、资源管理和性能调试研究。

### 6.4 同时覆盖 cloud 和 edge/IoT

Swarm 服务把无人机群协调引入 benchmark，比较 computation 在 edge 侧执行和 cloud 侧执行的取舍。2019 年这个方向已经预示了后来的 cloud-edge continuum、边缘推理、IoT swarm control 等研究问题。

### 6.5 用 benchmark 反推系统栈设计需求

论文的价值不是“我做了几个应用”，而是用这些应用证明：硬件、网络栈、OS、scheduler、serverless runtime 和 observability stack 都需要为微服务演进。

## 7. 现在真实具体应用 DeathStarBench 的情况

下面的“应用”指真实研究、工业文档、开源派生项目或平台集成。它们大多不是把 DeathStarBench 当作生产业务系统直接上线，而是把它作为 **可复现微服务系统、压力测试对象、资源管理评估对象、故障注入靶场或平台 benchmark workload**。

### 7.1 原始开源项目：持续作为通用微服务 benchmark

DeathStarBench 的 GitHub 仓库仍作为公共 benchmark 使用。当前仓库页面显示其包含 Social Network、Media Service、Hotel Reservation 等已发布服务，并列出 E-commerce、Banking、Drone coordination 等条目。页面还显示项目有数百 stars 和 forks。

Christina Delimitrou 的研究页面进一步说明，DeathStarBench 论文获得 IEEE Micro Top Pick 2020，软件在 GitHub 上有超过 20,000 次 unique clones，并被数十个研究组和若干云服务提供商使用。

**实际用途**：

- 作为微服务系统研究共同基准；
- 作为 reproducible research 的公共参考点；
- 作为教学、实验室课程、系统性能分析和容器/Kubernetes 部署练习对象；
- 作为后续研究系统的被测应用。

### 7.2 Microsoft Virtual Client：集成为云平台 workload

Microsoft 的 Virtual Client 文档将 DeathStarBench 作为 workload 集成。文档列出 Social Network、Media Service、Hotel Reservation 三个已集成端到端服务，并说明 Virtual Client 使用 DeathStarBench 研究微服务的网络/OS、cluster management 和 application/framework trade-off。该 workload 会输出不同百分位网络延迟、请求率、传输速率等指标。

**实际用途**：

- 云基础设施 benchmark；
- VM/裸金属/容器平台性能回归；
- 不同平台或配置下微服务网络通信 latency/throughput 对比；
- 自动化 workload orchestration 和指标采集。

### 7.3 Ampere Computing：将 Social Network 移植和调优到 AArch64/Arm 平台

Ampere Computing 发布了 **DSB Social Network - Transition and Tuning Guide**。文档说明原始 DeathStarBench Social Network 使用 x86 Docker images，若部署到 Ampere Altra 或其他 Arm-based system，需要为 AArch64 重新构建镜像；Ampere 提供了 arm64-port，并给出在裸金属 Kubernetes 集群上启动和运行 Social Network workload 的步骤。

**实际用途**：

- 评估 Arm server 上微服务 workload 性能；
- 移植 x86 容器镜像到 AArch64；
- 调整 Kubernetes/Helm chart、容器镜像、依赖服务；
- 为云服务器芯片/平台厂商提供真实风格微服务压测样例。

### 7.4 Sinan（ASPLOS 2021）：QoS-aware 微服务资源管理

Sinan 是 Cornell/Delimitrou 组后续的 ML-based resource manager。它直接使用 DeathStarBench 中的 Social Network 和 Hotel Reservation 作为评估对象，并在本地集群和 Google Compute Engine 上评估。Sinan 的 artifact appendix 明确说明其程序是 modified version of DeathStarBench suite，重点关注 Social Network 和 Hotel Reservation。

**实际用途**：

- 训练和评估 dependency-aware resource allocation 模型；
- 研究如何在满足 end-to-end tail latency QoS 的同时减少资源使用；
- 对比传统 autoscaling 和 queueing-based scheduler；
- 作为 ML-for-systems 在微服务管理中的代表性基准。

### 7.5 AIOpsLab（2025）：用于评估 LLM-based AIOps agents

AIOpsLab 是一个用于评估 autonomous cloud / AgentOps 的框架。论文说明 AIOpsLab 使用 DeathStarBench 的两个应用作为 testbeds，并结合其 workload generators；后续章节明确当前集成 HotelReservation 和 SocialNetwork。AIOpsLab 还集成 Kubernetes、Helm、ChaosMesh、Prometheus、Jaeger、Filebeat、Logstash 等，用于故障注入、telemetry 导出和 LLM agent 交互式诊断。

**实际用途**：

- 构造真实微服务事故场景；
- 评估 LLM agent 的 anomaly detection、fault localization、root cause analysis 和 mitigation 能力；
- 生成动态 telemetry，包括 metrics、traces、logs、配置和集群信息；
- 将 DeathStarBench 从性能 benchmark 扩展为 AIOps/AgentOps 靶场。

### 7.6 ChaosStarBench：基于 DeathStarBench 的故障注入 benchmark

ChaosStarBench 是 University of Sussex 相关团队的开源派生项目。其 README 说明该项目 adapted from original DeathStarBench，用于 cloud microservices fault experiments。当前主要支持 Social Network 的 fault injection，并计划扩展到其他服务。

**实际用途**：

- chaos engineering 实验；
- microservice fault injection；
- 故障传播、RCA、resilience testing；
- 在 DeathStarBench 的服务图基础上增加故障实验能力。

### 7.7 CC-DeathStarBench：迁移到 Intel SGX / TEE 的 confidential computing 场景

CC-DeathStarBench 是 DeathStarBench 的一个 fork，目标是将微服务迁移到 trusted execution environments（TEEs），使用 Intel SGX。项目说明目前主要将 Hotel Reservation benchmark 做了 confidential-computing 化。

**实际用途**：

- 评估 confidential computing 对微服务架构的性能和部署影响；
- 研究 SGX/TEE 下的微服务隔离、信任边界和 overhead；
- 为安全云计算实验提供端到端微服务应用。

### 7.8 Efficient Asynchronous RPC Calls for Microservices：研究 DeathStarBench RPC 实现瓶颈

2022 年论文 *Efficient Asynchronous RPC Calls for Microservices: DeathStarBench Study* 专门研究 DeathStarBench 中异步 RPC 调用实现。作者发现 DeathStarBench 的 threaded asynchronous call 实现会造成 thread management bottleneck，降低 peak throughput 并提高高请求率下的 latency；将 threaded implementation 替换为 fiber implementation 后，peak throughput 最高可提升 6x。

**实际用途**：

- 将 DeathStarBench 用作 RPC runtime 和并发模型优化对象；
- 说明 benchmark 本身也能暴露微服务框架/库实现问题；
- 为 fiber/coroutine-based RPC runtime 的设计提供实验依据。

### 7.9 Ursa / Dapr-based reimplementation：把 DeathStarBench 扩展到现代 cloud-native 通信模式

2024 年 *Analytically-Driven Resource Management for Cloud-Native Microservices* 提出 Ursa。该工作认为传统 DeathStarBench 主要使用 RPC/HTTP，现代 cloud-native 应用还大量使用 message queues、不同请求等级和更复杂业务逻辑。因此，作者使用 Dapr 重新实现多个 DeathStarBench 应用，加入 RPC、Redis streams/message queues、图像处理、情感分析、object detection、视频处理等逻辑。

**实际用途**：

- 将 DeathStarBench 升级为更现代的 cloud-native benchmark；
- 评估 RPC 与 MQ 下 backpressure 传播差异；
- 支持多请求类别、多 SLA、多优先级；
- 研究 resource management 如何适应微服务业务逻辑变化。

## 8. 对论文的批判性评价

### 8.1 优点

1. **问题选得准**：论文抓住了微服务区别于传统多层服务的本质：依赖图、级联、tail latency、网络/RPC 开销。
2. **benchmark 有复用价值**：它不是一次性实验代码，而是长期被后续研究、工业文档和平台集成反复使用。
3. **跨层视角完整**：从硬件、OS、网络、调度、serverless 到 tail-at-scale，覆盖面非常系统。
4. **影响力强**：后续大量微服务资源管理、AIOps、故障注入、RPC 优化、安全计算研究都使用或派生自 DeathStarBench。
5. **与真实系统足够接近**：多语言、多中间件、缓存、数据库、RPC/REST、Docker/Kubernetes 部署方式，使其比 toy microservice 更有研究价值。

### 8.2 局限

1. **不等同于真实生产服务**：DeathStarBench 是生产风格 benchmark，不是 Amazon/Netflix/Twitter 的真实内部系统。其服务图、数据规模、流量分布和故障模式仍是可控实验环境。
2. **当前仓库与论文版本存在演进差异**：论文中描述的服务和当前 GitHub released 服务并不完全一致，复现实验时要确认使用的 commit、branch、deployment method 和 workload generator。
3. **现代微服务生态已变化**：2019 年论文主要聚焦 RPC/REST、Docker、Thrift/gRPC、Lambda 等。今天常见的 service mesh、OpenTelemetry、eBPF observability、Kafka/Pulsar/Redis Streams、Dapr、sidecarless mesh、WASM plugin 等并未被原始 benchmark 完整覆盖。
4. **部署复杂度较高**：多服务、多语言、多依赖、多数据库使 benchmark 真实，但也提高了复现门槛。
5. **业务逻辑仍偏 benchmark 化**：虽然比 toy service 强很多，但有些服务逻辑和数据规模仍不能完全代表复杂企业生产负载。

### 8.3 现在使用这篇论文时的建议

如果你要基于 DeathStarBench 做实验或项目，建议：

1. **明确你使用的版本**：论文版、GitHub master、某个 fork、Microsoft VC 版本、Ampere Arm port、AIOpsLab 集成版或 Dapr reimplementation。
2. **记录 deployment substrate**：Docker Compose、Kubernetes、Helm、裸金属、VM、公有云实例类型、CPU pinning、网络拓扑都要写清楚。
3. **不要只看平均延迟**：应报告 P95/P99/P999、goodput under QoS、tail latency violation rate、per-service latency breakdown。
4. **结合 tracing/metrics/logs**：至少使用 Jaeger/OpenTelemetry、Prometheus、service-level logs；若研究内核/网络，也可以结合 eBPF 或 perf。
5. **引入故障和动态负载**：静态 QPS 压测不足以体现 DeathStarBench 价值。更有意义的是请求 skew、服务 CPU throttling、network delay、pod restart、配置错误、DB slowdown、cache miss storm 等场景。
6. **区分 culprit 和 symptom**：资源高的服务未必是根因；应分析依赖图上的 backpressure 传播。
7. **对现代系统可做扩展**：可加入 service mesh、MQ、OpenTelemetry、Dapr、LLM/AIOps agent、eBPF tracing 或 confidential computing 作为现代研究方向。

## 9. 如果用这篇论文做课程/项目，可以选哪些方向

下面是一些可落地的项目方向：

| 方向 | 可做内容 |
|---|---|
| 微服务性能基线 | 部署 Social Network 或 Hotel Reservation，使用 wrk2/Locust 压测，记录吞吐、P99 延迟、服务级 latency breakdown。 |
| eBPF observability | 用 eBPF 监控 TCP connect、send/recv、syscall latency、container-level network latency，并与 Jaeger trace 对齐。 |
| 故障注入 | 使用 Chaos Mesh 或 tc/netem 注入 CPU throttling、network delay、packet loss、pod kill，观察 QoS violation 传播。 |
| 自动扩缩容 | 比较 Kubernetes HPA、custom autoscaler、dependency-aware autoscaler 在动态负载下的表现。 |
| RCA / AIOps | 收集 metrics/traces/logs，训练或实现根因定位算法；也可把 LLM agent 接入日志和指标接口做诊断。 |
| 网络优化 | 对比 RPC/REST、连接池、异步 RPC、fiber/coroutine、keepalive、sidecar proxy 对 tail latency 的影响。 |
| 架构移植 | 将 x86 Docker image 移植到 ARM64，或移植到 containerd/K8s/service mesh/Dapr。 |
| 安全与隔离 | 将关键服务迁移到 TEE、gVisor、Kata Containers 或 seccomp/AppArmor，评估 overhead。 |

## 10. 结论

DeathStarBench 的核心价值在于：它把“微服务系统”作为一个可以被严肃实验研究的对象，而不是只给出若干 demo 服务。论文证明，微服务会系统性改变云平台的瓶颈结构：网络/RPC 开销上升，kernel 和 library overhead 变重，依赖图导致 backpressure 和 cascading QoS violation，传统 autoscaling 定位能力不足，tail-at-scale 效应被放大，serverless 的弹性与状态传递开销之间存在明显权衡。

从今天看，DeathStarBench 仍然是云原生系统研究中非常重要的 benchmark 基础。它已经被用于云平台性能测试、Arm 服务器移植、ML-based resource management、AIOps agent 评测、chaos engineering、RPC runtime 优化和 confidential computing 等多个方向。它的不足也很清楚：原始版本不能完全覆盖今天的 service mesh、event-driven microservices、OpenTelemetry/eBPF observability 和更复杂的业务逻辑。但这恰恰说明它适合作为“基础实验平台”，在其上继续扩展现代云原生组件。
