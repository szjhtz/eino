# Eino Code Wiki

## 项目概述

Eino 是一个用 Golang 编写的 LLM 应用开发框架，借鉴了 LangChain、Google ADK 等开源框架的设计理念，同时遵循 Golang 的开发约定。

### 核心功能

- **组件系统**：提供可重用的构建块，如 `ChatModel`、`Tool`、`Retriever` 和 `ChatTemplate`，并为 OpenAI、Ollama 等提供官方实现。
- **Agent 开发工具包 (ADK)**：构建具有工具使用、多代理协调、上下文管理、中断/恢复（人机协作）等能力的 AI 代理。
- **组合系统**：将组件连接成图和工作流，可以独立运行或作为代理的工具暴露。
- **流处理**：自动处理整个编排过程中的流数据，包括连接、合并和复制流。
- **回调机制**：在组件、图和代理的固定点注入日志、跟踪和指标。
- **中断/恢复**：任何代理或工具都可以暂停执行以获取人工输入，并从检查点恢复。

## 架构设计

Eino 框架由以下部分组成：

1. **Eino (核心)**：类型定义、流机制、组件抽象、编排、代理实现、方面机制
2. **EinoExt**：组件实现、回调处理程序、使用示例、评估器、提示优化器
3. **Eino Devops**：可视化开发和调试
4. **EinoExamples**：示例应用程序和最佳实践

### 核心模块关系

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    components   │◄────┤      adk        │◄────┤     compose     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        ▲                      │                      ▲
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                       ┌─────────────────┐
                       │     schema      │
                       └─────────────────┘
