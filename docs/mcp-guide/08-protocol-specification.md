# 协议规范详解

本文深入讲解 MCP 使用的 JSON-RPC 2.0 协议规范和 MCP 自身的方法定义。

---

## 一、JSON-RPC 2.0 规范

### 1.1 协议概述

JSON-RPC 是一种轻量级的远程过程调用协议，使用 JSON 作为数据格式。

```
┌─────────────────────────────────────────────────────────────────┐
│                      JSON-RPC 特点                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • 简单: 仅使用 JSON，无复杂定义                                  │
│  • 无状态: 每个请求独立，不依赖之前请求                           │
│  • 双向: 双方都可以发送请求和响应                                 │
│  • 版本固定: 使用 "jsonrpc": "2.0" 标识                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 消息类型详解

#### Request（请求）

必须有 `id` 字段，需要响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "query",
    "arguments": { "query": "test" }
  }
}
```

字段说明：
| 字段 | 必需 | 说明 |
|------|------|------|
| jsonrpc | 是 | 固定值 "2.0" |
| id | 是 | 请求标识符，用于匹配响应 |
| method | 是 | 要调用的方法名 |
| params | 否 | 方法参数 |

#### Response（成功响应）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "查询结果..." }
    ]
  }
}
```

字段说明：
| 字段 | 必需 | 说明 |
|------|------|------|
| jsonrpc | 是 | 固定值 "2.0" |
| id | 是 | 对应请求的 id |
| result | 是 | 方法返回结果（成功时） |

#### Response（错误响应）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "field": "query",
      "reason": "required"
    }
  }
}
```

字段说明：
| 字段 | 必需 | 说明 |
|------|------|------|
| jsonrpc | 是 | 固定值 "2.0" |
| id | 是 | 对应请求的 id |
| error | 是 | 错误对象（失败时） |

error 对象：
| 字段 | 必需 | 说明 |
|------|------|------|
| code | 是 | 错误码（整数） |
| message | 是 | 错误描述 |
| data | 否 | 错误详情 |

#### Notification（通知）

无 `id` 字段，不需要响应：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "token-123",
    "progress": 50,
    "total": 100
  }
}
```

用途：
- 告知对方某事件已发生
- 发送进度更新
- 不需要对方回复

### 1.3 错误码定义

```
┌─────────────────────────────────────────────────────────────────┐
│                      JSON-RPC 错误码                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  标准错误码:                                                     │
│  ┌───────────┬─────────────────────────────────────────────────┐│
│  │ 错误码    │ 说明                                            ││
│  ├───────────┼─────────────────────────────────────────────────┤│
│  │ -32700    │ Parse error: JSON 解析失败                       ││
│  │ -32600    │ Invalid Request: 请求格式无效                    ││
│  │ -32601    │ Method not found: 方法不存在                     ││
│  │ -32602    │ Invalid params: 参数无效                         ││
│  │ -32603    │ Internal error: 服务器内部错误                   ││
│  └───────────┴─────────────────────────────────────────────────┘│
│                                                                 │
│  服务器自定义错误码:                                             │
│  ┌───────────┬─────────────────────────────────────────────────┐│
│  │ -32000    │ Server error (通用)                             ││
│  │ -32099    │ 保留给实现定义                                   ││
│  │ -32100+   │ 可自定义使用                                     ││
│  └───────────┴─────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 ID 处理规则

