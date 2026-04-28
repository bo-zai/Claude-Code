# Claude Code MCP 实现分析

本文基于 Claude Code 源码，深入分析 MCP Server 的处理流程和请求报文封装机制。

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Claude Code MCP 实现架构                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐     ┌─────────────────────────────────────────────┐  │
│   │   配置加载       │     │              MCP Connection Manager         │  │
│   │                 │     │                                             │  │
│   │  getClaudeCode  │     │  useManageMCPConnections (React Hook)       │  │
│   │  McpConfigs()   │ ──→ │    ├─ 初始化服务器为 pending 状态            │  │
│   │                 │     │    ├─ 批量连接服务器                          │  │
│   │  settings.json  │     │    ├─ 处理连接回调                            │  │
│   │  .mcp.json      │     │    ├─ 自动重连（SSE/HTTP）                   │  │
│   │                 │     │    └─ 工具/资源变更通知                        │  │
│   └─────────────────┘     └─────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ↓                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      MCP Client (client.ts)                          │  │
│   │                                                                      │  │
│   │   connectToServer (memoized)                                         │  │
│   │     ├─ 根据配置类型创建 Transport                                     │  │
│   │     │   ├─ stdio: StdioClientTransport                               │  │
│   │     │   ├─ sse: SSEClientTransport                                   │  │
│   │     │   ├─ http: StreamableHTTPClientTransport                       │  │
│   │     │   ├─ ws: WebSocketTransport                                    │  │
│   │     │   └─ claudeai-proxy: StreamableHTTPClientTransport             │  │
│   │     ├─ 创建 MCP SDK Client                                           │  │
│   │     ├─ 执行握手 (initialize → initialized)                           │  │
│   │     ├─ 获取 capabilities                                             │  │
│   │     └─ 注册事件处理器 (onerror, onclose)                              │  │
│   │                                                                      │  │
│   │   fetchToolsForClient                                                │  │
│   │     ├─ 调用 tools/list                                               │  │
│   │     ├─ 转换为 Tool 对象                                               │  │
│   │     └─ 添加 mcpInfo 元数据                                           │  │
│   │                                                                      │  │
│   │   callMCPToolWithUrlElicitationRetry                                 │  │
│   │     ├─ 调用 tools/call                                               │  │
│   │     └─ 处理结果和错误                                                 │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ↓                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      API Layer (claude.ts)                           │  │
│   │                                                                      │  │
│   │   queryModel                                                         │  │
│   │     ├─ 构建 System Prompt                                            │  │
│   │     ├─ 转换消息格式                                                   │  │
│   │     ├─ 转换工具 Schema (toolToAPISchema)                             │  │
│   │     └─ 发送 API 请求                                                  │  │
│   │                                                                      │  │
│   │   toolToAPISchema                                                    │  │
│   │     ├─ name: 工具名                                                   │  │
│   │     ├─ description: 工具描述                                          │  │
│   │     ├─ input_schema: JSON Schema                                     │  │
│   │     ├─ cache_control: 缓存控制                                        │  │
│   │     └─ defer_loading: 延迟加载                                        │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ↓                                      │
│                             Anthropic API                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、MCP Server 连接流程

### 2.1 配置加载

配置来源（`src/services/mcp/config.ts`）：

```typescript
// 配置层级
type ConfigScope =
  | 'local'      // 本地 .mcp.json
  | 'user'       // 用户全局 settings.json
  | 'project'    // 项目级配置
  | 'dynamic'    // 插件动态配置
  | 'enterprise' // 企业级配置
  | 'claudeai'   // claude.ai 连接器
  | 'managed'    // 受管配置

// 服务器配置类型
type McpServerConfig =
  | McpStdioServerConfig     // stdio 进程
  | McpSSEServerConfig       // SSE 连接
  | McpHTTPServerConfig      // HTTP 连接
  | McpWebSocketServerConfig // WebSocket 连接
  | McpSdkServerConfig       // SDK 内置
  | McpClaudeAIProxyServerConfig // claude.ai 代理
```

### 2.2 连接管理器

核心 React Hook：`useManageMCPConnections` (`src/services/mcp/useManageMCPConnections.ts`)：

