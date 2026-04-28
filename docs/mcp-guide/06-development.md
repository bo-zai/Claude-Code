# MCP 开发指南

## 开发环境准备

### 方式一：Python SDK

```bash
# 安装 MCP Python SDK
pip install mcp
```

### 方式二：TypeScript SDK

```bash
# 安装 MCP TypeScript SDK
npm install @modelcontextprotocol/sdk
```

---

## 最简示例（Python）

### 完整服务器代码

```python
# server.py
from mcp.server import Server
from mcp.types import Tool, TextContent

# 创建服务器实例
app = Server("my-first-server")

# 1. 声明可用工具
@app.list_tools()
async def list_tools():
    return [
        Tool(
            name="hello",
            description="打招呼工具",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {
                        "type": "string",
                        "description": "要打招呼的名字"
                    }
                },
                "required": ["name"]
            }
        ),
        Tool(
            name="add",
            description="两数相加",
            inputSchema={
                "type": "object",
                "properties": {
                    "a": {"type": "number", "description": "第一个数"},
                    "b": {"type": "number", "description": "第二个数"}
                },
                "required": ["a", "b"]
            }
        )
    ]

# 2. 实现工具逻辑
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "hello":
        return [TextContent(
            type="text",
            text=f"你好，{arguments['name']}！"
        )]

    elif name == "add":
        result = arguments['a'] + arguments['b']
        return [TextContent(
            type="text",
            text=f"{arguments['a']} + {arguments['b']} = {result}"
        )]

    else:
        raise ValueError(f"未知工具: {name}")

# 3. 运行服务器
if __name__ == "__main__":
    import asyncio
    asyncio.run(app.run(transport="stdio"))
```

### 运行服务器

```bash
python server.py
```

---

## 最简示例（TypeScript）

### 完整服务器代码

```typescript
// server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

// 创建服务器实例
const server = new Server({
  name: "my-first-server",
  version: "1.0.0"
}, {
  capabilities: {
    tools: {}
  }
});

// 1. 声明可用工具
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "hello",
        description: "打招呼工具",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "要打招呼的名字"
            }
          },
          required: ["name"]
        }
      },
      {
        name: "add",
        description: "两数相加",
        inputSchema: {
          type: "object",
          properties: {
            a: { type: "number", description: "第一个数" },
            b: { type: "number", description: "第二个数" }
          },
          required: ["a", "b"]
        }
      }
    ]
  };
});

// 2. 实现工具逻辑
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "hello") {
    return {
      content: [{
        type: "text",
        text: `你好，${args.name}！`
      }]
    };
  }

  if (name === "add") {
    const result = args.a + args.b;
    return {
      content: [{
        type: "text",
        text: `${args.a} + ${args.b} = ${result}`
      }]
    };
  }

  throw new Error(`未知工具: ${name}`);
});

// 3. 运行服务器
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 配置给 Claude Code 使用

### 编辑配置文件

编辑 `~/.claude/settings.json`（全局）或项目目录下的 `.claude/settings.json`（项目）：

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["path/to/server.py"]
    }
  }
}
```

### 配置说明

| 字段 | 说明 |
|------|------|
| command | 启动命令（python、node、npx 等） |
| args | 命令行参数 |
| env | 环境变量（可选） |
| cwd | 工作目录（可选） |

### 带环境变量的配置

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["server.py"],
      "env": {
        "API_KEY": "your-api-key",
        "DEBUG": "true"
      },
      "cwd": "/path/to/server/directory"
    }
  }
}
```

---

## 添加资源支持

### Python 示例

```python
from mcp.types import Resource, ResourceContents

# 声明可用资源
@app.list_resources()
async def list_resources():
    return [
        Resource(
            uri="config://app.json",
            name="应用配置",
            description="应用程序的配置文件",
            mimeType="application/json"
        ),
        Resource(
            uri="readme://project",
            name="项目说明",
            description="项目的 README 文档",
            mimeType="text/markdown"
        )
    ]

# 实现资源读取
@app.read_resource()
async def read_resource(uri: str):
    if uri == "config://app.json":
        config = load_config()  # 加载配置
        return ResourceContents(
            uri=uri,
            mimeType="application/json",
            text=json.dumps(config)
        )

    elif uri == "readme://project":
        with open("README.md") as f:
            return ResourceContents(
                uri=uri,
                mimeType="text/markdown",
                text=f.read()
            )

    raise ValueError(f"未知资源: {uri}")
