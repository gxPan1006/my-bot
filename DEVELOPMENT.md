# Nanobot 开发指南

## 1. 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                        Channels Layer                        │
│  (Telegram, WhatsApp, Discord, Feishu, CLI)                 │
└────────────────┬────────────────────────────────────────────┘
                 │ InboundMessage
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                       Message Bus                            │
│  (Async Queue - 消息路由中枢)                                  │
└────────────────┬────────────────────────────────────────────┘
                 │ consume_inbound()
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                       Agent Loop                             │
│  ├─ Context Builder (构建 prompt)                            │
│  ├─ Session Manager (会话历史)                               │
│  ├─ Tool Registry (工具注册)                                  │
│  ├─ Subagent Manager (后台任务)                              │
│  └─ LLM Provider (调用 LLM)                                   │
└────────────────┬────────────────────────────────────────────┘
                 │ publish_outbound()
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                    Channel Manager                           │
│  (分发响应到对应的 Channel)                                    │
└─────────────────────────────────────────────────────────────┘
```

## 2. 关键文件

| 文件 | 作用 |
|------|------|
| `nanobot/agent/loop.py` | 核心循环：消息处理、LLM 调用、工具执行 |
| `nanobot/agent/context.py` | 构建系统 prompt（身份、记忆、技能） |
| `nanobot/agent/tools/base.py` | Tool 基类 |
| `nanobot/agent/tools/registry.py` | 工具注册和执行 |
| `nanobot/channels/base.py` | Channel 基类 |
| `nanobot/channels/manager.py` | Channel 管理器 |
| `nanobot/config/schema.py` | 配置定义 (Pydantic) |
| `nanobot/config/loader.py` | 配置加载 |
| `nanobot/bus/queue.py` | 消息总线 |
| `nanobot/session/manager.py` | 会话管理 |
| `nanobot/providers/litellm_provider.py` | LLM Provider 实现 |

## 3. Agent Loop 工作流程

```python
async def _process_message(msg: InboundMessage):
    # 1. 获取会话
    session = sessions.get_or_create(msg.session_key)

    # 2. 构建上下文 (system prompt + history + current message)
    messages = context.build_messages(
        history=session.get_history(),
        current_message=msg.content,
        media=msg.media
    )

    # 3. Agent 迭代循环
    for iteration in range(max_iterations):
        # 3.1 调用 LLM
        response = provider.chat(
            messages=messages,
            tools=tools.get_definitions()
        )

        # 3.2 如果有 tool calls
        if response.has_tool_calls:
            # 添加 assistant 消息 (包含 tool_calls)
            messages.append({
                "role": "assistant",
                "content": response.content,
                "tool_calls": [...]
            })

            # 执行工具
            for tool_call in response.tool_calls:
                result = await tools.execute(
                    tool_call.name,
                    tool_call.arguments
                )
                # 添加 tool 结果
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "name": tool_call.name,
                    "content": result
                })
        else:
            # 3.3 没有 tool calls，完成
            final_content = response.content
            break

    # 4. 保存会话
    session.add_message("user", msg.content)
    session.add_message("assistant", final_content)
    sessions.save(session)

    # 5. 返回响应
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=final_content
    )
```

## 4. 添加新 Tool

### 4.1 创建 Tool 类

在 `nanobot/agent/tools/` 目录下创建新文件：

```python
# nanobot/agent/tools/my_tool.py
from typing import Any
from nanobot.agent.tools.base import Tool

class MyTool(Tool):
    """Tool 描述"""

    @property
    def name(self) -> str:
        return "my_tool"  # 工具名称

    @property
    def description(self) -> str:
        return "这个工具的功能描述，LLM 会看到"

    @property
    def parameters(self) -> dict[str, Any]:
        """JSON Schema 格式的参数定义"""
        return {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "参数1的描述"
                },
                "param2": {
                    "type": "integer",
                    "description": "参数2的描述"
                }
            },
            "required": ["param1"]  # 必填参数
        }

    async def execute(self, param1: str, param2: int = 0, **kwargs: Any) -> str:
        """执行工具逻辑，返回字符串结果"""
        try:
            result = f"处理 {param1} 和 {param2}"
            return result
        except Exception as e:
            return f"Error: {str(e)}"
```

### 4.2 注册 Tool

在 `nanobot/agent/loop.py` 的 `_register_default_tools()` 方法中添加:

```python
def _register_default_tools(self) -> None:
    # ... 现有工具 ...

    # 注册新工具
    from nanobot.agent.tools.my_tool import MyTool
    self.tools.register(MyTool())
```

### 4.3 Tool 开发最佳实践

1. **错误处理**：总是捕获异常并返回友好的错误消息
2. **参数验证**：使用 JSON Schema 定义参数，基类会自动验证
3. **异步执行**：工具方法必须是 `async`
4. **结果长度**：如果输出很长，考虑截断（参考 `ExecTool` 的实现）
5. **上下文传递**：如果需要上下文（如 channel/chat_id），使用 `set_context()` 方法（参考 `MessageTool`）

## 5. 添加新 Channel

### 5.1 创建 Channel 类

在 `nanobot/channels/` 目录下创建文件：

```python
# nanobot/channels/my_channel.py
import asyncio
from loguru import logger
from nanobot.channels.base import BaseChannel
from nanobot.bus.events import OutboundMessage
from nanobot.bus.queue import MessageBus
from nanobot.config.schema import MyChannelConfig

