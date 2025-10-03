# Codex 项目架构分析

## 项目概述

Codex 是 OpenAI 开发的本地运行的代码智能助手CLI工具，采用 Rust(后端核心) + TypeScript(CLI包装/SDK) 的混合架构。

---

## 1. 整体架构图

```mermaid
graph TB
    subgraph "用户交互层"
        CLI[Codex CLI<br/>命令行入口]
        TUI[交互式终端界面<br/>codex-tui]
        EXEC[非交互模式<br/>codex-exec]
    end
    
    subgraph "核心服务层"
        CORE[核心引擎<br/>codex-core]
        AUTH[认证管理<br/>AuthManager]
        CONFIG[配置管理<br/>Config]
        SESSION[会话管理<br/>Session]
    end
    
    subgraph "AI交互层"
        CLIENT[模型客户端<br/>ModelClient]
        OPENAI[OpenAI API]
        OSS[开源模型<br/>Ollama]
    end
    
    subgraph "工具执行层"
        EXECUTOR[执行器<br/>Executor]
        TOOLS[工具路由<br/>ToolRouter]
        SANDBOX[沙箱环境<br/>Seatbelt/Landlock]
    end
    
    subgraph "协议与扩展"
        MCP[MCP服务器<br/>Model Context Protocol]
        PROTOCOL[协议层<br/>codex-protocol]
        APPSERVER[应用服务器<br/>app-server]
    end
    
    CLI --> TUI
    CLI --> EXEC
    CLI --> MCP
    
    TUI --> CORE
    EXEC --> CORE
    MCP --> CORE
    
    CORE --> AUTH
    CORE --> CONFIG
    CORE --> SESSION
    CORE --> CLIENT
    CORE --> TOOLS
    
    CLIENT --> OPENAI
    CLIENT --> OSS
    
    TOOLS --> EXECUTOR
    EXECUTOR --> SANDBOX
    
    SESSION --> PROTOCOL
    APPSERVER --> PROTOCOL
```

---

## 2. 核心工作流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as CLI入口
    participant TUI as TUI界面
    participant Core as Codex核心
    participant Auth as 认证管理
    participant Session as 会话管理
    participant Client as 模型客户端
    participant AI as AI模型(OpenAI)
    participant Tools as 工具系统
    participant Executor as 命令执行器
    participant Sandbox as 沙箱环境
    
    User->>CLI: 运行 codex
    CLI->>Auth: 检查认证状态
    Auth-->>CLI: 返回认证令牌
    
    CLI->>TUI: 启动交互界面
    TUI->>Core: 初始化Codex实例
    Core->>Session: 创建新会话
    Core->>Config: 加载配置
    
    User->>TUI: 输入问题/命令
    TUI->>Core: 提交Submission
    Core->>Session: 处理用户输入
    
    Session->>Client: 构建Prompt发送请求
    Client->>AI: 调用Chat Completions API
    
    loop 流式响应
        AI-->>Client: 返回Token流
        Client-->>Session: ResponseEvent
        Session-->>Core: Event
        Core-->>TUI: 更新UI显示
    end
    
    alt AI请求执行工具
        AI-->>Client: FunctionCall(shell/exec)
        Client-->>Session: 工具调用请求
        Session->>Tools: 路由到对应工具
        
        Tools->>Executor: 准备执行上下文
        
        alt 需要审批
            Executor->>TUI: 请求用户审批
            TUI->>User: 显示审批弹窗
            User-->>TUI: 批准/拒绝
            TUI-->>Executor: 审批结果
        end
        
        Executor->>Sandbox: 在沙箱中执行
        Sandbox-->>Executor: 执行结果
        Executor-->>Tools: ExecOutput
        Tools-->>Session: 工具响应
        Session->>Client: 将结果反馈给AI
        Client->>AI: 继续对话
    end
    
    AI-->>Client: 最终响应
    Client-->>Session: 完成事件
    Session-->>Core: TokenUsage/完成信息
    Core-->>TUI: 显示完成状态
    TUI-->>User: 展示结果
```

---

## 3. Codex核心引擎架构

```mermaid
graph LR
    subgraph "Codex 核心 (codex-core)"
        subgraph "会话管理"
            Session[Session<br/>会话状态]
            History[ConversationHistory<br/>对话历史]
            TurnContext[TurnContext<br/>回合上下文]
        end
        
        subgraph "任务系统"
            RegularTask[RegularTask<br/>常规任务]
            CompactTask[CompactTask<br/>压缩任务]
            ReviewTask[ReviewTask<br/>审查任务]
        end
        
        subgraph "工具系统"
            ToolRouter[ToolRouter<br/>工具路由器]
            ToolRegistry[ToolRegistry<br/>工具注册表]
            ToolHandlers[ToolHandlers<br/>工具处理器]
        end
        
        subgraph "执行引擎"
            Executor[Executor<br/>执行器]
            ExecMode[ExecutionMode<br/>执行模式]
            ExecCache[ExecutionCache<br/>执行缓存]
        end
    end
    
    Session --> History
    Session --> TurnContext
    
    TurnContext --> RegularTask
    TurnContext --> CompactTask
    TurnContext --> ReviewTask
    
    RegularTask --> ToolRouter
    ToolRouter --> ToolRegistry
    ToolRegistry --> ToolHandlers
    
    ToolHandlers --> Executor
    Executor --> ExecMode
    Executor --> ExecCache
