# MCP 核心组件

MCP Server 提供三种核心能力：Tools（工具）、Resources（资源）、Prompts（提示模板）。

## Tool 和 Resource 的本质区别

Tool 是"动作"，Resource 是"数据"。这个区别决定了它们的使用方式、权限控制和交互模式。

### 动作 vs 数据

Tool 对应 HTTP 的 POST 请求，意味着执行某个操作。操作可能产生结果，可能修改状态，可能触发副作用。比如查询数据库是一个 Tool，因为它执行了查询动作；调用外部 API 是一个 Tool，因为它发起了网络请求。

Resource 对应 HTTP 的 GET 请求，意味着读取某个数据源。数据源是静态存在的，读取不会改变它。比如配置文件是一个 Resource，它本身就存在，读取只是获取其内容；项目文档是一个 Resource，它不会因为被读取而改变。

```
Tool:  动作 → 执行 → 结果（可能有副作用）
Resource: 数据 → 读取 → 内容（无副作用）
```

### 权限差异

Tool 执行操作，可能修改数据、可能访问敏感系统，通常需要用户审批。Agent 会询问用户是否允许执行某个 Tool。

Resource 读取数据，通常只读不写，权限相对宽松。Agent 可以直接读取已声明的 Resource，不需要每次审批。

不过 Resource 的"宽松"是相对的。如果 Resource 包含敏感数据（如密码配置），Agent 可能仍然会提示用户。关键在于数据本身的敏感程度。

### 调用时机

Tool 由模型决策调用。模型根据用户问题，判断是否需要使用某个 Tool，决定调用时机、参数、次数。

Resource 的调用时机分两种情况：静态 Resource 在 Agent 启动时就可能预加载；动态 Resource 在模型需要时才读取。但无论哪种，Resource 都不需要模型"决策是否调用"，它更像是一个可随时访问的数据源。

---

## Tool 深入解析

### Tool 的完整结构

一个 Tool 包含三部分：定义（告诉 Agent 有什么工具）、调用（执行工具）、结果（返回执行结果）。

**定义部分**：

```json
{
  "name": "query",
  "description": "查询代码知识图谱，返回相关代码符号和文件位置",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "查询内容，可以是关键词或自然语言描述"
      },
      "limit": {
        "type": "number",
        "description": "返回结果数量限制",
        "default": 5
      }
    },
    "required": ["query"]
  },
  "_meta": {
    "readOnlyHint": false,
    "destructiveHint": false,
    "openWorldHint": true
  }
}
```

`inputSchema` 使用 JSON Schema 格式，定义了工具接受的参数。Agent 会把这个 Schema 转换成 API 调用格式发给模型，模型根据 Schema 生成调用参数。

`_meta` 是可选的元数据，提供权限提示：
- `readOnlyHint`: 只读操作，不修改数据
- `destructiveHint`: 破坏性操作，可能删除数据
- `openWorldHint`: 访问外部系统，如网络请求

这些提示帮助 Agent 和用户判断工具的风险程度。

**调用部分**：

Agent 通过 `tools/call` 方法调用工具：

```json
{
  "method": "tools/call",
  "params": {
    "name": "query",
    "arguments": {
      "query": "用户登录流程",
      "limit": 10
    }
  }
}
```

MCP Server 收到请求，执行工具逻辑，返回结果。

**结果部分**：

```json
{
  "content": [
    {
      "type": "text",
      "text": "找到 3 个相关代码符号：\n1. login() - src/auth/login.ts:15\n2. authenticate() - src/auth/authenticate.ts:42\n3. UserLogin - src/auth/types.ts:8"
    }
  ],
  "isError": false
}
```

`content` 是数组，可以包含多个内容块。每个块有 `type`，常见类型：`text`（文本）、`image`（图片）、`resource`（嵌入资源）。

`isError` 标识是否出错。如果为 `true`，`content` 包含错误信息。

### Tool 的输入验证

Agent 收到模型的调用请求后，会验证参数是否符合 `inputSchema`：

```typescript
// 验证必填参数
if (schema.required && schema.required.includes("query")) {
  if (!arguments.query) {
    return { isError: true, content: [{ type: "text", text: "缺少必填参数 query" }] }
  }
}

// 验证参数类型
if (arguments.limit && typeof arguments.limit !== "number") {
  return { isError: true, content: [{ type: "text", text: "limit 必须是数字" }] }
}

// 验证枚举值
if (schema.properties.unit && schema.properties.unit.enum) {
  if (!schema.properties.unit.enum.includes(arguments.unit)) {
    return { isError: true, content: [{ type: "text", text: `unit 必须是 ${schema.properties.unit.enum.join(",")} 之一` }] }
  }
}
```

验证失败时，Agent 返回错误结果，模型看到错误后可以调整参数重新调用。