class MyChannel(BaseChannel):
    """新 Channel 实现"""

    name = "my_channel"  # Channel 名称

    def __init__(self, config: MyChannelConfig, bus: MessageBus):
        super().__init__(config, bus)
        self.config: MyChannelConfig = config
        # 初始化 Channel 特定的客户端

    async def start(self) -> None:
        """启动 Channel，监听消息"""
        self._running = True

        while self._running:
            try:
                # 从平台获取消息
                platform_msg = await self._fetch_message()

                # 转换并发送到 MessageBus
                await self._handle_message(
                    sender_id=platform_msg.user_id,
                    chat_id=platform_msg.chat_id,
                    content=platform_msg.text,
                    media=platform_msg.media_urls,
                    metadata={"platform_specific": "data"}
                )
            except asyncio.TimeoutError:
                continue

    async def stop(self) -> None:
        """停止 Channel"""
        self._running = False
        # 清理资源

    async def send(self, msg: OutboundMessage) -> None:
        """发送消息到平台"""
        try:
            await self._platform_send(
                chat_id=msg.chat_id,
                text=msg.content
            )
        except Exception as e:
            logger.error(f"发送失败: {e}")
```

### 5.2 添加配置 Schema

在 `nanobot/config/schema.py` 中添加:

```python
class MyChannelConfig(BaseModel):
    """MyChannel 配置"""
    enabled: bool = False
    api_token: str = ""
    allow_from: list[str] = Field(default_factory=list)

class ChannelsConfig(BaseModel):
    # ... 现有配置 ...
    my_channel: MyChannelConfig = Field(default_factory=MyChannelConfig)
```

### 5.3 注册 Channel

在 `nanobot/channels/manager.py` 的 `_init_channels()` 方法中添加:

```python
def _init_channels(self) -> None:
    # ... 现有 Channel ...

    # MyChannel
    if self.config.channels.my_channel.enabled:
        try:
            from nanobot.channels.my_channel import MyChannel
            self.channels["my_channel"] = MyChannel(
                self.config.channels.my_channel, self.bus
            )
            logger.info("MyChannel enabled")
        except ImportError as e:
            logger.warning(f"MyChannel not available: {e}")
```

### 5.4 Channel 开发最佳实践

1. **权限检查**：使用 `is_allowed()` 方法检查用户权限
2. **消息格式化**：在 `send()` 方法中处理 Markdown → 平台格式的转换
3. **错误恢复**：长时间运行的 Channel 应该自动重连
4. **媒体处理**：下载媒体文件到 `~/.nanobot/media/`

## 6. 添加新 LLM Provider

### 6.1 创建 Provider 类

```python
# nanobot/providers/my_provider.py
from typing import Any
from nanobot.providers.base import LLMProvider, LLMResponse, ToolCallRequest

class MyLLMProvider(LLMProvider):
    """新 LLM Provider"""

    def __init__(self, api_key: str | None = None, api_base: str | None = None):
        super().__init__(api_key, api_base)
        self.default_model = "my-default-model"

    async def chat(
        self,
        messages: list[dict[str, Any]],
        tools: list[dict[str, Any]] | None = None,
        model: str | None = None,
        max_tokens: int = 4096,
        temperature: float = 0.7,
    ) -> LLMResponse:
        """发送聊天请求"""
        model = model or self.default_model

        # 调用平台 API
        response = await self._call_api(
            messages=messages,
            tools=tools,
            model=model,
            max_tokens=max_tokens,
            temperature=temperature
        )

        return self._parse_response(response)

    def _parse_response(self, response: Any) -> LLMResponse:
        """解析平台响应为标准格式"""
        tool_calls = []

        if hasattr(response, 'tool_calls') and response.tool_calls:
            for tc in response.tool_calls:
                tool_calls.append(ToolCallRequest(
                    id=tc.id,
                    name=tc.function.name,
                    arguments=tc.function.arguments
                ))

        return LLMResponse(
            content=response.content,
            tool_calls=tool_calls,
            finish_reason=response.finish_reason or "stop",
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens,
            }
        )

    def get_default_model(self) -> str:
        return self.default_model
```

### 6.2 集成到配置系统

在 `nanobot/config/schema.py` 中添加:

```python
class ProvidersConfig(BaseModel):
    # ... 现有 providers ...
    my_provider: ProviderConfig = Field(default_factory=ProviderConfig)
```

## 7. Skills 系统

### 7.1 Skill 结构

每个 Skill 是一个目录，包含 `SKILL.md` 文件:

```
nanobot/skills/
├── my-skill/
│   └── SKILL.md
```

### 7.2 SKILL.md 格式

```markdown
---
name: my-skill
description: 技能简短描述
metadata: |
  {
    "nanobot": {
      "always": false,
      "requires": {
        "bins": ["some-cli"],
        "env": ["API_KEY"]
      }
    }
  }
