# Flink内核阅读
## 1. Flink 源码结构与核心组件总览

### 1.1 主要源码模块
- **flink-core**：Flink 的核心模块，包含基础设施和通用组件。
- **flink-java**：Java API 模块，提供 Java 编程接口。
- **flink-streaming-java**：流处理模块，包含流处理 API 及相关组件。
- **flink-table**：表处理模块，提供表 API 和 SQL 支持。
- **flink-connector-\***：连接器模块，支持多种数据源和接收器。
- **flink-runtime**：运行时模块，包含调度器、任务管理器等。
- **flink-clients**：客户端模块，包含命令行工具和 REST API。
- **flink-examples**：示例模块，包含各类示例程序。

### 1.2 核心组件
- **StreamExecutionEnvironment**：流处理环境，管理流处理作业。
- **DataStream**：流数据抽象，表示数据流。
- **DataSet**：批处理数据抽象，表示数据集。
- **Transformations**：转换操作，如 map、filter、reduce 等。
- **Windows**：窗口机制，支持滚动、滑动、会话等窗口类型。
- **State**：状态管理，支持键控状态、操作符状态等。
- **Checkpoints**：检查点机制，支持作业容错与恢复。
- **Connectors**：连接器，支持 Kafka、HDFS、JDBC 等数据源和接收器。

---

## 2. Flink 流处理核心梳理
### 2.1 Cli启动流程
- **Flink CLI**：Flink 提供命令行工具 `flink`，用于提交和管理作业。
- **启动过程**：
  1. 用户通过命令行提交作业。
  2. CLI 解析命令并构建作业图。
  3. 作业图通过 REST API 提交到 JobManager。
  4. JobManager 接收作业并进行调度。
  
- **CliFrontend类**：处理命令行输入，解析参数，构建作业图，封装命令行接口，
按顺序加载GenericCLI,Yarn,DefaultCLI，挨个判断是否active
- 获取相应的命令行配置
- 获取有效的配置：
  HA的id、Target(本地、yarn、kubernetes)、配置文件、JobMannger内存、TaskManager内存、TaskManager数量、slots数量等

### 2.2 提交作业流程
- **Job Submission**：作业提交过程包括以下步骤：
  1. **创建 StreamExecutionEnvironment**：用户代码中创建流处理环境。
  2. **定义数据流**：通过 DataStream API 定义数据源、转换操作和数据接收器。
  3. **生成 JobGraph**：将数据流转换为 JobGraph，表示作业的执行计划。
  4. **提交 JobGraph**：通过 REST API 将 JobGraph 提交到 JobManager。
  5. **调度执行**：JobManager 接收 JobGraph，进行任务调度和资源分配。
- 生成JobGraph的流程（大量使用工厂模式）
  - PipelineExcutorUtils
    - createPipelineAndRun
      - 创建Pipeline
      - 运行Pipeline
    - createJobGraph
      - 创建JobGraph
        - 获取用户代码的类加载器
        - 创建ExecutionConfig
        - 创建StreamGraphGenerator
            - StreamGraphGenerator.generate()
              - 生成StreamGraph
                - StreamGraphGenerator.transform()
                  - 转换DataStream操作为StreamNode节点
                    - DataStream API调用transform()方法，创建Transformation对象，并添加到StreamGraph中
                - 设置StreamGraph属性，如并行度、状态后端等
              - 返回生成的StreamGraph对象
        - 将StreamGraph转换为JobGraph
          - StreamGraph.toJobGraph()
            - 创建JobGraph对象
            - 遍历StreamGraph中的StreamNode节点，创建对应的JobVertex，并添加到JobGraph中
            - 设置JobGraph属性，如作业名称、检查点配置等
          - 返回生成的JobGraph对象
### 2.3 创建ResourceManager过程
- **ResourceManager**：资源管理器负责管理集群资源，协调 TaskManager 的注册和资源分配。
- **创建过程**：
  1. JobManager 启动时，初始化 ResourceManager。
  2. ResourceManager 根据配置选择具体的实现，如 StandaloneResourceManager、YarnResourceManager 等。
  3. ResourceManager 启动并监听 TaskManager 的注册请求。
  4. TaskManager 启动时，向 ResourceManager 注册自身信息。
  5. ResourceManager 接收注册请求，记录 TaskManager 信息，并分配资源。 
### 2.4 创建JobManager过程（三大组件：ResourceManager、Dispatcher、JobManager）
- **JobManager**：作业管理器负责作业调度、任务分配和故障恢复。
- **创建过程**：
  1. 启动时，加载配置文件，初始化日志系统。
  2. 创建 ActorSystem，用于管理分布式通信。
  3. 初始化 ResourceManager，负责资源管理和 TaskManager 注册。
  4. 初始化 Dispatcher，负责作业调度和生命周期管理。
  5. 启动 REST 服务，提供作业提交和管理接口。
  6. 等待 TaskManager 注册，并进行资源分配和任务调度。
### 2.5 启动Dispatcher过程
- **Dispatcher**：调度器负责接收作业提交请求，管理作业生命周期
- 其使用**RPC框架Akka**进行通信
- **启动过程**：
  1. JobManager 启动时，初始化 Dispatcher。
  2. Dispatcher 启动并监听作业提交请求。
  3. 用户通过 CLI 或 REST API 提交作业，Dispatcher 接收请求。
  4. Dispatcher 将作业提交给 JobManager，JobManager 进行调度和执行。
