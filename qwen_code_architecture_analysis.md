# Qwen Code 项目架构分析

## 项目概述

Qwen Code 是阿里云通义千问团队基于 **Gemini CLI** Fork并深度改造的AI代码助手，专门为 **Qwen3-Coder** 模型优化。项目采用完全的 TypeScript + Node.js 架构，重点在**解析器层面**进行了适配，以更好地支持Qwen模型的输出特性。

---

## 1. 整体架构图

```mermaid
graph TB
    subgraph "用户交互层"
        CLI[Qwen CLI<br/>命令行入口]
        InkUI[Ink终端UI<br/>React组件]
        NonInteractive[非交互模式<br/>脚本执行]
    end
    
    subgraph "核心包 (packages/core)"
        QwenContentGen[QwenContentGenerator<br/>Qwen专用生成器]
        OpenAIGen[OpenAIContentGenerator<br/>OpenAI兼容生成器]
        ToolRegistry[工具注册表]
        TurnManager[回合管理器]
    end
    
    subgraph "认证层"
        QwenOAuth[Qwen OAuth2<br/>通义千问认证]
        TokenManager[SharedTokenManager<br/>Token管理器]
        OpenAIAuth[OpenAI兼容<br/>API Key认证]
    end
    
    subgraph "API层"
        QwenAPI[Qwen API<br/>通义千问]
        DashScope[DashScope<br/>阿里云百炼]
        ModelScope[ModelScope<br/>魔搭社区]
        OpenRouter[OpenRouter<br/>国际版]
    end
    
    subgraph "核心增强"
        StreamingParser[StreamingToolCallParser<br/>流式解析器]
        ErrorHandler[EnhancedErrorHandler<br/>增强错误处理]
        Pipeline[ContentGenerationPipeline<br/>内容生成管道]
    end
    
    subgraph "工具系统"
        FileTools[文件工具]
        ShellTool[Shell执行]
        WebTools[网络工具]
        MCPTools[MCP工具]
    end
    
    CLI --> InkUI
    CLI --> NonInteractive
    
    InkUI --> QwenContentGen
    InkUI --> OpenAIGen
    
    QwenContentGen --> QwenOAuth
    QwenContentGen --> TokenManager
    OpenAIGen --> OpenAIAuth
    
    QwenOAuth --> QwenAPI
    TokenManager --> DashScope
    OpenAIAuth --> ModelScope
    OpenAIAuth --> OpenRouter
    
    QwenContentGen --> StreamingParser
    OpenAIGen --> StreamingParser
    QwenContentGen --> Pipeline
    OpenAIGen --> Pipeline
    
    Pipeline --> ErrorHandler
    Pipeline --> ToolRegistry
    ToolRegistry --> FileTools
    ToolRegistry --> ShellTool
    ToolRegistry --> WebTools
    ToolRegistry --> MCPTools
```

---

## 2. 核心工作流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant CLI as CLI入口
    participant UI as Ink UI
    participant QwenGen as QwenContentGenerator
    participant TokenMgr as SharedTokenManager
    participant OAuth as QwenOAuth2
    participant Parser as StreamingToolCallParser
    participant API as Qwen/DashScope API
    participant Tools as 工具系统
    
    User->>CLI: 运行 qwen
    CLI->>CLI: 检查认证状态
    
    alt 首次使用
        CLI->>OAuth: 启动OAuth流程
        OAuth->>User: 打开浏览器授权
        User->>OAuth: 完成授权
        OAuth->>TokenMgr: 保存Token
    end
    
    CLI->>UI: 启动Ink UI
    User->>UI: 输入提示词
    
    UI->>QwenGen: 提交消息
    QwenGen->>TokenMgr: 获取有效Token
    
    alt Token已过期
        TokenMgr->>OAuth: 刷新Token
        OAuth-->>TokenMgr: 返回新Token
    end
    
    TokenMgr-->>QwenGen: 返回Token+Endpoint
    
    QwenGen->>API: generateContentStream(流式)
    
    loop 流式响应
        API-->>QwenGen: 返回Chunk
        QwenGen->>Parser: 解析工具调用
        
        alt 工具调用Chunk
            Parser->>Parser: 累积Buffer
            Parser->>Parser: 跟踪JSON深度
            
            alt JSON完整
                Parser-->>QwenGen: 返回完整工具调用
                QwenGen-->>UI: 显示工具请求
                UI->>User: 请求确认
                User-->>UI: 批准
                UI-->>Tools: 执行工具
                Tools-->>QwenGen: 返回结果
                QwenGen->>API: 发送FunctionResponse
            end
        else 文本Chunk
            QwenGen-->>UI: 流式更新文本
            UI-->>User: 实时显示
        end
    end
    
    API-->>QwenGen: 完成信号
    QwenGen-->>UI: 更新完成状态
    UI-->>User: 显示Token使用
