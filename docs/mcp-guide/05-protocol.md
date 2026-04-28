# MCP 通信协议

MCP 使用 JSON-RPC 2.0 作为通信协议，支持 Stdio 和 HTTP/SSE 两种传输方式。

## 协议概述

```
┌─────────────────────────────────────────────────────────────────┐
│                      MCP 通信层                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  传输方式:                                                       │
│  ├─ Stdio (标准输入输出) ← 最常用                                │
│  └─ HTTP/SSE (Server-Sent Events) ← 远程部署                     │
│                                                                 │
│  消息格式: JSON-RPC 2.0                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## JSON-RPC 基础

### 消息类型

| 类型 | 说明 |
|------|------|
| Request | 请求，需要响应 |
| Response | 响应，对应某个请求 |
| Notification | 通知，不需要响应 |

### 请求格式

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "北京"
    }
  }
}
```

### 成功响应格式

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "北京：晴，18°C"
      }
    ]
  }
}
```

### 错误响应格式

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params: missing required 'city'"
  }
}
```

### 通知格式

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progress": 50,
    "total": 100
  }
}
```

---

## 传输方式

### 方式一：Stdio（标准输入输出）

最常用的通信方式，MCP Server 作为子进程运行。

```
┌─────────────┐   stdin    ┌─────────────┐
│  AI 客户端   │ ─────────→ │ MCP 服务器   │
│   (Agent)   │ ←───────── │ (子进程)     │
└─────────────┘   stdout   └─────────────┘
```

**特点**：
- 简单可靠
- 本地运行
- 进程隔离
- 零网络开销

**配置示例**：
```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["server.py"]
    }
  }
}
```

### 方式二：HTTP/SSE

用于远程部署场景。

```
┌─────────────┐   HTTP    ┌─────────────┐
│  AI 客户端   │ ←───────→ │ MCP 服务器   │
│   (Agent)   │   SSE     │ (远程服务)   │
└─────────────┘           └─────────────┘
```

**特点**：
- 支持远程部署
- 可以多个客户端共享
- 需要处理认证

**配置示例**：
```json
{
  "mcpServers": {
    "remote-server": {
      "url": "https://api.example.com/mcp"
    }
  }
}
```

---

## 完整握手流程

### 初始化序列

```
Agent                                    MCP Server
  │                                           │
  │                                           │
  │  1. 发送初始化请求                         │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 1,                               │
  │    "method": "initialize",                │
  │    "params": {                            │
  │      "protocolVersion": "2024-11-05",     │
  │      "clientInfo": {                      │
  │        "name": "claude-code",             │
  │        "version": "1.0.0"                 │
  │      },                                   │
  │      "capabilities": {}                   │
  │    }                                      │
  │  }                                        │
  │                                           │
  │  2. 接收初始化响应                         │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 1,                               │
  │    "result": {                            │
  │      "protocolVersion": "2024-11-05",     │
  │      "serverInfo": {                      │
  │        "name": "my-mcp-server",           │
  │        "version": "1.0.0"                 │
  │      },                                   │
  │      "capabilities": {                    │
  │        "tools": {},                       │
  │        "resources": {}                    │
  │      }                                   │
  │    }                                      │
  │  }                                        │
  │                                           │
  │  3. 发送初始化完成通知                     │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "method": "notifications/initialized" │
  │  }                                        │
  │                                           │
  │         === 握手完成，可以开始正常通信 ===  │
  │                                           │
