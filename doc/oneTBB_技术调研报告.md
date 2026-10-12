# oneTBB 技术调研与工程采用分析

> 调研对象：本仓库 `master` 分支的 oneTBB 源码快照
> 证据基线：提交 `885b7813d38de698a9942e5026ad1bd9d16125d9`，工作区无本地修改
> 代码声明版本：`2023.2.0`；oneTBB 规范版本 `104`；接口版本 `12200`；二进制兼容主版本 `12`
> 调研日期：2026-10-12
> 调研方法：静态源码、构建脚本、测试/CI 配置与官方公开文档交叉核验；未在本次调研中编译、运行或做性能基准。

## 摘要

oneAPI Threading Building Blocks（oneTBB）是面向 C++ 的通用并行运行时与模板库。它的核心价值不是让应用直接管理线程，而是让应用表达“可并行的工作、依赖和并发约束”，再由进程级任务调度器把工作映射到可用 CPU 资源。其最适合的场景是 CPU 密集型循环、递归分治、并行归约、内存内并发容器，以及包含多阶段依赖的计算流水线。

本仓库的实现由四个主要二进制组件组成：任务运行时 `tbb`、可扩展分配器 `tbbmalloc`、动态替换 C 运行库分配函数的 `tbbmalloc_proxy`，以及提供 CPU 拓扑/NUMA 代理能力的 `tbbbind`。另有 Thread Composability Manager（TCM），用于在同一进程内与其他并行运行时协调 CPU 资源。默认交付形态是共享库；这不是单纯的部署偏好，而是维持一个进程级调度器和工作线程池、避免多副本过度订阅及对象跨副本错误的架构前提。

结论：对需要可移植 CPU 并行的 C++ 应用，oneTBB 是成熟且功能覆盖面广的基础设施，建议以共享库形式通过 CMake 目标 `TBB::tbb` 集成。采用时应先审计进程内已有的 OpenMP、Parallel STL、旧 TBB、静态库和插件，确认不会加载多个 oneTBB 副本；之后从 `parallel_for`/`parallel_reduce` 等粗粒度计算热点开始试点。不要将预览 API 作为长期 ABI 契约，也不应把静态链接、WASM、`fork()` 后继续使用运行时、或多运行时资源协调视为“无需验证的默认路径”。

## 1. 调研范围、结论边界与证据可信度

本报告分析的是工作区中的源码快照，而非对任一生产环境的性能承诺。仓库 `origin` 指向个人 Fork（`MarvinGeng/oneTBB`），且当前提交不被本地 tag 包含；因此版本宏和 `RELEASE_NOTES.md` 能证明源码自称为 `2023.2.0`，但不能单独证明它等同于某个上游已发布二进制包。生产使用前应以受控的上游 tag、包哈希和 SBOM 再做一次供应链确认。

下列结论属于高可信的源码事实：公共头文件/API 面、CMake 默认值、构建目标、版本宏、测试目录结构，以及源码中已编码的限制。调度开销、加速比、内存峰值和目标工作负载下的公平性则必须通过本项目的基准与压力测试验证，不能从 API 设计直接推导。

## 2. 产品定位与设计取向

oneTBB 的设计中心是“逻辑并行（logical parallelism）”而不是“线程数量”。应用提交可分割的范围、任务或图节点；运行时在可用工作线程之间调度，并通过本地任务池、全局任务流和窃取机制维持负载均衡。相较于直接维护 `std::thread`，这将线程池生命周期、嵌套并行、任务等待、异常传播、取消和资源上限收敛到一个运行时内。

主要取向如下：

