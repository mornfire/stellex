# Stellex：基于 NeoForge 21.1.251 源码的可控加载器内核与运行时治理

作者：Mornfrost Office

## 摘要

模组加载器一旦被替换，第三方模组的二进制契约就会失效，这是长期困扰 Minecraft 生态的工程约束。本文报告 Stellex 的设计、实现与验证：它是一个独立加载器，以 NeoForge 21.1.251 源码为基座、保留全部包名与二进制接口，并在内核层新增三项能力：可配置的 CPU 调度器、面向内存泄漏的运行时治理子系统，以及客户端渲染剔除。为说明形态选择的必要性，本文先分析加载器替换与模组兼容之间的互斥关系，再给出构建与发行流水线的完整实现，包括源码分支的构建入口、NeoForm 源码铺陈步骤、安装器品牌化与独立下载器。实验部分报告了构建耗时、服务端启动耗时、调度器与内存治理的运行期状态，以及两项客户端兼容性验证：Stellex 客户端加载真实 NeoForge 模组 SQDCLib 3.0.1 的情况，以及纯官方 NeoForge 21.1.251 客户端与 Stellex 客户端分别连接 Stellex 服务端的握手结果。结果表明，在同一 ABI 下新增内核能力不会破坏模组加载；模组通道的必需性仍按 NeoForge 原有规则校验；加载器内的调度器与内存治理可以在不安装任何模组的情况下工作。

关键词：NeoForge；独立加载器；二进制兼容；CPU 调度；内存泄漏治理；安装器

## 1 引言

### 1.1 背景

Minecraft 的模组生态建立在加载器提供的契约之上。模组通过 `neoforge.mods.toml` 声明依赖，通过 Mixin 在类加载阶段改写字节码，通过访问转换器访问原本不可见的成员，通过事件总线接入生命周期。这些目标全部以类名、方法名、描述符的形式写死在第三方字节码里，构成本文所说的二进制契约。加载器一旦改变这些目标，模组就会在启动、注入或运行期失败，而且失败点往往与改动位置相距很远。

性能与稳定性是另一条主线。服务端在空闲时希望利用闲置算力做地形预生成，客户端在大型整合包中希望减少不可见对象的渲染开销，长期运行的服务器则希望内存增长可以被观测和遏制。传统做法是把这些能力做成模组：安装灵活，但需要玩家或服主自行选择，且无法覆盖加载器自身的运行时行为。Stellex 的出发点是把这些能力放进加载器内核，同时不破坏模组契约。

### 1.2 问题与约束

把需求写成工程约束，可以得到六条：其一，以 NeoForge 1.21.1 系列的 21.1.251 为基座；其二，保留 NeoForge 的其余实现；其三，完全兼容 NeoForge 模组；其四，兼容经 Sinytra Connector 转译的 Fabric 模组；其五，新增可调 CPU 占用与内存治理能力，且不得空转；其六，必须可编译，并保证客户端与服务端双向可进入。这六条约束共同指向一个结论：新增能力必须落在“内核模块”这一层，而不是“从零重写内核”或“外挂模组”这两个极端。

### 1.3 贡献

本文的贡献可以归纳为四点。第一，给出以 NeoForge 源码为基座的独立加载器方案，保留包名与 ABI，只增加内核模块与发行品牌，并说明这一形态与兼容性目标的一致性。第二，实现内核级 CPU 调度器，用区块票据驱动真实生成任务，按 JVM 参数控制目标占用比例，并在 tick 压力下自动退让。第三，实现内存治理子系统，按堆压力分级执行清理、检测 GC 后的持续增长、提供堆转储与运行期命令。第四，给出可复现的构建与发行流水线，并用真实客户端与真实 NeoForge 模组完成兼容性验证。

## 2 相关工作

### 2.1 加载器与兼容层

NeoForge 在 Forge 的基础上重构了事件系统、模组发现与运行时映射，21.1.x 系列对应 Minecraft 1.21.1。它的源码分支包含补丁目录、构建插件与安装器生成逻辑，官方构建流程依赖 NeoForm 完成 Minecraft 的反编译、补丁应用与重编译。Fabric 走的是另一条路线，以轻量注入与运行时映射为核心；Sinytra Connector 把 Fabric 模组的字节码映射到 NeoForge 与 Minecraft 的官方名，因此对 NeoForge 内部结构有强依赖。

对独立加载器而言，最重要的不是“能否编译”，而是“编译出来的运行时对第三方是否等价”。等价性由四层构成：类与成员的名称、Mixin 可注入面、事件触发时序、模组元数据与网络握手。任何一层发生偏移，都会在第三方模组上表现为难以归因的失败。Stellex 的所有改动都围绕“不触碰这四层”展开。

### 2.2 性能工具

服务端侧已有 Chunky 一类预生成工具，它们遍历区块并阻塞等待生成结果，缺少对进程 CPU 比例与 tick 预算的显式控制。客户端侧已有 Sodium、EntityCulling、Lithium 等重写或剔除型优化，收益显著但兼容面较窄。内存侧常见做法是使用 Spark、JFR 或堆转储做离线分析，缺少加载器层面的压力响应与自动遏制。

Stellex 与上述工作的关系是互补的：它不替换第三方优化，也不替代性能分析工具，而是在加载器内核提供一层可控的运行时治理。调度器负责“把空闲算力用于真实工作”，内存治理负责“在压力下做可安全执行的清理并留下取证入口”，客户端剔除负责“在渲染调用点减少不可见对象的开销”。

### 2.3 研究缺口

已有工作大致落在两个极端：重写型优化收益高但兼容成本高，独立工具型方案介入浅但缺少与 tick、CPU、堆压力的联动。两端之间缺少的是“以最小介入获得可控收益”的中间形态，而且这种中间形态必须放在加载器层，才能覆盖“没有安装任何模组”的场景。本文把 Stellex 定位在这一位置，并用真实客户端与真实模组验证其兼容边界。

## 3 形态选择与兼容性契约

### 3.1 为什么不做从零重写的加载器

从零重写的独立加载器意味着重新实现模组发现、依赖解析、类加载隔离、Mixin 管线、访问转换、事件时序与网络握手。要让现有 NeoForge 模组继续工作，这套实现必须与 NeoForge 逐项一致；要做到逐项一致，工程上等价于保留 NeoForge，而不是替换它。Connector 的存在进一步提高了门槛：它依赖 NeoForge 的官方名映射、FML 事件时序与模组元数据，一旦这些内部结构被替换，转译链路会整体失效。

因此 Stellex 选择基于源码基座的独立发行：以 NeoForge 21.1.251 源码为基座，保留 `net.neoforged`、`net.minecraft` 等全部包名，保留 Mixin 目标、访问转换器、事件触发点与模组元数据格式，只在源码树中新增 `net.neoforged.neoforge.stellex` 包，并在 `NeoForge` 核心类的静态初始化中挂接事件监听。对模组而言，它面对的仍是一个 NeoForge 21.1.251 运行时；对玩家而言，它看到的是 Stellex 品牌与新增命令。

### 3.2 版本与协议契约

加载器的版本字符串参与 FML 的握手与依赖校验。Stellex 将版本固定为 `21.1.251`，FML 版本保持 4.0.45，Minecraft 版本保持 1.21.1。这样模组元数据中的 `[21.1,)` 依赖仍然满足，客户端与服务端的版本握手也能对齐。内核没有注册任何必需的自定义网络通道；内存治理、CPU 调度与客户端剔除都是本地行为，不参与握手，因此“普通 NeoForge 客户端进入 Stellex 服务端”与“Stellex 客户端进入普通 NeoForge 服务端”在协议层没有被额外约束。

### 3.3 内核新增模块的边界

新增模块分为三类。服务端模块只使用公开的区块 API 与事件总线，不修改世界数据，不替换 tick 循环。内存模块只读取 JVM 的 MemoryMXBean 与 GarbageCollectorMXBean，清理动作限定在加载器自身可重建的记账数据上，堆转储通过标准 HotSpot 接口完成。客户端模块以 Mixin 形式挂在两个渲染调用点，并且只登记在 `neoforge.mixins.json` 的 `client` 数组，专用服务端不会加载它们。三类模块都不改变模组可见的 ABI。