```typescript
export function useManageMCPConnections(
  dynamicMcpConfig: Record<string, ScopedMcpServerConfig> | undefined,
  isStrictMcpConfig = false,
) {
  // 核心职责:
  // 1. 初始化服务器为 pending 状态
  // 2. 批量连接服务器
  // 3. 处理连接回调 (onConnectionAttempt)
  // 4. 自动重连 (SSE/HTTP 断线后)
  // 5. 工具/资源变更通知处理

  // 状态批量更新机制
  const MCP_BATCH_FLUSH_MS = 16
  const pendingUpdatesRef = useRef<PendingUpdate[]>([])

  // 自动重连配置
  const MAX_RECONNECT_ATTEMPTS = 5
  const INITIAL_BACKOFF_MS = 1000
  const MAX_BACKOFF_MS = 30000
}
```

**连接回调处理流程**：

```typescript
const onConnectionAttempt = useCallback(({
  client, tools, commands, resources
}) => {
  // 1. 更新服务器状态
  updateServer({ ...client, tools, commands, resources })

  // 2. 处理不同状态
  switch (client.type) {
    case 'connected':
      // 注册 elicitation 处理器
      registerElicitationHandler(client.client, client.name, setAppState)

      // 设置关闭回调 (自动重连)
      client.client.onclose = () => {
        // 清除缓存
        clearServerCache(client.name, client.config)

        // 检查是否被禁用
        if (isMcpServerDisabled(client.name)) return

        // SSE/HTTP 自动重连 (指数退避)
        if (configType !== 'stdio' && configType !== 'sdk') {
          reconnectWithBackoff() // 最多 5 次，每次间隔加倍
        }
      }

      // 注册工具/资源变更通知处理器
      if (client.capabilities?.tools?.listChanged) {
        client.client.setNotificationHandler(
          ToolListChangedNotificationSchema,
          async () => {
            // 刷新工具列表
            const newTools = await fetchToolsForClient(client)
            updateServer({ ...client, tools: newTools })
          }
        )
      }
      break

    case 'needs-auth':
    case 'failed':
    case 'pending':
    case 'disabled':
      break
  }
}, [updateServer])
```

---

## 三、MCP Server 连接实现

### 3.1 connectToServer 核心逻辑

