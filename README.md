# DingTalk Stream + FastGPT

无页面的钉钉 Stream 中间件：

```text
钉钉 Stream -> bridge.py -> FastGPT SSE -> 钉钉 AI 卡片或 Markdown 消息
```

Stream 模式通过 WebSocket 长连接接收消息，不需要公网 IP、公网域名、HTTPS 回调地址或内网穿透。

## 安装

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

配置 `.env`：

```dotenv
DINGTALK_APP_KEY=钉钉企业内部应用AppKey
DINGTALK_APP_SECRET=钉钉企业内部应用AppSecret
FASTGPT_BASE_URL=http://127.0.0.1:3000
FASTGPT_API_KEY=FastGPT应用APIKey
FASTGPT_APP_ID=FastGPT应用ID
FASTGPT_TIMEOUT=180
FASTGPT_CHAT_ID_MODE=user
FASTGPT_DETAIL=false
DINGTALK_REACTION_EMOJI=🤔Thinking
DINGTALK_DONE_EMOJI=✅Done
DINGTALK_CARD_TEMPLATE_ID=
DINGTALK_CARD_TEMPLATE_KEY=content
DINGTALK_CARD_ENABLED=true
DINGTALK_STATE_DB=./dingtalk-state.sqlite3
```

`FASTGPT_BASE_URL` 填 FastGPT 根地址，不要带 `/api/v1`。`FASTGPT_APP_ID` 必须填写目标应用的 ID，通常可以从 FastGPT 应用页面 URL 中找到。
`FASTGPT_CHAT_ID_MODE=user` 表示每个钉钉用户使用独立 FastGPT 上下文；群里多人共用上下文时才改成 `conversation`。
`bridge.py` 从 FastGPT SSE 中只读取用户可见的 `delta.content`，不会把 `reasoning_content` 发往钉钉。无需卡片模板时，它在生成结束后发送一条 Markdown 回复；钉钉不支持的 Markdown 表格会转成字段列表。

要让同一张消息卡片边生成边更新，请先在钉钉开放平台创建 AI 卡片模板，并将它的模板 ID 填入 `DINGTALK_CARD_TEMPLATE_ID`。模板需有名为 `content` 的文本变量；变量名不同可通过 `DINGTALK_CARD_TEMPLATE_KEY` 指定。钉钉应用还需开通 `Card.Streaming.Write` 权限。程序会在每轮回复时自动创建卡片并维护 `outTrackId`，无需提前提供卡片实例 ID。卡片创建失败时会回退到一条最终 Markdown 回复。设 `DINGTALK_CARD_ENABLED=false` 可暂时关闭卡片而保留模板配置。

`DINGTALK_REACTION_EMOJI` 和 `DINGTALK_DONE_EMOJI` 控制原消息上的处理状态表情；设为 `none` 可分别关闭。排查知识库命中问题时，可以临时设置 `FASTGPT_DETAIL=true`。

机器人支持 `/new`、`/reset`、`/clear`、`/help`、`/status`、`/compact`、`/model` 和 `/usage`；完整行为、持久化要求及多应用配置见 [钉钉命令](./COMMANDS.md)。

## 钉钉后台

1. 创建或打开钉钉企业内部应用。
2. 开启机器人能力。
3. 消息接收模式选择 **Stream 模式**。
4. 配置机器人收消息和发消息权限；使用流式 AI 卡片时还需 `Card.Streaming.Write`。
5. 发布应用版本。

不需要填写回调 URL。

## 启动

```bash
python3 bridge.py
```

看到下面的日志，就会开始等待钉钉消息：

```text
DingTalk Stream bridge started; AI card enabled=True
```

机器人收到问题后，中间件会调用：

```text
POST {FASTGPT_BASE_URL}/api/v1/chat/completions
```

并使用钉钉用户或会话 ID 作为 FastGPT `chatId`。

运行局部测试：

```bash
python3 -m unittest test_bridge -v
```

原有 `dingtalk_fastgpt.py` 入口仍可运行，使用一次性文字回复；新功能由 `bridge.py` 提供。