```

---

## 添加提示模板支持

### Python 示例

```python
from mcp.types import Prompt, PromptMessage

# 声明可用提示
@app.list_prompts()
async def list_prompts():
    return [
        Prompt(
            name="code_review",
            description="代码审查提示模板",
            arguments=[
                {
                    "name": "language",
                    "description": "编程语言",
                    "required": True
                },
                {
                    "name": "file_path",
                    "description": "文件路径",
                    "required": True
                }
            ]
        )
    ]

# 实现提示内容
@app.get_prompt()
async def get_prompt(name: str, arguments: dict):
    if name == "code_review":
        language = arguments["language"]
        file_path = arguments["file_path"]

        return [
            PromptMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""请用 {language} 的最佳实践审查 {file_path} 文件。

审查要点：
1. 代码风格是否符合规范
2. 是否存在潜在的 bug
3. 性能是否有优化空间
4. 安全性考虑"""
                )
            )
        ]

    raise ValueError(f"未知提示: {name}")
```

---

## 高级特性

### 1. 进度通知

长时间操作可以发送进度更新：

```python
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "long_operation":
        # 创建进度通知
        progress_token = arguments.get("_meta", {}).get("progressToken")

        for i in range(10):
            # 执行部分工作
            await do_part_of_work(i)

            # 发送进度通知
            if progress_token:
                await app.send_progress(
                    progress_token=progress_token,
                    progress=i + 1,
                    total=10
                )

        return [TextContent(type="text", text="操作完成")]
```

### 2. 日志记录

```python
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    # 发送日志消息
    await app.send_log("info", f"调用工具: {name}")

    try:
        result = await do_something(arguments)
        await app.send_log("debug", f"工具执行成功")
        return result
    except Exception as e:
        await app.send_log("error", f"工具执行失败: {e}")
        raise
```

### 3. 资源变更通知

当资源内容变化时通知客户端：

```python
# 订阅资源
subscribed_resources = set()

@app.list_resources()
async def list_resources():
    ...

@app.read_resource()
async def read_resource(uri: str):
    ...

# 当资源变化时
async def on_resource_changed(uri: str):
    if uri in subscribed_resources:
        await app.send_resource_updated(uri)
```

---

## 调试技巧

### 1. 查看服务器日志

```bash
# 启动服务器时输出日志到文件
python server.py 2> server.log
```

### 2. 使用 MCP Inspector

MCP Inspector 是一个调试工具，可以可视化查看 MCP 服务器的工具和资源：

```bash
# 安装
npm install -g @modelcontextprotocol/inspector

# 运行
mcp-inspector python server.py
```

### 3. 测试工具调用

```bash
# 使用命令行测试
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python server.py
```

---

## 最佳实践

### 1. 工具设计原则

| 原则 | 说明 |
|------|------|
| 单一职责 | 每个工具只做一件事 |
| 清晰描述 | 描述要让模型理解工具的用途 |
| 完整 Schema | 参数 Schema 要完整，包含类型和描述 |
| 错误处理 | 提供有意义的错误信息 |

### 2. 安全考虑

```python
# 验证输入
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    # 1. 验证必需参数
    if "file_path" not in arguments:
        raise ValueError("缺少必需参数: file_path")

    # 2. 验证参数格式
    file_path = arguments["file_path"]
    if not file_path.startswith("/safe/directory"):
        raise ValueError("不允许访问此目录")

    # 3. 限制操作范围
    # ...

    # 4. 执行操作
    ...
```

### 3. 性能优化

```python
# 批量操作
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "batch_query":
        # 一次性查询多个，而不是循环查询
        results = await batch_fetch(arguments["ids"])
        return [TextContent(type="text", text=str(results))]

# 异步操作
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "parallel_query":
        # 并行执行多个查询
        results = await asyncio.gather(
            query1(arguments),
            query2(arguments),
            query3(arguments)
        )
        ...
```

---

## 常见问题

### Q: 服务器启动失败

检查命令路径是否正确：
```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

### Q: 工具调用没有反应

确保服务器正确实现了 `list_tools` 和 `call_tool` 处理器。

### Q: 中文乱码

确保 Python 文件使用 UTF-8 编码，并在文件开头添加：
```python
# -*- coding: utf-8 -*-
```

---

## 参考资源

- [MCP 官方规范](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP 示例服务器](https://github.com/modelcontextprotocol/servers)