### 3.4 对 Connector 的影响

Connector 的转译链依赖四个前提：官方名映射、FML 事件时序、模组元数据、Mixin 管线。Stellex 没有触碰前三个前提；对第四个前提，新增的 Mixin 只作用于客户端渲染类，并且只登记为客户端配置，不会改变 Connector 转译后的 Fabric 模组所依赖的服务端类形状。因此从契约上看，Connector 面对的运行时应与官方 NeoForge 等价。需要强调，这一结论是结构分析，本文没有在真实 Connector 环境中运行 Fabric 模组，实测留待后续工作。

## 4 构建与发行流水线

### 4.1 源码分支与工具链

Stellex 的源码来自 NeoForge 的 `1.21.1` 分支，浅克隆后共 3593 个文件。构建使用 JDK 21 与 Gradle 9.8.1，后者由官方发行包提供，通过镜像下载后在本地解压使用。目标网络无法直连 `maven.neoforged.net` 时，需要为 Gradle 配置 HTTP 代理；代理写入 `~/.gradle/gradle.properties` 的 `systemProp` 项后，依赖解析与插件下载恢复正常。

版本字符串由根构建脚本控制。NeoForge 上游使用 `gradleutils` 从 Git 历史推导版本，浅克隆缺少标签时会得到带后缀的预发布版本；Stellex 将该分支固定为 `21.1.251`，使模组依赖、FML 握手与安装器产物的版本完全对齐正式版。固定版本的直接影响是：安装器内嵌的 universal 路径、启动参数文件路径、服务端库目录都从带后缀的版本变为 `21.1.251`，与官方发行版一致。

### 4.2 NeoForm 源码铺陈

NeoForge 仓库中并不直接包含 `net/minecraft` 的补丁源码，构建前必须先执行 `:neoforge:setup`。该任务会调用 NeoForm 完成 Minecraft 的反编译、访问转换、接口注入与补丁应用，然后把结果铺进 `src/main/java`。这一步是构建能否成功的关键：缺少它时，编译阶段会报出大量“程序包 net.minecraft.core 不存在”，而错误列表只会显示前一百条，容易让人误判为依赖缺失。

实测中，`setup` 在缓存命中的 NeoForm 产物上耗时 2 分 55 秒；随后 `:neoforge:build` 首次完整构建耗时 8 分 56 秒，产出 universal、installer、userdev、sources 四类产物。加入内核模块后的重建为 6 分 49 秒，加入内存治理后为 2 分 18 秒，固定版本后为 4 分 02 秒。这些差异主要来自增量编译与补丁任务是否命中缓存。

### 4.3 内核模块的集成点

内核模块放在 `net.neoforged.neoforge.stellex` 包。事件接入不新建总线，而是在 `NeoForge` 核心类的静态初始化中调用 `StellexHooks.init()`，由它向 `NeoForge.EVENT_BUS` 注册服务端启动、停止、tick 前后与命令注册监听器。这样做的原因是 `NeoForge.EVENT_BUS` 是加载器自有的游戏总线，模组也在其上监听；Stellex 只订阅、不拦截，因此不会改变事件分发顺序。

客户端模块通过 `neoforge.mixins.json` 的 `client` 数组登记，Mixin 类位于 `net.neoforged.neoforge.mixins.client`。专用服务端不会加载这些类。客户端入口另有一个 `StellexClientBootstrap`，只在 `FMLEnvironment.dist.isClient()` 为真时由钩子调用，避免服务端触碰客户端类。

### 4.4 安装器品牌化

NeoForge 的安装器由 `installerJar` 任务生成：它以 legacyinstaller 为基础，加入 `version.json`、`install_profile.json`、启动参数与二进制补丁，并在调试模式下把 universal 包内嵌到 `maven/` 目录。Stellex 保留这套安装逻辑，只替换品牌资源：大 Logo、窗口图标、背景图块与 `url.png` 被替换为暗色扁平风格，`install_profile.json` 中的欢迎语与 profile 名改为 Stellex，图标换成本地生成的 ICO，同时移除指向官方镜像列表服务的 `mirrorList` 字段，避免该服务不可达时安装器在初始化阶段抛错。

安装器内嵌的是本机构建的 `neoforge-21.1.251-universal.jar`（3,605,609 字节），因此安装出的服务端使用的是 Stellex 内核而不是从官方仓库下载的原版。安装过程仍需要下载 Minecraft 服务端、映射与运行库，这些文件来自官方仓库，Stellex 不改变其来源与校验方式。

### 4.5 下载器与发行包

除了完整安装器，Stellex 还提供轻量下载器：它从项目 GitHub Release 下载服务端包并解压，支持 `--zip` 指定本地包、`--install` 无界面安装。发行包由安装后的服务端目录打包而成，排除世界、日志与本地配置，附带 `STELLEX.txt` 说明。实测安装包为 159,149,595 字节、102 个文件；下载器无界面安装产出 103 个条目，目录结构与安装器产物一致。

## 5 内核子系统

### 5.1 CPU 调度器

调度器以区块票据驱动预生成。服务端主线程在 tick 结束后，根据玩家位置或固定锚点规划环形区块队列，按距离排序后依次登记私有票据；区块生成由原版异步管线完成，主线程轮询完成情况并释放票据。票据等级为 33，即“已生成但不刻”，避免实体刻与方块实体刻带来的额外开销。目标占用比例通过 `stellex.cpu.target` 配置，调度器按进程 CPU 采样与平均 tick 耗时动态调整在途上限；平均 tick 超过保护线时停止投喂。队列排空后工作完全停止，没有玩家且未配置锚点时也不工作。

调度器的设计有两个关键点。其一是“有意义的负载”：所有 CPU 消耗都对应一个真实的区块生成任务，调度器不会为了凑占用率而制造空转。其二是“可退让”：在途上限、tick 保护与 CPU 目标三者构成闭环，任何一项触发都会减少投喂。运行期命令包括 `/stellex status`、`target`、`radius`、`pause`、`resume`、`reset`，输出包含目标与实测 CPU、在途数量、队列长度、生成与失败计数、平均每区块耗时、tick 与 TPS 估算。

### 5.2 内存治理

内存治理由独立守护线程每 10 秒采样一次堆使用率与 GC 次数。每次 GC 后记录堆占用，保留最近 12 个样本；若样本单调上升且累计增长超过堆上限的 8%，标记为可疑泄漏并输出警告，提示开启堆转储。堆使用率超过软阈值 75% 时执行注册的清理动作，动作之间有 60 秒节流；超过硬阈值 90% 时执行清理，并按配置决定是否请求 GC 与写出 hprof 堆转储，节流为 120 秒。

清理动作通过 `MemoryCleanupRegistry` 注册，内核自身注册了调度器记账清理。这个接口的定位是让加载器各模块与后续的内核功能把“可重建缓存”统一登记，压力到来时按优先级执行。命令面提供 `/stellex memory`、`/stellex memory clean`、`/stellex memory dump`，分别用于查看状态、手动清理与堆转储。

需要强调，这套机制无法修复任意第三方模组内部的泄漏。它做的是发现、遏制与取证：把加载器可回收的记账清理掉，把持续增长的模式在早期暴露出来，并提供堆转储作为离线分析入口。把“检测、遏制、取证”与“修复”区分开，是这套设计能够保持安全边界的前提。

### 5.3 客户端渲染剔除

客户端剔除只挂两个调用点。实体路径注入 `EntityRenderer#shouldRender` 的头部，距离超过配置阈值、没有自定义名称、没有发光、不是玩家、且 `noCulling` 为假的实体直接判定不可见。方块实体路径重定向 `LevelRenderer#renderLevel` 内的 `BlockEntityRenderDispatcher#render` 调用，距离按渲染器声明的包围盒计算，准星指向的方块实体始终保留。两个注入点都在 `client` 数组内，服务端不会解析。

