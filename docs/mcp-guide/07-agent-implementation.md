# Agent 内部实现

Agent 是 MCP 系统的核心协调者，本文深入剖析其内部架构和实现细节。

## Agent 架构总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Agent 内部架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐      │
│   │   用户接口层     │ ──→ │   会话管理层     │ ──→ │   Prompt 构建层  │      │
│   │                 │     │                 │     │                 │      │
│   │  - 接收输入     │     │  - 上下文存储   │     │  - 模板渲染     │      │
│   │  - 展示输出     │     │  - 消息历史     │     │  - 工具注入     │      │
│   │  - 权限审批     │     │  - 状态追踪     │     │  - 资源嵌入     │      │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘      │
│           │                       │                       │                │
│           └───────────────────────┼───────────────────────┘                │
│                                   ↓                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        核心循环控制器                                  │  │
│   │                                                                      │  │
│   │   while (对话未结束) {                                                │  │
│   │       1. 构建当前 Prompt                                             │  │
│   │       2. 调用模型                                                    │  │
│   │       3. 解析响应                                                    │  │
│   │       4. 处理工具调用                                                 │  │
│   │       5. 更新上下文                                                  │  │
│   │   }                                                                  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                   │                                         │
│           ┌───────────────────────┼───────────────────────┐                │
│           ↓                       ↓                       ↓                │
│   ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐      │
│   │   模型通信层     │     │   MCP 通信层     │     │   输出处理层     │      │
│   │                 │     │                 │     │                 │      │
│   │  - API 调用     │     │  - Server 管理  │     │  - 文本渲染     │      │
│   │  - 流式响应     │     │  - 工具调用     │     │  - 工具结果     │      │
│   │  - 错误处理     │     │  - 结果解析     │     │  - 格式化输出   │      │
│   └─────────────────┘     └─────────────────┘     └─────────────────┘      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 一、用户接口层

### 职责

用户接口层负责与用户的直接交互，包括：
- 接收用户输入（文本、文件、图片等）
- 展示 AI 回复
- 处理权限审批请求
- 显示工具调用状态

### 输入处理

```
用户输入 → 输入解析 → 类型识别 → 上下文构建

┌─────────────────────────────────────────────────────┐
│                    输入类型                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  文本输入:                                          │
│    "帮我分析这个代码仓库"                            │
│    → 直接作为用户消息                               │
│                                                     │
│  文件输入:                                          │
│    @file: src/main.py                              │
│    → 读取文件内容，作为上下文                       │
│                                                     │
│  图片输入:                                          │
│    [图片附件]                                       │
│    → 转换为 base64，作为视觉内容                    │
│                                                     │
│  斜杠命令:                                          │
│    /skill: code-review                             │
│    → 调用对应技能                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 输出展示

```
┌─────────────────────────────────────────────────────┐
│                    输出类型                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  文本输出:                                          │
│    AI 生成的文字回复                                │
│    → 直接展示                                       │
│                                                     │
│  工具调用状态:                                       │
│    "正在调用 query 工具..."                          │
│    → 显示进度指示器                                 │
│                                                     │
│  工具结果:                                          │
│    MCP 返回的数据                                   │
│    → 格式化展示（代码块、表格等）                    │
│                                                     │
│  权限请求:                                          │
│    "工具 read_file 需要您的授权"                    │
│    → 弹出审批对话框                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 权限审批实现

```python
class PermissionManager:
    def __init__(self, config):
        self.config = config  # 权限配置

    def check_permission(self, tool_name: str, arguments: dict) -> str:
        """
        检查工具调用权限

        返回值:
        - "allow": 直接执行
        - "ask": 需要用户确认
        - "deny": 拒绝执行
        """
        # 检查白名单
        if tool_name in self.config.allow:
            return "allow"

        # 检查黑名单
        if tool_name in self.config.deny:
            return "deny"

        # 检查需要确认的列表
        if tool_name in self.config.ask:
            return "ask"

        # 默认行为
        return self.config.default_policy

    async def request_user_approval(self, tool_name: str, arguments: dict):
        """
        请求用户审批
        """
        # 构建审批请求
        request = PermissionRequest(
            tool=tool_name,
            arguments=arguments,
            options=["允许本次", "总是允许", "拒绝"]
        )

        # 显示给用户并等待响应
        response = await self.ui.show_permission_dialog(request)

        # 处理响应
        if response == "总是允许":
            self.config.allow.add(tool_name)

        return response in ["允许本次", "总是允许"]
```