```
┌─────────────────────────────────────────────────────────────────┐
│                      ID 规则                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  类型:                                                          │
│  • String: "1", "abc", "request-123"                            │
│  • Number: 1, 2, 100                                            │
│  • 不能是 null、数组、对象                                       │
│                                                                 │
│  用途:                                                          │
│  • 匹配请求和响应                                                │
│  • 客户端生成请求 ID                                             │
│  • 服务器响应必须使用相同 ID                                      │
│                                                                 │
│  最佳实践:                                                       │
│  • 使用递增数字: 1, 2, 3, ...                                    │
│  • 或使用 UUID: "uuid-550e8400..."                               │
│  • 每个请求唯一 ID                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、MCP 方法定义

### 2.1 方法分类

```
┌─────────────────────────────────────────────────────────────────┐
│                      MCP 方法分类                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  初始化方法:                                                     │
│  ├─ initialize           (请求)  初始化连接                      │
│  └─ initialized          (通知)  确认初始化完成                   │
│                                                                 │
│  工具方法:                                                       │
│  ├─ tools/list           (请求)  获取工具列表                     │
│  └─ tools/call           (请求)  调用工具                        │
│                                                                 │
│  资源方法:                                                       │
│  ├─ resources/list       (请求)  获取资源列表                     │
│  ├─ resources/read       (请求)  读取资源内容                     │
│  ├─ resources/subscribe  (请求)  订阅资源变更                    │
│  ├─ resources/unsubscribe(请求)  取消订阅                        │
│  └─ notifications/resources/updated (通知) 资源变更通知           │
│                                                                 │
│  提示方法:                                                       │
│  ├─ prompts/list         (请求)  获取提示列表                     │
│  └─ prompts/get          (请求)  获取提示内容                     │
│                                                                 │
│  日志方法:                                                       │
│  ├─ logging/setLevel     (请求)  设置日志级别                     │
│  └─ notifications/message(通知)  日志消息                        │
│                                                                 │
│  进度方法:                                                       │
│  └─ notifications/progress(通知)  进度更新                       │
│                                                                 │
│  采样方法（可选）:                                               │
│  └─ sampling/createMessage(请求) 请求 LLM 采样                   │
│                                                                 │
│  根目录方法（可选）:                                             │
│  ├─ roots/list           (请求)  获取根目录列表                   │
│  └─ notifications/roots/list_changed (通知) 根目录变更           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、初始化流程详解

### 3.1 完整握手序列

```
┌─────────────────────────────────────────────────────────────────┐
│                      初始化握手                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                                     Server               │
│     │                                         │                  │
│     │  1. initialize 请求                      │                  │
│     │────────────────────────────────────────→│                  │
│     │                                         │                  │
│     │  请求内容:                               │                  │
│     │ {                                       │                  │
│     │   "jsonrpc": "2.0",                     │                  │
│     │   "id": 1,                              │                  │
│     │   "method": "initialize",               │                  │
│     │   "params": {                           │                  │
│     │     "protocolVersion": "2024-11-05",    │                  │
│     │     "capabilities": {                   │                  │
│     │       "roots": { "listChanged": true }, │                  │
│     │       "sampling": {}                    │                  │
│     │     },                                  │                  │
│     │     "clientInfo": {                     │                  │
│     │       "name": "claude-code",            │                  │
│     │       "version": "1.0.0"                │                  │
│     │     }                                   │                  │
│     │   }                                     │                  │
│     │ }                                       │                  │
│     │                                         │                  │
│     │  2. initialize 响应                     │                  │
│     │←────────────────────────────────────────│                  │
│     │                                         │                  │
│     │  响应内容:                               │                  │
│     │ {                                       │                  │
│     │   "jsonrpc": "2.0",                     │                  │
│     │   "id": 1,                              │                  │
│     │   "result": {                           │                  │
│     │     "protocolVersion": "2024-11-05",    │                  │
│     │     "capabilities": {                   │                  │
│     │       "tools": {},                      │                  │
│     │       "resources": {                    │                  │
│     │         "subscribe": true               │                  │
│     │       },                                │                  │
│     │       "prompts": {},                    │                  │
│     │       "logging": {}                     │                  │
│     │     },                                  │                  │
│     │     "serverInfo": {                     │                  │
│     │       "name": "code-memory",            │                  │
│     │       "version": "1.0.0"                │                  │
│     │     }                                   │                  │
│     │   }                                     │                  │
│     │ }                                       │                  │
│     │                                         │                  │
│     │  3. initialized 通知                    │                  │
│     │────────────────────────────────────────→│                  │
│     │                                         │                  │
│     │ {                                       │                  │
│     │   "jsonrpc": "2.0",                     │                  │
│     │   "method": "notifications/initialized" │                  │
│     │ }                                       │                  │
│     │                                         │                  │
│     │         === 握手完成 ===                │                  │
│     │                                         │                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 版本协商

```
┌─────────────────────────────────────────────────────────────────┐
│                      版本协商                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  协议版本:                                                       │
│  • 当前版本: "2024-11-05"                                        │
│  • 格式: YYYY-MM-DD                                              │
│                                                                 │
│  协商规则:                                                       │
│  1. Client 发送支持的版本                                        │
│  2. Server 返回它支持的版本                                       │
│  3. 双方使用 Server 返回的版本                                   │
│                                                                 │
│  版本不匹配处理:                                                 │
│  • Client 版本高于 Server → 使用 Server 版本                    │
│  • Client 版本低于 Server → 使用 Client 版本                    │
│  • 完全不兼容 → 错误                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 能力协商