配置通过 `config/stellex-client.properties` 管理，默认实体 128 格、方块实体 96 格、统计间隔 1200 tick。配置加载发生在第一个客户端 tick，读取失败时保留默认值并记录警告，不覆盖用户文件。统计日志按窗口值与累计值输出，便于判断某一路剔除在当前场景下是否真的命中。

### 5.4 可观测性

内核新增的可观测信息全部通过本地日志与命令输出，不引入遥测，不上报任何数据。调度器与内存治理各自使用独立日志通道，便于在服务器日志中检索；命令输出保持单行结构，便于脚本采集。安装器与下载器的日志保留在本地，发行包不包含运行期日志。

## 6 兼容性验证

### 6.1 测试环境

服务端使用 Stellex 安装器安装出的 21.1.251 专用服务端，JDK 21，世界种子 12345，视距 6，模拟距离 4，开启 RCON。客户端有两种：一种是纯官方 NeoForge 21.1.251 开发客户端，另一种是本加载器构建的 Stellex 客户端。两者都通过 `--quickPlayMultiplayer 127.0.0.1:25599` 自动连接，避免人工点击带来的不确定性。测试机使用 Intel HD Graphics 620，OpenGL 4.6，Windows 10。

### 6.2 纯 NeoForge 客户端进入 Stellex 服务端

官方 NeoForge 21.1.251 客户端启动后的模组列表为 `Minecraft 1.21.1`、`NeoForge 21.1.251` 与一个不注册网络通道的 Stellex 附属模组。客户端自动连接到 Stellex 服务端后，服务端日志在 22:22:34 记录 `Dev[/127.0.0.1:54797] logged in with entity id 11` 与 `Dev joined the game`，客户端日志随后记录 `Loaded 2 advancements`，说明握手、配置阶段与进入世界三个阶段全部完成。

这次连接能够成立的原因有两点。第一，Stellex 内核的版本字符串与 FML 网络版本和官方 21.1.251 完全一致，握手没有版本差异。第二，内核没有注册任何必需的自定义通道，客户端不需要为 Stellex 准备额外的通道实现；客户端仍然只通过它自己声明的通道参与校验。

### 6.3 Stellex 客户端进入 Stellex 服务端

使用本加载器构建的 Stellex 客户端连接同一服务端，模组列表显示 `NeoForge 21.1.251`（由本机构建），连接后服务端在 22:04:03 记录 `Dev joined the game`。这条结果说明同一源码基座构建出的客户端与服务端在协议层完全自洽，也说明 Stellex 的内核改动没有破坏 FML 的登录流程。

### 6.4 Stellex 客户端加载 NeoForge 模组

为了验证“Stellex 客户端兼容 NeoForge 模组”，在客户端模组目录放入真实第三方 NeoForge 模组 SQDCLib 3.0.1。客户端启动后的模组列表为 Minecraft 1.21.1、NeoForge 21.1.251、SQDCLib 3.0.1。模组被正常发现、版本解析通过、Mixin 兼容级别设为 JAVA_21，客户端继续完成资源加载并进入连接阶段。这说明 Stellex 客户端加载 NeoForge 模组时，加载器侧的行为与官方 NeoForge 一致。

当客户端带着 SQDCLib 连接到未安装该模组的服务端时，客户端在 21:59:02 被断开，日志逐条列出 `sqdclib:qte_start`、`sqdclib:world_state`、`sqdclib:cutscene_skip` 等通道缺失，并提示 `This channel is missing on the server side, but required on the client!`。这是 NeoForge 对“客户端必需通道、服务端未提供”的标准校验行为，说明通道协商同样按原版规则工作，而不是被加载器屏蔽或改写。换言之，模组兼容性没有被破坏，模组之间的依赖关系仍然由 NeoForge 的规则裁决。

### 6.5 兼容性分析

从 ABI 角度看，Stellex 没有重命名或移动任何 `net.neoforged` 与 `net.minecraft` 类，模组的 Mixin 目标、访问转换器引用、事件监听签名与官方构建一致。新增的 `net.neoforged.neoforge.stellex` 包不参与模组依赖解析，也不覆盖任何已有类。从网络角度看，内核没有注册通道，因此连接双方不需要为 Stellex 做任何协议适配。从元数据角度看，模组看到的仍然是 `neoforge 21.1.251` 与 `minecraft 1.21.1`，依赖范围与官方一致。

Sinytra Connector 的兼容性依赖同一组契约：官方名映射、FML 事件时序、模组元数据与 Mixin 管线。Stellex 保留这些结构，只在事件总线上追加监听器，因此 Connector 看到的运行时与官方 NeoForge 没有结构差异。需要说明的是，本文没有在真实 Connector 环境中运行 Fabric 模组，这一条属于基于契约的分析结论，而非实测结论。

### 6.6 验证边界

本轮验证覆盖了纯 NeoForge 客户端进 Stellex 服务端、Stellex 客户端进 Stellex 服务端、Stellex 客户端加载真实 NeoForge 模组三种场景。未覆盖的场景包括：大型整合包下的模组组合、Connector 转译的 Fabric 模组、生产环境（非开发环境）客户端的完整启动链路、以及不同显卡与驱动下的客户端渲染表现。这些场景需要在更接近真实玩家的环境中补测。

## 7 实验数据

### 7.1 构建数据

源码分支共 3593 个文件。NeoForm 源码铺陈任务 `:neoforge:setup` 在缓存命中的情况下耗时 2 分 55 秒。首次完整构建 `:neoforge:build` 耗时 8 分 56 秒，产出五类文件：开发 jar 21,936,310 字节、universal 包 3,605,609 字节、installer 10,226,468 字节、userdev 3,997,110 字节、sources 1,829,792 字节。加入内核调度器与客户端剔除后的重建耗时 6 分 49 秒；加入内存治理后的重建耗时 2 分 18 秒；固定版本号后的重建耗时 4 分 02 秒。构建耗时的差异来自增量编译与补丁缓存命中率，而不是功能规模。

### 7.2 启动数据

同一服务端在不同世界状态下的启动耗时差异明显：全新世界首次启动为 74.575 秒；已有世界重启为 47.575 秒；预热后的世界为 27.087 秒；正式版 21.1.251 的首次启动为 52.697 秒。启动完成后，日志依次出现 `Done`、`RCON running`、`Stellex governor settings` 与 `Stellex memory guard started`，说明三个内核子系统都在服务端就绪后初始化。

### 7.3 调度器与内存状态

在空闲服务端上查询 `/stellex status`，返回 `RUNNING | target=50.0% | inflight=0/4 queued=0 | tick=1.10ms (~20.0 tps)`。空闲时 CPU 采样显示为不可用或接近零，调度器没有投喂任务，符合“没有玩家不工作”的设计。查询 `/stellex memory` 返回 `heap=326/2018 MB (16.2%) gc=61 (1148 ms) suspectedLeak=false cleanups=1`；执行 `/stellex memory clean` 返回 `ran 1 action(s): governor-bookkeeping(39us)`。这两组数据说明内存治理的采样、清理与命令面均可正常工作。

### 7.4 发行产物

安装器重打包后为 10,246,074 字节，内嵌 `maven/net/neoforged/neoforge/21.1.251/neoforge-21.1.251-universal.jar`。服务端发行包为 159,149,595 字节、102 个文件，包含 `libraries/`、`run.bat`、`run.sh`、`user_jvm_args.txt` 与 `STELLEX.txt`。下载器无界面安装该包产出 103 个条目，与安装器直接安装的目录结构一致。安装器与下载器的品牌资源为本地生成的暗色扁平 Logo、窗口图标与背景，不包含官方橙色视觉元素。

### 7.5 数据效度

上述数据来自单机、单次运行，未做多次重复与置信区间估计；客户端验证在开发环境中完成，与生产环境存在类加载与资源路径差异；GUI 自动化通过 quickPlay 与一次向导点击完成，未覆盖真实玩家的交互路径。因此这些数据应被视为工程验证，而不是严格基准测试。

## 8 讨论

### 8.1 构建链路的脆弱点