---

## 二、会话管理层

### 职责

会话管理层负责维护对话状态：
- 存储消息历史
- 管理上下文窗口
- 追踪当前状态

### 上下文结构

```python
class ConversationContext:
    def __init__(self):
        self.messages = []       # 消息历史
        self.tools = {}          # 可用工具定义
        self.resources = {}      # 可用资源定义
        self.active_mcp_servers = []  # 已连接的 MCP Server
        self.state = {}          # 当前状态

    def add_message(self, role: str, content: Any):
        """添加消息到历史"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": time.now()
        })

    def add_tool_result(self, tool_call_id: str, result: Any):
        """添加工具调用结果"""
        self.messages.append({
            "role": "tool",
            "tool_call_id": tool_call_id,
            "content": result
        })

    def get_context_window(self, max_tokens: int):
        """
        获取适合当前窗口大小的上下文

        当消息历史超过窗口限制时，需要裁剪：
        - 保留最近的工具调用和结果
        - 压缩或移除旧消息
        """
        # 计算当前消息总 token 数
        total_tokens = self.calculate_tokens()

        if total_tokens <= max_tokens:
            return self.messages

        # 需要裁剪
        return self.trim_messages(max_tokens)
```

### 消息历史结构

```
┌─────────────────────────────────────────────────────┐
│                    消息历史                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  [                                                    │
│    {                                                  │
│      "role": "user",                                  │
│      "content": "帮我分析代码仓库结构"                 │
│    },                                                 │
│                                                       │
│    {                                                  │
│      "role": "assistant",                             │
│      "content": null,                                 │
│      "tool_calls": [                                  │
│        {                                              │
│          "id": "call_001",                            │
│          "type": "function",                          │
│          "function": {                                │
│            "name": "query",                           │
│            "arguments": "{...}"                       │
│          }                                            │
│        }                                              │
│      ]                                                │
│    },                                                 │
│                                                       │
│    {                                                  │
│      "role": "tool",                                  │
│      "tool_call_id": "call_001",                      │
│      "content": "仓库包含以下模块..."                  │
│    },                                                 │
│                                                       │
│    {                                                  │
│      "role": "assistant",                             │
│      "content": "这个代码仓库的结构如下..."            │
│    }                                                  │
│  ]                                                    │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### 上下文压缩策略

```
当上下文超过限制时的处理策略:

┌─────────────────────────────────────────────────────┐
│                    压缩策略                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  1. 保留关键部分                                     │
│     - 用户最近的问题                                 │
│     - 最近一轮的工具调用和结果                        │
│     - AI 的最终回答                                  │
│                                                     │
│  2. 移除或压缩                                       │
│     - 较早的对话内容                                 │
│     - 长的工具输出结果                               │
│                                                     │
│  3. 生成摘要                                         │
│     - 对早期对话生成摘要                             │
│     - 用摘要替换原始消息                             │
│                                                     │
│  示例:                                               │
│  [摘要] 用户之前问了关于项目配置的问题，              │
│         AI 分析了 config.json 并提供了建议。          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 三、Prompt 构建层

### 职责

Prompt 构建层负责将各部分信息组合成完整的请求，发送给模型。

### Prompt 结构