```
┌─────────────────────────────────────────────────────────────────┐
│                      能力协商                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client capabilities:                                           │
│  {                                                               │
│    "roots": { "listChanged": true },  // 支持根目录变更通知       │
│    "sampling": {}                     // 支持 LLM 采样           │
│  }                                                               │
│                                                                 │
│  Server capabilities:                                           │
│  {                                                               │
│    "tools": {},                       // 提供工具                │
│    "resources": { "subscribe": true }, // 支持资源订阅           │
│    "prompts": {},                     // 提供提示模板            │
│    "logging": {}                      // 支持日志                │
│  }                                                               │
│                                                                 │
│  协商后双方知道:                                                 │
│  • Server 提供什么功能                                           │
│  • Client 支持什么功能                                           │
│  • 如何正确交互                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、工具方法详解

### 4.1 tools/list

获取服务器提供的工具列表。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}
```

**响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "query",
        "description": "查询代码知识图谱，返回相关的执行流程和代码符号",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "查询内容"
            },
            "repo": {
              "type": "string",
              "description": "仓库名（可选）"
            },
            "limit": {
              "type": "integer",
              "description": "返回数量限制",
              "default": 5
            }
          },
          "required": ["query"]
        }
      },
      {
        "name": "context",
        "description": "获取单个代码符号的完整上下文（调用者、被调用者等）",
        "inputSchema": {
          "type": "object",
          "properties": {
            "name": {
              "type": "string",
              "description": "符号名称"
            },
            "repo": {
              "type": "string",
              "description": "仓库名（可选）"
            },
            "include_content": {
              "type": "boolean",
              "description": "是否包含源代码",
              "default": false
            }
          },
          "required": ["name"]
        }
      }
    ]
  }
}
```

### 4.2 tools/call

调用指定工具。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "query",
    "arguments": {
      "query": "用户登录流程",
      "repo": "my-project",
      "limit": 3
    }
  }
}
```

**成功响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "找到以下相关执行流程:\n\n1. UserLogin Process\n   - 入口: /src/api/auth.ts:login()\n   - 步骤: 验证 → 查询用户 → 生成Token\n\n2. OAuthLogin Process\n   - 入口: /src/api/oauth.ts:handleOAuth()\n   - 步骤: 获取授权 → 验证 → 创建会话"
      }
    ]
  }
}
```

**错误响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "errors": [
        {
          "path": ["query"],
          "message": "必需参数缺失"
        }
      ]
    }
  }
}
```

### 4.3 工具定义 Schema

