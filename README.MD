# DeepSeek2API

[![](https://img.shields.io/github/license/a0yark/deepseek2api.svg)](LICENSE)
![](https://img.shields.io/github/stars/a0yark/deepseek2api.svg)
![](https://img.shields.io/github/forks/a0yark/deepseek2api.svg)

支持高速流式输出、支持多轮对话、支持深度思考(reasoning)，多路账号支持，支持对话结束后自动删除 DeepSeek 会话，支持工具函数调用。

与 ChatGPT 接口完全兼容。

> **本仓库是 [iidamie/deepseek2api](https://github.com/iidamie/deepseek2api) 的 fork。** 上游已 archive,但 DeepSeek 网页端在 2026-04 升级到 V4 协议后原实现失效,本 fork 基于实测抓包重写关键路径并增加 [claude-code-router](https://github.com/musistudio/claude-code-router) 适配,可直接用于 Claude Code。详见 [v1.4.0 commit](https://github.com/a0yark/deepseek2api/commit/c6808ff)。

## v1.4.0 更新摘要

* **DeepSeek V4 网页端协议**:升级 BASE_HEADERS、`chat_session/create` 空 body、`completion` 新增 `model_type` / `preempt`、SSE 改 fragments OT 协议
* **新模型命名**:`deepseek-v4-flash` (Instant) / `deepseek-v4-pro` (Expert),含 `-think` / `-search` 变体;旧 `deepseek-chat` / `deepseek-reasoner` 别名保留
* **Claude / Anthropic 工具兼容**:接受 Anthropic 风格 `tools` 数组、扁平化 `system` content、流式工具调用 buffer + emit;detector 同时识别 OpenAI / Anthropic / 单对象 / 数组 / markdown 包裹的 5 种格式
* **依赖瘦身**:删除未实际使用的 `transformers` (镜像减重 ~1GB)
* **稳定性**:WASM bytes 缓存、账号队列与 config 写入加锁、防双重 release

## 目录

* [免责声明](#免责声明)
* [接入准备](#接入准备)
  * [多账号接入](#多账号接入)
* [Docker部署](#Docker部署)
* [Docker-compose部署](#Docker-compose部署)
* [Vercel部署](#Vercel部署)
* [原生部署](#原生部署)
* [接入 Claude Code](#接入-claude-code)
* [接口列表](#接口列表)
  * [模型列表](#模型列表)
  * [对话补全](#对话补全)
* [注意事项](#注意事项)
  * [Nginx反代优化](#Nginx反代优化)
  * [Token统计](#Token统计)
* [Star History](#star-history)
* [鸣谢](#鸣谢)

## 免责声明

**逆向API是不稳定的，建议前往DeepSeek官方 https://platform.deepseek.com/ 付费使用API，避免封禁的风险。**

**本组织和个人不接受任何资金捐助和交易，此项目是纯粹研究交流学习性质！**

**仅限自用，禁止对外提供服务或商用，避免对官方造成服务压力，否则风险自担！**

**仅限自用，禁止对外提供服务或商用，避免对官方造成服务压力，否则风险自担！**

**仅限自用，禁止对外提供服务或商用，避免对官方造成服务压力，否则风险自担！**

## 接入准备

一个或多个 DeepSeek 账号

### 多账号接入

目前同个账号同时只能有*一路*输出，你可以通过提供多个账号并使用。

每次请求服务会从中挑选一个。

## Docker部署

请准备一台具有公网IP的服务器并将5001端口开放。

配置 config.json

在 `deepseek` 目录下，创建 config.json文件
```json
{
  "keys": [
    "key1",
    "key2"
  ],
  "accounts": [
    {
      "email": "example1@example.com",
      "password": "password1",
      "token": ""
    },
    {
      "email": "example2@example.com",
      "password": "password2",
      "token": ""
    },
    {
      "mobile": "12345678901",
      "password": "password3",
      "token": ""
    },
    {
      "mobile": "12345678901",
      "password": "password4",
      "token": ""
    }
  ]
}
```
 * keys - 你的 API 鉴权密钥
 * accounts - DeepSeek 账号列表，支持多个账号轮换，避免单账号受限

拉取镜像并启动服务。

```shell
docker run -d -p 5001:5001 -v "$(pwd)/config.json:/app/config.json" --name deepseek2api ghcr.io/iidamie/deepseek2api:latest
```

查看服务实时日志

```shell
docker logs -f deepseek2api
```

重启服务

```shell
docker restart deepseek2api
```

停止服务

```shell
docker stop deepseek2api
```

## Docker-compose部署

拉取该项目
```shell
git clone https://github.com/iidamie/deepseek2api.git
cd deepseek2api
```
配置 config.json

在当前目录下，修改 config.json 文件
 * keys - 你的 API 鉴权密钥
 * accounts - DeepSeek 账号列表，支持多个账号轮换，避免单账号受限

启动服务

```shell
docker-compose up -d
```

查看服务实时日志

```shell
docker logs -f deepseek2api
```

重启服务

```shell
docker restart deepseek2api
```

停止服务

```shell
docker stop deepseek2api
```

## Vercel部署

> [!NOTE]
> Vercel免费账户的请求响应超时时间为10秒，但接口响应通常较久，可能会遇到Vercel返回的504超时错误！

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/import/project?template=https://github.com/iidamie/deepseek2api)

部署完后，返回你的 Github 仓库配置 config.json
 * keys - 你的 API 鉴权密钥
 * accounts - DeepSeek 账号列表，支持多个账号轮换，避免单账号受限

> [!WARNING]  
> 一定要是私库，防止信息泄露！

配置完后，返回 Vercel 控制台，重新部署即可

## 原生部署

请准备一台具有公网IP的服务器并将5001端口开放。

请先安装好 Python 环境并且配置好环境变量，确认 python 命令可用。

安装依赖

```shell
git clone https://github.com/iidamie/deepseek2api.git
cd deepseek2api
pip install -r requirements.txt
```

配置 config.json

在当前目录下，修改 config.json 文件
 * keys - 你的 API 鉴权密钥
 * accounts - DeepSeek 账号列表，支持多个账号轮换，避免单账号受限

启动服务

```shell
python app.py
```

使用 nohup 启动

```shell
nohup uvicorn app:app --host 0.0.0.0 --port 5001 > uvicorn.log 2>&1 &
```

## 接入 Claude Code

通过 [claude-code-router](https://github.com/musistudio/claude-code-router) 把本服务包成 Anthropic 兼容端点,Claude Code 可直接使用,**支持 MCP 工具调用与 Skill** (实测多步工具循环可用,但 DeepSeek 模型一次只 emit 一个 tool_call,并行多工具会被串行化,不是协议层问题)。

1. 启动本服务(`python app.py`,默认 `http://127.0.0.1:5001`)

2. 安装 ccr:
   ```shell
   npm install -g @musistudio/claude-code-router
   ```

3. 编辑 `~/.claude-code-router/config.json`,加一个指向本服务的 OpenAI 兼容 provider:
   ```json
   {
     "Providers": [
       {
         "name": "deepseek",
         "api_base_url": "http://127.0.0.1:5001/v1/chat/completions",
         "api_key": "<你的 deepseek 网页 token,或 config.json keys 中的一个>",
         "models": [
           "deepseek-v4-flash",
           "deepseek-v4-pro",
           "deepseek-v4-pro-think",
           "deepseek-v4-pro-think-search",
           "deepseek-v4-flash-search"
         ]
       }
     ],
     "Router": {
       "default":     "deepseek,deepseek-v4-flash",
       "background":  "deepseek,deepseek-v4-flash",
       "think":       "deepseek,deepseek-v4-pro-think",
       "longContext": "deepseek,deepseek-v4-flash",
       "longContextThreshold": 60000,
       "webSearch":   "deepseek,deepseek-v4-flash-search"
     }
   }
   ```
   `api_key` 字段:如果填的是本仓库 `config.json` `keys` 里的值,服务会用账号池里的某个账号 token;否则它直接被当作 deepseek 网页 token 发给 chat.deepseek.com。

4. 启动并使用:
   ```shell
   ccr restart
   ccr code
   ```

实测限制(请按预期使用):

* 并行多工具调用会被 DeepSeek 模型串行化
* `prompt_caching` 未实现,长会话每轮全量重发
* Expert 模式输入上限约 163,840 字符,超长 `system` + 大量工具 schema 时切到 `deepseek-v4-flash` (上限约 2,621,440 字符)
* 图片输入未支持

## 接口列表

目前支持与openai兼容的 `/v1/chat/completions` 接口，可自行使用与openai或其他兼容的客户端接入接口。

### 模型列表

获取模型列表接口

**GET /v1/models**

支持的 model 名(全部映射到 DeepSeek 网页端的 `model_type` + `thinking_enabled` + `search_enabled` 三元组):

| model 名 | DeepSeek 内部档位 | think | search |
|---|---|---|---|
| `deepseek-v4-flash` / `deepseek-chat` / `deepseek-v3` / `deepseek-v3.1` / `deepseek-v3.2` / `deepseek-instant` | default (Instant) | ✗ | ✗ |
| `deepseek-v4-flash-search` / `deepseek-chat-search` / `deepseek-instant-search` / `deepseek-v3-search` | default | ✗ | ✓ |
| `deepseek-v4-flash-think` | default | ✓ | ✗ |
| `deepseek-v4-flash-think-search` | default | ✓ | ✓ |
| `deepseek-v4-pro` / `deepseek-expert` | expert (Expert) | ✗ | ✗ |
| `deepseek-v4-pro-think` / `deepseek-reasoner` / `deepseek-r1` / `deepseek-v3.1-think` / `deepseek-v3.2-think` / `deepseek-expert-think` | expert | ✓ | ✗ |
| `deepseek-v4-pro-search` / `deepseek-expert-search` | expert | ✗ | ✓ |
| `deepseek-v4-pro-think-search` / `deepseek-reasoner-search` / `deepseek-r1-search` / `deepseek-expert-think-search` | expert | ✓ | ✓ |

> Instant 输入字符上限 ~2,621,440;Expert ~163,840(实测,以 chat.deepseek.com 当时的 settings 为准)。

### 对话补全

对话补全接口，与openai的 [chat-completions-api](https://platform.openai.com/docs/guides/text-generation/chat-completions-api) 兼容。

**POST /v1/chat/completions**

header 需要设置 Authorization 头部：

```
Authorization: Bearer [userToken value or keys]
```

请求数据:
```json
{
    // model 名称(参考上方表格,推荐使用 V4 命名)
    // 默认:           deepseek-v4-flash
    // 深度思考:        deepseek-v4-pro-think
    // 联网搜索:        deepseek-v4-flash-search
    // 联网搜索+深度思考: deepseek-v4-pro-think-search
    "model": "deepseek-v4-flash",
    // 多轮对话基于消息合并实现,某些场景可能导致能力下降且受单轮最大 token 数限制
    "messages": [
        {
            "role": "user",
            "content": "你是谁？"
        }
    ],
    // 如果使用流式响应请设置为 true,默认 false
    "stream": false
}
```

响应数据：
```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": "您好！我是由中国的深度求索（DeepSeek）公司开发的智能助手DeepSeek-R1。如您有任何任何问题，我会尽我所能为您提供帮助。",
        "reasoning_content": "您好！我是由中国的深度求索（DeepSeek）公司开发的智能助手DeepSeek-R1。如您有任何任何问题，我会尽我所能为您提供帮助。\n",
        "role": "assistant"
      }
    }
  ],
  "created": 1738598554,
  "id": "bc223da6-f091-4687-9f59-b9f2a917bd49",
  "model": "deepseek-r1",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 37,
    "prompt_tokens": 1,
    "total_tokens": 38
  }
}
```

## 注意事项

### Nginx反代优化

如果您正在使用Nginx反向代理deepseek2api，请添加以下配置项优化流的输出效果，优化体验感。

```nginx
# 关闭代理缓冲。当设置为off时，Nginx会立即将客户端请求发送到后端服务器，并立即将从后端服务器接收到的响应发送回客户端。
proxy_buffering off;
# 启用分块传输编码。分块传输编码允许服务器为动态生成的内容分块发送数据，而不需要预先知道内容的大小。
chunked_transfer_encoding on;
# 开启TCP_NOPUSH，这告诉Nginx在数据包发送到客户端之前，尽可能地发送数据。这通常在sendfile使用时配合使用，可以提高网络效率。
tcp_nopush on;
# 开启TCP_NODELAY，这告诉Nginx不延迟发送数据，立即发送小数据包。在某些情况下，这可以减少网络的延迟。
tcp_nodelay on;
# 设置保持连接的超时时间，这里设置为120秒。如果在这段时间内，客户端和服务器之间没有进一步的通信，连接将被关闭。
keepalive_timeout 120;
```

### Token统计

符合 OPENAI 接口规范

示例:
```json
"usage": {
  "completion_tokens": 37,
  "prompt_tokens": 1,
  "total_tokens": 38
}
```

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=a0yark/deepseek2api&type=Date)](https://star-history.com/#a0yark/deepseek2api&Date)

## 鸣谢

* 上游原仓库 [iidamie/deepseek2api](https://github.com/iidamie/deepseek2api) — 本仓库 fork 自此并基于实测重写 V4 适配
* [LLM-Red-Team/deepseek-free-api](https://github.com/LLM-Red-Team/deepseek-free-api) — 上游参考的部分实现
* [musistudio/claude-code-router](https://github.com/musistudio/claude-code-router) — 用于把本服务暴露成 Anthropic 兼容端点