```
┌─────────────────────────────────────────────────────────────────┐
│                      完整 Prompt 结构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      系统消息                               │ │
│  │                                                            │ │
│  │  - Agent 角色定义                                          │ │
│  │  - 工作目录信息                                            │ │
│  │  - 项目上下文（来自 CLAUDE.md）                            │ │
│  │  - 安全指引                                                │ │
│  │  - 输出格式要求                                            │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      工具定义                               │ │
│  │                                                            │ │
│  │  tools: [                                                  │ │
│  │    {                                                       │ │
│  │      "name": "query",                                      │ │
│  │      "description": "查询代码知识图谱...",                 │ │
│  │      "input_schema": {                                     │ │
│  │        "type": "object",                                   │ │
│  │        "properties": {                                     │ │
│  │          "query": {"type": "string", "description": "..."} │ │
│  │        },                                                  │ │
│  │        "required": ["query"]                               │ │
│  │      }                                                     │ │
│  │    },                                                      │ │
│  │    ...更多工具...                                          │ │
│  │  ]                                                         │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      消息历史                               │ │
│  │                                                            │ │
│  │  messages: [                                               │ │
│  │    {"role": "user", "content": "用户消息1"},              │ │
│  │    {"role": "assistant", "content": "AI回复1"},           │ │
│  │    {"role": "user", "content": "工具调用请求"},            │ │
│  │    {"role": "tool", "content": "工具结果"},               │ │
│  │    ...                                                     │ │
│  │  ]                                                         │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                      当前用户输入                           │ │
│  │                                                            │ │
│  │  {"role": "user", "content": "帮我分析代码仓库结构"}       │ │
│  │                                                            │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Prompt 构建实现

```python
class PromptBuilder:
    def __init__(self, system_template, tool_manager, context_manager):
        self.system_template = system_template
        self.tool_manager = tool_manager
        self.context_manager = context_manager

    async def build_prompt(self, user_input: str) -> dict:
        """
        构建完整的请求 Prompt
        """
        # 1. 构建系统消息
        system_message = await self.build_system_message()

        # 2. 获取可用工具定义
        tools = await self.tool_manager.get_available_tools()

        # 3. 获取消息历史（在窗口限制内）
        messages = self.context_manager.get_context_window(
            max_tokens=self.config.max_context_tokens
        )

        # 4. 添加当前用户输入
        messages.append({
            "role": "user",
            "content": user_input
        })

        # 5. 组装完整请求
        return {
            "model": self.config.model,
            "system": system_message,
            "tools": tools,
            "messages": messages,
            "max_tokens": self.config.max_output_tokens
        }

    async def build_system_message(self) -> str:
        """
        构建系统消息
        """
        # 渲染系统模板
        template_vars = {
            "working_directory": self.context_manager.get_working_directory(),
            "project_context": await self.load_project_context(),
            "current_date": datetime.now().strftime("%Y/%m/%d"),
            "user_info": self.context_manager.get_user_info()
        }

        return self.system_template.render(template_vars)

    async def load_project_context(self) -> str:
        """
        加载项目上下文（CLAUDE.md）
        """
        # 尝试读取项目目录下的 CLAUDE.md
        claude_md_path = self.context_manager.get_working_directory() + "/CLAUDE.md"

        if os.path.exists(claude_md_path):
            return await read_file(claude_md_path)

        # 尝试读取用户全局 CLAUDE.md
        global_claude_md = "~/.claude/CLAUDE.md"
        if os.path.exists(global_claude_md):
            return await read_file(global_claude_md)

        return ""
```

### 工具注入策略

```
┌─────────────────────────────────────────────────────┐
│                    工具注入                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  每次请求都需要注入工具定义，但不是全部工具：         │
│                                                     │
│  1. 基础工具（始终注入）                             │
│     - read_file                                     │
│     - write_file                                    │
│     - bash                                          │
│                                                     │
│  2. MCP 工具（从已连接的 Server 获取）               │
│     - 查询每个 MCP Server 的工具列表                │
│     - 合并所有工具定义                              │
│                                                     │
│  3. 动态工具（根据当前上下文）                       │
│     - 如果用户提到"PR"，注入 GitHub 工具            │
│     - 如果用户提到"代码"，注入 code-memory 工具     │
│                                                     │
│  4. 工具数量限制                                     │
│     - 工具定义占用 token                            │
│     - 需要控制在合理范围内                          │
│     - 可以分批加载或智能选择                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 四、核心循环控制器

### 职责

核心循环控制器是 Agent 的"心脏"，负责协调整个对话流程。

### 主循环实现