```
┌─────────────────────────────────────────────────────────────────┐
│                      工具定义 Schema                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tool 定义:                                                      │
│  {                                                               │
│    "name": string,           // 工具名称（唯一标识）              │
│    "description": string,    // 工具描述（让模型理解用途）        │
│    "inputSchema": {          // 参数 Schema (JSON Schema)        │
│      "type": "object",                                           │
│      "properties": {                                             │
│        "参数名": {                                                │
│          "type": "string|number|integer|boolean|array|object",   │
│          "description": "参数描述",                              │
│          "enum": ["可选值1", "可选值2"],  // 可选                 │
│          "default": 默认值               // 可选                 │
│        }                                                         │
│      },                                                          │
│      "required": ["必需参数1", "必需参数2"],                      │
│      "additionalProperties": false      // 不允许额外参数         │
│    }                                                             │
│  }                                                               │
│                                                                 │
│  类型定义:                                                       │
│  • string: 字符串                                                │
│  • number: 数字（浮点）                                          │
│  • integer: 整数                                                 │
│  • boolean: 布尔值                                               │
│  • array: 数组                                                   │
│  • object: 对象                                                  │
│  • null: 空值                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、资源方法详解

### 5.1 resources/list

获取可用资源列表。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/list"
}
```

**响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "resources": [
      {
        "uri": "code-memory://repos",
        "name": "All Indexed Repositories",
        "description": "列出所有已索引的仓库",
        "mimeType": "text/yaml"
      },
      {
        "uri": "code-memory://repo/my-project/context",
        "name": "My Project Context",
        "description": "项目上下文信息",
        "mimeType": "text/markdown"
      },
      {
        "uri": "file:///config/app.json",
        "name": "App Config",
        "description": "应用配置文件",
        "mimeType": "application/json"
      }
    ]
  }
}
```

### 5.2 resources/read

读取资源内容。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "resources/read",
  "params": {
    "uri": "code-memory://repos"
  }
}
```

**响应（文本资源）：**
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "contents": [
      {
        "uri": "code-memory://repos",
        "mimeType": "text/yaml",
        "text": "repos:\n  - name: my-project\n    path: /path/to/project\n    indexed: 2024-01-01"
      }
    ]
  }
}
```

**响应（二进制资源）：**
```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "result": {
    "contents": [
      {
        "uri": "file:///image/logo.png",
        "mimeType": "image/png",
        "blob": "iVBORw0KGgoAAAANSUhEUgAA..."
      }
    ]
  }
}
```

### 5.3 resources/subscribe

订阅资源变更通知（可选功能）。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "resources/subscribe",
  "params": {
    "uri": "file:///config/app.json"
  }
}
```

**响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {}
}
```

**变更通知：**
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///config/app.json"
  }
}
```

### 5.4 资源 URI 设计

```
┌─────────────────────────────────────────────────────────────────┐
│                      资源 URI 设计                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  URI 格式: scheme://path                                        │
│                                                                 │
│  常见 scheme:                                                   │
│  • file:///     - 本地文件                                      │
│  • http://      - HTTP 资源                                     │
│  • https://     - HTTPS 资源                                    │
│  • custom://    - 自定义 scheme                                  │
│                                                                 │
│  MCP 常用 scheme:                                               │
│  • code-memory://   - CodeMemory 资源                           │
│  • github://        - GitHub 资源                               │
│  • db://            - 数据库资源                                 │
│  • config://        - 配置资源                                  │
│                                                                 │
│  URI 编码规则:                                                  │
│  • 使用 UTF-8 编码                                              │
│  • 特殊字符需要 URL 编码                                        │
│  • 不能超过最大长度限制                                         │
│                                                                 │
│  示例:                                                          │
│  • file:///C:/Users/project/src/main.ts                        │
│  • code-memory://repo/my-project/process/UserLogin             │
│  • config://database/connection                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、提示方法详解

### 6.1 prompts/list

获取可用提示模板列表。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "method": "prompts/list"
}
```