```

### 能力发现

```
Agent                                    MCP Server
  │                                           │
  │  查询可用工具                               │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 2,                               │
  │    "method": "tools/list"                 │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 2,                               │
  │    "result": {                            │
  │      "tools": [                           │
  │        {                                  │
  │          "name": "query",                 │
  │          "description": "查询知识图谱",    │
  │          "inputSchema": {...}             │
  │        },                                 │
  │        {                                  │
  │          "name": "read",                  │
  │          "description": "读取文件",        │
  │          "inputSchema": {...}             │
  │        }                                  │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
  │  查询可用资源                               │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 3,                               │
  │    "method": "resources/list"             │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 3,                               │
  │    "result": {                            │
  │      "resources": [                       │
  │        {                                  │
  │          "uri": "file:///config.json",    │
  │          "name": "配置文件",               │
  │          "mimeType": "application/json"   │
  │        }                                  │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
```

---

## 工具调用详解

### 调用工具

```
Agent                                    MCP Server
  │                                           │
  │  调用工具                                  │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 4,                               │
  │    "method": "tools/call",                │
  │    "params": {                            │
  │      "name": "query",                     │
  │      "arguments": {                       │
  │        "query": "代码仓库结构",            │
  │        "repo": "my-project"               │
  │      }                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 4,                               │
  │    "result": {                            │
  │      "content": [                         │
  │        {                                  │
  │          "type": "text",                  │
  │          "text": "仓库包含以下模块: ..."   │
  │        }                                  │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
```

### 工具返回内容类型

```json
// 文本内容
{
  "type": "text",
  "text": "查询结果..."
}

// 图片内容
{
  "type": "image",
  "data": "base64编码的图片数据",
  "mimeType": "image/png"
}

// 资源引用
{
  "type": "resource",
  "resource": {
    "uri": "file:///result.json",
    "mimeType": "application/json"
  }
}
```

---

## 资源访问详解

### 读取资源

```
Agent                                    MCP Server
  │                                           │
  │  读取资源                                  │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 5,                               │
  │    "method": "resources/read",            │
  │    "params": {                            │
  │      "uri": "config://app.json"           │
  │    }                                      │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "jsonrpc": "2.0",                      │
  │    "id": 5,                               │
  │    "result": {                            │
  │      "contents": [                        │
  │        {                                  │
  │          "uri": "config://app.json",      │
  │          "mimeType": "application/json",  │
  │          "text": "{\"port\": 3306}"       │
  │        }                                  │
  │        // 或对于二进制文件:                 │
  │        // {                               │
  │        //   "uri": "...",                  │
  │        //   "mimeType": "image/png",       │
  │        //   "blob": "base64..."            │
  │        // }                                │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
```

### 资源订阅（可选功能）

某些 MCP Server 支持资源变更通知：

```
Agent                                    MCP Server
  │                                           │
  │  订阅资源变更                              │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "method": "resources/subscribe",       │
  │    "params": {                            │
  │      "uri": "config://app.json"           │
  │    }                                      │
  │  }                                        │
  │                                           │
  │         ... 资源变更时 ...                  │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "method": "notifications/resources/   │
  │               updated",                   │
  │    "params": {                            │
  │      "uri": "config://app.json"           │
  │    }                                      │
  │  }                                        │
  │                                           │
```

---

## 提示模板使用

```
Agent                                    MCP Server
  │                                           │
  │  列出可用提示                              │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "method": "prompts/list"               │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "result": {                            │
  │      "prompts": [                         │
  │        {                                  │
  │          "name": "code_review",           │
  │          "description": "代码审查",        │
  │          "arguments": [...]               │
  │        }                                  │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
  │  获取提示内容                              │
  │─────────────────────────────────────────→│
  │  {                                        │
  │    "method": "prompts/get",               │
  │    "params": {                            │
  │      "name": "code_review",               │
  │      "arguments": {                       │
  │        "language": "Python",              │
  │        "file_path": "src/auth.py"         │
  │      }                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
  │←─────────────────────────────────────────│
  │  {                                        │
  │    "result": {                            │
  │      "messages": [                        │
  │        {                                  │
  │          "role": "user",                  │
  │          "content": {                     │
  │            "type": "text",                │
  │            "text": "请用 Python 最佳..."  │
  │          }                                │
  │        }                                  │
  │      ]                                    │
  │    }                                      │
  │  }                                        │
  │                                           │
```

---

## 错误码定义

| 错误码 | 说明 |
|--------|------|
| -32700 | Parse error（解析错误） |
| -32600 | Invalid Request（无效请求） |
| -32601 | Method not found（方法未找到） |
| -32602 | Invalid params（无效参数） |
| -32603 | Internal error（内部错误） |
| -32000 to -32099 | Server error（服务器错误） |

### 错误响应示例

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "field": "city",
      "reason": "required field missing"
    }
  }
}
```

---

## 下一步

- 了解如何开发 MCP 服务器，请阅读 [开发指南](./06-development.md)