```python
class ConversationLoop:
    def __init__(self, model_client, tool_executor, permission_manager, context_manager):
        self.model_client = model_client
        self.tool_executor = tool_executor
        self.permission_manager = permission_manager
        self.context_manager = context_manager

    async def run(self, user_input: str) -> str:
        """
        主对话循环
        """
        # 添加用户输入到上下文
        self.context_manager.add_message("user", user_input)

        # 循环直到生成最终回答
        while True:
            # 1. 构建当前 Prompt
            prompt = await self.build_current_prompt()

            # 2. 调用模型
            response = await self.model_client.generate(prompt)

            # 3. 解析响应
            parsed = self.parse_response(response)

            # 4. 处理响应
            if parsed.has_tool_calls:
                # 有工具调用，执行并继续循环
                await self.handle_tool_calls(parsed.tool_calls)
                continue

            elif parsed.is_final_answer:
                # 最终回答，结束循环
                self.context_manager.add_message("assistant", parsed.content)
                return parsed.content

            else:
                # 其他情况（如思考过程），继续
                self.context_manager.add_message("assistant", parsed.content)
                continue

    async def handle_tool_calls(self, tool_calls: list):
        """
        处理工具调用
        """
        for tool_call in tool_calls:
            # 检查权限
            permission = self.permission_manager.check_permission(
                tool_call.function.name,
                tool_call.function.arguments
            )

            if permission == "deny":
                # 拒绝执行
                self.context_manager.add_tool_result(
                    tool_call.id,
                    f"工具 {tool_call.function.name} 被拒绝执行"
                )
                continue

            if permission == "ask":
                # 需要用户确认
                approved = await self.permission_manager.request_user_approval(
                    tool_call.function.name,
                    tool_call.function.arguments
                )
                if not approved:
                    self.context_manager.add_tool_result(
                        tool_call.id,
                        f"工具 {tool_call.function.name} 被用户拒绝"
                    )
                    continue

            # 执行工具调用
            result = await self.tool_executor.execute(
                tool_call.function.name,
                tool_call.function.arguments
            )

            # 添加结果到上下文
            self.context_manager.add_tool_result(tool_call.id, result)

        # 记录 assistant 的工具调用请求
        self.context_manager.add_message("assistant", {
            "content": None,
            "tool_calls": tool_calls
        })
```

### 循环状态图

```
┌─────────────────────────────────────────────────────────────────┐
│                      主循环状态图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        ┌───────────┐                            │
│                        │   开始    │                            │
│                        └───────────┘                            │
│                             │                                   │
│                             ↓                                   │
│                        ┌───────────┐                            │
│                        │ 构建Prompt│                            │
│                        └───────────┘                            │
│                             │                                   │
│                             ↓                                   │
│                        ┌───────────┐                            │
│                        │ 调用模型  │                            │
│                        └───────────┘                            │
│                             │                                   │
│                             ↓                                   │
│                        ┌───────────┐                            │
│                        │ 解析响应  │                            │
│                        └───────────┘                            │
│                             │                                   │
│               ┌─────────────┼─────────────┐                    │
│               │             │             │                    │
│               ↓             ↓             ↓                    │
│          ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│          │工具调用 │  │最终回答 │  │思考过程 │                 │
│          └─────────┘  └─────────┘  └─────────┘                 │
│               │             │             │                    │
│               ↓             │             │                    │
│          ┌─────────┐        │             │                    │
│          │权限检查 │        │             │                    │
│          └─────────┘        │             │                    │
│               │             │             │                    │
│      ┌────────┼────────┐    │             │                    │
│      │        │        │    │             │                    │
│      ↓        ↓        ↓    │             │                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ │             │                    │
│  │执行 │ │确认 │ │拒绝 │ │             │                    │
│  └──────┘ └──────┘ └──────┘ │             │                    │
│      │        │        │    │             │                    │
│      ↓        ↓        ↓    │             │                    │
│  ┌──────┐ ┌──────┐ ┌──────┐ │             │                    │
│  │MCP  │ │用户 │ │记录 │ │             │                    │
│  │调用 │ │审批 │ │拒绝 │ │             │                    │
│  └──────┘ └──────┘ └──────┘ │             │                    │
│      │        │        │    │             │                    │
│      └────────┼────────┘    │             │                    │
│               │             │             │                    │
│               ↓             │             │                    │
│          ┌─────────┐        │             │                    │
│          │记录结果 │        │             │                    │
│          └─────────┘        │             │                    │
│               │             │             │                    │
│               │             │             │                    │
│               ↓             ↓             ↓                    │
│          ┌─────────────────────────────────────┐               │
│          │              继续循环               │               │
│          └─────────────────────────────────────┘               │
│               │ (回到构建Prompt)                                │
│               │                                                 │
│               │                                                 │
│               └─────────────────────────────────────────────→  │
│               │                                                 │
│               ↓                                                 │
│          ┌─────────────────────────────────────┐               │
│          │              结束循环               │               │
│          │         返回最终回答给用户           │               │
│          └─────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、模型通信层

### 职责

模型通信层负责与 AI 模型的实际通信：
- 发送 API 请求
- 处理流式响应
- 错误处理和重试

### API 调用实现

```python
class ModelClient:
    def __init__(self, api_key, model, base_url):
        self.api_key = api_key
        self.model = model
        self.base_url = base_url

    async def generate(self, prompt: dict) -> ModelResponse:
        """
        调用模型 API
        """
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {self.api_key}"
        }

        payload = {
            "model": self.model,
            "system": prompt.get("system"),
            "tools": prompt.get("tools"),
            "messages": prompt.get("messages"),
            "max_tokens": prompt.get("max_tokens"),
            "stream": False  # 或 True for streaming
        }

        try:
            response = await self.http_client.post(
                f"{self.base_url}/v1/messages",
                headers=headers,
                json=payload
            )

            return self.parse_response(response.json())

        except APIError as e:
            return await self.handle_error(e)

    async def generate_stream(self, prompt: dict):
        """
        流式调用模型 API
        """
        payload = {
            ...,
            "stream": True
        }

        response = await self.http_client.post_stream(...)

        for chunk in response:
            parsed_chunk = self.parse_stream_chunk(chunk)
            yield parsed_chunk