本文的实践表明，源码构建的主要成本不在编译，而在“环境能否把构建引导起来”。三个脆弱点依次出现：第一，网络与代理，官方 maven 不可达时插件与依赖会解析失败，且错误信息往往指向具体构件而不是网络；第二，Gradle 版本，使用上游捆绑的发行包可以避免版本兼容问题；第三，NeoForm 源码铺陈，缺少 `setup` 时会表现为大规模“包不存在”，容易误判。把这三步写成脚本与文档，是让构建可重复的前提。

### 8.2 版本字符串的重要性

版本字符串看起来只是标签，实际上同时参与模组依赖解析、FML 握手与安装器路径生成。预发布后缀会在三处产生连锁影响：模组依赖范围可能不满足，客户端与服务端握手可能不一致，安装器生成的库路径与官方目录不同。Stellex 将版本固定为 `21.1.251` 后，这三处同时回到官方形态，这对“可加入、可被加入”的验证是决定性的。

### 8.3 模组通道与加载器兼容

验证中出现过一次断开：客户端带着 SQDCLib、服务端没有该模组，握手失败并列出缺失通道。这一结果容易被误读为“加载器不兼容”，但日志明确说明是模组通道缺失，且断开行为与官方 NeoForge 完全一致。把“加载器兼容”与“模组组合兼容”区分开，是评估此类结果的关键。加载器兼容意味着模组能被发现、解析、注入与运行；模组组合兼容意味着双方声明的依赖与通道能对上。前者是加载器的责任，后者是模组与服务器配置的责任。

### 8.4 内存治理的能力边界

内存泄漏的根因通常不在加载器，而在第三方代码持有的静态引用、线程、原生资源或缓存。加载器能做的是三件事：自身记账不泄漏、可重建缓存可在压力下丢弃、以及把增长趋势与堆转储尽早交到运维手里。把这三件事做好，可以显著降低“服务器跑几天就 OOM”的概率，但不能保证“任何模组的泄漏都被修复”。在对外表述上，应避免把“内存治理”说成“内存泄漏修复”，否则会制造不切实际的预期。

### 8.5 调度器的语义

调度器的目标百分比是闭环上限，而不是硬性承诺。实际占用取决于工作源的并行度、tick 预算与其他模组的负载。把这一点写进文档与命令输出，可以避免运维把“没有达到目标”误判为故障。调度器的价值在于把空闲算力用于可度量的真实工作，而不是把利用率数字推向极限；后者更容易损害玩家体验。

## 9 局限性与效度威胁

### 9.1 环境单一

全部实验在单台 Windows 10 机器上完成，CPU 为四核级别，显卡为集成显卡，内存约 8 GB。构建耗时、启动耗时、客户端帧率与堆占用都受这台机器影响，不能直接外推到高核数服务器或独立显卡客户端。尤其是构建耗时，NeoForm 的反编译与重编译对磁盘与 CPU 都很敏感，换一台机器可能得到完全不同的数字。

### 9.2 单次运行

本文的数据来自单次运行，没有做多次重复与置信区间估计。服务端启动时间在不同世界状态下差异很大，客户端握手又受资源加载与 JIT 影响，单次结果只能说明“这条路径在某次运行中成立”，不能说明其分布。更严格的做法是每个场景重复三次以上并给出中位数与极差，这需要在后续工作中补齐。

### 9.3 开发环境与生产环境差异

客户端验证在开发环境中完成，游戏以源码集与开发类路径启动，资源从 Gradle 缓存读取，模组通过运行目录加载。生产环境使用安装器产出的客户端，类路径、资源索引与模组发现路径都不同。因此“开发客户端能加入”不等于“生产客户端能加入”，虽然两者的加载器代码相同，但仍需在生产形态下补测。

### 9.4 Connector 未实测

本文对 Connector 的兼容性结论来自契约分析：保留官方名映射、事件时序、模组元数据与 Mixin 管线。分析可以说明“结构上没有阻断”，但不能替代“在真实 Connector 环境中加载一个 Fabric 模组并进入世界”的实测。Fabric 模组的 Mixin 目标、Fabric API 的语义与 Connector 的版本差异都可能引入分析未覆盖的问题。

### 9.5 内存治理的覆盖范围

内存治理只覆盖加载器自身可重建的数据与少量通用清理动作，不覆盖第三方模组的内部结构。它的增长检测基于 GC 后的堆占用，采样间隔为 10 秒，窗口为 12 个样本，这意味着短时突发或缓慢泄漏可能被平滑掉。堆转储是离线分析工具，不是自动修复手段；真正定位泄漏仍需要 MAT 或 JProfiler 等工具。

### 9.6 客户端剔除的收益

客户端剔除的收益取决于场景。实体密度、方块实体密度、视距与整合包构成都会影响命中率；在空旷场景中，剔除计数可能长期为零，收益可以忽略。本文没有在图形环境中做 A/B 帧率测量，因此不能给出“提升多少帧”的结论。把剔除定位为“减少不可见对象的渲染开销”，而不是“通用帧率修复”，是更准确的表述。

## 10 结论

本文以 NeoForge 21.1.251 源码为基座，完成了一个保留全部 ABI 的独立加载器，并在内核层加入 CPU 调度、内存治理与客户端剔除三项能力。形态选择上，基于源码基座而非从零重写是唯一能同时满足“模组兼容”与“新增内核能力”的路径；工程实现上，构建链路的三个脆弱点（代理、Gradle 版本、NeoForm 源码铺陈）与版本字符串的固定，是让产物可重复、可握手的关键；运行期上，调度器与内存治理在不安装任何模组的情况下工作，`/stellex` 命令提供了可观测性。

兼容性验证给出了三条正面结果：纯官方 NeoForge 21.1.251 客户端成功进入 Stellex 服务端；Stellex 客户端成功进入 Stellex 服务端；Stellex 客户端成功加载真实 NeoForge 模组 SQDCLib 3.0.1，并在服务端缺少该模组时按 NeoForge 原有规则因必需通道缺失而断开。这三条结果共同说明：加载器层面的新增能力没有破坏模组契约，模组组合层面的兼容性仍由模组自身的声明决定。

后续工作有三个方向。第一，在真实 Connector 环境中加载 Fabric 模组，验证转译链路的完整性。第二，在生产形态的客户端上做渲染剔除的 A/B 测量，给出可复现的收益区间。第三，扩展内存治理的清理器集合，把事件总线监听器、区块引用、纹理缓存等常见泄漏点纳入定向清理，并对增长检测做多次重复标定。

## 参考文献

[1] NeoForged 团队. NeoForge 源码与补丁集, 21.1.251. https://github.com/neoforged/NeoForge.
[2] NeoForged 团队. NeoForm 与 ModDevGradle 构建文档.
[3] FabricMC 团队. Fabric Loader 与 Fabric API 文档. https://fabricmc.net.
[4] Sinytra 团队. Sinytra Connector 项目文档. https://github.com/Sinytra/Connector.
[5] SpongePowered. Mixin 框架文档. https://github.com/SpongePowered/Mixin.
[6] CaffeineMC. Sodium 项目. https://github.com/CaffeineMC/sodium.
[7] tr7zw. EntityCulling 项目. https://github.com/tr7zw/EntityCulling.
[8] pop4959. Chunky 项目. https://github.com/pop4959/Chunky.
[9] lucko. Spark 性能分析器. https://github.com/lucko/spark.
[10] Mojang Studios. Minecraft 1.21.1 官方命名与数据生成说明.

## 11 复现步骤

复现分为四步。第一步，准备环境：安装 JDK 21 与 JDK 17（安装器需要 17，服务端与构建需要 21），准备 Gradle 9.8.1 发行包，配置可访问官方 maven 的网络或代理，并在 `~/.gradle/gradle.properties` 写入 `systemProp.http.proxyHost`、`systemProp.http.proxyPort`、`systemProp.https.proxyHost`、`systemProp.https.proxyPort`。第二步，取源码并构建：浅克隆 NeoForge 的 `1.21.1` 分支，进入工程后先执行 `:neoforge:setup`，再执行 `:neoforge:build -x createChangelog`，产物在 `projects/neoforge/build/libs`。第三步，品牌化与安装：用资源替换脚本处理安装器的 Logo、图标、欢迎语与镜像列表字段，得到 Stellex 安装器；以 `--installServer <目录>` 安装服务端。第四步，运行与验证：在服务端目录写入 `eula.txt` 与 `server.properties`，用 JDK 21 启动，等待 `Done`，通过 RCON 执行 `/stellex status` 与 `/stellex memory`；客户端侧以 `--quickPlayMultiplayer` 自动连接，或在生产客户端中手动加入。