### Tool 的错误处理

Tool 执行可能出错，错误分两类：参数错误（验证失败）、执行错误（运行时失败）。

参数错误在调用前发生，Agent 直接返回错误，不执行工具。

执行错误在调用中发生，MCP Server 捕获异常，返回错误结果：

```json
{
  "content": [
    {
      "type": "text",
      "text": "查询失败：数据库连接超时"
    }
  ],
  "isError": true
}
```

模型收到错误后，可能尝试其他方案，或者向用户解释问题。

### Tool 的幂等性

幂等性指多次调用产生相同结果。设计 Tool 时应考虑幂等性：

- **查询类 Tool**：天然幂等，多次查询结果相同（假设数据未变）
- **修改类 Tool**：需要设计幂等，比如用唯一标识防止重复创建
- **删除类 Tool**：天然幂等，删除已删除的数据返回相同结果

幂等性影响重试策略。幂等的 Tool 可以安全重试；非幂等的 Tool 重试可能导致重复操作。

### 实际案例：code-memory MCP 的 query Tool

code-memory MCP Server 提供一个 `query` 工具，查询代码知识图谱：

```typescript
// 工具定义
{
  name: "query",
  description: "Query the code knowledge graph for execution flows related to a concept.",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string", description: "Natural language or keyword search query" },
      limit: { type: "number", default: 5, description: "Max processes to return" },
      include_content: { type: "boolean", default: false, description: "Include full symbol source code" }
    },
    required: ["query"]
  }
}

// 工具调用
{
  method: "tools/call",
  params: {
    name: "query",
    arguments: {
      query: "用户认证流程",
      limit: 5
    }
  }
}

// 返回结果
{
  content: [
    {
      type: "text",
      text: "找到执行流程：\n\nProcess: UserLogin\nSteps:\n1. validateInput() - 检查输入参数\n2. authenticate() - 验证用户身份\n3. createSession() - 创建会话\n4. returnToken() - 返回令牌\n\n相关文件：\n- src/auth/login.ts\n- src/auth/session.ts"
    }
  ]
}
```

这个工具是只读查询（`readOnlyHint: true`），不修改知识图谱数据。Agent 通常自动允许，不需要每次审批。

---

## Resource 深入解析

### Resource 的本质

Resource 是 MCP Server 暴露的数据源。Agent 可以读取 Resource 内容，但 Resource 本身由 MCP Server 管理。

类比：Tool 是 API 接口，Resource 是文件系统。Tool 需要调用才能执行，Resource 随时可以读取。

### Resource URI

每个 Resource 有唯一 URI，格式类似 URL：

```
config://app.json        配置资源
file:///path/to/file     文件资源
db://users/123          数据库记录
memory://context        内存数据
```

URI 是 Resource 的标识符，不是真实的 URL。Agent 通过 URI 读取 Resource，但 URI 如何映射到实际数据由 MCP Server 决定。

比如 `config://app.json` 可能映射到 `/etc/app/config.json` 文件，也可能映射到数据库中的配置表，这完全由 MCP Server 实现决定。

### Resource 的两种类型

**静态 Resource**：内容固定，列出时就知道所有 Resource。比如配置文件、文档模板、静态数据。

```json
{
  "resources": [
    {
      "uri": "config://app.json",
      "name": "应用配置",
      "mimeType": "application/json"
    },
    {
      "uri": "template://review.md",
      "name": "代码审查模板",
      "mimeType": "text/markdown"
    }
  ]
}
```

Agent 启动时通过 `resources/list` 获取所有 Resource 列表，可以预加载或按需读取。

**动态 Resource**：内容动态变化，可能需要订阅或按需生成。比如实时日志、数据库查询结果、运行状态。

动态 Resource 通常使用订阅机制：

```json
// 订阅资源
{
  "method": "resources/subscribe",
  "params": {
    "uri": "log://app.log"
  }
}

// 资源更新通知
{
  "method": "notifications/resources/updated",
  "params": {
    "uri": "log://app.log"
  }
}
```

订阅后，Resource 内容变化时 MCP Server 会通知 Agent，Agent 可以重新读取获取最新内容。

### Resource 的读取

Agent 通过 `resources/read` 读取 Resource：

```json
{
  "method": "resources/read",
  "params": {
    "uri": "config://app.json"
  }
}
```

返回内容：

```json
{
  "contents": [
    {
      "uri": "config://app.json",
      "mimeType": "application/json",
      "text": "{\"database\": \"mysql\", \"port\": 3306, \"host\": \"localhost\"}"
    }
  ]
}
```

`contents` 是数组，可以返回多个内容块。`text` 是文本内容，如果是二进制文件用 `blob` 字段（base64 编码）。