```

---

## 4. TUI交互界面架构

```mermaid
graph TB
    subgraph "TUI应用 (codex-tui)"
        App[App<br/>应用主状态]
        
        subgraph "聊天组件"
            ChatWidget[ChatWidget<br/>聊天窗口]
            MessageStream[MarkdownStream<br/>消息流]
            HistoryCell[HistoryCell<br/>历史消息]
        end
        
        subgraph "底部面板"
            Composer[ChatComposer<br/>输入编辑器]
            ApprovalOverlay[ApprovalOverlay<br/>审批弹窗]
            FileSearch[FileSearchPopup<br/>文件搜索]
            CommandPopup[CommandPopup<br/>命令弹窗]
        end
        
        subgraph "状态与控制"
            Status[StatusIndicator<br/>状态指示器]
            Updates[UpdateChecker<br/>更新检查]
            Onboarding[OnboardingScreen<br/>引导界面]
        end
        
        subgraph "渲染系统"
            Markdown[MarkdownRender<br/>Markdown渲染]
            DiffRender[DiffRender<br/>差异渲染]
            Highlight[SyntaxHighlight<br/>语法高亮]
        end
    end
    
    App --> ChatWidget
    App --> Composer
    App --> Status
    
    ChatWidget --> MessageStream
    ChatWidget --> HistoryCell
    
    Composer --> ApprovalOverlay
    Composer --> FileSearch
    Composer --> CommandPopup
    
    MessageStream --> Markdown
    MessageStream --> DiffRender
    Markdown --> Highlight
    
    App --> Updates
    App --> Onboarding
```

---

## 5. 认证与配置流程

```mermaid
stateDiagram-v2
    [*] --> 检查配置
    
    检查配置 --> 加载配置文件: ~/.codex/config.toml
    加载配置文件 --> 应用CLI覆盖: -c参数
    应用CLI覆盖 --> 检查认证
    
    检查认证 --> 已登录ChatGPT: 使用OAuth Token
    检查认证 --> 已配置API密钥: 使用API Key
    检查认证 --> 未认证: 首次使用
    
    未认证 --> 显示登录选项
    显示登录选项 --> ChatGPT登录: 默认方式
    显示登录选项 --> API密钥登录: --with-api-key
    显示登录选项 --> 设备码登录: 实验性
    
    ChatGPT登录 --> 打开浏览器授权
    打开浏览器授权 --> 获取Token
    获取Token --> 保存凭证
    
    API密钥登录 --> 从stdin读取
    从stdin读取 --> 保存凭证
    
    已登录ChatGPT --> 验证Token有效性
    已配置API密钥 --> 验证密钥有效性
    
    验证Token有效性 --> 启动会话: 有效
    验证Token有效性 --> 刷新Token: 过期
    验证密钥有效性 --> 启动会话: 有效
    验证密钥有效性 --> 显示登录选项: 无效
    
    刷新Token --> 启动会话
    保存凭证 --> 启动会话
    
    启动会话 --> [*]
```

---

## 6. 工具执行与沙箱隔离

```mermaid
graph TD
    subgraph "工具调用流程"
        ToolCall[AI发起工具调用]
        Router[ToolRouter路由]
        Handler[对应Handler处理]
    end
    
    subgraph "执行准备"
        ParseCmd[解析命令]
        SafetyCheck[安全检查]
        BuildContext[构建执行上下文]
    end
    
    subgraph "审批流程"
        NeedApproval{需要审批?}
        ShowApproval[显示审批弹窗]
        UserDecision{用户决策}
        Approved[批准执行]
        Rejected[拒绝执行]
    end
    
    subgraph "沙箱执行"
        SelectMode{选择执行模式}
        
        subgraph "macOS"
            Seatbelt[Seatbelt沙箱]
        end
        
        subgraph "Linux"
            Landlock[Landlock + seccomp]
        end
        
        subgraph "Windows"
            WindowsExec[直接执行<br/>受限权限]
        end
        
        Execute[执行命令]
        CaptureOutput[捕获输出]
    end
    
    subgraph "结果处理"
        FormatOutput[格式化输出]
        UpdateDiff[更新文件差异]
        SendToAI[发送结果给AI]
    end
    
    ToolCall --> Router
    Router --> Handler
    Handler --> ParseCmd
    ParseCmd --> SafetyCheck
    SafetyCheck --> BuildContext
    
    BuildContext --> NeedApproval
    NeedApproval -->|是| ShowApproval
    NeedApproval -->|否| SelectMode
    
    ShowApproval --> UserDecision
    UserDecision -->|批准| Approved
    UserDecision -->|拒绝| Rejected
    
    Approved --> SelectMode
    Rejected --> FormatOutput
    
    SelectMode -->|macOS| Seatbelt
    SelectMode -->|Linux| Landlock
    SelectMode -->|Windows| WindowsExec
    
    Seatbelt --> Execute
    Landlock --> Execute
    WindowsExec --> Execute
    
    Execute --> CaptureOutput
    CaptureOutput --> FormatOutput
    FormatOutput --> UpdateDiff
    UpdateDiff --> SendToAI