| 设计目标 | 代码与接口体现 | 对工程的含义 |
| --- | --- | --- |
| 数据并行优先 | `parallel_for`、`parallel_reduce`、`parallel_scan`、range 与 partitioner | 先识别可切分、计算量足够大的范围；极小任务会被调度成本淹没。 |
| 可组合性 | `task_group`、`task_arena`、`global_control`、Flow Graph | 库与应用可各自提交任务，但仍应共享同一 oneTBB 动态库。 |
| 硬件适配 | `info`、`task_arena::constraints`、NUMA 分配、`tbbbind`/hwloc | 高端 NUMA/混合核优化可逐步启用，不能假定每个平台都有同等效果。 |
| 可移植性 | C++11 基线、Windows/Linux/macOS/Android 以及社区平台配置 | 通用算法能跨平台；拓扑、分配器代理和工具链特性需单独验证。 |
| 长期二进制兼容 | `dN`/`rN` 内联命名空间、接口版本、SONAME | 消费方可在同一二进制主版本内升级运行时，但不应绕过公开 API 使用内部命名空间。 |

公开实体同时可在 `oneapi::tbb` 与兼容别名 `tbb` 下使用：`include/oneapi/tbb/detail/_namespace_injection.h` 将前者定义为后者的别名。新代码应优先使用 `oneapi::tbb`，以清晰表达 oneAPI 归属；迁移遗留代码时可保留 `tbb`。

## 3. 源码版图与交付物

本快照包含 148 个 `include/` 文件、126 个 `src/` 文件、226 个测试文件、279 个示例文件和 634 个文档文件（文件计数含非源码资源）。它不是纯头文件库：模板和公共声明在头文件中，而调度器、线程管理、动态加载与分配器依赖二进制运行时。

| 位置/目标 | 职责 | 采用影响 |
| --- | --- | --- |
| `include/oneapi/tbb/` | 49 个公共头，含算法、容器、任务、图、锁、分配器和版本接口 | 消费方唯一应直接依赖的 API 层；`include/tbb/` 提供旧路径兼容。 |
| `src/tbb/` → `TBB::tbb` | 33 个 C++ 源文件：arena、market、任务分发、线程控制、动态加载、NUMA、诊断等 | 调度器核心；应作为进程内单一共享实例。 |
| `src/tbbmalloc/` → `TBB::tbbmalloc` | 可扩展内存分配器 | 仅在分配竞争确为热点时引入并做碎片与尾延迟评估。 |
| `src/tbbmalloc_proxy/` → `TBB::tbbmalloc_proxy` | 以动态库方式拦截/替换 C 分配接口 | 影响进程级分配行为，必须评估与 Sanitizer、第三方分配器和插件的交互。 |
| `src/tbbbind/` → `TBB::tbbbind*` | 与 hwloc 协作的动态拓扑代理 | NUMA、核类型和约束功能的运行时依赖；必须随主库正确部署。 |
| `thread_composability_manager/` | TCM：与其他线程运行时协调，降低过度订阅 | 适用于混用 oneTBB/OpenMP 等的进程，但不能替代整进程并发预算设计。 |
| `integration/`、`cmake/`、`BUILD.bazel` | CMake、pkg-config、平台打包和 Bazel 集成 | CMake 是主支持路径；Bazel 是受限、社区维护的替代路径。 |

`CMakeLists.txt` 默认构建 `tbb`、`tbbmalloc`、`tbbmalloc_proxy`、TCM，默认构建共享库，默认开启测试，默认关闭示例。安装时会导出 CMake 包配置、pkg-config 文件和头文件。`find_package(TBB REQUIRED)` 后链接 `TBB::tbb` 是最小且稳定的集成方式。

## 4. 运行时架构

### 4.1 分层视图

```text
应用/库代码
  ├─ 并行算法、并发容器、Flow Graph、task_group、task_arena
  │     （多数模板/内联逻辑位于公共头文件）
  ▼
公开 ABI 边界：tbb::detail::d1 类型  ←→  tbb::detail::r1 导出运行时函数
  ▼
进程级 oneTBB 运行时（TBB::tbb）
  ├─ governor：外部线程的 TLS 与按需初始化
  ├─ market：跨 arena 的并发需求、优先级和配额分派
  ├─ arena / slot：局部任务池、FIFO/resume 流、外部线程与 worker 槽位
  ├─ task_dispatcher：本地取任务、全局接收、窃取、执行/取消
  └─ thread_dispatcher / RML：worker 生命周期与资源管理接口
  ▼
操作系统线程、CPU 拓扑、NUMA、动态依赖
  ├─ tbbbind + hwloc（可选的动态拓扑能力）
  └─ TCM（可选的跨并行运行时资源协调）
```

