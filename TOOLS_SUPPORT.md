# Tools 支持说明

## 概述

deepseek2api 现已支持 OpenAI 格式的 `tools` 参数，可用于：
- **MCP (Model Context Protocol)** 工具调用
- **OpenClaw** 本地技能集成
- 任何符合 OpenAI tools 规范的应用

## 功能特性

✅ 支持 OpenAI 标准的 `tools` 参数格式
✅ 自动检测模型返回的 tool_calls
✅ 支持流式和非流式响应
✅ 兼容现有的 `/v1/chat/completions` 端点
✅ 返回标准的 `finish_reason: "tool_calls"`

## 使用方法

### 1. 基本示例

```python
import requests
import json

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "城市名称"
                    }
                },
                "required": ["location"]
            }
        }
    }
]

response = requests.post(
    "http://localhost:8000/v1/chat/completions",
    json={
        "model": "deepseek-chat",
        "messages": [
            {"role": "user", "content": "北京今天天气怎么样？"}
        ],
        "tools": tools
    }
)

result = response.json()
print(result["choices"][0]["message"])
```

### 2. 响应格式

当模型决定调用工具时，响应格式如下：

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "deepseek-chat",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "",
        "tool_calls": [
          {
            "id": "call_001",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\": \"北京\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ],
  "usage": {...}
}
```

### 3. 工具调用流程

1. **发送请求**：在 `messages` 中包含用户问题，在 `tools` 中定义可用工具
2. **模型决策**：模型分析是否需要调用工具
3. **返回 tool_calls**：如果需要，返回 `tool_calls` 数组和 `finish_reason: "tool_calls"`
4. **执行工具**：客户端执行工具并获取结果
5. **继续对话**：将工具结果作为 `tool` 角色的消息发送回模型

### 4. 完整对话示例

```python
# 第一轮：用户提问
messages = [
    {"role": "user", "content": "北京今天天气怎么样？"}
]

response1 = requests.post(url, json={
    "model": "deepseek-chat",
    "messages": messages,
    "tools": tools
})

# 模型返回 tool_calls
tool_calls = response1.json()["choices"][0]["message"]["tool_calls"]

# 执行工具
tool_result = get_weather(location="北京")  # 你的工具实现

# 第二轮：提供工具结果
messages.append({
    "role": "assistant",
    "content": "",
    "tool_calls": tool_calls
})
messages.append({
    "role": "tool",
    "tool_call_id": tool_calls[0]["id"],
    "content": json.dumps(tool_result)
})

response2 = requests.post(url, json={
    "model": "deepseek-chat",
    "messages": messages,
    "tools": tools
})

# 模型基于工具结果生成最终回答
final_answer = response2.json()["choices"][0]["message"]["content"]
```

## MCP 集成

对于 MCP 客户端（如 Claude Desktop、Cursor 等），只需配置 API 端点：

```json
{
  "mcpServers": {
    "deepseek": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-everything"],
      "env": {
        "OPENAI_API_BASE": "http://localhost:8000/v1",
        "OPENAI_API_KEY": "your-key"
      }
    }
  }
}
```

## OpenClaw 集成

OpenClaw 可以通过标准的 OpenAI 客户端调用：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="your-key"
)

# OpenClaw 会自动处理 tools 参数
response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[...],
    tools=[...]
)
```

## 流式响应

流式模式下，tool_calls 会在流的末尾发送：

```python
response = requests.post(url, json={
    "model": "deepseek-chat",
    "messages": messages,
    "tools": tools,
    "stream": True
}, stream=True)

for line in response.iter_lines():
    if line.startswith(b'data: '):
        data = json.loads(line[6:])
        delta = data["choices"][0]["delta"]
        
        if "tool_calls" in delta:
            print(f"Tool call: {delta['tool_calls']}")
        elif "content" in delta:
            print(delta["content"], end="")
```

## 注意事项

1. **模型能力**：DeepSeek 模型需要理解工具调用的指令格式，建议使用 deepseek-chat 或更新的模型
2. **JSON 格式**：模型需要返回特定格式的 JSON 才能被识别为 tool_calls
3. **参数验证**：客户端应验证 tool_calls 的参数是否符合工具定义
4. **错误处理**：如果模型没有正确返回 tool_calls 格式，会作为普通文本返回

## 测试

运行测试脚本：

```bash
python3 test_tools.py
```

## 相关 Issues

- Issue #50: MCP 工具支持 ✅
- Issue #52: OpenClaw 支持 ✅

## 更新日志

- 2026-03-21: 添加 tools 参数支持，实现 MCP 和 OpenClaw 集成