**响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "description": "代码审查提示模板",
        "arguments": [
          {
            "name": "language",
            "description": "编程语言",
            "required": true
          },
          {
            "name": "file_path",
            "description": "文件路径",
            "required": true
          },
          {
            "name": "focus",
            "description": "审查重点（可选）",
            "required": false
          }
        ]
      },
      {
        "name": "debug_trace",
        "description": "调试追踪提示模板",
        "arguments": [
          {
            "name": "error_message",
            "description": "错误信息",
            "required": true
          },
          {
            "name": "context",
            "description": "上下文信息（可选）",
            "required": false
          }
        ]
      }
    ]
  }
}
```

### 6.2 prompts/get

获取具体提示内容。

**请求：**
```json
{
  "jsonrpc": "2.0",
  "id": 9,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "language": "TypeScript",
      "file_path": "src/auth.ts",
      "focus": "安全性"
    }
  }
}
```

**响应：**
```json
{
  "jsonrpc": "2.0",
  "id": 9,
  "result": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "请用 TypeScript 的最佳实践审查 src/auth.ts 文件。\n\n重点关注：安全性\n\n审查要点：\n1. 代码风格是否符合规范\n2. 是否存在潜在的安全漏洞\n3. 密码和敏感数据处理\n4. 认证和授权逻辑\n5. 输入验证"
        }
      }
    ]
  }
}
```

---

## 七、通知类型详解

### 7.1 进度通知

```
┌─────────────────────────────────────────────────────────────────┐
│                      进度通知                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  使用场景:                                                      │
│  • 长时间运行的操作                                              │
│  • 大文件处理                                                    │
│  • 复杂查询                                                      │
│                                                                 │
│  通知格式:                                                       │
│  {                                                               │
│    "jsonrpc": "2.0",                                             │
│    "method": "notifications/progress",                           │
│    "params": {                                                   │
│      "progressToken": "token-123",   // 进度标识                  │
│      "progress": 50,                  // 当前进度                 │
│      "total": 100                     // 总量（可选）             │
│    }                                                             │
│  }                                                               │
│                                                                 │
│  progressToken:                                                 │
│  • 由 Client 在请求中提供                                        │
│  • 用于匹配对应的请求                                            │
│                                                                 │
│  请求中添加 progressToken:                                       │
│  {                                                               │
│    "method": "tools/call",                                       │
│    "params": {                                                   │
│      "name": "long_operation",                                   │
│      "arguments": {...},                                         │
│      "_meta": {                                                  │
│        "progressToken": "token-123"                              │
│      }                                                           │
│    }                                                             │
│  }                                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 日志通知

```
┌─────────────────────────────────────────────────────────────────┐
│                      日志通知                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  日志级别:                                                       │
│  • debug: 调试信息                                               │
│  • info: 一般信息                                                │
│  • notice: 重要信息                                              │
│  • warning: 警告                                                 │
│  • error: 错误                                                   │
│  • critical: 严重错误                                            │
│                                                                 │
│  设置日志级别:                                                   │
│  {                                                               │
│    "method": "logging/setLevel",                                 │
│    "params": {                                                   │
│      "level": "info"                                             │
│    }                                                             │
│  }                                                               │
│                                                                 │
│  日志通知:                                                       │
│  {                                                               │
│    "method": "notifications/message",                            │
│    "params": {                                                   │
│      "level": "info",                                            │
│      "data": "工具 query 执行完成"                                │
│    }                                                             │
│  }                                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、采样方法（可选）

### 8.1 sampling/createMessage

Server 可以请求 Client 调用 LLM（让 Server 使用 AI 能力）。

```
┌─────────────────────────────────────────────────────────────────┐
│                      LLM 采样                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Server 请求:                                                   │
│  {                                                               │
│    "method": "sampling/createMessage",                           │
│    "params": {                                                   │
│      "messages": [                                               │
│        {                                                         │
│          "role": "user",                                         │
│          "content": {                                            │
│            "type": "text",                                       │
│            "text": "分析这段代码的结构..."                        │
│          }                                                       │
│        }                                                         │
│      ],                                                          │
│      "modelPreferences": {                                       │
│        "hints": [{ "name": "claude-sonnet-4-6" }]                │
│      },                                                          │
│      "maxTokens": 1000                                           │
│    }                                                             │
│  }                                                               │
│                                                                 │
│  Client 响应:                                                   │
│  {                                                               │
│    "result": {                                                   │
│      "role": "assistant",                                        │
│      "content": {                                                │
│        "type": "text",                                           │
│        "text": "这段代码包含以下结构..."                          │
│      },                                                          │
│      "model": "claude-sonnet-4-6",                               │
│      "stopReason": "endTurn"                                     │
│    }                                                             │
│  }                                                               │
│                                                                 │
│  用途:                                                          │
│  • Server 要AI能力处理数据                                      │
│  • Server 需要生成内容                                           │
│  • 复合操作中需要AI判断                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、安全考虑