这是从 `src/tbb/governor.h`、`market.h`、`arena.h`、`task_dispatcher.h`、`thread_dispatcher.h` 的类型关系和注释归纳出的架构图，而非保证的稳定内部 ABI。`detail` 命名空间是实现细节；应用不能依赖其中的类布局、调度顺序或任务窃取策略。

### 4.2 任务调度路径

1. 应用以算法调用、`task_group::run`、`task_arena::enqueue` 或 Flow Graph 提交工作。
2. 外部线程首次进入时，`governor` 建立线程本地的调度上下文；`task_arena` 可以限制并发槽位、绑定 NUMA 节点/核类型，或附着到当前 arena。
3. `market` 维护各 arena 按优先级提出的 worker 需求，并受 `global_control` 等进程内控制影响。
4. `task_dispatcher` 优先从局部任务池执行；无可执行本地工作时，从全局流/邮箱接收或窃取其他工作。`arena` 中另有 FIFO、恢复任务等流，以保障外部入队任务最终能够被服务。
5. `task_group_context` 负责取消范围和异常状态。等待时，调用线程可能参与执行任务，因此等待并不必然等价于阻塞一个 OS 线程。

这套模型解释了两个容易误用的点：第一，嵌套并行不应按“每层各开 N 个线程”理解，而应让同一调度器协调；第二，提交异步工作后若没有定义数据所有权、取消与等待边界，调度器不能替应用修复悬垂引用或业务级竞态。

### 4.3 并发资源控制的层级

| 层级 | 主要接口 | 适用问题 | 不应期待的能力 |
| --- | --- | --- | --- |
| 进程内全局 | `global_control` | 限制允许并行度、线程栈大小、异常终止策略、离开策略 | 不能控制同进程另一份静态/动态 oneTBB 副本。 |
| 调度域 | `task_arena`、`constraints` | 隔离子系统、限制 arena 并发、选择 NUMA/核类型 | 不是硬实时调度器；约束成功与否依赖运行时拓扑能力。 |
| 任务集合 | `task_group`、`task_group_context` | 生命周期、等待、取消、异常边界 | 不自动解决任务访问同一可变对象的同步问题。 |
| 图节点 | Flow Graph 的并发度/limiter/resource limiter | 多阶段依赖、背压与资源令牌 | 图结构正确不代表业务操作天然线程安全。 |

## 5. 公共能力与选型指引

### 5.1 并行算法与范围

核心算法包括 `parallel_for`、`parallel_for_each`、`parallel_invoke`、`parallel_reduce`、`parallel_scan`、`parallel_sort` 和 `parallel_pipeline`。`blocked_range`、二维/三维/ND range 描述切分边界，`partitioner` 决定切分与亲和等策略。这是绝大多数首次落地的首选：算法层表达清晰，运行时细节暴露最少。

适合：迭代独立或可归约的 CPU 计算、批处理、图像/几何/数值内核、排序和过滤。
谨慎：每项计算只有数十纳秒、循环体受锁或 I/O 主导、迭代间有顺序写依赖、性能关键在 GPU 或远端服务。

### 5.2 任务、arena 与取消

`task_group` 面向不规则/递归任务；`task_arena` 为子系统设定并发边界和运行位置；`task_scheduler_observer` 用于观测线程进入/退出。`task_group_context` 提供取消和异常传播边界。对于复杂任务树，应首先定义：任务输入是否在 `wait` 前保持有效、异常如何回传、取消后哪些副作用需要补偿，以及谁拥有最终 `wait` 的责任。