```

---

## 3. 关键差异：Qwen Code vs Gemini CLI

```mermaid
graph LR
    subgraph "Gemini CLI原始架构"
        GeminiClient[Gemini SDK<br/>@google/genai]
        GeminiParser[标准解析器]
        GeminiAuth[Google OAuth]
    end
    
    subgraph "Qwen Code改造架构"
        QwenAdapter[OpenAI兼容适配器]
        QwenParser[StreamingToolCallParser<br/>增强解析器]
        QwenAuthSys[多认证系统]
        
        subgraph "解析器增强"
            BufferMgmt[Buffer管理]
            DepthTrack[深度追踪]
            IndexCollision[索引碰撞处理]
            JsonRepair[JSON修复]
        end
        
        subgraph "认证系统"
            QwenOAuth2[Qwen OAuth2]
            APIKey[API Key模式]
            TokenRefresh[自动刷新]
        end
    end
    
    GeminiClient -.Fork改造.-> QwenAdapter
    GeminiParser -.大幅增强.-> QwenParser
    GeminiAuth -.扩展.-> QwenAuthSys
    
    QwenParser --> BufferMgmt
    QwenParser --> DepthTrack
    QwenParser --> IndexCollision
    QwenParser --> JsonRepair
    
    QwenAuthSys --> QwenOAuth2
    QwenAuthSys --> APIKey
    QwenAuthSys --> TokenRefresh
```

---

## 4. StreamingToolCallParser 核心创新

```mermaid
graph TB
    subgraph "问题场景"
        Problem1[工具调用索引碰撞<br/>同索引被复用]
        Problem2[JSON片段化<br/>跨多个Chunk]
        Problem3[缺失ID/Name<br/>元数据不完整]
        Problem4[格式不一致<br/>空字符串/部分JSON]
    end
    
    subgraph "解析器核心机制"
        BufferMap[buffers: Map<index, string><br/>累积每个工具调用的Buffer]
        DepthMap[depths: Map<index, number><br/>追踪JSON嵌套深度]
        StringMap[inStrings: Map<index, boolean><br/>是否在字符串内]
        MetaMap[toolCallMeta: Map<index, metadata><br/>存储ID和Name]
        IDIndexMap[idToIndexMap: Map<id, actualIndex><br/>解决索引碰撞]
    end
    
    subgraph "处理流程"
        AddChunk[addChunk<br/>接收新Chunk]
        ResolveIndex[解决索引碰撞<br/>ID映射]
        AccumulateBuffer[累积到Buffer]
        TrackState[更新解析状态<br/>深度/字符串]
        CheckComplete{深度==0?}
        AttemptParse[尝试解析JSON]
        RepairJSON[修复不完整JSON]
        ReturnResult[返回解析结果]
    end
    
    Problem1 --> BufferMap
    Problem2 --> DepthMap
    Problem3 --> MetaMap
    Problem4 --> StringMap
    
    BufferMap --> AddChunk
    DepthMap --> AddChunk
    StringMap --> AddChunk
    MetaMap --> AddChunk
    IDIndexMap --> AddChunk
    
    AddChunk --> ResolveIndex
    ResolveIndex --> AccumulateBuffer
    AccumulateBuffer --> TrackState
    TrackState --> CheckComplete
    CheckComplete -->|是| AttemptParse
    CheckComplete -->|否| ReturnResult
    AttemptParse -->|失败| RepairJSON
    AttemptParse -->|成功| ReturnResult
    RepairJSON --> ReturnResult