### 9.1 输入验证

```
┌─────────────────────────────────────────────────────────────────┐
│                      输入验证                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JSON Schema 验证:                                              │
│  • 所有参数必须符合 Schema 定义                                  │
│  • 必需参数必须存在                                              │
│  • 类型必须匹配                                                  │
│  • 超出范围应报错                                                │
│                                                                 │
│  实现建议:                                                       │
│  def validate_input(schema: dict, input: dict):                 │
│      # 检查必需参数                                              │
│      for required in schema.get("required", []):                │
│          if required not in input:                              │
│              raise ValidationError(f"缺少必需参数: {required}") │
│                                                                 │
│      # 检查参数类型                                              │
│      properties = schema.get("properties", {})                  │
│      for key, value in input.items():                           │
│          if key in properties:                                  │
│              expected_type = properties[key]["type"]            │
│              if not check_type(value, expected_type):           │
│                  raise ValidationError(...)                     │
│                                                                 │
│      # 检查不允许的额外参数                                      │
│      if not schema.get("additionalProperties", True):           │
│          for key in input:                                      │
│              if key not in properties:                          │
│                  raise ValidationError(...)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 权限控制

```
┌─────────────────────────────────────────────────────────────────┐
│                      权限控制                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  工具权限分级:                                                   │
│  ├─ 读取类工具: 通常自动允许                                     │
│  │   • read_file                                               │
│  │   • query                                                   │
│  │   • list_*                                                  │
│  │                                                              │
│  ├─ 写入类工具: 需要确认                                         │
│  │   • write_file                                              │
│  │   • edit_file                                               │
│  │                                                              │
│  ├─ 执行类工具: 需要确认                                         │
│  │   • bash                                                    │
│  │   • execute                                                 │
│  │                                                              │
│  └─ 高危操作: 默认拒绝                                           │
│      • delete_file                                              │
│      • system_command                                           │
│                                                                 │
│  资源访问控制:                                                   │
│  • 只允许访问特定目录                                            │
│  • 禁止访问敏感文件                                              │
│  • URI 白名单                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 9.3 错误处理最佳实践