本快照还包含动态依赖任务、单任务等待、并行 phase、核心类型选择等较新的能力；其支持级别需以对应 RFC 和正式 API 标记为准。不要仅因头文件可见就把带 `TBB_PREVIEW_*` 宏的能力固化进对外 SDK。

### 5.3 Flow Graph

`flow_graph.h` 提供 sender/receiver 抽象，以及 `function_node`、`multifunction_node`、`continue_node`、`buffer_node`、`queue_node`、`limiter_node`、`join_node`、`async_node`、`composite_node` 等节点。它适合“数据到达—变换—分支/汇聚—限流”的有向工作流，特别是阶段速度不一致且需要表达背压的场景。

其代价是拓扑、消息生命周期与节点并发策略都变成可维护对象。简单线性循环不应为“看起来更异步”而改造成图；反之，手写多个条件变量和队列的流水线通常值得评估 Flow Graph。

### 5.4 并发容器、同步与内存

公共头覆盖 `concurrent_hash_map`、`concurrent_queue`、`concurrent_priority_queue`、`concurrent_vector`、并发有序/无序 map/set、`enumerable_thread_specific`，以及 spin/queueing/RW/mutex 系列锁。它们解决的是容器或短临界区的线程安全，不替代容量控制、业务事务一致性或无锁正确性证明。

内存能力分为显式使用的 `scalable_allocator`、`cache_aligned_allocator`、`tbb_allocator` 与可选的全局分配代理。应优先将分配器限定在明确的高竞争热点；全进程代理需要与现有 jemalloc/tcmalloc、Sanitizer、插件和跨 CRT 分配/释放关系一并压测。

### 5.5 拓扑、NUMA 与异构核

`info.h` 和 `task_arena::constraints` 可查询/利用 NUMA 节点、核类型和每核线程数；预览 NUMA 分配接口可指定内存位置。该能力在多插槽或混合核服务器上可能有价值，但必须带着真实访问模式测量：错误的亲和性或过细的 arena 划分可能降低调度弹性。`tbbbind` 借助 hwloc 在运行时提供这些能力，因此部署包不能漏掉匹配的动态库。

## 6. 构建、发布和消费模型

### 6.1 支持基线

源码顶层 CMake 要求 `3.5.0...3.31.3`；非 MSVC 环境未指定时以 C++11 为默认基线，关闭 GNU 扩展。`SYSTEM_REQUIREMENTS.md` 所列主支持面涵盖 Windows、Linux、macOS 与 Android，并列出 GCC 8–16、Clang 7–22、MSVC 14.2/14.3/14.5 和 oneAPI 编译器。实际项目应以自己的 OS、编译器、标准库和 CPU 架构组合做验证，不要把“可配置”解读为“所有组合均同等受支持”。

### 6.2 推荐集成方式

```cmake
find_package(TBB REQUIRED COMPONENTS tbb)
target_link_libraries(my_target PRIVATE TBB::tbb)
```

建议：

- 默认使用共享库。Linux 通过 `libtbb.so.12` SONAME 保持二进制主版本兼容；Windows 主库名为 `tbb12.dll`。运行时仍应通过 `TBB_runtime_version()` 与 `TBB_runtime_interface_version()` 在诊断中记录实际加载版本。
- 优先按需包含具体头文件，如 `<oneapi/tbb/parallel_for.h>`；只有确需全量接口时才用 `<oneapi/tbb.h>`。
- 将 oneTBB 作为全项目统一依赖，不让不同插件、SDK 或静态库各自私带一个副本。
- 发布产物中保持 `tbb` 与所需的 `tbbbind` 变体相邻，并在干净机器/容器里验证动态加载路径。

### 6.3 静态链接为何是高风险例外

源码和官方文档都明确将静态构建标为“强烈不建议且支持有限”。原因并非静态归档无法链接，而是一个进程中出现多个 oneTBB 副本时，每个副本可能创建自己的 worker 池，产生过度订阅、缓存争用和难诊断的正确性问题；携带内部调度状态的 `task_arena`、`task_group`、Flow Graph 等对象跨副本传递属于未定义行为。