```

---

## 7. MCP (Model Context Protocol) 架构

```mermaid
graph TB
    subgraph "MCP服务器模式"
        MCPServer[codex-mcp-server<br/>MCP服务器]
        StdIO[stdio传输层]
        
        subgraph "MCP消息处理"
            MessageProc[MessageProcessor<br/>消息处理器]
            ToolRunner[CodexToolRunner<br/>工具运行器]
        end
    end
    
    subgraph "MCP客户端模式"
        MCPClient[codex-mcp-client<br/>MCP客户端]
        ConnMgr[McpConnectionManager<br/>连接管理器]
        
        subgraph "外部MCP服务器"
            External1[文件系统MCP]
            External2[Git MCP]
            External3[自定义MCP服务器]
        end
    end
    
    subgraph "Codex核心集成"
        CoreSession[Codex Session]
        ToolRegistry[工具注册表]
    end
    
    subgraph "外部IDE/工具"
        VSCode[VS Code]
        Cursor[Cursor IDE]
        CustomTool[自定义工具]
    end
    
    VSCode --> StdIO
    Cursor --> StdIO
    CustomTool --> StdIO
    
    StdIO <--> MCPServer
    MCPServer --> MessageProc
    MessageProc --> ToolRunner
    ToolRunner --> CoreSession
    
    CoreSession --> ConnMgr
    ConnMgr --> MCPClient
    
    MCPClient --> External1
    MCPClient --> External2
    MCPClient --> External3
    
    External1 --> ToolRegistry
    External2 --> ToolRegistry
    External3 --> ToolRegistry
    
    ToolRegistry --> CoreSession
```

---

## 8. 数据流与事件系统

```mermaid
graph LR
    subgraph "输入流"
        UserInput[用户输入<br/>Submission]
        OpTypes[操作类型<br/>Op enum]
    end
    
    subgraph "事件流"
        EventChannel[事件通道<br/>async-channel]
        EventTypes[事件类型<br/>Event enum]
        
        subgraph "事件类型"
            AgentDelta[AgentMessageDelta<br/>AI消息流]
            ExecBegin[ExecCommandBegin<br/>命令开始]
            ExecEnd[ExecCommandEnd<br/>命令结束]
            ApprovalReq[ApprovalRequest<br/>审批请求]
            TokenCount[TokenCount<br/>Token统计]
        end
    end
    
    subgraph "状态管理"
        SessionState[SessionState<br/>会话状态]
        TurnState[TurnState<br/>回合状态]
        History[ConversationHistory<br/>对话历史]
    end
    
    UserInput --> OpTypes
    OpTypes --> EventChannel
    
    EventChannel --> EventTypes
    
    EventTypes --> AgentDelta
    EventTypes --> ExecBegin
    EventTypes --> ExecEnd
    EventTypes --> ApprovalReq
    EventTypes --> TokenCount
    
    AgentDelta --> TurnState
    ExecBegin --> TurnState
    ExecEnd --> TurnState
    ApprovalReq --> TurnState
    TokenCount --> SessionState
    
    TurnState --> History
    History --> SessionState