```typescript
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: {...}
  ): Promise<MCPServerConnection> => {
    let transport

    // 根据配置类型创建 Transport
    if (serverRef.type === 'sse') {
      // SSE Transport
      const authProvider = new ClaudeAuthProvider(name, serverRef)
      const combinedHeaders = await getMcpServerHeaders(name, serverRef)

      transport = new SSEClientTransport(
        new URL(serverRef.url),
        {
          authProvider,
          fetch: wrapFetchWithTimeout(...),
          requestInit: { headers: {...} }
        }
      )

    } else if (serverRef.type === 'http') {
      // HTTP Transport (Streamable HTTP)
      const authProvider = new ClaudeAuthProvider(name, serverRef)
      transport = new StreamableHTTPClientTransport(
        new URL(serverRef.url),
        {...}
      )

    } else if (serverRef.type === 'ws') {
      // WebSocket Transport
      transport = new WebSocketTransport(wsClient)

    } else if (serverRef.type === 'stdio' || !serverRef.type) {
      // Stdio Transport (最常用)
      transport = new StdioClientTransport({
        command: serverRef.command,
        args: serverRef.args,
        env: { ...subprocessEnv(), ...serverRef.env },
        stderr: 'pipe'
      })
    }

    // 创建 MCP SDK Client
    const client = new Client(
      {
        name: 'claude-code',
        title: 'Claude Code',
        version: MACRO.VERSION,
        description: "Anthropic's agentic coding tool"
      },
      {
        capabilities: {
          roots: {},
          elicitation: {}
        }
      }
    )

    // 注册 roots 处理器
    client.setRequestHandler(ListRootsRequestSchema, async () => {
      return { roots: [{ uri: `file://${getOriginalCwd()}` }] }
    })

    // 执行连接（带超时）
    const connectPromise = client.connect(transport)
    await Promise.race([
      connectPromise,
      timeoutPromise(getConnectionTimeoutMs())
    ])

    // 获取 capabilities
    const capabilities = client.getServerCapabilities()
    const serverVersion = client.getServerVersion()
    const instructions = client.getInstructions()

    // 返回连接结果
    return {
      name,
      client,
      type: 'connected',
      capabilities,
      serverInfo: serverVersion,
      instructions,
      config: serverRef,
      cleanup: wrappedCleanup
    }
  }
)
```

### 3.2 Transport 类型详解

| Transport | 使用场景 | 特点 |
|-----------|----------|------|
| `StdioClientTransport` | 本地 MCP Server | 进程通信，最常用 |
| `SSEClientTransport` | 远程 SSE Server | 支持自动重连、OAuth |
| `StreamableHTTPClientTransport` | HTTP MCP Server | 新标准，支持双向流 |
| `WebSocketTransport` | WebSocket Server | 长连接，IDE 集成 |
| `claudeai-proxy` | claude.ai 连接器 | OAuth 代理 |

---

## 四、工具获取与转换

### 4.1 fetchToolsForClient

```typescript
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.tools) return []

    // 调用 MCP tools/list
    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema
    )

    // 转换 MCP 工具为 Claude Code Tool 格式
    return result.tools.map((tool): Tool => {
      const fullyQualifiedName = buildMcpToolName(client.name, tool.name)

      return {
        // 基础属性
        name: fullyQualifiedName,
        mcpInfo: { serverName: client.name, toolName: tool.name },
        isMcp: true,

        // 描述
        description: async () => tool.description ?? '',
        prompt: async () => {
          const desc = tool.description ?? ''
          return desc.length > MAX_MCP_DESCRIPTION_LENGTH
            ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
            : desc
        },

        // 输入 Schema
        inputJSONSchema: tool.inputSchema,

        // 工具特性
        isConcurrencySafe: () => tool.annotations?.readOnlyHint ?? false,
        isReadOnly: () => tool.annotations?.readOnlyHint ?? false,
        isDestructive: () => tool.annotations?.destructiveHint ?? false,
        isOpenWorld: () => tool.annotations?.openWorldHint ?? false,

        // 工具调用实现
        call: async (args, context, ...) => {
          const connectedClient = await ensureConnectedClient(client)
          const mcpResult = await callMCPToolWithUrlElicitationRetry({
            client: connectedClient,
            tool: tool.name,
            args,
            meta: { 'claudecode/toolUseId': toolUseId },
            signal: context.abortController.signal,
            onProgress: ...
          })

          return { data: mcpResult.content }
        },

        // 权限检查
        checkPermissions: async () => ({
          behavior: 'passthrough',
          message: 'MCPTool requires permission.',
          suggestions: [{
            type: 'addRules',
            rules: [{ toolName: fullyQualifiedName }],
            behavior: 'allow',
            destination: 'localSettings'
          }]
        })
      }
    })
  }
)
```

### 4.2 工具命名规则

```typescript
// 构建完整的 MCP 工具名
function buildMcpToolName(serverName: string, toolName: string): string {
  // 格式: mcp__{serverName}__{toolName}
  // 例如: mcp__code-memory__query
  return `mcp__${normalizeNameForMCP(serverName)}__${normalizeNameForMCP(toolName)}`
}
```

---

## 五、工具调用流程

### 5.1 callMCPToolWithUrlElicitationRetry

```typescript
async function callMCPToolWithUrlElicitationRetry({
  client,
  clientConnection,
  tool,
  args,
  meta,
  signal,
  onProgress,
  handleElicitation
}) {
  // 1. 发送 tools/call 请求
  const result = await client.request(
    {
      method: 'tools/call',
      params: {
        name: tool,
        arguments: args,
        _meta: meta
      }
    },
    CallToolResultSchema
  )

  // 2. 处理结果
  if (result.isError) {
    throw new McpToolCallError(result.content)
  }

  // 3. 处理内容
  // - text: 文本内容
  // - image: 图片内容（base64）
  // - resource: 资源引用

  return result
}
```

### 5.2 进度通知

```typescript
// MCP 进度通知格式
type MCPProgress = {
  type: 'mcp_progress'
  status: 'started' | 'completed' | 'failed'
  serverName: string
  toolName: string
  elapsedTimeMs?: number
}