静态构建还会禁用或削弱若干能力：`tbbbind` 动态拓扑代理不构建，`info`/`constraints` 的拓扑功能不可用；`tbbmalloc_proxy` 不构建；TCM 无法动态加载；IPO 不生效；相关测试覆盖也会减少。只有在能证明整个进程只有一个副本、不会跨组件传递 oneTBB 对象、且已完成正确性与性能验证时，才考虑此路径。

### 6.4 Bazel 与 Python

仓库提供 Bazel 文件和 Bzlmod 元数据，但 `Bazel.md` 明确说明其为社区支持、维护者不以其作为内部构建路径，且目标仅覆盖受限子集，可能缺少推荐优化/安全参数。需要 Bazel 时应把它作为独立集成项目验证，而非假定与 CMake 发行物等价。

`TBB4PY_BUILD` 可构建 Python 模块（依赖 Python 与 SWIG），这更接近打包/绑定工作流，不应与 C++ 运行时部署混为一谈。

## 7. 版本、API 与 ABI 策略

oneTBB 区分四种版本概念：规范版本、库发行版本、接口版本和 API 兼容版本。本快照的宏位于 `include/oneapi/tbb/version.h`：

| 标识 | 值 | 用途 |
| --- | ---: | --- |
| `TBB_VERSION_STRING` | `2023.2.0` | 编译所见的库发行版本。 |
| `ONETBB_SPEC_VERSION` | `104` | 实现宣称完整支持的 oneTBB 规范版本。 |
| `TBB_INTERFACE_VERSION` | `12200` | 头文件和导出接口的契约版本。 |
| `__TBB_BINARY_VERSION` | `12` | SONAME/DLL 名称使用的二进制兼容主版本。 |

为在次版本演进中维持二进制兼容，公开类型与导出函数分别使用 `detail::d1` 与 `detail::r1` 等版本化内部命名空间。它是实现 ABI 演化的机制，不是公共扩展点。消费者应只依赖文档化的 `oneapi::tbb`/`tbb` API；直接引用 `detail` 将绕过兼容承诺。

## 8. 质量保障与可维护性观察

测试划分为 `test/abi`、`test/conformance`、`test/tbb`、`test/tbbmalloc` 和公共辅助目录。CMake 将每个测试注册到 CTest，并给多数测试标记 `RUN_SERIAL`；CI 包含代码拼写/格式、文档、Python 模块、Linux/macOS/Windows、静态构建、TCM、示例和文档示例等作业。仓库还包括 CodeQL、Coverity 和 OpenSSF Scorecard 工作流配置。

这些信号表明项目对跨平台回归、ABI 和示例可构建性有系统化投入，但不能替代本项目的验证，原因包括：

1. 上游 CI 无法覆盖调用方的数据竞争、对象生命周期和业务取消语义。
2. 静态构建与拓扑相关功能被特别标记为覆盖较轻或依赖运行时环境。
3. oneTBB 是性能基础设施，CPU 型号、NUMA、容器配额、任务粒度和邻近运行时都会显著改变结果。
4. 本报告未执行构建或测试，因此不能据此声明该工作区在当前 Windows 环境可构建通过。

建议的最小验收矩阵：Debug/Release × 目标编译器 × 目标 OS；并另加 1 核、物理核数、受限 CPU 配额、嵌套并行、异常取消、长时间压力、NUMA（若启用）和与 OpenMP/Parallel STL 同进程的场景。性能报告至少记录吞吐、p50/p99 延迟、CPU 利用率、上下文切换、峰值 RSS、任务粒度和线程数。

## 9. 风险、限制与缓解措施

