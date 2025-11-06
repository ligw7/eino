# 图式编排（Graph）

<cite>
**本文档引用的文件**
- [compose/graph.go](file://compose/graph.go)
- [compose/generic_graph.go](file://compose/generic_graph.go)
- [compose/dag.go](file://compose/dag.go)
- [compose/pregel.go](file://compose/pregel.go)
- [compose/graph_node.go](file://compose/graph_node.go)
- [compose/branch.go](file://compose/branch.go)
- [compose/graph_run.go](file://compose/graph_run.go)
- [compose/types.go](file://compose/types.go)
- [compose/state.go](file://compose/state.go)
- [compose/types_composable.go](file://compose/types_composable.go)
- [adk/react.go](file://adk/react.go)
- [flow/agent/multiagent/host/compose.go](file://flow/agent/multiagent/host/compose.go)
- [flow/agent/react/react.go](file://flow/agent/react/react.go)
</cite>

## 目录
1. [简介](#简介)
2. [核心架构](#核心架构)
3. [Graph结构体详解](#graph结构体详解)
4. [节点管理](#节点管理)
5. [边连接机制](#边连接机制)
6. [分支控制](#分支控制)
7. [运行模式](#运行模式)
8. [状态管理](#状态管理)
9. [复杂决策流程示例](#复杂决策流程示例)
10. [性能特性](#性能特性)
11. [与Chain模式对比](#与chain模式对比)
12. [最佳实践](#最佳实践)

## 简介

Eino框架的Graph编排模式是一种基于有向无环图（DAG）的高级流程控制机制，它提供了比传统链式模式更强大和灵活的工作流编排能力。Graph模式支持复杂的控制流和数据流，能够实现基于条件的运行时分支决策、全局状态管理和并发执行。

Graph编排模式的核心优势在于：
- **灵活性**：支持复杂的控制流和数据流
- **可扩展性**：易于添加新的节点和分支
- **并发性**：支持并行执行多个独立路径
- **状态共享**：提供全局状态管理机制
- **类型安全**：编译时类型检查和验证

## 核心架构

Graph编排模式采用分层架构设计，主要包含以下核心组件：

```mermaid
graph TB
subgraph "Graph编排架构"
Graph[Graph[I, O]] --> GraphStruct[graph结构体]
GraphStruct --> Nodes[nodes映射]
GraphStruct --> ControlEdges[controlEdges映射]
GraphStruct --> DataEdges[dataEdges映射]
GraphStruct --> Branches[branches映射]
Nodes --> GraphNode[graphNode]
GraphNode --> ComposableRunnable[composableRunnable]
ControlEdges --> EdgeConnection[边连接]
DataEdges --> DataFlow[数据流]
Branches --> GraphBranch[GraphBranch]
GraphStruct --> StateManagement[状态管理]
StateManagement --> GlobalState[全局状态]
StateManagement --> LocalState[局部状态]
end
```

**图表来源**
- [compose/graph.go](file://compose/graph.go#L57-L90)
- [compose/graph_node.go](file://compose/graph_node.go#L60-L70)

**章节来源**
- [compose/graph.go](file://compose/graph.go#L57-L90)
- [compose/generic_graph.go](file://compose/generic_graph.go#L89-L91)

## Graph结构体详解

`Graph[I, O]`结构体是整个编排系统的核心，它管理着所有节点、边和分支的关系：

### 内部结构组成

```mermaid
classDiagram
class Graph {
+nodes map[string]*graphNode
+controlEdges map[string][]string
+dataEdges map[string][]string
+branches map[string][]*GraphBranch
+startNodes []string
+endNodes []string
+stateType reflect.Type
+stateGenerator func(ctx) any
+expectedInputType reflect.Type
+expectedOutputType reflect.Type
+genericHelper *genericHelper
+compiled bool
+buildError error
}
class graphNode {
+cr *composableRunnable
+g AnyGraph
+nodeInfo *nodeInfo
+executorMeta *executorMeta
+instance any
+opts []GraphAddNodeOpt
}
class GraphBranch {
+invoke func(ctx, input) (output, error)
+collect func(ctx, input) (output, error)
+inputType reflect.Type
+endNodes map[string]bool
+idx int
+noDataFlow bool
}
Graph --> graphNode
Graph --> GraphBranch
```

**图表来源**
- [compose/graph.go](file://compose/graph.go#L57-L90)
- [compose/graph_node.go](file://compose/graph_node.go#L60-L70)
- [compose/branch.go](file://compose/branch.go#L40-L50)

### 关键字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| `nodes` | `map[string]*graphNode` | 存储所有节点及其配置信息 |
| `controlEdges` | `map[string][]string` | 控制依赖关系，决定执行顺序 |
| `dataEdges` | `map[string][]string` | 数据流连接，传递计算结果 |
| `branches` | `map[string][]*GraphBranch` | 条件分支定义 |
| `startNodes` | `[]string` | 起始节点列表 |
| `endNodes` | `[]string` | 结束节点列表 |
| `stateType` | `reflect.Type` | 全局状态类型 |
| `stateGenerator` | `func(ctx) any` | 状态生成函数 |

**章节来源**
- [compose/graph.go](file://compose/graph.go#L57-L90)

## 节点管理

Graph提供了丰富的节点管理功能，支持多种类型的组件作为节点：

### 节点类型分类

```mermaid
graph LR
subgraph "节点类型"
ComponentNodes[组件节点]
LambdaNodes[Lambda节点]
GraphNodes[Graph节点]
PassthroughNodes[透传节点]
ComponentNodes --> Embedding[嵌入节点]
ComponentNodes --> Retriever[检索节点]
ComponentNodes --> ChatModel[聊天模型节点]
ComponentNodes --> ToolsNode[工具节点]
LambdaNodes --> Invokable[可调用Lambda]
LambdaNodes --> Streamable[可流式Lambda]
LambdaNodes --> Collectable[可收集Lambda]
LambdaNodes --> Transformable[可变换Lambda]
GraphNodes --> SubGraph[子图节点]
GraphNodes --> Chain[链式节点]
PassthroughNodes --> PassThrough[透传节点]
end
```

**图表来源**
- [compose/graph.go](file://compose/graph.go#L296-L420)

### 节点添加方法

Graph提供了专门的方法来添加不同类型的节点：

| 方法 | 用途 | 示例 |
|------|------|------|
| `AddEmbeddingNode()` | 添加嵌入组件节点 | 文本向量化 |
| `AddRetrieverNode()` | 添加检索组件节点 | 文档检索 |
| `AddChatModelNode()` | 添加聊天模型节点 | 大语言模型调用 |
| `AddToolsNode()` | 添加工具节点 | 工具调用 |
| `AddLambdaNode()` | 添加Lambda函数节点 | 自定义逻辑处理 |
| `AddGraphNode()` | 添加子图或链式节点 | 复杂工作流 |
| `AddPassthroughNode()` | 添加透传节点 | 数据传递 |

**章节来源**
- [compose/graph.go](file://compose/graph.go#L296-L420)

## 边连接机制

边连接是Graph编排模式的核心机制，负责建立节点间的控制依赖和数据流关系：

### 边的类型

```mermaid
graph TD
subgraph "边连接机制"
Start[START] --> ControlEdge1[控制边]
ControlEdge1 --> Node1[节点1]
Node1 --> DataEdge1[数据边]
DataEdge1 --> Node2[节点2]
Node2 --> End[END]
Node1 -.-> Branch1[分支1]
Branch1 --> Path1[路径1]
Branch1 --> Path2[路径2]
ControlEdge1 -.->|控制依赖| Node1
DataEdge1 -.->|数据传递| Node2
Branch1 -.->|条件分支| Path1
Branch1 -.->|条件分支| Path2
end
```

**图表来源**
- [compose/graph.go](file://compose/graph.go#L232-L294)

### AddEdge方法详解

`AddEdge`方法用于建立节点间的连接：

```mermaid
sequenceDiagram
participant Client as 客户端
participant Graph as Graph实例
participant Validator as 验证器
participant EdgeManager as 边管理器
Client->>Graph : AddEdge(startNode, endNode)
Graph->>Validator : 检查节点存在性
Validator-->>Graph : 验证结果
Graph->>EdgeManager : 建立控制边和数据边
EdgeManager-->>Graph : 连接成功
Graph-->>Client : 返回错误或成功
```

**图表来源**
- [compose/graph.go](file://compose/graph.go#L232-L294)

### 边验证规则

| 规则 | 描述 | 错误处理 |
|------|------|----------|
| 节点存在性 | 起始和结束节点必须已添加 | 抛出节点不存在错误 |
| 循环检测 | 不允许形成循环依赖 | 检测并阻止循环 |
| 类型匹配 | 输入输出类型必须兼容 | 类型转换或报错 |
| 控制依赖 | 控制边不能同时为数据边 | 报告冲突错误 |

**章节来源**
- [compose/graph.go](file://compose/graph.go#L232-L294)

## 分支控制

Graph编排模式提供了强大的分支控制机制，支持基于条件的运行时决策：

### GraphBranch结构

```mermaid
classDiagram
class GraphBranch {
+invoke func(ctx, input) (output, error)
+collect func(ctx, input) (output, error)
+inputType reflect.Type
+endNodes map[string]bool
+idx int
+noDataFlow bool
+GetEndNode() map[string]bool
}
class BranchCondition {
<<interface>>
+condition(ctx, input) (endNode, error)
}
class MultiBranchCondition {
<<interface>>
+condition(ctx, input) (endNodes, error)
}
GraphBranch --> BranchCondition
GraphBranch --> MultiBranchCondition
```

**图表来源**
- [compose/branch.go](file://compose/branch.go#L40-L50)

### 分支类型

| 分支类型 | 条件函数 | 用途 | 示例场景 |
|----------|----------|------|----------|
| 单一分支 | `GraphBranchCondition[T]` | 单一条件判断 | 检查输入内容类型 |
| 多分支 | `GraphMultiBranchCondition[T]` | 多个条件选择 | 复杂业务逻辑分支 |
| 流式分支 | `StreamGraphBranchCondition[T]` | 流式输入条件 | 实时内容分析 |
| 流式多分支 | `StreamGraphMultiBranchCondition[T]` | 流式多条件 | 并行内容处理 |

### 分支决策流程

```mermaid
flowchart TD
Start([开始分支]) --> Condition{条件判断}
Condition --> |满足条件A| PathA[路径A]
Condition --> |满足条件B| PathB[路径B]
Condition --> |满足条件C| PathC[路径C]
Condition --> |无匹配| DefaultPath[默认路径]
PathA --> Merge[合并路径]
PathB --> Merge
PathC --> Merge
DefaultPath --> Merge
Merge --> End([结束])
```

**图表来源**
- [compose/branch.go](file://compose/branch.go#L142-L172)

**章节来源**
- [compose/branch.go](file://compose/branch.go#L40-L50)

## 运行模式

Graph编排模式支持两种主要的运行模式，每种模式适用于不同的应用场景：

### runTypePregel模式

Pregel模式适用于大规模图处理任务，支持循环图和任意前驱触发：

```mermaid
graph TD
subgraph "Pregel运行模式"
Input[输入] --> Node1[节点1]
Node1 --> Node2[节点2]
Node2 --> Node3[节点3]
Node3 --> Node1
Node1 -.->|任意前驱触发| Node4[节点4]
Node2 -.->|任意前驱触发| Node5[节点5]
Node3 -.->|任意前驱触发| Node6[节点6]
end
```

**图表来源**
- [compose/pregel.go](file://compose/pregel.go#L21-L23)

### runTypeDAG模式

DAG模式代表严格的有向无环图，适用于任务表示为有向无环图的场景：

```mermaid
graph TD
subgraph "DAG运行模式"
Input[输入] --> Node1[节点1]
Node1 --> Node2[节点2]
Node2 --> Node3[节点3]
Node3 --> Output[输出]
Node1 -.->|所有前驱完成| Node4[节点4]
Node2 -.->|所有前驱完成| Node5[节点5]
Node3 -.->|所有前驱完成| Node6[节点6]
end
```

**图表来源**
- [compose/dag.go](file://compose/dag.go#L23-L40)

### 运行模式对比

| 特性 | Pregel模式 | DAG模式 |
|------|------------|---------|
| 循环支持 | ✅ 支持 | ❌ 不支持 |
| 触发模式 | 任意前驱触发 | 所有前驱完成触发 |
| 性能特点 | 更灵活但可能较慢 | 更高效但受限 |
| 应用场景 | 复杂图处理 | 严格线性流程 |
| 类型检查 | 编译时检查 | 运行时检查 |

**章节来源**
- [compose/graph.go](file://compose/graph.go#L639-L681)
- [compose/dag.go](file://compose/dag.go#L23-L40)
- [compose/pregel.go](file://compose/pregel.go#L21-L23)

## 状态管理

Graph编排模式提供了强大的全局状态管理机制，支持节点间的数据共享：

### WithState选项

通过`WithGenLocalState`选项可以启用全局状态管理：

```mermaid
sequenceDiagram
participant Graph as Graph实例
participant StateManager as 状态管理器
participant Node1 as 节点1
participant Node2 as 节点2
participant State as 全局状态
Graph->>StateManager : 初始化状态生成器
StateManager->>State : 创建初始状态
StateManager->>Node1 : 注入状态
Node1->>State : 读取/修改状态
State-->>Node1 : 返回状态值
Node1->>StateManager : 完成处理
StateManager->>Node2 : 传递更新后的状态
Node2->>State : 继续处理
```

**图表来源**
- [compose/generic_graph.go](file://compose/generic_graph.go#L33-L40)
- [compose/state.go](file://compose/state.go#L34-L37)

### 状态处理器

| 处理器类型 | 触发时机 | 用途 | 示例 |
|------------|----------|------|------|
| `StatePreHandler` | 节点执行前 | 状态预处理 | 记录访问日志 |
| `StatePostHandler` | 节点执行后 | 状态后处理 | 更新统计信息 |
| `StreamStatePreHandler` | 流式处理前 | 流式状态处理 | 实时状态监控 |
| `StreamStatePostHandler` | 流式处理后 | 流式状态处理 | 流式统计汇总 |

### 状态并发安全

```mermaid
graph LR
subgraph "状态并发控制"
Thread1[线程1] --> Mutex[互斥锁]
Thread2[线程2] --> Mutex
Thread3[线程3] --> Mutex
Mutex --> State[状态对象]
State --> SharedData[共享数据]
end
```

**图表来源**
- [compose/state.go](file://compose/state.go#L34-L37)

**章节来源**
- [compose/generic_graph.go](file://compose/generic_graph.go#L33-L40)
- [compose/state.go](file://compose/state.go#L34-L37)

## 复杂决策流程示例

以下是一个基于Graph编排模式的复杂决策流程示例，展示了一个智能Agent如何根据LLM输出决定是检索文档还是调用外部工具：

### Agent决策流程

```mermaid
graph TD
subgraph "智能Agent决策流程"
Input[用户输入] --> LLM[大语言模型]
LLM --> Decision{决策分支}
Decision --> |需要文档| Retrieval[文档检索]
Decision --> |需要工具| ToolCall[工具调用]
Decision --> |直接回答| DirectAnswer[直接回复]
Retrieval --> Embedding[向量化]
Embedding --> VectorDB[向量数据库]
VectorDB --> Context[上下文构建]
ToolCall --> ToolNode[工具节点]
ToolNode --> ToolExecution[工具执行]
Context --> FinalResponse[最终响应]
ToolExecution --> FinalResponse
DirectAnswer --> FinalResponse
FinalResponse --> Output[输出结果]
end
```

**图表来源**
- [adk/react.go](file://adk/react.go#L104-L154)
- [flow/agent/multiagent/host/compose.go](file://flow/agent/multiagent/host/compose.go#L189-L222)

### 实现代码结构

这个复杂流程的实现涉及多个关键组件：

| 组件 | 功能 | 实现方式 |
|------|------|----------|
| 主机代理 | 接收用户输入并做出决策 | `addHostAgent()` |
| 专家代理 | 专门处理特定领域的任务 | `addSpecialistAgent()` |
| 分支逻辑 | 基于工具调用判断的分支 | `addDirectAnswerBranch()` |
| 多专家分支 | 处理多个专家的协作 | `addMultiSpecialistsBranch()` |
| 汇总节点 | 合并多个专家的回答 | `addMultiIntentsSummarizeNode()` |

### 流程控制机制

```mermaid
sequenceDiagram
participant User as 用户
participant Host as 主机代理
participant Branch as 分支控制器
participant Specialist as 专家代理
participant Summarizer as 汇总器
User->>Host : 提交请求
Host->>Branch : 分析是否需要工具调用
Branch->>Specialist : 如果需要，路由到专家
Specialist->>Branch : 返回专家结果
Branch->>Summarizer : 汇总多个结果
Summarizer->>Host : 最终综合答案
Host->>User : 返回结果
```

**图表来源**
- [flow/agent/react/react.go](file://flow/agent/react/react.go#L267-L303)

**章节来源**
- [adk/react.go](file://adk/react.go#L104-L154)
- [flow/agent/multiagent/host/compose.go](file://flow/agent/multiagent/host/compose.go#L189-L222)
- [flow/agent/react/react.go](file://flow/agent/react/react.go#L267-L303)

## 性能特性

Graph编排模式在性能方面具有显著优势：

### 类型检查优化

```mermaid
graph LR
subgraph "编译时类型检查"
SourceCode[源代码] --> TypeCheck[类型检查]
TypeCheck --> OptimizedCode[优化代码]
TypeCheck --> |编译时| ErrorDetection[错误检测]
TypeCheck --> |编译时| TypeSafety[类型安全]
end
```

### 并发执行能力

| 特性 | 描述 | 性能影响 |
|------|------|----------|
| 并行节点 | 多个独立节点可并行执行 | 显著提升吞吐量 |
| 流式处理 | 支持流式数据处理 | 减少内存占用 |
| 异步操作 | 非阻塞I/O操作 | 提高响应速度 |
| 状态缓存 | 全局状态本地化缓存 | 减少状态访问开销 |

### 流式数据传递

Graph编排模式支持高效的流式数据传递机制：

```mermaid
sequenceDiagram
participant Producer as 生产者节点
participant Channel as 通道
participant Consumer as 消费者节点
Producer->>Channel : 发送数据块
Channel->>Consumer : 流式传递
Consumer->>Channel : 处理完成确认
Channel->>Producer : 下一块准备就绪
Note over Producer,Consumer : 支持实时数据流处理
```

**章节来源**
- [compose/graph_run.go](file://compose/graph_run.go#L810-L853)

## 与Chain模式对比

Graph编排模式与传统的Chain模式在多个方面存在显著差异：

### 架构对比

| 方面 | Chain模式 | Graph模式 |
|------|-----------|-----------|
| 流程结构 | 线性序列 | 有向无环图 |
| 分支能力 | 有限分支 | 强大分支控制 |
| 并发性 | 串行执行 | 并行执行 |
| 状态管理 | 局部状态 | 全局状态 |
| 复杂度 | 简单直观 | 复杂灵活 |

### 使用场景对比

```mermaid
graph TD
subgraph "Chain模式适用场景"
LinearProcess[线性处理流程]
SimplePipeline[简单管道]
SequentialTasks[顺序任务]
end
subgraph "Graph模式适用场景"
ComplexWorkflow[复杂工作流]
ConditionalLogic[条件逻辑]
ParallelProcessing[并行处理]
DynamicRouting[动态路由]
end
LinearProcess --> ChainPattern[Chain模式]
SimplePipeline --> ChainPattern
SequentialTasks --> ChainPattern
ComplexWorkflow --> GraphPattern[Graph模式]
ConditionalLogic --> GraphPattern
ParallelProcessing --> GraphPattern
DynamicRouting --> GraphPattern
```

### 性能对比

| 指标 | Chain模式 | Graph模式 |
|------|-----------|-----------|
| 执行效率 | 高（线性） | 中等（图遍历） |
| 内存使用 | 低 | 中等 |
| 开发复杂度 | 低 | 高 |
| 调试难度 | 低 | 中等 |
| 扩展性 | 有限 | 优秀 |

**章节来源**
- [compose/types.go](file://compose/types.go#L28-L46)

## 最佳实践

基于对Eino框架Graph编排模式的深入分析，以下是推荐的最佳实践：

### 设计原则

1. **单一职责**：每个节点应该只负责一个明确的功能
2. **类型安全**：充分利用泛型和类型检查机制
3. **状态最小化**：尽量减少全局状态的使用
4. **错误处理**：完善的错误传播和恢复机制

### 性能优化建议

```mermaid
graph LR
subgraph "性能优化策略"
Parallelization[并行化处理]
Caching[状态缓存]
LazyLoading[延迟加载]
ResourcePool[资源池化]
Parallelization --> Throughput[提升吞吐量]
Caching --> Latency[降低延迟]
LazyLoading --> Memory[节省内存]
ResourcePool --> Efficiency[提高效率]
end
```

### 常见陷阱避免

| 陷阱 | 描述 | 解决方案 |
|------|------|----------|
| 循环依赖 | 节点间形成循环引用 | 使用DAG验证 |
| 状态竞争 | 多线程状态访问冲突 | 使用互斥锁 |
| 内存泄漏 | 未正确释放资源 | 实现资源清理 |
| 类型不匹配 | 输入输出类型不兼容 | 编译时检查 |

### 调试和监控

```mermaid
flowchart TD
Start[开始调试] --> EnableLogging[启用日志记录]
EnableLogging --> MonitorState[监控状态变化]
MonitorState --> TraceExecution[跟踪执行路径]
TraceExecution --> AnalyzePerf[分析性能瓶颈]
AnalyzePerf --> Optimize[优化改进]
Optimize --> Start
```

通过遵循这些最佳实践，可以充分发挥Graph编排模式的强大功能，构建高效、可靠的应用程序。