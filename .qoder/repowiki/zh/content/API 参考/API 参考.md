# API 参考

<cite>
**本文档中引用的文件**  
- [compose/doc.go](file://compose/doc.go)
- [callbacks/doc.go](file://callbacks/doc.go)
- [schema/doc.go](file://schema/doc.go)
- [components/model/doc.go](file://components/model/doc.go)
- [components/tool/doc.go](file://components/tool/doc.go)
- [components/prompt/doc.go](file://components/prompt/doc.go)
- [components/retriever/doc.go](file://components/retriever/doc.go)
- [components/embedding/doc.go](file://components/embedding/doc.go)
- [components/document/doc.go](file://components/document/doc.go)
- [components/indexer/doc.go](file://components/indexer/doc.go)
- [compose/types.go](file://compose/types.go)
- [compose/types_lambda.go](file://compose/types_lambda.go)
- [components/types.go](file://components/types.go)
</cite>

## 目录
1. [简介](#简介)
2. [compose 包](#compose-包)
3. [callbacks 包](#callbacks-包)
4. [schema 包](#schema-包)
5. [components 子包](#components-子包)

## 简介
本文档为 Eino 框架提供权威的 API 参考，涵盖 `compose`、`callbacks`、`schema` 以及 `components` 下的 `model`、`tool`、`prompt`、`retriever`、`embedding`、`document` 和 `indexer` 等核心包。文档内容严格基于各包的 `doc.go` 文件和源码注释生成，旨在为开发者提供完整的结构体、接口、函数和方法参考，包括签名、参数说明、返回值、错误类型及使用示例。

## compose 包

`compose` 包是 Eino 框架的核心编排模块，提供构建和管理可组合组件（如图、链、Lambda 函数）的能力。

### 核心类型与常量

#### Component 枚举
表示图节点中内置组件的类型。

**Section sources**
- [compose/types.go](file://compose/types.go#L27-L35)

#### NodeTriggerMode
控制图节点的触发模式。

**Section sources**
- [compose/types.go](file://compose/types.go#L37-L46)

#### AnyGraph 接口
表示可组合和可编译的图或链的通用接口。

**Section sources**
- [compose/types_composable.go](file://compose/types_composable.go#L25-L32)

### Lambda 函数支持

`compose` 包提供了强大的 Lambda 函数支持，允许用户将任意函数包装为可执行节点。

#### Lambda 函数类型
- `Invoke[I, O, TOption]`: 可调用的 Lambda 函数。
- `Stream[I, O, TOption]`: 可流式输出的 Lambda 函数。
- `Collect[I, O, TOption]`: 可收集流式输入的 Lambda 函数。
- `Transform[I, O, TOption]`: 可转换流式输入的 Lambda 函数。

**Section sources**
- [compose/types_lambda.go](file://compose/types_lambda.go#L26-L55)

#### Lambda 创建函数
- `InvokableLambda[I, O any](i InvokeWOOpt[I, O], opts ...LambdaOpt) *Lambda`: 创建一个可调用的 Lambda。
- `StreamableLambda[I, O any](s StreamWOOpt[I, O], opts ...LambdaOpt) *Lambda`: 创建一个可流式输出的 Lambda。
- `CollectableLambda[I, O any](c CollectWOOpt[I, O], opts ...LambdaOpt) *Lambda`: 创建一个可收集流式输入的 Lambda。
- `TransformableLambda[I, O any](t TransformWOOpts[I, O], opts ...LambdaOpt) *Lambda`: 创建一个可转换流式输入的 Lambda。
- `AnyLambda[I, O, TOption any](i Invoke[I, O, TOption], s Stream[I, O, TOption], c Collect[I, O, TOption], t Transform[I, O, TOption], opts ...LambdaOpt) (*Lambda, error)`: 创建一个支持多种模式的 Lambda。

**Section sources**
- [compose/types_lambda.go](file://compose/types_lambda.go#L104-L182)

#### Lambda 选项
- `WithLambdaCallbackEnable(y bool) LambdaOpt`: 启用或禁用 Lambda 的回调切面。
- `WithLambdaType(t string) LambdaOpt`: 设置 Lambda 的类型。

**Section sources**
- [compose/types_lambda.go](file://compose/types_lambda.go#L83-L95)

#### 辅助 Lambda 函数
- `ToList[I any](opts ...LambdaOpt) *Lambda`: 创建一个将单个输入转换为列表的 Lambda。
- `MessageParser[T any](p schema.MessageParser[T], opts ...LambdaOpt) *Lambda`: 创建一个将消息解析为对象的 Lambda。

**Section sources**
- [compose/types_lambda.go](file://compose/types_lambda.go#L224-L265)

## callbacks 包

`callbacks` 包为 Eino 组件的执行提供回调机制，可用于实现日志记录、监控和指标收集等治理能力。

### 核心功能

该包提供了两种创建回调处理器的方式：

1.  **使用 HandlerBuilder**: 创建一个通用的回调处理器，可以为开始、结束和错误等阶段设置处理函数。
2.  **使用 HandlerHelper**: 使用 `utils/callbacks` 包中的 `HandlerHelper`，可以为不同类型的组件（如模型、提示词、工具等）分别设置特定的处理器。

**Section sources**
- [callbacks/doc.go](file://callbacks/doc.go#L17-L93)

### 使用示例
```go
// 使用 HandlerHelper 为模型和提示词组件创建处理器
modelHandler := &model.CallbackHandler{
    OnStart: func(ctx context.Context, info *RunInfo, input *model.CallbackInput) context.Context {
        log.Printf("模型执行开始: %s", info.ComponentName)
        return ctx
    },
}

promptHandler := &prompt.CallbackHandler{
    OnEnd: func(ctx context.Context, info *RunInfo, output *prompt.CallbackOutput) context.Context {
        log.Printf("提示词执行完成: %s", output.Result)
        return ctx
    },
}

handler := callbacks.NewHandlerHelper().
    ChatModel(modelHandler).
    Prompt(promptHandler).
    Handler()

// 在调用组件时使用回调
runnable.Invoke(ctx, input, compose.WithCallbacks(handler))
```

**Section sources**
- [callbacks/doc.go](file://callbacks/doc.go#L58-L78)

## schema 包

`schema` 包定义了 Eino 框架中的核心数据结构和类型，如消息、文档和流。

### 核心数据结构
该包主要包含以下结构：
- `Message`: 表示对话消息，包含角色、内容等。
- `Document`: 表示文档，包含内容、元数据等。
- `StreamReader[T]`: 表示泛型流读取器，用于处理流式数据。

**Section sources**
- [schema/doc.go](file://schema/doc.go)

## components 子包

`components` 包是 Eino 框架中所有可执行组件的基础定义。

### 核心接口

#### Typer 接口
用于获取组件实现的类型名称。

**Section sources**
- [components/types.go](file://components/types.go#L23-L25)

#### Checker 接口
用于告知框架组件是否已自行处理回调切面。

**Section sources**
- [components/types.go](file://components/types.go#L38-L40)

### 组件类型常量
定义了框架支持的各种组件类型。

**Section sources**
- [components/types.go](file://components/types.go#L54-L62)

### 子包概述

#### model 子包
提供与聊天模型交互的组件接口。

**Section sources**
- [components/model/doc.go](file://components/model/doc.go)

#### tool 子包
提供工具组件的接口和实现。

**Section sources**
- [components/tool/doc.go](file://components/tool/doc.go)

#### prompt 子包
提供提示词模板组件的接口。

**Section sources**
- [components/prompt/doc.go](file://components/prompt/doc.go)

#### retriever 子包
提供检索器组件的接口。

**Section sources**
- [components/retriever/doc.go](file://components/retriever/doc.go)

#### embedding 子包
提供嵌入式（Embedding）模型组件的接口。

**Section sources**
- [components/embedding/doc.go](file://components/embedding/doc.go)

#### document 子包
提供文档加载和转换组件的接口。

**Section sources**
- [components/document/doc.go](file://components/document/doc.go)

#### indexer 子包
提供索引器组件的接口。

**Section sources**
- [components/indexer/doc.go](file://components/indexer/doc.go)