```

---

## 5. Qwen OAuth2 认证流程

```mermaid
sequenceDiagram
    participant User
    participant CLI
    participant OAuth as QwenOAuth2
    participant TokenMgr as SharedTokenManager
    participant Browser
    participant QwenServer as chat.qwen.ai
    participant FileSystem as ~/.qwen/oauth_creds.json
    
    User->>CLI: 首次运行 qwen
    CLI->>OAuth: 检查本地凭证
    OAuth->>FileSystem: 读取oauth_creds.json
    FileSystem-->>OAuth: 文件不存在
    
    OAuth->>OAuth: 生成PKCE pair<br/>(code_verifier, code_challenge)
    OAuth->>QwenServer: POST /device/code<br/>client_id + code_challenge
    QwenServer-->>OAuth: device_code + user_code + verification_uri
    
    OAuth->>User: 显示二维码/链接
    OAuth->>Browser: 打开verification_uri_complete
    User->>Browser: 扫码/点击授权
    Browser->>QwenServer: 用户授权
    
    loop 轮询Token
        OAuth->>QwenServer: POST /token<br/>device_code + code_verifier
        
        alt 等待授权
            QwenServer-->>OAuth: authorization_pending
            OAuth->>OAuth: 等待5秒
        else 授权完成
            QwenServer-->>OAuth: access_token + refresh_token + resource_url
            OAuth->>TokenMgr: 保存Token
            TokenMgr->>FileSystem: 写入oauth_creds.json
        end
    end
    
    OAuth-->>CLI: 认证成功
    CLI->>User: 开始会话
    
    Note over User,FileSystem: 后续使用
    
    CLI->>TokenMgr: 获取Token
    TokenMgr->>FileSystem: 读取oauth_creds.json
    
    alt Token有效
        TokenMgr-->>CLI: 返回Token
    else Token过期
        TokenMgr->>QwenServer: POST /token<br/>refresh_token
        QwenServer-->>TokenMgr: 新的access_token
        TokenMgr->>FileSystem: 更新oauth_creds.json
        TokenMgr-->>CLI: 返回新Token
    end
```

---

## 6. OpenAI兼容层架构

```mermaid
graph TB
    subgraph "Provider抽象层"
        OpenAIProvider[OpenAICompatibleProvider<br/>基础接口]
        
        subgraph "具体Provider实现"
            DefaultProvider[DefaultProvider<br/>标准OpenAI]
            DashScopeProvider[DashScopeProvider<br/>阿里云百炼]
            DeepSeekProvider[DeepSeekProvider<br/>DeepSeek]
            OpenRouterProvider[OpenRouterProvider<br/>OpenRouter]
        end
    end
    
    subgraph "Pipeline组件"
        Pipeline[ContentGenerationPipeline]
        
        subgraph "Pipeline内部"
            Converter[请求转换器<br/>Gemini→OpenAI]
            Client[OpenAI客户端]
            StreamParser[StreamingToolCallParser]
            ErrorHandler[EnhancedErrorHandler]
            Telemetry[TelemetryService]
        end
    end
    
    subgraph "内容生成器"
        OpenAIGen[OpenAIContentGenerator]
        QwenGen[QwenContentGenerator]
    end
    
    OpenAIProvider --> DefaultProvider
    OpenAIProvider --> DashScopeProvider
    OpenAIProvider --> DeepSeekProvider
    OpenAIProvider --> OpenRouterProvider
    
    DefaultProvider --> Pipeline
    DashScopeProvider --> Pipeline
    DeepSeekProvider --> Pipeline
    OpenRouterProvider --> Pipeline
    
    Pipeline --> Converter
    Pipeline --> Client
    Pipeline --> StreamParser
    Pipeline --> ErrorHandler
    Pipeline --> Telemetry
    
    OpenAIGen --> Pipeline
    QwenGen -.继承.-> OpenAIGen
    QwenGen --> DashScopeProvider
```

---

## 7. Vision Model自动切换机制

```mermaid
stateDiagram-v2
    [*] --> 检测输入
    
    检测输入 --> 包含图片: 发现图片
    检测输入 --> 无图片: 纯文本
    
    无图片 --> 使用当前模型
    
    包含图片 --> 检查设置
    
    检查设置 --> 显示对话框: vlmSwitchMode未设置
    检查设置 --> 一次性切换: vlmSwitchMode=once
    检查设置 --> 会话切换: vlmSwitchMode=session
    检查设置 --> 不切换: vlmSwitchMode=persist
    检查设置 --> 禁用视觉: visionModelPreview=false
    
    显示对话框 --> 用户选择
    用户选择 --> 一次性切换: 选择"once"
    用户选择 --> 会话切换: 选择"session"
    用户选择 --> 不切换: 选择"persist"
    
    一次性切换 --> 切换到Vision模型
    会话切换 --> 切换到Vision模型
    不切换 --> 使用当前模型
    禁用视觉 --> 使用当前模型
    
    切换到Vision模型 --> 处理图片请求
    处理图片请求 --> 恢复原模型: mode=once
    处理图片请求 --> 保持Vision模型: mode=session
    
    恢复原模型 --> [*]
    保持Vision模型 --> [*]
    使用当前模型 --> [*]