`mimeType` 标识内容类型，Agent 可以根据类型决定如何处理。比如 JSON 可以解析，图片可以展示。

### Resource Template

有些 Resource 是参数化的，比如读取特定用户的信息：

```
user://{userId}/profile
```

这类 Resource 使用 Resource Template 定义：

```json
{
  "resourceTemplates": [
    {
      "uriTemplate": "user://{userId}/profile",
      "name": "用户资料",
      "description": "获取指定用户的资料信息",
      "mimeType": "application/json",
      "parameters": [
        {
          "name": "userId",
          "type": "string",
          "description": "用户ID",
          "required": true
        }
      ]
    }
  ]
}
```

Agent 通过模板知道有哪些参数化的 Resource，读取时传入具体 URI：

```json
{
  "method": "resources/read",
  "params": {
    "uri": "user://123/profile"
  }
}
```

Resource Template 和 Tool 的区别：Template 返回数据，Tool 执行操作。如果只是读取数据用 Template，如果需要计算或处理用 Tool。

### Resource vs Tool 的选择

读取数据时，应该用 Resource 还是 Tool？

用 Resource：
- 数据本身就是静态存在的（文件、配置、数据库记录）
- 读取不产生副作用
- 数据格式固定（JSON、文本、图片）

用 Tool：
- 数据需要计算或处理后返回（查询结果、分析报告）
- 需要参数化查询（搜索、筛选）
- 返回结果依赖执行过程

实践中很多场景两者都能实现，选择取决于语义。如果强调"读取某物"用 Resource，如果强调"执行某操作"用 Tool。

---

## 实际案例对比

### 案例 1：获取配置信息

**Resource 方式**：

```json
// 定义
{
  "uri": "config://app.json",
  "name": "应用配置"
}

// 读取
resources/read { "uri": "config://app.json" }

// 返回
{ "contents": [{ "text": "{...}" }] }
```

语义：配置是一个数据源，可以随时读取。

**Tool 方式**：

```json
// 定义
{
  "name": "get_config",
  "description": "获取应用配置"
}

// 调用
tools/call { "name": "get_config", "arguments": {} }

// 返回
{ "content": [{ "text": "{...}" }] }
```

语义：获取配置是一个操作，需要执行。

这个场景用 Resource 更自然，配置本身就是存在的数据。

### 案例 2：查询代码知识图谱

**Tool 方式**：

```json
{
  "name": "query",
  "description": "查询代码知识图谱",
  "inputSchema": {
    "properties": {
      "query": { "type": "string" }
    }
  }
}
```

需要传入查询参数，查询过程涉及计算（搜索、匹配、排序），结果依赖执行过程。用 Tool 更合适。

### 案例 3：读取项目文档

**Resource 方式**：

```json
{
  "resources": [
    { "uri": "docs://README.md", "name": "项目说明" },
    { "uri": "docs://API.md", "name": "API文档" },
    { "uri": "docs://CHANGELOG.md", "name": "更新日志" }
  ]
}
```

文档是静态文件，直接读取即可。用 Resource。

---

## Tool 和 Resource 的协作

Tool 和 Resource 经常协作使用：

```
用户: "帮我分析 src/auth.ts 的代码质量"

Agent 流程:
1. Tool: read_file({ path: "src/auth.ts" })  → 获取文件内容
2. Resource: config://rules.json             → 读取代码质量规则
3. Tool: context({ name: "auth" })           → 获取函数调用关系
4. 分析并生成报告
```

Tool 获取动态数据（文件内容、调用关系），Resource 获取静态配置（质量规则）。两者配合完成任务。

---

## Prompts（提示模板）

预定义的可复用提示模板，用户可以快速调用。

```json
{
  "name": "code_review",
  "description": "代码审查提示模板",
  "arguments": [
    { "name": "language", "required": true },
    { "name": "file_path", "required": true }
  ]
}
```

Agent 通过 `prompts/get` 获取模板内容，填入参数后生成 Prompt 发给模型。

---

## 组件对比总结

| 特性 | Tool | Resource | Prompt |
|------|------|----------|--------|
| **本质** | 动作/操作 | 数据源 | 提示模板 |
| **HTTP类比** | POST | GET | - |
| **副作用** | 可能有 | 无 | 无 |
| **权限** | 通常需审批 | 相对宽松 | 自动允许 |
| **调用方式** | tools/call | resources/read | prompts/get |
| **参数** | inputSchema | URI | arguments |
| **返回** | 执行结果 | 数据内容 | Prompt文本 |

一句话：Tool 执行操作，Resource 读取数据，Prompt 提供模板。

---

## 下一步

- 了解通信协议细节，请阅读 [通信协议](./05-protocol.md)
- 了解如何开发 MCP 服务器，请阅读 [开发指南](./06-development.md)