---

# My Skill 使用说明

这里是给 Agent 看的详细说明。

## 如何使用

1. 调用 some-cli 命令
2. ...
```

### 7.3 Skill 加载机制

**渐进式加载 (Progressive Loading)**:

1. **Always-loaded Skills** (`always: true`)
   - 完整内容加载到系统 prompt
   - 适合核心功能

2. **按需加载 (On-demand)**
   - 只在系统 prompt 中显示摘要 (名称、描述、路径)
   - Agent 使用 `read_file` 工具读取完整内容
   - 减少 token 消耗

### 7.4 创建新 Skill

1. 在 `~/.nanobot/workspace/skills/` 创建目录
2. 创建 `SKILL.md` 文件
3. Agent 会自动发现并可以使用

## 8. 配置系统

### 8.1 配置文件位置

```
~/.nanobot/
├── config.json      # 主配置
├── workspace/       # 工作目录
│   ├── AGENTS.md    # Agent 指令
│   ├── SOUL.md      # 人格设定
│   ├── USER.md      # 用户信息
│   ├── memory/      # 长期记忆
│   │   └── MEMORY.md
│   └── skills/      # 自定义技能
└── sessions/        # 会话历史 (JSONL)
```

### 8.2 配置结构 (Pydantic)

```python
Config (根配置)
├── agents: AgentsConfig
│   └── defaults: AgentDefaults
│       ├── workspace: str           # 工作目录
│       ├── model: str               # 默认模型
│       ├── max_tokens: int          # 最大 token
│       ├── temperature: float       # 温度
│       └── max_tool_iterations: int # 最大工具迭代次数
├── channels: ChannelsConfig
│   ├── telegram: TelegramConfig
│   ├── whatsapp: WhatsAppConfig
│   ├── discord: DiscordConfig
│   └── feishu: FeishuConfig
├── providers: ProvidersConfig
│   ├── openrouter: ProviderConfig
│   ├── anthropic: ProviderConfig
│   ├── openai: ProviderConfig
│   ├── deepseek: ProviderConfig
│   ├── groq: ProviderConfig
│   ├── gemini: ProviderConfig
│   ├── vllm: ProviderConfig
│   └── moonshot: ProviderConfig
├── gateway: GatewayConfig
│   ├── host: str
│   └── port: int
└── tools: ToolsConfig
    ├── web: WebToolsConfig
    │   └── search: WebSearchConfig
    ├── exec: ExecToolConfig
    └── restrict_to_workspace: bool  # 安全沙箱
```

### 8.3 配置加载示例

```python
from nanobot.config.loader import load_config

config = load_config()  # 从 ~/.nanobot/config.json 加载
api_key = config.get_api_key()  # 自动匹配 provider
workspace = config.workspace_path  # 展开路径
```

### 8.4 安全配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `tools.restrictToWorkspace` | `false` | 限制所有工具在 workspace 目录内 |
| `channels.*.allowFrom` | `[]` | 用户白名单，空表示允许所有 |

## 9. 设计模式

### 9.1 消息总线模式 (Message Bus)
- **优点**：解耦 Channel 和 Agent，易于扩展
- **实现**：`asyncio.Queue`

### 9.2 工具注册模式 (Tool Registry)
- **优点**：动态注册，支持插件化
- **实现**：字典存储 + OpenAI 格式工具定义

### 9.3 渐进式上下文加载
- **优点**：减少 token 消耗
- **实现**：Skills 摘要 + 按需 read_file

### 9.4 子代理模式 (Subagent)
- **优点**：后台任务隔离，避免阻塞主循环
- **实现**：独立 Agent 实例 + 完成后通知

## 10. 快速开发示例

### 10.1 添加计算器工具

```python
# nanobot/agent/tools/calculator.py
from typing import Any
from nanobot.agent.tools.base import Tool

class CalculatorTool(Tool):
    @property
    def name(self) -> str:
        return "calculator"

    @property
    def description(self) -> str:
        return "执行数学计算表达式"

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 '2 + 2 * 3'"
                }
            },
            "required": ["expression"]
        }

    async def execute(self, expression: str, **kwargs: Any) -> str:
        try:
            # 注意：生产环境应使用安全的表达式解析器
            result = eval(expression)
            return f"结果: {result}"
        except Exception as e:
            return f"计算错误: {e}"
```

### 10.2 测试

```bash
nanobot agent -m "计算 2 + 2 * 3"
```

## 11. CLI 命令参考

| 命令 | 说明 |
|------|------|
| `nanobot onboard` | 初始化配置和工作空间 |
| `nanobot agent -m "..."` | 单次对话 |
| `nanobot agent` | 交互模式 |
| `nanobot gateway` | 启动网关 (连接聊天平台) |
| `nanobot status` | 查看状态 |
| `nanobot channels login` | WhatsApp 扫码登录 |
| `nanobot cron list` | 查看定时任务 |
| `nanobot cron add` | 添加定时任务 |

## 12. 调试技巧

```bash
# 启用详细日志
nanobot gateway --verbose

# 查看配置
nanobot status

# 测试单条消息
nanobot agent -m "测试消息"
```
