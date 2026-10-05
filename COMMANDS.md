# 钉钉命令

启动 `python3 bridge.py` 后，中转服务会在钉钉侧直接处理以下命令，不将命令文本交给 FastGPT：

| 命令 | 功能 |
| --- | --- |
| `/new`、`/reset`、`/clear` | 生成新的 FastGPT `chatId`，后续提问从空上下文开始。旧聊天记录仍保留在 FastGPT。 |
| `/help` | 列出可用命令。 |
| `/status` | 查看当前应用配置、`chatId` 和本地统计的轮次。 |
| `/compact` | 让 FastGPT 总结当前会话，生成新 `chatId`，并将摘要带入新会话的下一次成功提问。 |
| `/model` | 查看已配置的 FastGPT 应用配置。 |
| `/model 名称` | 切换到该应用配置，同时开始新会话。 |
| `/usage` | 查看当前会话由中转服务统计的轮次和输入/输出字符数。 |

原有 `reset`、`重置`、`清空上下文`、`重新开始` 仍可重置会话。未知斜杠命令只返回帮助提示。`/compact` 生成的摘要请求会留在旧 FastGPT 会话记录中；如果请求失败，当前会话不会切换。

## 状态保存

中转服务使用 SQLite 保存当前 `chatId`、应用配置、统计与待注入的摘要。默认文件为运行目录中的 `dingtalk-state.sqlite3`；可以设置 `DINGTALK_STATE_DB` 更改位置。容器部署时应将此文件放到持久化卷，例如设置 `DINGTALK_STATE_DB=/data/dingtalk-state.sqlite3` 并挂载 `/data`。状态库含会话摘要，应按聊天记录保护。升级前已有的钉钉会话会继续使用原 `chatId`，首次执行重置后改用新 ID。

## 应用配置

默认应用继续使用 `FASTGPT_APP_ID` 和 `FASTGPT_API_KEY`。可选环境变量 `DINGTALK_MODEL_PROFILES` 是 JSON 对象，例如：

```dotenv
DINGTALK_MODEL_PROFILES={"fast":{"app_id":"<应用 ID>","api_key":"<应用 API Key>"}}
```

`/model` 只切换已经配置的 FastGPT 应用，不会修改应用工作流里的模型节点。`/usage` 不显示 token 和费用，因为当前中转服务没有可靠的计费数据来源；统计只覆盖新版中转服务处理成功的回答。