// 发送进度
if (onProgress && toolUseId) {
  onProgress({
    toolUseID: toolUseId,
    data: {
      type: 'mcp_progress',
      status: 'started',
      serverName: client.name,
      toolName: tool.name
    }
  })
}
```

---

## 六、请求报文封装

### 6.1 toolToAPISchema

将 Tool 转换为 API 请求格式：

```typescript
export async function toolToAPISchema(
  tool: Tool,
  options: {...}
): Promise<BetaToolUnion> {
  // 使用缓存避免重复计算
  const cacheKey = tool.name
  const cache = getToolSchemaCache()

  // 基础 Schema
  const base = {
    name: tool.name,
    description: await tool.prompt({...}),
    input_schema: tool.inputJSONSchema ?? zodToJsonSchema(tool.inputSchema)
  }

  // 添加 strict 模式（如果支持）
  if (strictToolsEnabled && tool.strict && modelSupportsStructuredOutputs(model)) {
    base.strict = true
  }

  // 添加 fine-grained tool streaming
  if (isFirstPartyAnthropicBaseUrl()) {
    base.eager_input_streaming = true
  }

  // 添加 defer_loading（工具搜索功能）
  if (options.deferLoading) {
    base.defer_loading = true
  }

  // 添加 cache_control
  if (options.cacheControl) {
    base.cache_control = options.cacheControl
  }

  return base as BetaTool
}
```

### 6.2 API 请求构建

```typescript
export async function queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options
) {
  // 1. 转换消息格式
  const normalizedMessages = normalizeMessagesForAPI(messages)

  // 2. 转换工具 Schema
  const toolSchemas = await Promise.all(
    tools.map(tool => toolToAPISchema(tool, {...}))
  )

  // 3. 构建 API 请求
  const params: BetaMessageStreamParams = {
    model: options.model,
    max_tokens: getMaxOutputTokens(options.model),
    system: systemPrompt,
    messages: normalizedMessages,
    tools: toolSchemas,
    ...(toolChoice && { tool_choice: toolChoice }),
    ...(betas.length > 0 && { betas }),
    metadata: getAPIMetadata(),
    ...getExtraBodyParams()
  }

  // 4. 发送请求
  const stream = await anthropic.beta.messages.stream(params)

  // 5. 处理流式响应
  for await (const event of stream) {
    yield normalizeStreamEvent(event)
  }
}
```

### 6.3 请求报文格式

```typescript
// 最终发送给 API 的请求格式
{
  model: "claude-sonnet-4-6",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: "系统提示内容...",
      cache_control: { type: "ephemeral" }
    }
  ],
  messages: [
    {
      role: "user",
      content: "帮我分析代码仓库结构"
    },
    {
      role: "assistant",
      content: [
        { type: "text", text: "..." },
        {
          type: "tool_use",
          id: "toolu_001",
          name: "mcp__code-memory__query",
          input: { query: "..." }
        }
      ]
    },
    {
      role: "user",
      content: [
        {
          type: "tool_result",
          tool_use_id: "toolu_001",
          content: "查询结果..."
        }
      ]
    }
  ],
  tools: [
    {
      name: "mcp__code-memory__query",
      description: "查询代码知识图谱...",
      input_schema: {
        type: "object",
        properties: {
          query: { type: "string", description: "..." }
        },
        required: ["query"]
      },
      cache_control: { type: "ephemeral" }
    },
    {
      name: "Read",
      description: "读取文件...",
      input_schema: {...}
    }
  ],
  betas: ["prompt-caching...", "token-efficient-tools..."],
  metadata: {
    user_id: "{\"device_id\":\"...\",\"session_id\":\"...\"}"
  }
}
```

---

## 七、状态管理

### 7.1 AppState MCP 结构

```typescript
interface AppState {
  mcp: {
    clients: MCPServerConnection[]  // 所有服务器连接
    tools: Tool[]                   // 所有可用工具
    commands: Command[]             // 所有可用命令（prompts）
    resources: Record<string, ServerResource[]>  // 资源列表
    pluginReconnectKey: number      // 重连触发器
  }
}

// 服务器连接状态
type MCPServerConnection =
  | ConnectedMCPServer    // 已连接
  | FailedMCPServer       // 连接失败
  | NeedsAuthMCPServer    // 需要认证
  | PendingMCPServer      // 等待连接
  | DisabledMCPServer     // 已禁用