```

### 响应解析

```python
class ResponseParser:
    def parse_response(self, raw_response: dict) -> ParsedResponse:
        """
        解析模型响应
        """
        # 检查响应类型
        content_blocks = raw_response.get("content", [])

        tool_calls = []
        text_content = []
        thinking_content = []

        for block in content_blocks:
            if block["type"] == "text":
                text_content.append(block["text"])

            elif block["type"] == "tool_use":
                tool_calls.append(ToolCall(
                    id=block["id"],
                    type=block["type"],
                    function=FunctionCall(
                        name=block["name"],
                        arguments=json.loads(block["input"])
                    )
                ))

            elif block["type"] == "thinking":
                thinking_content.append(block["thinking"])

        return ParsedResponse(
            has_tool_calls=len(tool_calls) > 0,
            tool_calls=tool_calls,
            content="\n".join(text_content),
            thinking="\n".join(thinking_content),
            is_final_answer=len(tool_calls) == 0 and len(text_content) > 0
        )
```

### 流式响应处理

```
┌─────────────────────────────────────────────────────┐
│                    流式响应                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  流式响应格式:                                       │
│                                                     │
│  {                                                  │
│    "type": "content_block_start",                   │
│    "index": 0,                                      │
│    "content_block": {                               │
│      "type": "text",                                │
│      "text": ""                                     │
│    }                                                │
│  }                                                  │
│                                                     │
│  {                                                  │
│    "type": "content_block_delta",                   │
│    "index": 0,                                      │
│    "delta": {                                       │
│      "type": "text_delta",                          │
│      "text": "这个"                                 │
│    }                                                │
│  }                                                  │
│                                                     │
│  {                                                  │
│    "type": "content_block_delta",                   │
│    "index": 0,                                      │
│    "delta": {                                       │
│      "type": "text_delta",                          │
│      "text": "代码仓库"                             │
│    }                                                │
│  }                                                  │
│                                                     │
│  ...更多增量...                                      │
│                                                     │
│  {                                                  │
│    "type": "content_block_stop",                    │
│    "index": 0                                       │
│  }                                                  │
│                                                     │
│  处理流程:                                          │
│  1. 收集增量内容                                    │
│  2. 实时展示给用户（可选）                           │
│  3. 最终组装完整响应                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 六、MCP 通信层

### 职责

MCP 通信层负责与 MCP Server 的所有交互：
- Server 生命周期管理
- 工具调用执行
- 资源访问