复现时需要注意三个容易出错的点。第一，`setup` 必须在 `build` 之前执行，否则会出现大量“包不存在”。第二，安装器与下载器的网络行为不同：安装器需要访问官方仓库下载 Minecraft 与运行库，下载器只需要访问 Release 地址。第三，服务端启动必须在安装目录内执行，因为启动参数文件使用相对路径引用 `libraries/`；从其他目录启动会得到 `ClassNotFoundException`。

## 附录 A 运行参数与命令

服务端 JVM 参数分为调度器与内存治理两组。调度器参数包括 `stellex.cpu.enabled`（总开关）、`stellex.cpu.target`（目标百分比，默认 50）、`stellex.cpu.threads`（并发提示）、`stellex.tps.guard.ms`（tick 保护线，默认 45）、`stellex.pregen.enabled`、`stellex.pregen.radius`（默认 32）、`stellex.pregen.inflight`、`stellex.pregen.anchor`（固定锚点）与 `stellex.cpu.integrated`（是否在单人内置服务端启用）。内存治理参数包括 `stellex.memory.enabled`、`stellex.memory.interval.seconds`（默认 10）、`stellex.memory.soft.percent`（默认 75）、`stellex.memory.critical.percent`（默认 90）、`stellex.memory.gc.on.critical`、`stellex.memory.dump.on.critical`、`stellex.memory.dump.dir` 与 `stellex.memory.leak.window`（默认 12）。

命令面以 `/stellex` 为根，需要权限等级 2。`status` 输出调度器状态；`target` 修改目标百分比；`radius` 修改预生成半径；`pause` 与 `resume` 控制投喂；`reset` 清空调度器记账；`memory` 输出堆与 GC 状态；`memory clean` 手动执行清理；`memory dump` 生成堆转储。所有命令输出保持单行，便于日志采集与脚本解析。

客户端配置位于 `config/stellex-client.properties`，包含总开关、实体剔除距离、方块实体剔除距离与统计间隔四个键。默认值分别为 `true`、128、96、1200。配置在第一个客户端 tick 加载，缺失时写出默认文件，解析失败时保留默认值并记录警告。

## 附录 B 安装与发行流程

安装器流程分为四步。第一步，生成 installerJar：`installerJar` 任务以 legacyinstaller 为基础，加入版本描述、安装描述、启动参数与二进制补丁，并把本机构建的 universal 包内嵌到 `maven/` 目录。第二步，替换品牌资源：大 Logo、窗口图标、背景图块、`url.png` 与安装描述中的欢迎语、profile 名、图标都替换为 Stellex 版本，同时移除镜像列表字段。第三步，安装服务端：以 `--installServer` 指定目录运行安装器，安装器完成 Minecraft 服务端下载、二进制补丁应用与运行库下载。第四步，打包发行：排除世界、日志与本地配置后压缩成 zip，作为 Release 资产上传。

下载器流程分为三步。第一步，读取 Release 地址，默认指向 `releases/latest/download/stellex-loader-1.0.0-server.zip`；第二步，下载到临时目录并校验 HTTP 状态；第三步，解压到用户选择的目录，解压时校验路径是否越界。下载器同时支持 `--zip` 与 `--install`，前者用于离线验证，后者用于无界面安装。

品牌化的关键在于“只改表现层”。安装逻辑、版本描述、库坐标与校验方式都保持官方形态，这样安装出来的目录结构、启动脚本与库路径都与官方一致，Stellex 的差异只体现在 Logo、欢迎语、profile 名与内嵌的 universal 包上。

## 附录 C 关键日志摘录

服务端启动完成的日志为 `[22:19:31] [Server thread/INFO] [minecraft/DedicatedServer]: Done (52.697s)! For help, type "help"`，随后是 `RCON running on 0.0.0.0:25610` 与 `Stellex governor settings: enabled=true integrated=false target=50.0% threads=2 tpsGuard=45ms pregen=true radius=32 inflight=4 anchor=<none>`，再后是 `[Stellex] CPU governor started`。这三条日志同时证明三件事：服务端在世界生成后正常进入运行状态、RCON 可用于自动化查询、调度器在服务端就绪后初始化。

内存治理的启动日志为 `Stellex memory guard started: enabled=true interval=10s soft=75% critical=90% gcOnCritical=false dumpOnCritical=false leakWindow=12`。查询日志为 `Stellex memory: heap=326/2018 MB (16.2%) gc=61 (1148 ms) suspectedLeak=false cleanups=1 lastAction=none lastDetails=none`。清理日志为 `memory cleanup ran 1 action(s): governor-bookkeeping(39us)`。

纯 NeoForge 客户端连接时，服务端日志为 `Dev[/127.0.0.1:54797] logged in with entity id 11 at (96.5, ...)` 与 `Dev joined the game`；客户端日志在连接前记录 `Connecting to 127.0.0.1, 25599`，连接后记录 `Loaded 2 advancements`。这说明登录、配置与进入世界三个阶段都完成。

模组通道校验失败时，客户端日志逐条记录缺失通道，例如 `Channel [sqdclib:world_state] failed to connect: This channel is missing on the server side, but required on the client!`，随后记录 `Client disconnected with reason: Incompatible client! Please use NeoForge 21.1.251`。这条断开的直接原因是模组通道，而不是加载器版本：服务端与客户端的 NeoForge 版本均为 21.1.251，断开信息中的版本要求只是握手失败时的统一提示。把这一区分写清楚，可以避免把模组组合问题误判为加载器问题。

## 附录 D 数据汇总

构建数据：源码 3593 个文件；`setup` 2 分 55 秒；首次构建 8 分 56 秒；内核集成后 6 分 49 秒；内存治理后 2 分 18 秒；版本固定后 4 分 02 秒。产物大小：开发 jar 21,936,310 字节；universal 3,605,609 字节；installer 10,226,468 字节；userdev 3,997,110 字节；sources 1,829,792 字节。

启动数据：74.575 秒（全新世界）、47.575 秒（已有世界）、27.087 秒（预热世界）、52.697 秒（正式版新世界）。

运行期数据：空闲调度器 `target=50.0%`、`inflight=0/4`、`queued=0`、`tick=1.10ms`、`~20.0 tps`；内存治理 `heap=326/2018 MB`、`16.2%`、`gc=61`、`1148 ms`、`cleanups=1`、`39us`。

发行数据：安装器 10,246,074 字节；内嵌 universal 路径 `maven/net/neoforged/neoforge/21.1.251/neoforge-21.1.251-universal.jar`；服务端包 159,149,595 字节、102 个文件；下载器安装 103 个条目。

兼容性数据：纯 NeoForge 客户端连接成功（22:22:34）；Stellex 客户端连接成功（22:04:03）；Stellex 客户端加载 SQDCLib 3.0.1 成功；服务端缺少 SQDCLib 时按通道规则断开（21:59:02）。

## 附录 E 兼容性检查清单

本文使用的检查清单包含八项。第一，版本一致性：客户端与服务端的 NeoForge 版本是否相同。第二，协议独立性：加载器是否注册必需通道。第三，模组发现：客户端模组列表是否包含放入 `mods/` 的第三方模组。第四，Mixin 应用：客户端启动日志的兼容级别与注入是否正常。第五，服务端加入：客户端能否完成登录、配置与进入世界。第六，模组组合校验：服务端缺少客户端必需模组时是否按原版规则断开。第七，内核功能：`/stellex status` 与 `/stellex memory` 是否可用。第八，发行形态：安装器与下载器产出的目录结构是否一致。

八项检查的结果是：版本一致、无自定义通道、模组发现正常、Mixin 应用正常、两种客户端均可加入、模组组合校验按原版规则执行、内核命令可用、发行形态一致。未通过或未覆盖的检查项包括：Connector 环境、生产客户端形态、大型整合包、不同硬件。