```

---

## 9. 项目模块关系

```mermaid
graph TB
    subgraph "Rust工作空间 (codex-rs)"
        CLI_MOD[cli<br/>CLI入口]
        CORE_MOD[core<br/>核心引擎]
        TUI_MOD[tui<br/>交互界面]
        EXEC_MOD[exec<br/>非交互执行]
        
        subgraph "基础设施"
            PROTOCOL_MOD[protocol<br/>协议定义]
            COMMON_MOD[common<br/>公共工具]
            AUTH_MOD[login<br/>认证逻辑]
        end
        
        subgraph "工具与扩展"
            MCP_SERVER_MOD[mcp-server<br/>MCP服务器]
            MCP_CLIENT_MOD[mcp-client<br/>MCP客户端]
            MCP_TYPES_MOD[mcp-types<br/>MCP类型]
            OLLAMA_MOD[ollama<br/>Ollama集成]
        end
        
        subgraph "执行环境"
            SANDBOX_LINUX[linux-sandbox<br/>Linux沙箱]
            EXECPOLICY[execpolicy<br/>执行策略]
            APPLY_PATCH[apply-patch<br/>补丁应用]
        end
        
        subgraph "辅助工具"
            GIT_TOOLS[git-tooling<br/>Git工具]
            FILE_SEARCH[file-search<br/>文件搜索]
            OTEL[otel<br/>遥测]
        end
    end
    
    subgraph "Node.js包"
        CLI_TS[codex-cli<br/>TypeScript包装]
        SDK_TS[sdk/typescript<br/>TypeScript SDK]
    end
    
    CLI_MOD --> TUI_MOD
    CLI_MOD --> EXEC_MOD
    CLI_MOD --> MCP_SERVER_MOD
    
    TUI_MOD --> CORE_MOD
    EXEC_MOD --> CORE_MOD
    MCP_SERVER_MOD --> CORE_MOD
    
    CORE_MOD --> PROTOCOL_MOD
    CORE_MOD --> COMMON_MOD
    CORE_MOD --> AUTH_MOD
    CORE_MOD --> MCP_CLIENT_MOD
    CORE_MOD --> OLLAMA_MOD
    CORE_MOD --> SANDBOX_LINUX
    CORE_MOD --> EXECPOLICY
    CORE_MOD --> APPLY_PATCH
    CORE_MOD --> GIT_TOOLS
    CORE_MOD --> FILE_SEARCH
    CORE_MOD --> OTEL
    
    MCP_CLIENT_MOD --> MCP_TYPES_MOD
    MCP_SERVER_MOD --> MCP_TYPES_MOD
    
    CLI_TS -.->|打包| CLI_MOD
    SDK_TS --> PROTOCOL_MOD
```

---

## 10. 关键特性实现

### 10.1 会话持久化与恢复

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant RolloutRecorder
    participant FileSystem
    participant Session
    
    User->>CLI: 正常使用Codex
    Session->>RolloutRecorder: 记录每个回合
    RolloutRecorder->>FileSystem: 写入 ~/.codex/sessions/{uuid}.jsonl
    
    Note over FileSystem: 每一行是一个Event
    
    User->>CLI: Ctrl+C 或退出
    CLI->>User: 显示: codex resume {session-id}
    
    User->>CLI: codex resume {session-id}
    CLI->>FileSystem: 读取session文件
    FileSystem-->>CLI: 历史Events
    CLI->>Session: 重建会话状态
    Session-->>User: 继续对话
```

### 10.2 文件差异追踪

```mermaid
graph LR
    subgraph "差异追踪系统"
        TurnDiff[TurnDiffTracker]
        
        subgraph "文件监控"
            PreExec[执行前快照]
            PostExec[执行后快照]
            DiffCalc[计算差异]
        end
        
        subgraph "差异展示"
            DiffUI[差异UI展示]
            ApplyPatch[应用补丁]
            GitDiff[Git差异格式]
        end
    end
    
    TurnDiff --> PreExec
    PreExec --> PostExec
    PostExec --> DiffCalc
    
    DiffCalc --> DiffUI
    DiffCalc --> ApplyPatch
    DiffCalc --> GitDiff
```

---

## 总结

### 架构亮点

1. **分层清晰**: CLI → TUI/EXEC → Core → Tools/Executor → Sandbox
2. **异步驱动**: 全面使用 Tokio 异步运行时，事件驱动架构
3. **类型安全**: Rust + TypeScript，强类型系统保证安全性
4. **沙箱隔离**: 多平台沙箱支持（Seatbelt/Landlock/seccomp）
5. **可扩展**: MCP协议支持，工具系统插件化
6. **用户友好**: TUI交互式界面，审批机制，会话恢复

### 核心技术栈

- **语言**: Rust (核心) + TypeScript (CLI/SDK)
- **UI框架**: ratatui (终端UI)
- **异步运行时**: Tokio
- **序列化**: serde + serde_json
- **协议**: JSON-RPC (MCP), SSE (流式响应)
- **沙箱**: Seatbelt (macOS), Landlock + seccomp (Linux)

### 数据流向

```
用户输入 → Submission → Codex核心 → ModelClient → AI API
                ↓                            ↓
            Event流 ← ResponseEvent ← 流式响应
                ↓
            TUI更新 ← 工具执行 ← FunctionCall
```

