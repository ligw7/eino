# 组件系统 (Components) API

<cite>
**本文档中引用的文件**  
- [types.go](file://components/types.go)
- [model/interface.go](file://components/model/interface.go)
- [tool/interface.go](file://components/tool/interface.go)
- [prompt/interface.go](file://components/prompt/interface.go)
- [retriever/interface.go](file://components/retriever/interface.go)
- [embedding/interface.go](file://components/embedding/interface.go)
- [indexer/interface.go](file://components/indexer/interface.go)
- [document/interface.go](file://components/document/interface.go)
- [prompt/chat_template.go](file://components/prompt/chat_template.go)
- [model/option.go](file://components/model/option.go)
- [tool/option.go](file://components/tool/option.go)
- [prompt/option.go](file://components/prompt/option.go)
- [retriever/option.go](file://components/retriever/option.go)
- [embedding/option.go](file://components/embedding/option.go)
- [indexer/option.go](file://components/indexer/option.go)
- [document/option.go](file://components/document/option.go)
- [tool/utils/create_options.go](file://components/tool/utils/create_options.go)
- [tool/utils/error_handler.go](file://components/tool/utils/error_handler.go)
- [tool/utils/common.go](file://components/tool/utils/common.go)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件接口](#核心组件接口)
   - [模型 (Model)](#模型-model)
   - [工具 (Tool)](#工具-tool)
   - [提示模板 (Prompt)](#提示模板-prompt)
   - [检索器 (Retriever)](#检索器-retriever)
   - [嵌入 (Embedding)](#嵌入-embedding)
   - [索引器 (Indexer)](#索引器-indexer)
   - [文档 (Document)](#文档-document)
3. [Option 配置模式](#option-配置模式)
4. [自定义组件实现指南](#自定义组件实现指南)

## 简介
Eino 框架的 `components` 包提供了构建 AI 应用的核心组件系统。该系统采用接口驱动的设计，每个组件都定义了清晰的公共接口和构造函数，支持灵活的扩展和组合。本文档详细说明了 `model`、`tool`、`prompt`、`retriever`、`embedding`、`indexer` 和 `document` 等核心组件的 API，以及统一的 `Option` 配置模式。

## 核心组件接口

### 模型 (Model)

`model` 组件是 AI 应用的核心，负责生成文本响应。其主要接口定义在 `components/model/interface.go` 中。

**BaseChatModel** 接口定义了聊天模型的基础能力，包括同步生成和流式生成：

- `Generate(ctx context.Context, input []*schema.Message, opts ...Option) (*schema.Message, error)`：根据输入消息列表生成单条响应消息。
- `Stream(ctx context.Context, input []*schema.Message, opts ...Option) (*schema.StreamReader[*schema.Message], error)`：以流式方式生成响应消息。

**ChatModel** 接口（已弃用）在 `BaseChatModel` 基础上增加了 `BindTools(tools []*schema.ToolInfo) error` 方法，用于绑定工具。由于存在并发问题，推荐使用更安全的 `ToolCallingChatModel` 接口。

**ToolCallingChatModel** 接口提供了更安全的工具调用方式：
- `WithTools(tools []*schema.ToolInfo) (ToolCallingChatModel, error)`：返回一个绑定了指定工具的新实例，避免了状态突变和并发问题。

**Section sources**
- [model/interface.go](file://components/model/interface.go#L25-L57)

### 工具 (Tool)

`tool` 组件用于扩展模型的能力，使其能够调用外部函数。其接口定义在 `components/tool/interface.go` 中。

**BaseTool** 接口是所有工具的基础，提供获取工具信息的方法：
- `Info(ctx context.Context) (*schema.ToolInfo, error)`：返回工具的元信息，用于模型的意图识别。

**InvokableTool** 接口定义了可调用工具的能力：
- `InvokableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (string, error)`：执行工具逻辑，输入为 JSON 格式的参数字符串，返回结果字符串。

**StreamableTool** 接口定义了支持流式输出的工具：
- `StreamableRun(ctx context.Context, argumentsInJSON string, opts ...Option) (*schema.StreamReader[string], error)`：以流式方式执行工具逻辑。

**Section sources**
- [tool/interface.go](file://components/tool/interface.go#L25-L44)

### 提示模板 (Prompt)

`prompt` 组件负责将变量和模板渲染为模型可理解的消息序列。其核心接口定义在 `components/prompt/interface.go` 中。

**ChatTemplate** 接口定义了模板渲染的统一方法：
- `Format(ctx context.Context, vs map[string]any, opts ...Option) ([]*schema.Message, error)`：将变量 `vs` 应用到模板中，生成消息列表。

`DefaultChatTemplate` 是 `ChatTemplate` 的默认实现，它通过 `FromMessages(formatType schema.FormatType, templates ...schema.MessagesTemplate)` 构造函数创建。渲染逻辑在 `Format` 方法中实现，该方法会处理回调（callbacks）的生命周期，确保在开始、结束和出错时触发相应的钩子。

**Section sources**
- [prompt/interface.go](file://components/prompt/interface.go#L27-L30)
- [prompt/chat_template.go](file://components/prompt/chat_template.go#L27-L90)

### 检索器 (Retriever)

`retriever` 组件用于从数据源中检索相关文档。其接口定义在 `components/retriever/interface.go` 中。

**Retriever** 接口提供了核心的检索方法：
- `Retrieve(ctx context.Context, query string, opts ...Option) ([]*schema.Document, error)`：根据查询字符串检索最相关的文档列表。

**Section sources**
- [retriever/interface.go](file://components/retriever/interface.go#L39-L42)

### 嵌入 (Embedding)

`embedding` 组件负责将文本转换为向量表示。其接口定义在 `components/embedding/interface.go` 中。

**Embedder** 接口定义了文本嵌入的方法：
- `EmbedStrings(ctx context.Context, texts []string, opts ...Option) ([][]float64, error)`：将一组文本字符串转换为对应的向量数组。

**Section sources**
- [embedding/interface.go](file://components/embedding/interface.go#L22-L25)

### 索引器 (Indexer)

`indexer` 组件负责将文档存储到向量数据库中。其接口定义在 `components/indexer/interface.go` 中。

**Indexer** 接口提供了文档存储的方法：
- `Store(ctx context.Context, docs []*schema.Document, opts ...Option) (ids []string, err error)`：将文档列表存储到索引中，并返回分配的 ID 列表。

**Section sources**
- [indexer/interface.go](file://components/indexer/interface.go#L29-L33)

### 文档 (Document)

`document` 组件处理文档的加载和转换。其接口定义在 `components/document/interface.go` 中。

**Loader** 接口负责从源加载文档：
- `Load(ctx context.Context, src Source, opts ...LoaderOption) ([]*schema.Document, error)`：从指定的 `Source` 加载文档列表。

**Transformer** 接口负责转换文档：
- `Transform(ctx context.Context, src []*schema.Document, opts ...TransformerOption) ([]*schema.Document, error)`：对文档列表进行转换，如分块或过滤。

`Source` 结构体定义了文档源，包含一个 `URI` 字段。

**Section sources**
- [document/interface.go](file://components/document/interface.go#L34-L43)

## Option 配置模式

Eino 框架在 `components` 包的各个子包中广泛使用了 `Option` 模式来配置组件行为。该模式的核心思想是将配置选项封装为函数，允许用户以声明式的方式设置参数。

每个子包都定义了 `Option` 类型和一系列 `WithXXX` 函数。例如，在 `model` 包中：

- `Option` 是一个结构体，包含一个 `apply` 函数，用于修改 `Options` 结构体。
- `WithTemperature(temperature float32)` 等函数返回一个 `Option`，该 `Option` 的 `apply` 函数会设置温度值。

用户可以通过传递多个 `WithXXX` 选项来配置组件调用。框架通过 `GetCommonOptions(base *Options, opts ...Option)` 函数来应用这些选项，该函数会遍历所有 `Option` 并执行其 `apply` 函数。

此外，`WrapImplSpecificOptFn[T any](optFn func(*T)) Option` 和 `GetImplSpecificOptions[T any](base *T, opts ...Option)` 函数支持组件实现者定义自己的特定选项，实现了配置的灵活性和扩展性。

**Section sources**
- [model/option.go](file://components/model/option.go#L1-L160)
- [tool/option.go](file://components/tool/option.go#L1-L79)
- [prompt/option.go](file://components/prompt/option.go#L1-L49)
- [retriever/option.go](file://components/retriever/option.go#L1-L147)
- [embedding/option.go](file://components/embedding/option.go#L1-L96)
- [indexer/option.go](file://components/indexer/option.go#L1-L87)
- [document/option.go](file://components/document/option.go#L1-L41)

## 自定义组件实现指南

要实现自定义组件，需要遵循以下步骤：

1.  **实现核心接口**：根据组件类型，实现相应的接口，如 `ChatModel`、`InvokableTool` 或 `ChatTemplate`。

2.  **处理 Option 模式**：如果组件需要配置，应使用 `GetImplSpecificOptions` 函数来解析用户传递的 `Option` 列表。可以定义自己的选项结构体，并通过 `WrapImplSpecificOptFn` 将其包装为通用的 `Option` 类型。

3.  **利用工具函数**：`components/tool/utils` 包提供了许多实用工具。
    - `create_options.go` 中的 `WithSchemaModifier` 等函数可用于自定义工具参数的 JSON Schema 生成。
    - `error_handler.go` 中的 `WrapInvokableToolWithErrorHandler` 等函数可用于为工具添加统一的错误处理逻辑，将错误转换为字符串结果。
    - `common.go` 中的 `marshalString` 函数可用于将任意响应安全地序列化为字符串。

4.  **集成回调**：如果需要，可以实现 `Typer` 和 `Checker` 接口来自定义组件的类型名称和回调行为。

通过遵循这些指南，开发者可以创建与 Eino 框架无缝集成的自定义组件。

**Section sources**
- [tool/utils/create_options.go](file://components/tool/utils/create_options.go#L1-L226)
- [tool/utils/error_handler.go](file://components/tool/utils/error_handler.go#L1-L159)
- [tool/utils/common.go](file://components/tool/utils/common.go#L1-L29)
- [types.go](file://components/types.go#L23-L48)