```

### 7.2 连接状态流转

```
初始状态:
  配置加载 → pending

连接成功:
  pending → connected
  └─ 获取 tools、commands、resources

连接失败:
  pending → failed
  └─ 记录错误信息

需要认证:
  SSE/HTTP 401 → needs-auth
  └─ 缓存 15 分钟

断线重连:
  connected → pending → connected
  └─ SSE/HTTP 自动重连（指数退避）

手动禁用:
  connected/disabled → disabled
  └─ 断开连接，清除缓存

手动启用:
  disabled → pending → connected
  └─ 重新连接
```

---

## 八、缓存机制

### 8.1 连接缓存

```typescript
// connectToServer 使用 memoize 缓存
export const connectToServer = memoize(async (...) => {...})

// 缓存键
function getServerCacheKey(name: string, config: ScopedMcpServerConfig): string {
  return `${name}:${JSON.stringify(config)}`
}

// 断开时清除缓存
client.onclose = () => {
  const key = getServerCacheKey(name, config)
  connectToServer.cache.delete(key)

  // 同时清除工具/资源缓存
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
}
```

### 8.2 工具 Schema 缓存

```typescript
// toolSchemaCache.ts
export function getToolSchemaCache(): Map<string, BetaToolUnion> {
  // 缓存工具的 name、description、input_schema
  // 避免 mid-session GrowthBook 翻转导致重复计算
}

// 缓存键
const cacheKey = 'inputJSONSchema' in tool && tool.inputJSONSchema
  ? `${tool.name}:${JSON.stringify(tool.inputJSONSchema)}`
  : tool.name
```

---

## 九、错误处理

### 9.1 连接错误

```typescript
// 增强错误处理
client.onerror = (error: Error) => {
  const transportType = serverRef.type || 'stdio'

  // 记录详细错误信息
  if (error.message.includes('ECONNRESET')) {
    logMCPDebug(name, 'Connection reset - server may have crashed')
  } else if (error.message.includes('ETIMEDOUT')) {
    logMCPDebug(name, 'Connection timeout - network issue')
  }

  // SSE/HTTP 终端错误计数
  if (isTerminalConnectionError(error.message)) {
    consecutiveConnectionErrors++
    if (consecutiveConnectionErrors >= MAX_ERRORS_BEFORE_RECONNECT) {
      closeTransportAndRejectPending('max consecutive terminal errors')
    }
  }
}
```

### 9.2 工具调用错误

```typescript
// McpToolCallError 携带 _meta 元数据
export class McpToolCallError extends TelemetrySafeError {
  constructor(message: string, telemetryMessage: string, readonly mcpMeta?) {
    super(message, telemetryMessage)
  }
}

// Session expired 错误检测
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = error.code
  if (httpStatus !== 404) return false

  // 检查 JSON-RPC 错误码 -32001
  return error.message.includes('"code":-32001')
}
```

---

## 十、关键文件清单

| 文件路径 | 核心功能 |
|----------|----------|
| `src/services/mcp/MCPConnectionManager.tsx` | React Context，连接管理 |
| `src/services/mcp/useManageMCPConnections.ts` | 连接 Hook，状态管理 |
| `src/services/mcp/client.ts` | 连接实现，工具获取 |
| `src/services/mcp/config.ts` | 配置加载 |
| `src/services/mcp/types.ts` | 类型定义 |
| `src/services/api/claude.ts` | API 调用，请求构建 |
| `src/utils/api.ts` | toolToAPISchema 转换 |
| `src/tools/MCPTool/MCPTool.ts` | MCP 工具定义 |

---

## 总结

Claude Code 的 MCP 实现：

1. **连接管理**：使用 React Hook + Context 管理所有 MCP 连接
2. **多 Transport 支持**：stdio、SSE、HTTP、WebSocket
3. **自动重连**：SSE/HTTP 断线后指数退避重连
4. **工具转换**：MCP 工具 → Claude Tool 格式，注入 API Schema
5. **缓存优化**：连接、工具 Schema 都有缓存机制
6. **请求封装**：完整的消息 + 工具定义 → API 请求格式