```

---

## 8. 会话Token管理与压缩

```mermaid
graph LR
    subgraph "Token监控"
        InputTokens[输入Token统计]
        OutputTokens[输出Token统计]
        HistoryTokens[历史Token统计]
        TotalCalc[总计计算]
    end
    
    subgraph "Session配置"
        SessionLimit[sessionTokenLimit<br/>会话Token上限]
        DefaultValue[默认值: 32000]
    end
    
    subgraph "达到上限处理"
        CheckLimit{超过限制?}
        
        subgraph "用户选项"
            Compress[/compress命令<br/>压缩历史]
            Clear[/clear命令<br/>清空历史]
            Stats[/stats命令<br/>查看统计]
        end
        
        subgraph "压缩机制"
            SelectHistory[选择压缩范围]
            CallSummarizer[调用Summarizer]
            ReplaceHistory[替换为摘要]
            RecalcTokens[重新计算Token]
        end
    end
    
    InputTokens --> TotalCalc
    OutputTokens --> TotalCalc
    HistoryTokens --> TotalCalc
    
    TotalCalc --> CheckLimit
    SessionLimit --> CheckLimit
    
    CheckLimit -->|是| Compress
    CheckLimit -->|是| Clear
    CheckLimit -->|是| Stats
    
    Compress --> SelectHistory
    SelectHistory --> CallSummarizer
    CallSummarizer --> ReplaceHistory
    ReplaceHistory --> RecalcTokens
    
    Clear --> TotalCalc
    RecalcTokens --> TotalCalc
```

---

## 9. 错误处理与重试机制

```mermaid
graph TB
    subgraph "EnhancedErrorHandler"
        CatchError[捕获错误]
        ClassifyError{错误分类}
        
        subgraph "错误类型"
            AuthError[认证错误<br/>401/403]
            RateLimitError[限流错误<br/>429]
            QuotaError[配额错误<br/>quota_exceeded]
            NetworkError[网络错误<br/>ECONNRESET]
            TimeoutError[超时错误<br/>ETIMEDOUT]
            ServerError[服务器错误<br/>5xx]
        end
        
        subgraph "处理策略"
            RefreshToken[刷新Token]
            BackoffRetry[退避重试]
            FriendlyMessage[友好错误提示]
            LogError[记录错误日志]
            SuppressLog[抑制日志<br/>避免噪音]
        end
    end
    
    CatchError --> ClassifyError
    
    ClassifyError --> AuthError
    ClassifyError --> RateLimitError
    ClassifyError --> QuotaError
    ClassifyError --> NetworkError
    ClassifyError --> TimeoutError
    ClassifyError --> ServerError
    
    AuthError --> RefreshToken
    AuthError --> SuppressLog
    RateLimitError --> BackoffRetry
    RateLimitError --> FriendlyMessage
    QuotaError --> FriendlyMessage
    QuotaError --> LogError
    NetworkError --> BackoffRetry
    TimeoutError --> BackoffRetry
    ServerError --> BackoffRetry
    ServerError --> LogError
```

---

## 10. 包结构与模块依赖

```mermaid
graph TB
    subgraph "Workspace Packages"
        CLI[packages/cli<br/>CLI包]
        CORE[packages/core<br/>核心包]
        VSCODE[packages/vscode-ide-companion<br/>VS Code扩展]
        TEST[packages/test-utils<br/>测试工具]
    end
    
    subgraph "Core内部模块"
        QwenMod[qwen/<br/>Qwen专用]
        CoreLogic[core/<br/>核心逻辑]
        ToolsMod[tools/<br/>工具实现]
        MCPMod[mcp/<br/>MCP集成]
        SubAgents[subagents/<br/>子代理]
    end
    
    subgraph "Qwen模块详情"
        QwenOAuth2[qwenOAuth2.ts<br/>OAuth认证]
        QwenContentGen[qwenContentGenerator.ts<br/>内容生成器]
        SharedTokenMgr[sharedTokenManager.ts<br/>Token管理]
    end
    
    subgraph "OpenAI兼容模块"
        OpenAIGen[openaiContentGenerator/<br/>生成器]
        Pipeline[pipeline.ts<br/>管道]
        StreamParser[streamingToolCallParser.ts<br/>流式解析]
        Providers[provider/<br/>Provider实现]
    end
    
    CLI --> CORE
    VSCODE --> CORE
    
    CORE --> QwenMod
    CORE --> CoreLogic
    CORE --> ToolsMod
    CORE --> MCPMod
    CORE --> SubAgents
    
    QwenMod --> QwenOAuth2
    QwenMod --> QwenContentGen
    QwenMod --> SharedTokenMgr
    
    CoreLogic --> OpenAIGen
    OpenAIGen --> Pipeline
    OpenAIGen --> StreamParser
    OpenAIGen --> Providers
    
    QwenContentGen -.继承.-> OpenAIGen