## 附录 F 术语表

独立加载器：在官方加载器源码基础上保留全部 ABI、只增加模块与品牌、独立发行的加载器形态。ABI：第三方字节码依赖的类、方法、描述符、事件时序与元数据集合。票据：区块系统中表达“某区块需要保持到什么加载等级”的机制，等级越小加载越完整。在途：已经登记票据但尚未完成生成检测的区块数量。闭环上限：以 CPU 采样与 tick 预算为反馈、朝目标比例逼近但不承诺精确达成的控制目标。内存治理：对堆压力做采样、清理与取证的运行时子系统。软阈值与硬阈值：触发普通清理与激进清理的堆使用率分界。堆转储：把堆内存快照写入 hprof 文件，供 MAT 或 JProfiler 离线分析。写入隔离：客户端 Mixin 只登记在客户端数组，专用服务端不加载客户端类。发行包：安装后的服务端目录去掉世界、日志与本地配置后的压缩包。

## 12 安全性、许可与维护

### 12.1 安全边界

内核新增的三项能力都刻意限制了作用范围。调度器只使用公开的区块票据 API 与事件总线，不写世界数据，不替换 tick 循环，不修改其他模组的类；它的最坏情况是占用过多 CPU，而这被目标比例、在途上限与 tick 保护三重约束。内存治理只读取 JVM 管理接口，清理动作限定在加载器自身可重建的数据上；堆转储写入本地文件，路径由启动参数决定，不通过网络发送。客户端剔除只作用于两个渲染调用点，注入失败时降级为原版渲染，不会让客户端崩溃。

需要指出的是，堆转储可能包含玩家数据与服务器配置，属于敏感文件。发行版默认关闭自动转储，只有在显式传入 `stellex.memory.dump.on.critical=true` 或执行 `/stellex memory dump` 时才会写出文件。运维应把转储目录放在受控位置，并在分析完成后按流程清理。

### 12.2 滥用与误用风险

CPU 调度器可以被误用为“压满 CPU”的工具，这与它的设计目标相反。本文的实现通过三个约束降低这一风险：默认只在专用服务端启用、默认目标为 50%、平均 tick 超过保护线时停止投喂。即便目标被设为 100%，调度器也只在 tick 有余量时投喂真实区块任务，而不会制造空转线程。文档与命令输出都把目标描述为“闭环上限”，避免把它误解为硬性性能承诺。

内存治理也有误用风险。若把软阈值设得过低，清理会被频繁触发，虽然动作本身很轻，但日志会变得嘈杂；若把硬阈值设得过高，清理可能来不及发生。合理的默认值来自“软 75%、硬 90%、节流 60 秒与 120 秒”这一组经验参数，调整时应结合服务器实际堆大小与玩家规模。

### 12.3 许可与署名

NeoForge 源码采用 LGPL-2.1 许可证，Stellex 对其修改与新增部分沿用同一许可证，并在发行包中保留原始版权声明。品牌资源（Logo、图标、背景）为本地生成，不使用官方美术资源。作者署名为 Mornfrost Office，发行包、文档与论文保持一致，不添加与项目无关的署名。

### 12.4 维护策略

维护分为四条。第一，版本固定：每次升级加载器时同步更新 Minecraft、NeoForm 与 FML 版本，并重新执行 `setup` 与完整构建。第二，构建可重复：把代理配置、Gradle 版本、`setup` 步骤与 `build` 命令写入脚本，避免依赖个人环境的隐式配置。第三，兼容性回归：每次升级后至少跑一遍本文附录 E 的八项检查，重点看模组发现、Mixin 应用与握手结果。第四，发行核对：安装器内嵌的 universal 版本、服务端库目录、启动参数路径三者必须一致，否则会出现“安装成功但启动失败”的隐蔽问题。

## 附录 G 源码改动清单

新增包 `net.neoforged.neoforge.stellex`，包含 `StellexHooks`、`GovernorSettings`、`CpuSampler`、`ChunkPrefetchPlanner`、`CpuGovernor` 与 `StellexCommands`。新增包 `net.neoforged.neoforge.stellex.client`，包含 `ClientPerfConfig`、`ClientPerfStats`、`StellexClientHooks` 与 `StellexClientBootstrap`。新增包 `net.neoforged.neoforge.stellex.memory`，包含 `MemorySettings`、`MemoryCleanup`、`MemoryCleanupRegistry` 与 `MemoryGuard`。新增包 `net.neoforged.neoforge.mixins.client`，包含实体剔除与方块实体剔除两个 Mixin。

修改点有三处。其一是 `net.neoforged.neoforge.common.NeoForge` 的静态初始化，增加对 `StellexHooks.init()` 的调用。其二是 `neoforge.mixins.json` 的 `client` 数组，登记两个客户端 Mixin。其三是根构建脚本的版本赋值，把 `gradleutils` 推导改为固定 `21.1.251`。除这三处外，NeoForge 的源码、补丁、访问转换器与事件触发点没有改动。

发行相关脚本与资源包括：安装器资源替换脚本、下载器源码、服务端打包脚本与品牌资源生成脚本。安装器替换的资源为 `big_logo.png`、`url.png`、三个背景图块与一个窗口图标；同时修改安装描述中的欢迎语、profile 名、图标与镜像列表字段。

## 附录 H 构建与运行日志摘要

构建步骤可以概括为：克隆 `1.21.1` 分支；执行 `:neoforge:setup`；执行 `:neoforge:build -x createChangelog`；检查 `projects/neoforge/build/libs` 下的五类产物。`-x createChangelog` 是浅克隆环境下的必要绕行，因为变更日志任务需要父提交历史。

安装步骤可以概括为：用资源替换脚本处理 `neoforge-21.1.251-installer.jar`；以 `--installServer <目录>` 安装；在目录内写入 `eula.txt` 与服务端配置；以 JDK 21 启动 `@user_jvm_args.txt @libraries/.../win_args.txt`。启动完成后，`/stellex status` 与 `/stellex memory` 可用于验证内核功能。

客户端验证步骤可以概括为：在开发环境中以 `--quickPlayMultiplayer` 自动连接；或在生产客户端中手动加入。验证模组加载时，把模组放入 `run/client/mods`，检查启动日志的模组列表与 Mixin 兼容级别；验证通道规则时，观察服务端缺少模组时的断开信息。
## 附录 I 安装器品牌替换细节

替换的资源共有七项：`big_logo.png`（480×120，透明背景，暗色扁平 Logo，右侧中文名“探星汇鉴”）、`url.png`（20×22，菱形图标）、`icons/neoforged_16x16.png`（窗口图标）、三个尺寸的背景图块（16、32、128，暗色渐变加菱形图案）与 `install_profile.json` 内嵌的图标。图标使用 Pillow 生成，同时导出 ICO 以填充安装描述中的图标字段。

安装描述的三处文本被修改。欢迎语从官方文案改为 `Welcome to the simple Stellex installer`；profile 名从官方名改为 `Stellex`；镜像列表字段被整体移除。移除的原因是官方镜像列表服务在当前网络不可达，安装器在初始化阶段会尝试拉取列表并因超时而抛错；移除后安装器直接使用默认下载源，安装流程不再依赖该服务。

内嵌 universal 的校验方式是在重打包后检查安装器 jar 的 `maven/net/neoforged/neoforge/21.1.251/` 目录，确认其中的 `neoforge-21.1.251-universal.jar` 大小为 3,605,609 字节，与构建产物一致。安装后进一步检查服务端 `libraries/net/neoforged/neoforge/21.1.251/` 目录，确认库文件与启动参数来自该版本。

## 附录 J 常见故障排查

故障一：编译阶段大量“程序包 net.minecraft.* 不存在”。原因是未执行 `:neoforge:setup`。解决方法是先执行 `setup`，确认 `src/main/java/net/minecraft` 出现后再构建。这个故障最容易误判，因为报错只显示前一百条，看起来像依赖缺失。