```

## 核心模块

### 1. ADK (Agent Development Kit)

ADK 模块是 Eino 的核心，提供了构建 AI 代理的工具和框架。

#### 主要功能

- **代理接口**：定义了 Agent 接口，是所有代理实现的基础。
- **运行器**：提供了运行代理的 Runner，处理事件流和状态管理。
- **工具集成**：允许代理使用各种工具，增强其能力。
- **中断/恢复**：支持代理在执行过程中暂停并在之后恢复。
- **多代理协调**：支持代理之间的协作和任务委托。

#### 关键组件

- **Agent 接口**：定义了代理的基本行为，包括名称、描述和运行方法。
- **Runner**：管理代理的执行，处理事件流和状态。
- **AsyncIterator**：处理异步事件流，支持流式输出。
- **AgentEvent**：表示代理执行过程中的事件，包含输出、动作和错误信息。

### 2. Components

Components 模块定义了各种可重用的构建块，为 Eino 提供了基础功能。

#### 主要组件类型

- **Model**：包含 BaseChatModel 接口，定义了与聊天模型交互的方法。
- **Tool**：定义了工具的接口，允许代理使用各种功能。
- **Retriever**：用于从数据源检索信息。
- **Embedding**：处理文本嵌入。
- **Document**：处理文档加载和转换。
- **Indexer**：处理索引功能。
- **Prompt**：处理提示模板。

#### 关键接口

- **BaseChatModel**：定义了与聊天模型交互的核心方法，包括 Generate 和 Stream。
- **Tool**：定义了工具的基本行为，包括 Info 和 InvokableRun。
- **Retriever**：定义了检索信息的方法。

### 3. Compose

Compose 模块提供了构建和运行工作流的能力，允许用户创建复杂的处理管道。

#### 主要功能

- **图构建**：允许用户创建由节点和边组成的图。
- **节点类型**：支持各种类型的节点，包括 Lambda、ChatModel、Tool 等。
- **分支和条件**：支持基于条件的分支执行。
- **流处理**：自动处理节点之间的数据流。
- **检查点**：支持工作流的中断和恢复。

#### 关键组件

- **Graph**：表示一个由节点和边组成的计算图。
- **Node**：图中的基本计算单元，可以是各种类型的组件。
- **Edge**：连接节点的边，定义了数据流向。
- **Branch**：表示条件分支，根据条件选择不同的执行路径。

### 4. Schema

Schema 模块定义了 Eino 中使用的数据结构和类型，为整个框架提供了统一的数据表示。

#### 主要类型

- **Message**：表示聊天消息，支持多种角色和内容类型。
- **ToolInfo**：描述工具的信息，包括名称、描述和参数。
- **Stream**：处理流式数据。

### 5. Flow

Flow 模块提供了预定义的流程和模式，简化常见任务的实现。

#### 主要流程

- **Agent**：包含 React 代理实现。
- **Retriever**：包含多种检索器实现，如 MultiQuery 和 Router。
- **Indexer**：包含索引器实现。

## 关键类与函数

### 1. Agent 接口

```go
type Agent interface {
    Name(ctx context.Context) string
    Description(ctx context.Context) string
    Run(ctx context.Context, input *AgentInput, options ...AgentRunOption) *AsyncIterator[*AgentEvent]
}
```

**功能**：定义了代理的基本行为，是所有代理实现的基础。

**参数**：
- `ctx`：上下文，用于传递超时、取消信号等。
- `input`：代理输入，包含消息和流式启用标志。
- `options`：运行选项，用于配置代理行为。

**返回值**：
- `*AsyncIterator[*AgentEvent]`：异步事件迭代器，用于处理代理执行过程中的事件。

### 2. BaseChatModel 接口

```go
type BaseChatModel interface {
    Generate(ctx context.Context, input []*schema.Message, opts ...Option) (*schema.Message, error)
    Stream(ctx context.Context, input []*schema.Message, opts ...Option) (*schema.StreamReader[*schema.Message], error)
}
```

**功能**：定义了与聊天模型交互的核心方法。

**参数**：
- `ctx`：上下文。
- `input`：消息列表，表示对话历史。
- `opts`：选项，用于配置模型行为。

**返回值**：
- `Generate`：返回完整的消息响应。
- `Stream`：返回消息流读取器，用于流式接收响应。

### 3. Graph 结构

```go
type graph struct {
    nodes        map[string]*graphNode
    controlEdges map[string][]string
    dataEdges    map[string][]string
    branches     map[string][]*GraphBranch
    startNodes   []string
    endNodes     []string
    // ... 其他字段
}
```

**功能**：表示一个由节点和边组成的计算图，是 Compose 模块的核心。

**主要方法**：
- `AddChatModelNode`：添加聊天模型节点。
- `AddLambdaNode`：添加 lambda 函数节点。
- `AddEdge`：添加边连接节点。
- `AddBranch`：添加条件分支。
- `Compile`：编译图为可执行的运行时。

### 4. Runner 结构

```go
type Runner struct {
    a               Agent
    enableStreaming bool
    store           compose.CheckPointStore
    // ... 其他字段
}
```

**功能**：管理代理的执行，处理事件流和状态。

**主要方法**：
- `Query`：运行代理并返回事件迭代器。
- `Resume`：从检查点恢复代理执行。

### 5. NewGraph 函数

```go
func NewGraph[I, O any](opts ...NewGraphOption) *Graph[I, O]
```

**功能**：创建一个新的图，用于构建工作流。

**参数**：
- `opts`：图选项，用于配置图的行为。

**返回值**：
- `*Graph[I, O]`：新创建的图，支持泛型输入和输出类型。

### 6. NewChatModelAgent 函数

```go
func NewChatModelAgent(ctx context.Context, config *ChatModelAgentConfig) (Agent, error)
```

**功能**：创建一个基于聊天模型的代理。

**参数**：
- `ctx`：上下文。
- `config`：代理配置，包含模型和工具信息。

**返回值**：
- `Agent`：创建的代理实例。
- `error`：错误信息。

## 依赖关系

### 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| github.com/bytedance/sonic | v1.15.0 | 高性能 JSON 处理 |
| github.com/google/uuid | v1.6.0 | UUID 生成 |
| github.com/nikolalohinski/gonja | v1.5.3 | 模板引擎 |
| github.com/slongfield/pyfmt | v0.0.0-20220222012616-ea85ff4c361f | Python 风格的字符串格式化 |
| github.com/stretchr/testify | v1.10.0 | 测试工具 |
| go.uber.org/mock | v0.4.0 | 模拟对象生成 |
| gopkg.in/yaml.v3 | v3.0.1 | YAML 处理 |

### 技术栈

- **语言**：Go 1.18+
- **构建工具**：Go Modules
- **测试框架**：Testify, GoConvey
- **代码质量**：golangci-lint

## 开发与运行

### 开发环境设置

Eino 提供了 `dev_setup.sh` 脚本，用于设置本地多模块工作空间：

1. 克隆 eino-ext 到 ext/ 目录
2. 克隆 eino-examples 到 examples/ 目录
3. 在 .git/info/exclude 中注册这些目录，使它们不被 git 跟踪
4. 创建 go.work 文件，将 eino、ext/ 和 examples/ 中的所有模块连接在一起

### 运行方式

#### 基本用法

1. **创建聊天模型**：

```go
chatModel, _ := openai.NewChatModel(ctx, &openai.ChatModelConfig{
    Model:  "gpt-4o",
    APIKey: os.Getenv("OPENAI_API_KEY"),
})
```

2. **创建代理**：

```go
agent, _ := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    Model: chatModel,
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{weatherTool, calculatorTool},
        },
    },
})
```

3. **运行代理**：

```go
runner := adk.NewRunner(ctx, adk.RunnerConfig{Agent: agent})
iter := runner.Query(ctx, "Hello, who are you?")
for {
    event, ok := iter.Next()
    if !ok {
        break
    }
    fmt.Println(event.Message.Content)
}
```

#### 构建工作流

1. **创建图**：

```go
graph := compose.NewGraph[*Input, *Output]()
graph.AddLambdaNode("validate", validateFn)
graph.AddChatModelNode("generate", chatModel)
graph.AddLambdaNode("format", formatFn)

graph.AddEdge(compose.START, "validate")
graph.AddEdge("validate", "generate")
graph.AddEdge("generate", "format")
graph.AddEdge("format", compose.END)
```

2. **编译并运行**：

```go
runnable, _ := graph.Compile(ctx)
result, _ := runnable.Invoke(ctx, input)
```

## 最佳实践

1. **组件复用**：尽可能重用现有的组件，而不是重新实现。
2. **流处理**：对于大型模型响应，使用流式处理以提高用户体验。
3. **错误处理**：正确处理组件和代理执行过程中的错误。
4. **检查点**：对于长时间运行的任务，使用检查点功能以支持中断和恢复。
5. **工具使用**：为代理提供适当的工具，增强其能力。
6. **监控**：使用回调机制注入日志、跟踪和指标，以便监控系统运行状态。

## 示例应用

Eino 提供了丰富的示例应用，位于 [eino-examples](https://github.com/cloudwego/eino-examples) 仓库中，包括：

- **ADK 介绍**：基本的代理使用示例。
- **多代理**：多代理协作的示例。
- **人机协作**：包含人工输入的示例。
- **工具使用**：使用各种工具的示例。
- **工作流**：构建复杂工作流的示例。

## 总结

Eino 是一个功能强大的 LLM 应用开发框架，提供了构建 AI 代理和工作流的完整工具集。它的模块化设计和丰富的组件系统使得开发复杂的 LLM 应用变得简单而高效。通过合理使用 Eino 的各种功能，开发者可以快速构建出功能强大、响应迅速的 AI 应用。