### 2.6 创建心跳服务：TaskManager、JobMaster
- **心跳服务**：用于监控 TaskManager 和 JobMaster 的健康状态，确保集群稳定运行。
- **创建过程**：
  1. JobManager 启动时，初始化心跳服务。
  2. TaskManager 启动时，向 JobManager 注册，并启动心跳机制。
  3. JobMaster 启动时，向 JobManager 注册，并启动心跳机制。
  4. TaskManager 和 JobMaster 定期发送心跳信号给 JobManager。
  5. JobManager 接收心跳信号，更新状态信息，检测节点是否存活。
  6. 如果某个节点长时间未发送心跳信号，JobManager 将其标记为失效，并进行故障恢复。
### 2.7 启动slotManager （真正实现资源分配的模块）
- **SlotManager**：负责管理 TaskManager 的槽位资源，进行资源分配和回收。
- **启动过程**：
  1. JobManager 启动时，初始化 SlotManager。
  2. SlotManager 启动并监听 TaskManager 的注册请求。
  3. TaskManager 启动时，向 SlotManager 注册自身的槽位信息。
  4. SlotManager 接收注册请求，记录 TaskManager 的槽位信息。
  5. 当作业需要资源时，SlotManager 根据作业需求分配槽位。
  6. 作业完成后，SlotManager 回收槽位资源，供其他作业使用。
- ***重点:Slot的请求在旧版本(1.14之前)是由ResourceManager完成的,1.15之后是TaskManager进行请求***
- **官方解读SlotManager**:槽管理器负责维护所有注册的任务管理器槽，它们的分配和所有挂起的槽请求的视图，
每当一个新的槽被注册或已分配的槽被释放时，然后它试图满足另一个挂起的槽请求，
每当没有足够的槽可用时，槽管理器将通过{@link resourceallocator# declareresourcenented}
通知资源管理器，以便释放资源并避免资源泄漏。空闲任务管理器（其槽位当前未被使用的任务管理器）和挂起的槽位请求超时，分别触发它们的释放和失败
- 
  | 对比维度               | 旧版本（如 Flink 1.12，文档描述）                                         | 新版本（Flink 1.13+）                                   | 核心差异原因                                      |
  |------------------------|--------------------------------------------------------------------------|--------------------------------------------------------|---------------------------------------------------|
  | **调度模式**           | 集中式调度（以 ResourceManager 为核心）                                 | 分布式调度（JobMaster 与 TaskManager 直接交互）         | 提升大规模集群下的调度效率，减少 ResourceManager 瓶颈 |
  | **Slot 请求发起方**    | JobMaster 通过 SlotPool 向 ResourceManager 发送请求（`AllocateResourceRequest`） | JobMaster 通过 SlotPool 直接向 TaskManager 发送请求     | 减少中间环节，加快响应速度                         |
  | **分配决策主体**       | ResourceManager 统一决策（检查全局资源，匹配空闲 Slot）                 | TaskManager 自主决策（根据自身空闲资源直接判断是否分配） | 适配容器化环境（如 K8s），发挥 TaskManager 对本地资源的感知优势 |
  | **ResourceManager 角色** | 核心调度者：负责接收请求、分配资源、通知 TaskManager 执行               | 资源注册中心：仅维护 TaskManager 注册信息和资源总量，不参与具体分配决策 | 与外部调度器（如 K8s scheduler）解耦，避免功能重叠   |
  | **TaskManager 角色**   | 被动执行者：仅接收 ResourceManager 指令，执行 Slot 分配                 | 主动决策者：接收请求后自主判断，直接响应是否分配 Slot    | 支持细粒度资源隔离（如按请求规格动态分配），提升灵活性 |
  | **典型调用链路**       | JobMaster → ResourceManager → TaskManager                              | JobMaster → TaskManager                                 | 缩短调用链路，减少网络通信开销                     |
  | **适用场景**           | 传统 YARN 集群、中小规模部署                                             | 大规模集群、K8s 容器化环境、高频作业提交场景             | 适应云原生架构趋势，满足多样化部署需求             |
###2.8 启动TaskManager(Yarn模式)
- **TaskManager**：任务管理器负责执行具体的任务，管理槽位资源和任务生命周期。
- 内部通过***TaskExecutorRunner***启动
- 最终通过RPC服务启动TaskExecutor,找它的onstart方法
- 向ResourceManager注册自己的信息(设计模式可以借鉴,监听者模式) || 向JobMaster注册自己的信息(心跳机制)
- 

客户端提交  
├─ CliFrontend.parseParameters() → 解析命令  
├─ StreamExecutionEnvironment.execute() → 生成StreamGraph
├─ StreamingJobGraphGenerator.createJobGraph() → 转换为JobGraph
└─ ClusterClient.submitJob() → 提交到集群

集群启动  
├─ YarnApplicationMasterRunner → 启动JobManager组件  
├─ JobMaster → 生成ExecutionGraph  
└─ SlotPool.requestSlot() → 申请资源

任务部署  
├─ JobMaster.deployTask() → 发送部署描述符  
├─ TaskExecutor.submitTask() → 创建Task  
└─ Task.run() → 启动算子链执行数据处理  

      