| 风险/限制 | 触发条件 | 后果 | 建议控制 |
| --- | --- | --- | --- |
| 多个 oneTBB 副本 | 静态链接、插件私带依赖、不同 SDK 各自打包 | 过度订阅、负载失衡、跨副本对象未定义行为 | 依赖收敛到一个共享库；启动诊断记录运行时版本；发布前检查依赖树。 |
| 预览 API 演进 | 定义 `TBB_PREVIEW_*` 宏或使用 RFC/实验接口 | 升级时源/ABI 行为变化 | 在适配层封装，并锁定版本与回归测试；默认不用作公开接口。 |
| 静态库能力退化 | `BUILD_SHARED_LIBS=OFF` | 拓扑、malloc proxy、TCM、IPO 和测试覆盖受影响 | 默认拒绝；需要时做进程级单副本审计与专项验证。 |
| 拓扑/hwloc 平台差异 | Windows、特殊 NUMA、混合核、绑定库缺失 | 约束无效、检测错误或运行时失败 | 打包 `tbbbind`；在目标硬件验证；为不可用拓扑准备退化策略。 |
| 与其他运行时竞争 CPU | oneTBB 与 OpenMP、PSTL、线程池、插件并存 | 线程数爆炸、尾延迟上升 | 统一并发预算；评估 TCM；在容器/配额环境实测。 |
| WASM 并行不确定 | Emscripten/Web Worker 嵌套调度 | 可能退化为串行或出现意外性能 | 使用 pthread/`PROXY_TO_PTHREAD` 路径并预初始化线程池；将其列为单独支持级别。 |
| `fork()` 语义 | POSIX 进程在运行时初始化后 fork | 运行时不受支持 | fork 前用 `task_scheduler_handle` 完成 worker 回收，或改为 spawn/exec 架构。 |
| 分配器代理冲突 | 全局 malloc 替换与 Sanitizer/其他 allocator/插件混用 | 崩溃、跨库释放错误、诊断复杂 | 先采用显式分配器；代理模式做隔离环境和工具链组合测试。 |
| Windows 安全/加载策略 | 运行时动态依赖加载 | 依赖替换或加载路径风险 | 生产 Windows 构建显式设置 `TBB_VERIFY_DEPENDENCY_SIGNATURE=ON`，并校验发布目录与签名策略。 |

已知限制还包括：Debug oneTBB 与某些 SYCL 场景的双库冲突、特定旧版 `libstdc++` Parallel STL 与 oneTBB 的接口问题，以及 Linux 动态 malloc 替换与拓扑 API 同时使用时需要设定环境变量的情形。它们都说明“能链接成功”不足以证明部署正确。

## 10. 工程采用决策

| 工作负载/约束 | 建议 | 起步接口 | 关键验证 |
| --- | --- | --- | --- |
| 大量独立 CPU 循环或可结合归约 | 推荐 | `parallel_for`、`parallel_reduce` | 粒度、加速比、数值归约顺序/精度。 |
| 不规则递归、分治或动态依赖 | 推荐，但设计任务生命周期 | `task_group`、`task_arena` | 取消、异常、嵌套、数据所有权。 |
| 多阶段 CPU 流水线与背压 | 推荐评估 | Flow Graph | 节点并发度、队列上限、端到端尾延迟。 |
| 需要线程安全容器 | 视访问模式采用 | `concurrent_*`、`enumerable_thread_specific` | 语义与内存成本；不要用其掩盖业务级竞态。 |
| 主要瓶颈为磁盘/网络/数据库 I/O | 通常不是首选 | 现有异步 I/O/协程框架 | oneTBB 可处理 CPU 后处理，不解决 I/O 等待本身。 |
| 需严格可复现顺序/硬实时响应 | 谨慎 | 受控串行或专用调度器 | 任务执行顺序不是稳定契约；必须定义确定性策略。 |
| 插件生态、静态 SDK、多运行时混用 | 先做依赖治理再决定 | 共享 `TBB::tbb` + TCM 评估 | 单副本、加载次序、资源配额、跨边界对象。 |
| 浏览器 WASM | 可行但列为实验/专用支持 | Emscripten 构建路径 | 真机浏览器、worker、退化为串行的回退能力。 |

## 11. 推荐落地路线