### Server 管理

```python
class MCPManager:
    def __init__(self):
        self.servers = {}  # 已连接的 Server
        self.tools = {}    # 所有可用工具
        self.resources = {}  # 所有可用资源

    async def connect_server(self, config: ServerConfig):
        """
        连接 MCP Server
        """
        # 启动 Server 进程
        process = await asyncio.create_subprocess_exec(
            config.command,
            *config.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE
        )

        server = MCPServerConnection(process)

        # 执行握手
        await server.initialize()

        # 获取工具和资源列表
        tools = await server.list_tools()
        resources = await server.list_resources()

        # 注册到管理器
        self.servers[config.name] = server
        for tool in tools:
            self.tools[tool.name] = (config.name, tool)
        for resource in resources:
            self.resources[resource.uri] = (config.name, resource)

    async def disconnect_server(self, name: str):
        """
        断开 MCP Server
        """
        if name in self.servers:
            await self.servers[name].shutdown()
            del self.servers[name]

            # 移除相关工具和资源
            self.tools = {k: v for k, v in self.tools.items() if v[0] != name}
            self.resources = {k: v for k, v in self.resources.items() if v[0] != name}
```

### 工具调用执行

```python
class ToolExecutor:
    def __init__(self, mcp_manager):
        self.mcp_manager = mcp_manager

    async def execute(self, tool_name: str, arguments: dict) -> str:
        """
        执行工具调用
        """
        # 查找工具所属的 Server
        server_name, tool_def = self.mcp_manager.tools.get(tool_name)

        if not server_name:
            return f"工具 {tool_name} 不存在"

        server = self.mcp_manager.servers[server_name]

        # 发送调用请求
        result = await server.call_tool(tool_name, arguments)

        # 解析结果
        return self.format_result(result)

    def format_result(self, result: dict) -> str:
        """
        格式化工具结果
        """
        content = result.get("content", [])

        formatted_parts = []
        for block in content:
            if block["type"] == "text":
                formatted_parts.append(block["text"])
            elif block["type"] == "image":
                formatted_parts.append(f"[图片: {block['mimeType']}]")
            elif block["type"] == "resource":
                formatted_parts.append(f"[资源引用: {block['resource']['uri']}]")

        return "\n".join(formatted_parts)
```

### JSON-RPC 通信实现

```python
class MCPServerConnection:
    def __init__(self, process):
        self.process = process
        self.request_id = 0
        self.pending_requests = {}

    async def send_request(self, method: str, params: dict = None) -> dict:
        """
        发送 JSON-RPC 请求
        """
        request_id = self.request_id + 1
        self.request_id = request_id

        request = {
            "jsonrpc": "2.0",
            "id": request_id,
            "method": method,
            "params": params or {}
        }

        # 发送请求
        request_json = json.dumps(request) + "\n"
        self.process.stdin.write(request_json.encode())
        await self.process.stdin.drain()

        # 等待响应
        response_line = await self.process.stdout.readline()
        response = json.loads(response_line)

        return response.get("result")

    async def initialize(self):
        """
        执行初始化握手
        """
        # 1. 发送 initialize 请求
        result = await self.send_request("initialize", {
            "protocolVersion": "2024-11-05",
            "clientInfo": {
                "name": "claude-code",
                "version": "1.0.0"
            },
            "capabilities": {}
        })

        # 2. 发送 initialized 通知
        notification = {
            "jsonrpc": "2.0",
            "method": "notifications/initialized"
        }
        self.process.stdin.write(json.dumps(notification).encode())
        await self.process.stdin.drain()

    async def list_tools(self) -> list:
        """
        获取工具列表
        """
        result = await self.send_request("tools/list")
        return result.get("tools", [])

    async def call_tool(self, name: str, arguments: dict) -> dict:
        """
        调用工具
        """
        return await self.send_request("tools/call", {
            "name": name,
            "arguments": arguments
        })
```

---

## 七、输出处理层

### 职责

输出处理层负责将模型响应和工具结果格式化展示给用户。

### 文本格式化

