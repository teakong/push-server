# push-skill

一个 [Agent Skill](https://github.com/vercel-labs/agent-skills)，通过 [一封传话](https://push.phprm.com/mcp.html) 的 `pushServer` MCP 服务器，让 AI 助手（Trae、Claude、Cursor、OpenClaw 等）主动向您汇报任务成果或发送重要通知。

- 纯 MCP 调用，无需脚本与任何运行时依赖。
- 支持 Markdown 正文与点击跳转链接，推送到您的浏览器、飞书、钉钉、企业微信、邮件等通知通道。
- 内置发送前 URL 反引号清洗规则，保证接收端链接可点、图片不裂。

## 安装

通过 [`skills`](https://www.npmjs.com/package/skills) CLI 安装：

```bash
# 安装到当前项目
npx skills add teakong/push-skill

# 或安装到全局（所有 agent 可用）
npx skills add teakong/push-skill -g
```

## 配置

使用前，先在 AI 助手中添加远程 MCP 服务器：

- **名称**：`pushServer`
- **URL**：`https://www.phprm.com/services/push/mcp`
- **Headers**：
  - `X-Push-Channel-Code`：您的 32 位通道码（必须配置，否则无法接收推送）
  - `X-Mcp-Source`：客户端标识（可选，仅用于服务端日志区分来源，可填 `trae`、`cursor`、`claude-code` 等任意值）

通道码在 [一封传话](https://push.phprm.com/mcp.html) 注册账号/创建推送通道后获取。

> ⚠️ 通道码等同推送凭证，请勿提交进代码库或打印到日志。

## 用法

安装并配置好 MCP 后，直接让 agent「任务完成后给我推送一条通知」即可。底层调用 `pushServer` MCP 提供的 `send_push_message` 工具：

| 字段 | 必填 | 说明 |
|------|------|------|
| `head` | 是 | 消息标题，纯文本，200 字符以内 |
| `body` | 否 | 正文，仅支持 Markdown（不支持 HTML），50,000 字符以内 |
| `url` | 否 | 点击跳转链接（如 PR 地址、构建日志），500 字符以内 |
| `channelCode` | 否 | 指定通道码，覆盖 Header 中的 `X-Push-Channel-Code` |

简单任务可以只提供 `head`：

```json
{
  "head": "构建已完成"
}
```

任务汇报示例：

```json
{
  "head": "代码优化完成",
  "body": "**优化清单**\n- **性能**: 减少了冗余的会话查找\n- **稳定性**: 增加了异常捕获",
  "url": "https://example.com/pr/123"
}
```

## 工作原理

1. Agent 完成任务后，按 [SKILL.md](SKILL.md) 规范将成果总结为 `head` / Markdown `body`。
2. 发送前对 `body`、`url` 执行 URL 反引号清洗，避免接收端 showdown 转换后链接失效。
3. 通过 MCP 协议调用 `send_push_message`，由一封传话服务端投递到通道。
4. 服务端返回 `{"code":0, ...}` 且包含 `messageIdList` 即发送成功；非 `code:0` 时 Agent 必须如实告知失败原因，不得降级为 HTTP 直调或谎报成功。

## License

MIT