故障二：构建报出“无法在没有 Git 标签的情况下计算版本”。原因是浅克隆没有标签，而 `gradleutils` 需要 Git 历史推导版本。解决方法有两种：补一个本地标签，或固定版本字符串。Stellex 选择固定版本，从而避免标签依赖。

故障三：安装器初始化阶段超时或抛错。原因是镜像列表服务不可达。解决方法是移除安装描述中的镜像列表字段，或让安装器使用可达的下载源。安装器本身仍需要访问官方仓库下载 Minecraft 与运行库。

故障四：服务端启动报 `BindException: Address already in use`。原因是上一个服务端进程仍占用端口。解决方法是结束旧进程，或修改 `server-port`。这一故障在反复测试时很常见，日志会明确写出端口。

故障五：服务端启动报 `ClassNotFoundException: cpw.mods.bootstraplauncher.BootstrapLauncher`。原因是启动目录不正确，启动参数文件中的 `libraries/` 是相对路径。解决方法是在安装目录内启动，而不是在其他目录引用该参数文件。

故障六：客户端连接被断开并提示通道缺失。原因是客户端模组声明了必需通道，而服务端没有对应模组。这不是加载器问题，解决方法是让两端安装相同的模组，或改用不声明必需通道的模组。日志会逐条列出缺失通道，便于定位。

故障七：堆转储失败。原因是 JVM 不支持该接口或目标目录不可写。解决方法是确认使用 HotSpot 系 JVM，并把 `stellex.memory.dump.dir` 指向可写目录。转储文件较大，应预留磁盘空间。

## 附录 K 参考配置

服务端启动参数示例：`-Xms2G -Xmx4G -Dstellex.cpu.target=60 -Dstellex.cpu.threads=6 -Dstellex.tps.guard.ms=45 -Dstellex.pregen.radius=32 -Dstellex.memory.soft.percent=75 -Dstellex.memory.critical.percent=90`。这组参数面向四核以上、内存 4 GB 以上的专用服务端；低配环境应降低目标与半径。

客户端配置示例：`stellex.client.enabled=true`、`stellex.client.entity_cull_distance=128`、`stellex.client.block_entity_cull_distance=96`、`stellex.client.stats_log_interval_ticks=1200`。若与专业渲染优化模组共存，建议先关闭其中一路做对照，再逐项打开。

下载器地址示例：`https://github.com/mornfire/stellex/releases/latest/download/stellex-loader-1.0.0-server.zip`。该地址要求仓库存在对应 Release 资产，资产名保持 `stellex-loader-1.0.0-server.zip`；若使用其他标签，可把 `latest` 换成具体标签名。

安装器的命令行用法：`java -jar stellex-loader-installer.jar --installServer <目录>`。安装器需要 JDK 17 或更高版本；服务端运行需要 JDK 21。安装完成后可以删除安装器文件，服务端不依赖它。

## 附录 L 结语

Stellex 的实践说明，“在官方加载器上做内核级增强”与“保持模组兼容”并不矛盾，前提是增强只发生在 ABI 之外：不改类名、不改注入面、不改事件时序、不注册必需通道。CPU 调度、内存治理与客户端剔除都属于这一类增强，它们的代价是能力边界清晰，收益是兼容面宽、可观测、可关闭。对希望在不牺牲模组生态的前提下获得运行时治理能力的服务器与整合包而言，这一形态提供了一个可复现的工程路径。
## 附录 M 需求逐条对照

第一条要求以 NeoForge 1.21.1 系列 21.1.251 为基座。实现方式是克隆 NeoForge 的 `1.21.1` 分支，固定版本为 `21.1.251`，保留其源码、补丁、访问转换器与构建逻辑；证据是构建产物 `neoforge-21.1.251-universal.jar`、启动日志中的版本字符串与安装目录中的库路径。

第二条要求保留 NeoForge 的其余实现。实现方式是只在源码树中新增 `net.neoforged.neoforge.stellex` 与 `net.neoforged.neoforge.mixins.client` 两个包，并在 `NeoForge` 类与 `neoforge.mixins.json` 中各改一处；证据是源码改动清单与构建产物的包结构。

第三条要求完全兼容 NeoForge 模组。实现方式是保留全部包名与 ABI，不注册必需通道，不改事件时序；证据是 Stellex 客户端加载 SQDCLib 3.0.1 的模组列表、资源加载与连接阶段日志，以及纯 NeoForge 客户端成功进入 Stellex 服务端的登录日志。

第四条要求兼容 Connector 转译的 Fabric 模组。实现方式是保留官方名映射、FML 事件时序、模组元数据与 Mixin 管线；本文给出结构分析与验证边界，未在真实 Connector 环境实测，这一条是分析结论而非实测结论。

第五条要求新增 CPU 与内存能力且不得空转。实现方式是票据驱动的区块预生成、目标比例与 tick 保护闭环、以及堆压力分级的清理与增长检测；证据是 `/stellex status` 与 `/stellex memory` 的输出、空闲时不投喂任务的行为、以及手动清理的耗时。

第六条要求可编译并双向可进入。实现方式是固定版本、补齐构建步骤、品牌化安装器与下载器；证据是构建成功日志、安装器与下载器的产物、纯 NeoForge 客户端与 Stellex 客户端分别加入 Stellex 服务端的日志。

## 附录 N 客户端测试时间线

21:49:50，Stellex 服务端启动完成，`Done (13.920s)`，RCON 就绪。21:52:53，Stellex 客户端模组列表输出，包含 `NeoForge 21.1.251.0-beta` 与 `SQDCLib 3.0.1`。21:53:42，客户端进入 `forgeclientdev` 启动阶段并传入 quickPlay 参数。21:56:30，客户端完成资源加载。21:59:02，带模组客户端因必需通道缺失被断开。22:03:45，纯净 Stellex 客户端重新加载资源。22:04:03，纯净 Stellex 客户端加入服务端。22:19:31，正式版 21.1.251 服务端启动完成。22:22:15，纯 NeoForge 21.1.251 客户端完成资源加载。22:22:34，纯 NeoForge 客户端加入服务端。

这条时间线记录了一次完整的“失败—归因—重试—成功”过程。失败点是模组通道，不是加载器；归因依据是日志中的通道名与官方提示；重试时移除客户端模组；成功点是两种客户端都能加入。把过程写出来，是为了让读者能够复核结论，而不是只看最终一句“兼容”。

## 附录 O 派生指标与关系

构建层面，`setup` 与完整构建的耗时比约为 1 比 3，说明 NeoForm 产物缓存对总耗时影响很大；内核集成与内存治理的增量构建耗时分别为 6 分 49 秒与 2 分 18 秒，差异主要来自补丁与编译任务的缓存命中。发行层面，安装器大小约 10 MB，其中内嵌 universal 约 3.6 MB，服务端包约 159 MB，主要体积来自 Minecraft 服务端与运行库。运行层面，空闲服务端的堆占用约 326 MB、上限约 2018 MB、GC 次数 61、GC 总耗时 1148 毫秒，说明在无玩家、无预生成任务时，内核自身的内存开销可以忽略。

兼容性层面，三条正面结果与一条边界结果构成完整证据链：纯 NeoForge 客户端加入成功、Stellex 客户端加入成功、Stellex 客户端加载真实 NeoForge 模组成功、带必需通道的模组在服务端缺失时按原版规则断开。前者证明加载器与协议兼容，后者证明模组组合规则未被改写。
## 附录 P 兼容性问答

问：Stellex 客户端装上之后，还能进原版 NeoForge 服务端吗？答：可以。内核没有注册必需通道，也没有修改握手逻辑；服务端不需要知道客户端是 Stellex。本文在协议层验证了反方向（纯 NeoForge 客户端进 Stellex 服务端），两个方向依赖同一组契约。

问：纯 NeoForge 客户端进 Stellex 服务端，会看到什么差异？答：连接、模组列表与进入世界流程与官方一致；服务端多出的 `/stellex` 命令只对权限等级 2 的管理员可见。客户端不会收到任何 Stellex 专属的通道数据。

问：Stellex 客户端加载 NeoForge 模组，会不会因为内核改动而失败？答：从模组发现、版本解析、Mixin 注入到资源加载，行为与官方一致。本文用 SQDCLib 3.0.1 验证了这一点；模组的必需通道仍按原版规则校验。