1. **依赖审计**：列出进程内所有线程运行时、TBB/oneTBB 版本、静态归档、插件边界、OpenMP 与 Parallel STL 使用点；拒绝未解释的第二个 oneTBB 副本。
2. **最小 PoC**：选取一个 CPU 占比高、结果可校验的循环，以 `parallel_for` 或 `parallel_reduce` 改造；保留串行实现用于 A/B 对比。
3. **正确性与取消**：加入 ThreadSanitizer（可用平台）、地址/未定义行为检测、异常注入、取消、重复运行和单核模式；明确所有异步任务的输入/输出寿命。
4. **性能定标**：从粗粒度开始，测 1 核到目标并行度曲线；观察吞吐、p99、CPU 利用率、迁核/上下文切换和内存；只在数据证明需要时调整 grain size、partitioner、arena 或 NUMA 约束。
5. **集成固化**：统一由 CMake `TBB::tbb` 引入，锁定版本/哈希，生成 SBOM；发布时验证运行时库、`tbbbind` 和许可证文件齐全。
6. **高级能力准入**：将 TCM、NUMA、分配代理、Flow Graph 和预览 API 逐项作为独立特性开关；每项附平台矩阵、回退策略和监控指标。

## 12. 面向维护者的后续调研问题

在尚未给出具体业务工作负载的前提下，以下问题会实质改变选型和参数，建议在 PoC 前澄清：

- 目标进程是否同时使用 OpenMP、Intel/LLVM Parallel STL、Eigen、PyTorch、游戏引擎或其他自带线程池的 SDK？
- 运行环境是台式机、NUMA 服务器、容器 CPU quota、Windows 服务、移动端，还是浏览器 WASM？
- 主要目标是吞吐、单请求尾延迟、能耗、内存占用，还是构建可移植性？可接受怎样的数值非确定性？
- 是否需要跨 DLL/插件传递 `task_arena`、`task_group`、Flow Graph 或分配器拥有的对象？若需要，能否保证所有边界使用同一个动态 oneTBB？
- 是否真的存在分配器瓶颈，还是锁竞争、缓存局部性、I/O 或算法复杂度才是主因？

## 参考资料

### 本仓库证据

- `README.md`：项目定位、官方文档入口、许可证和治理说明。
- `include/oneapi/tbb/version.h`：版本、规范、接口和二进制兼容宏。
- `include/oneapi/tbb.h` 与 `include/oneapi/tbb/`：公共 API 面和预览接口开关。
- `src/tbb/CMakeLists.txt`、`src/tbb/{governor,market,arena,task_dispatcher,thread_dispatcher}.h`：运行时模块与调度结构。
- `CMakeLists.txt`、`cmake/README.md`、`integration/`：默认构建/安装/包集成和静态链接约束。
- `SYSTEM_REQUIREMENTS.md`、`VERSIONING.md`、`RELEASE_NOTES.md`、`WASM_Support.md`：支持面、兼容策略、功能变化和平台限制。
- `test/` 与 `.github/workflows/ci.yml`：测试分类和持续集成覆盖面。

### 官方公开资料（访问于 2026-10-12）

- [oneTBB：静态链接](https://uxlfoundation.github.io/oneTBB/main/intro/static_linking.html)：共享单例、多个副本的性能/正确性风险，以及静态构建缺失能力。
- [oneTBB：已知限制](https://uxlfoundation.github.io/oneTBB/main/intro/limitations.html)：SYCL、`fork()`、Parallel STL、动态分配替换与拓扑 API 的限制。
- [oneTBB：快速开始后续步骤](https://uxlfoundation.github.io/oneTBB/GSG/next_steps.html)：`tbbbind`/hwloc 部署与混合 CPU、NUMA 支持说明。
- [oneTBB：版本信息规范](https://uxlfoundation.github.io/oneTBB/main/specification/source/configuration/version_information.html)：运行时版本查询接口的规范描述。
- [oneTBB：发布页](https://github.com/uxlfoundation/oneTBB/releases)：上游发行物与签名/哈希应复核的入口。