```python
class OutputFormatter:
    def format_text(self, text: str) -> str:
        """
        格式化文本输出
        """
        # 处理 Markdown
        # 处理代码块
        # 处理特殊格式

        return text

    def format_tool_result(self, tool_name: str, result: str) -> str:
        """
        格式化工具结果
        """
        # 根据工具类型选择格式
        if tool_name in ["read_file", "query"]:
            # 文件内容或查询结果，用代码块
            return f"```\n{result}\n```"

        elif tool_name == "bash":
            # 命令输出，可能包含 stdout 和 stderr
            return self.format_command_output(result)

        else:
            return result

    def format_progress(self, tool_name: str, status: str) -> str:
        """
        格式化进度指示
        """
        return f"⏳ {tool_name}: {status}"
```

### 多媒体处理

```
┌─────────────────────────────────────────────────────┐
│                    多媒体输出                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  图片输出:                                          │
│    - MCP 返回图片数据                               │
│    - 转换为可显示格式                               │
│    - 在终端/界面中展示                              │
│                                                     │
│  代码块输出:                                        │
│    - 检测语言类型                                   │
│    - 应用语法高亮                                   │
│    - 添加行号                                       │
│                                                     │
│  表格输出:                                          │
│    - 解析结构化数据                                 │
│    - 格式化为 Markdown 表格                         │
│                                                     │
│  文件引用:                                          │
│    - 显示文件路径                                   │
│    - 可点击跳转                                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 八、完整数据流

```
用户输入: "帮我分析这个代码仓库的结构"

完整数据流:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  [用户接口层]                                                                │
│     │                                                                       │
│     │ 接收输入 → 解析 → 添加到上下文                                         │
│     ↓                                                                       │
│                                                                             │
│  [会话管理层]                                                                │
│     │                                                                       │
│     │ 上下文结构:                                                            │
│     │   messages: [{ role: "user", content: "帮我分析..." }]                │
│     │   tools: { query: {...}, read: {...} }                                │
│     │                                                                       │
│     ↓                                                                       │
│                                                                             │
│  [Prompt 构建层]                                                             │
│     │                                                                       │
│     │ 构建:                                                                  │
│     │   system: "你是一个代码助手..."                                        │
│     │   tools: [工具定义列表]                                                │
│     │   messages: [消息历史 + 当前输入]                                      │
│     │                                                                       │
│     ↓                                                                       │
│                                                                             │
│  [核心循环]                                                                  │
│     │                                                                       │
│     │ 第一轮:                                                                │
│     │ ────────────────────────────────────────                              │
│     │ [模型通信层]                                                           │
│     │   │ 发送请求                                                          │
│     │   │ 接收响应: { tool_calls: [{ name: "query" }] }                     │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [输出解析]                                                             │
│     │   │ 发现工具调用                                                       │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [权限检查]                                                             │
│     │   │ query 工具在白名单 → 直接执行                                      │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [MCP 通信层]                                                           │
│     │   │ 发送 tools/call 到 code-memory Server                             │
│     │   │ 接收结果: "仓库包含 src/core, src/api..."                          │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [会话管理层]                                                           │
│     │   │ 添加工具结果到上下文                                                │
│     │   │ messages: [..., { role: "tool", content: "..." }]                 │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ 继续循环 ────────────────────────────────→                             │
│     │                                                                       │
│     │ 第二轮:                                                                │
│     │ ────────────────────────────────────────                              │
│     │ [模型通信层]                                                           │
│     │   │ 发送请求（包含工具结果）                                            │
│     │   │ 接收响应: { content: "这个代码仓库的结构如下..." }                  │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [输出解析]                                                             │
│     │   │ 最终回答，结束循环                                                  │
│     │   ↓                                                                   │
│     │                                                                       │
│     │ [输出处理层]                                                           │
│     │   │ 格式化文本                                                         │
│     │   │ 展示给用户                                                         │
│     │   ↓                                                                   │
│     │                                                                       │
│     ↓                                                                       │
│                                                                             │
│  [用户接口层]                                                                │
│     │                                                                       │
│     │ 展示: "这个代码仓库的结构如下..."                                       │
│     │                                                                       │
│     ↓                                                                       │
│                                                                             │
│  用户看到最终答案                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 下一步

- 了解协议规范细节，请阅读 [协议规范详解](./08-protocol-specification.md)