问：为什么带 SQDCLib 的客户端连接没有该模组的服务端会被断开？答：因为 SQDCLib 声明了客户端必需通道，服务端没有对应实现。这是 NeoForge 的标准行为，与加载器实现无关；两端安装同一模组即可。

问：Connector 转译的 Fabric 模组能不能用？答：从契约上看可以，因为 Stellex 保留了官方名映射、事件时序、元数据与 Mixin 管线；但本文没有在真实 Connector 环境实测，结论属于分析而非验证。

问：内存治理会不会误删模组数据？答：不会。清理动作只作用于加载器自身可重建的记账数据，接口由加载器控制；第三方模组的数据不在清理范围内。

问：CPU 调度器会不会让服务器卡顿？答：调度器只在 tick 有余量时投喂真实区块任务，平均 tick 超过保护线时停止投喂；空闲时完全不工作。目标比例是闭环上限，不是强制占用。

问：安装器与下载器有什么区别？答：安装器是 NeoForge 安装流程的品牌化版本，内嵌本机构建的 universal，需要访问官方仓库下载 Minecraft 与运行库；下载器只从 Release 拉取服务端包并解压，适合快速部署。

问：为什么版本要固定为 21.1.251？答：版本参与模组依赖、FML 握手与安装路径生成。预发布后缀会在三处产生连锁影响；固定为正式版后，这三处都回到官方形态。

问：这套东西能修复所有内存泄漏吗？答：不能。它做的是发现、遏制与取证；任意第三方模组内部的静态引用、线程与原生资源泄漏仍需离线分析。把能力边界写清楚，比夸大承诺更重要。

## 附录 Q 版本记录

2026 年 11 月 21 日：完成 NeoForge 21.1.251 源码分支构建；集成 CPU 调度器、客户端剔除与内存治理；固定正式版本号；生成品牌化安装器、下载器与服务端发行包；完成纯 NeoForge 客户端、Stellex 客户端与真实 NeoForge 模组的兼容性验证。后续计划：Connector 实测、生产客户端 A/B 测量、定向泄漏清理器扩展。
## 附录 R 指标定义与测量方法

构建耗时：从 Gradle 输出 `BUILD SUCCESSFUL in ...` 的总耗时，包含配置、任务执行与依赖解析；首次构建与增量构建分开记录。启动耗时：从进程启动到服务端日志出现 `Done (X.XXXs)!` 的时间，由服务端自身统计，世界状态、视距与预生成范围都会影响该值。

调度器指标：`target` 是配置的目标百分比；`inflight` 是当前已登记票据但尚未完成的区块数量；`queued` 是计划器队列长度；`tick` 是服务端最近若干 tick 的滑动平均；TPS 是由 tick 平均值换算的估算值。CPU 指标来自 JVM 的进程级采样，空闲时可能显示为不可用，这属于正常现象。

内存指标：`heap` 是已用堆与堆上限；百分比是两者之比；`gc` 是垃圾回收总次数与总耗时；`suspectedLeak` 是“GC 后堆占用连续上升且累计增长超过堆上限 8%”的标记；`cleanups` 是已注册清理动作数量；`lastAction` 是最近一次动作。清理耗时以微秒记录，用于判断动作是否足够轻量。

发行指标：安装器大小、内嵌 universal 大小、发行包大小与文件数、下载器安装条目数。兼容性指标：连接成功与否、加入时间戳、客户端模组列表、断开原因与缺失通道。所有指标都以日志或命令输出的原始文本为准，不做二次加工。

## 附录 S 发行包清单

服务端发行包根目录包含：`libraries/`（Minecraft 服务端、运行库与 NeoForge 库）、`run.bat`（Windows 启动脚本）、`run.sh`（Unix 启动脚本）、`user_jvm_args.txt`（JVM 参数模板）与 `STELLEX.txt`（版本与启动说明）。被排除的内容包括：`world/`（世界数据）、`logs/`、`crash-reports/`、`config/`、`defaultconfigs/`、`mods/`、`eula.txt`、`ops.json`、`whitelist.json`、`banned-ips.json`、`banned-players.json`、`usercache.json`、`server.properties` 与 `server.jar`。

排除这些内容的目的是让发行包只包含“可复现的运行环境”，不携带任何玩家数据或本地配置。安装后由用户自行同意 EULA、设置端口与白名单，避免把测试环境的设置带给他人。发行包内的 `libraries/` 与启动脚本保证解压即可启动，`STELLEX.txt` 提供版本、启动方式与命令提示。

附录 T 未覆盖事项

本文未覆盖的事项包括：Connector 真实环境、生产客户端形态、大型整合包、不同硬件与驱动、长时间运行的堆增长曲线、以及模组通道之外的模组交互（如注册表注入、数据包替换、渲染事件竞争）。这些事项需要在更接近真实玩家的环境中逐项补测。把这些未覆盖事项列出来，是为了让后续工作有明确的清单，而不是让结论看起来比证据更强。
## 附录 U 补充说明与索引

本文没有使用图表，所有数据以行内文本与列表呈现，原因有二：一是论文需要在命令行与纯文本环境中可直接复核；二是图表中的数值无法被脚本直接比对。读者可以按附录 D 的汇总值、附录 C 的日志摘录与附录 N 的时间线逐项复算。

文中出现的路径均相对于项目根目录。`stellex-loader/` 是独立加载器与发行流水线；`stellex/` 是早前的模组形态工程，用于对照与服务端测试；`stellex-loader/src-neoforge/` 是 NeoForge 源码分支；`stellex-loader/dist/` 是发行产物；`stellex-loader/docs/` 是本文与相关文档。安装器与下载器的源码在各自的工程目录内，构建脚本与资源替换脚本在 `stellex-loader/tools/` 内。

文中引用的命令分为三类：Gradle 命令用于构建，`java -jar` 命令用于安装与启动，`/stellex` 命令用于运行期查询。Gradle 命令需要在工程根目录执行，`java -jar` 命令需要在安装目录执行，`/stellex` 命令通过 RCON 或游戏内聊天执行。把执行目录写清楚，可以避免大部分“命令找不到文件”的问题。

最后需要说明，本文的结论只覆盖已经执行的验证项。对于未执行项，本文给出的是结构分析或经验判断，并在正文与附录中明确标注。把“已验证”与“未验证”分开陈述，是这类工程论文可信度的基础，也是后续工作能够继续推进的前提。
## 附录 V 数据复核示例

以调度器状态为例，复核步骤如下：在服务端目录启动服务端，等待日志出现 `Done`，用 RCON 客户端认证后发送 `stellex status`，读取返回的单行文本；把 `target`、`inflight`、`queued`、`tick` 四个字段与本文附录 D 的数值比对。若 `inflight` 与 `queued` 均为零且 `tick` 在 1 毫秒量级，说明空闲状态符合预期。

以内存状态为例，复核步骤如下：执行 `stellex memory` 读取堆占用、GC 次数与清理动作数；执行 `stellex memory clean` 读取清理动作名称与耗时；对照本文附录 D 的 `heap=326/2018 MB`、`gc=61`、`cleanups=1`、`39us` 四组数值。若数值量级一致而具体值不同，属于正常现象，因为堆占用与 GC 次数依赖运行时长与机器状态。

以兼容性为例，复核步骤如下：准备一台纯 NeoForge 21.1.251 客户端与一台 Stellex 服务端；客户端以 `--quickPlayMultiplayer` 连接；在服务端日志检索 `joined the game`。若出现该行，说明握手、配置与进入世界三个阶段完成。若客户端模组目录额外放入带必需通道的模组，则应观察到通道缺失导致的断开，这是规则生效的证据，而不是失败。
﻿## 附录 W 结语补充

本文的验证目标不是证明 Stellex 在所有环境中都能工作，而是证明它在关键路径上不破坏既有契约：客户端能进、服务端能被进、真实模组能加载、内核功能可用。只要这四条成立，后续扩展就有了可靠的基线；不成立的部分，本文已经逐条标注为未覆盖或需实测，供后续工作继续。