```

---

## 11. 技术栈与依赖对比

```mermaid
graph LR
    subgraph "Qwen Code技术栈"
        QwenLang[TypeScript/Node.js 20+]
        QwenUI[Ink + React]
        QwenAPI[OpenAI兼容API]
        QwenAuth[OAuth2 + PKCE]
        QwenParser[自定义StreamingParser]
        QwenTest[Vitest]
    end
    
    subgraph "Gemini CLI技术栈"
        GeminiLang[TypeScript/Node.js 20+]
        GeminiUI[Ink + React]
        GeminiAPI[@google/genai SDK]
        GeminiAuth[Google OAuth]
        GeminiParser[标准解析器]
        GeminiTest[Vitest]
    end
    
    subgraph "关键差异"
        APILayer[API层<br/>OpenAI兼容 vs Google SDK]
        ParserLayer[解析层<br/>增强解析 vs 标准解析]
        AuthLayer[认证层<br/>多Provider vs 单一Google]
    end
    
    QwenAPI -.主要差异.-> APILayer
    GeminiAPI -.主要差异.-> APILayer
    
    QwenParser -.核心创新.-> ParserLayer
    GeminiParser -.核心创新.-> ParserLayer
    
    QwenAuth -.扩展.-> AuthLayer
    GeminiAuth -.扩展.-> AuthLayer
```

---

## 12. 核心创新点总结

### 12.1 StreamingToolCallParser

```mermaid
graph TB
    Innovation1[索引碰撞解决<br/>idToIndexMap机制]
    Innovation2[JSON深度追踪<br/>准确判断完整性]
    Innovation3[字符串边界检测<br/>避免误判]
    Innovation4[自动JSON修复<br/>处理不完整输出]
    Innovation5[元数据分离存储<br/>toolCallMeta Map]
    
    Problem[Qwen模型工具调用<br/>流式输出不稳定]
    
    Problem --> Innovation1
    Problem --> Innovation2
    Problem --> Innovation3
    Problem --> Innovation4
    Problem --> Innovation5
    
    Innovation1 --> Solution[稳定可靠的<br/>工具调用解析]
    Innovation2 --> Solution
    Innovation3 --> Solution
    Innovation4 --> Solution
    Innovation5 --> Solution
```

### 12.2 认证系统增强

```mermaid
graph LR
    QwenOAuth[Qwen OAuth2<br/>通义千问官方]
    DashScope[DashScope<br/>阿里云百炼]
    ModelScope[ModelScope<br/>魔搭社区]
    OpenRouter[OpenRouter<br/>国际路由]
    CustomAPI[自定义API<br/>OpenAI兼容]
    
    SharedMgr[SharedTokenManager<br/>统一Token管理]
    
    QwenOAuth --> SharedMgr
    DashScope --> SharedMgr
    ModelScope --> SharedMgr
    OpenRouter --> SharedMgr
    CustomAPI --> SharedMgr
    
    SharedMgr --> Features[功能特性]
    
    Features --> AutoRefresh[自动刷新Token]
    Features --> EndpointSync[Endpoint同步]
    Features --> CredPersist[凭证持久化]
    Features --> ErrorHandle[统一错误处理]