```
┌─────────────────────────────────────────────────────────────────┐
│                      错误处理                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  错误分类:                                                       │
│  ├─ 协议错误: 使用标准 JSON-RPC 错误码                           │
│  │   • 解析错误、无效请求等                                      │
│  │                                                              │
│  ├─ 业务错误: 使用自定义错误码                                   │
│  │   • 工具不存在、参数无效等                                    │
│  │                                                              │
│  └─ 系统错误: 使用 -32000 到 -32099                             │
│      • 内部错误、资源不可用等                                    │
│                                                                 │
│  错误信息要求:                                                   │
│  • 描述清楚错误原因                                              │
│  • 提供修复建议                                                  │
│  • 不要暴露敏感信息                                              │
│                                                                 │
│  示例:                                                          │
│  {                                                               │
│    "error": {                                                    │
│      "code": -32602,                                             │
│      "message": "Invalid params: 'repo' parameter is required", │
│      "data": {                                                   │
│        "parameter": "repo",                                     │
│        "suggestion": "请提供 repo 参数指定仓库名"                 │
│      }                                                           │
│    }                                                             │
│  }                                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十、协议版本历史

```
┌─────────────────────────────────────────────────────────────────┐
│                      协议版本                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2024-11-05 (当前版本):                                         │
│  ├─ 稳定的核心方法                                              │
│  ├─ 工具、资源、提示完整支持                                     │
│  └─ 采样和根目录可选支持                                         │
│                                                                 │
│  未来版本可能增加:                                              │
│  ├─ 更多通知类型                                                │
│  ├─ 批量操作                                                    │
│  ├─ 流式工具结果                                                │
│  └─ 更丰富的类型定义                                            │
│                                                                 │
│  版本兼容策略:                                                  │
│  • Client 应支持多个版本                                        │
│  • Server 应声明支持的最低版本                                  │
│  • 协商后使用双方都支持的版本                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十一、实现检查清单

### Server 实现检查

```
┌─────────────────────────────────────────────────────────────────┐
│                      Server 检查清单                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  必需实现:                                                       │
│  ☐ initialize 方法                                              │
│  ☐ 处理 initialized 通知                                        │
│  ☐ 返回正确的 capabilities                                      │
│  ☐ JSON-RPC 2.0 格式响应                                        │
│  ☐ 错误码和错误消息                                              │
│                                                                 │
│  工具实现:                                                       │
│  ☐ tools/list 方法                                              │
│  ☐ tools/call 方法                                              │
│  ☐ 完整的 inputSchema                                           │
│  ☐ 参数验证                                                      │
│  ☐ 错误处理                                                      │
│                                                                 │
│  资源实现（可选）:                                               │
│  ☐ resources/list 方法                                          │
│  ☐ resources/read 方法                                          │
│  ☐ URI 验证                                                      │
│  ☐ mimeType 正确设置                                            │
│                                                                 │
│  提示实现（可选）:                                               │
│  ☐ prompts/list 方法                                            │
│  ☐ prompts/get 方法                                             │
│  ☐ arguments 验证                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Client 实现检查

```
┌─────────────────────────────────────────────────────────────────┐
│                      Client 检查清单                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  连接管理:                                                       │
│  ☐ 启动 Server 进程                                             │
│  ☐ 发送 initialize 请求                                         │
│  ☐ 发送 initialized 通知                                        │
│  ☐ 处理版本协商                                                  │
│  ☐ 记录 capabilities                                            │
│                                                                 │
│  工具调用:                                                       │
│  ☐ 获取工具列表                                                  │
│  ☐ 构建工具定义给模型                                            │
│  ☐ 发送 tools/call 请求                                         │
│  ☐ 解析响应                                                      │
│  ☐ 权限检查                                                      │
│                                                                 │
│  资源访问:                                                       │
│  ☐ 获取资源列表                                                  │
│  ☐ 读取资源内容                                                  │
│  ☐ 处理二进制资源                                                │
│                                                                 │
│  错误处理:                                                       │
│  ☐ 处理 JSON-RPC 错误                                           │
│  ☐ 处理超时                                                      │
│  ☐ 处理 Server 断开                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 总结

MCP 协议建立在 JSON-RPC 2.0 之上，提供了一套完整的方法定义：

| 类别 | 核心方法 |
|------|----------|
| 初始化 | initialize, initialized |
| 工具 | tools/list, tools/call |
| 资源 | resources/list, resources/read |
| 提示 | prompts/list, prompts/get |
| 通知 | progress, message, updated |

理解这些协议细节后，你可以：
- 正确实现 MCP Server
- 正确实现 MCP Client
- 调试协议问题
- 扩展协议功能