```

---

## 13. 数据流与状态管理

```mermaid
graph LR
    subgraph "输入处理"
        UserInput[用户输入]
        ImageDetect[图片检测]
        VisionSwitch[Vision模型切换]
    end
    
    subgraph "内容生成"
        TokenGet[获取Token]
        BuildRequest[构建请求]
        StreamCall[流式API调用]
    end
    
    subgraph "流式解析"
        ChunkReceive[接收Chunk]
        ParseState[解析状态]
        
        subgraph "状态类型"
            TextChunk[文本Chunk]
            ToolChunk[工具调用Chunk]
            ThoughtChunk[思考过程Chunk]
        end
    end
    
    subgraph "输出渲染"
        UIUpdate[UI更新]
        ToolExec[工具执行]
        ResultDisplay[结果显示]
    end
    
    UserInput --> ImageDetect
    ImageDetect --> VisionSwitch
    VisionSwitch --> TokenGet
    
    TokenGet --> BuildRequest
    BuildRequest --> StreamCall
    
    StreamCall --> ChunkReceive
    ChunkReceive --> ParseState
    
    ParseState --> TextChunk
    ParseState --> ToolChunk
    ParseState --> ThoughtChunk
    
    TextChunk --> UIUpdate
    ToolChunk --> ToolExec
    ThoughtChunk --> UIUpdate
    
    UIUpdate --> ResultDisplay
    ToolExec --> ResultDisplay
```

---

## 14. Benchmark性能

```mermaid
graph TB
    subgraph "Terminal-Bench基准测试"
        TestSuite[测试套件]
        
        subgraph "测试结果"
            Qwen480[Qwen3-Coder-480A35<br/>准确率: 37.5%]
            Qwen30[Qwen3-Coder-30BA3B<br/>准确率: 31.3%]
        end
        
        subgraph "测试项目"
            FileOps[文件操作]
            ShellCmd[Shell命令]
            CodeEdit[代码编辑]
            MultiStep[多步骤任务]
        end
    end
    
    TestSuite --> Qwen480
    TestSuite --> Qwen30
    
    TestSuite --> FileOps
    TestSuite --> ShellCmd
    TestSuite --> CodeEdit
    TestSuite --> MultiStep
```

---

## 总结

### 核心特点

1. **基于Gemini CLI深度改造**: Fork自Google Gemini CLI，保留核心架构
2. **解析器层面大幅增强**: StreamingToolCallParser是核心创新
3. **多Provider认证支持**: 支持Qwen OAuth + 多个OpenAI兼容服务
4. **专为Qwen模型优化**: 处理Qwen3-Coder特有的输出特性
5. **Vision模型智能切换**: 自动检测图片并切换到视觉模型
6. **完善的Token管理**: SharedTokenManager统一管理所有认证

### 技术栈

- **语言**: TypeScript (Node.js >= 20)
- **UI框架**: Ink (React for Terminal)
- **API**: OpenAI Compatible API
- **认证**: OAuth2 + PKCE
- **测试**: Vitest
- **构建**: ESBuild

### 与Gemini CLI的主要差异

| 特性 | Qwen Code | Gemini CLI |
|------|-----------|------------|
| **API层** | OpenAI兼容 | Google GenAI SDK |
| **核心创新** | StreamingToolCallParser | 标准解析器 |
| **认证** | 多Provider (Qwen/DashScope/ModelScope) | Google OAuth |
| **模型支持** | Qwen3-Coder系列 | Gemini系列 |
| **解析器** | 索引碰撞处理+JSON修复 | 标准JSON解析 |
| **免费额度** | 2000请求/天 (Qwen OAuth) | 60请求/分钟 |

### 核心创新：StreamingToolCallParser

```
问题：Qwen模型工具调用输出不稳定
解决方案：
1. 索引碰撞处理 (idToIndexMap)
2. JSON深度追踪 (depths Map)
3. 字符串边界检测 (inStrings Map)
4. 自动JSON修复
5. Buffer累积机制
```

### 数据流总结

```
用户输入 → 图片检测 → Vision切换 → Token获取 → 请求构建
    ↓
流式API → Chunk接收 → StreamingParser → 解析完整工具调用
    ↓
工具执行 → 结果返回 → 继续流式 → 完成响应 → UI渲染
```

### 优势

- **免费额度高**: Qwen OAuth提供每天2000次请求
- **国内友好**: 支持阿里云百炼、魔搭社区
- **解析器强大**: 专门处理Qwen模型的输出特性
- **认证灵活**: 多种认证方式可选
- **开源透明**: 完全开源，Apache 2.0协议

Qwen Code通过在Gemini CLI基础上进行解析器层面的深度优化，为Qwen3-Coder模型提供了专业级的命令行AI助手体验。

