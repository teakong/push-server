# push-skill

一个 [Agent Skill](https://github.com/vercel-labs/agent-skills)，通过 [一封传话](https://push.phprm.com/mcp.html) 的 `push-server` MCP 服务器，让 AI 助手（Trae、Claude、Workbuddy、OpenClaw 等）主动向您汇报任务成果或发送重要通知。

- 纯 MCP 调用，无需脚本与任何运行时依赖。
- 支持 Markdown 正文与点击跳转链接，推送到您的浏览器、飞书、钉钉、企业微信、邮件等通知通道。
- 支持 `body` 传 JSON：把 AI 采集到的内容序列化成 JSON 文本放进 `body`，经 Webhook 类型的通道原样转发到您自己的业务服务接口（自建接收端），不参与 Markdown 渲染。
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

**安装 skill ≠ 配置通道**，两者互相独立，必须都完成才能推送：

1. 在 [一封传话](https://push.phprm.com/mcp.html) 注册账号并创建推送通道，获取 32 位通道码。
2. 在 AI 客户端添加远程 MCP 服务器。以 Trae CN 为例：AI 侧边对话框右上角 **设置 → MCP → + 添加 → 手动添加**，选择 Streamable HTTP 类型，粘贴下面的 JSON 并替换通道码：

```json
{
  "mcpServers": {
    "push-server": {
      "url": "https://www.phprm.com/services/push/mcp",
      "headers": {
        "X-Push-Channel-Code": "换成你自己的32位通道码",
        "X-Mcp-Source": "trae"
      }
    }
  }
}
```

字段说明：

- **名称**：`push-server`（固定，必须与 skill 中调用的服务器名一致）
- **URL**：`https://www.phprm.com/services/push/mcp`
- **Headers**：
  - `X-Push-Channel-Code`：您的 32 位通道码（必须配置，否则无法接收推送）
  - `X-Mcp-Source`：客户端标识（可选，仅用于服务端日志区分来源，可填 `trae`、`cursor`、`claude-code` 等任意值）

Claude Code、Cursor 等其他客户端同理，在各自的 MCP 配置中添加同名 HTTP 服务器即可。配置保存在本地客户端，不会进入任何代码仓库。

> ⚠️ 通道码等同推送凭证，请勿提交进代码库或打印到日志。

### 在豆包网页版使用

豆包网页版不支持 `npx skills` 安装技能，但电脑版支持自定义 HTTP 连接器，分两步接入：

1. **添加连接器**：左侧导航「技能·连接器·伙伴」→ 右上角「新建」→「新建自定义连接器」，填写：
   - 服务器名称：`push-server`
   - 传输类型：`HTTP`
   - 服务器 URL：`https://www.phprm.com/services/push/mcp`
   - 自定义 Headers：
     - `X-Push-Channel-Code`：你的 32 位通道码
     - `X-Mcp-Source`：`doubao`
2. **新建专属智能体**：创建一个智能体，把 [SKILL.md](SKILL.md) 的正文（去掉开头 `---` frontmatter）粘贴到「人设与回复逻辑/指令」中，并在技能里勾选刚建的 `push-server` 连接器。之后在该智能体对话中完成任务，它即会按本技能规范主动推送。

> 通道码只填在连接器 Headers 中，不要写进智能体的公开指令。不建智能体也能在对话中直接要求「用 push-server 推送通知」，但规范遵守不如专属智能体稳定。

## 用法

安装并配置好 MCP 后，直接让 agent「任务完成后给我推送一条通知」即可。底层调用 `push-server` MCP 提供的 `send_push_message` 工具：

| 字段 | 必填 | 说明 |
|------|------|------|
| `head` | 是 | 消息标题，纯文本，200 字符以内 |
| `body` | 否 | 正文，支持 Markdown/JSON（不支持 HTML），50,000 字符以内；JSON 用于 Webhook 通道推送，接收端按原文解析 |
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

采集内容推送到业务服务（Webhook 通道，`body` 为 JSON 文本）：

```json
{
  "head": "采集结果已就绪",
  "body": "{\"title\":\"xxx\",\"items\":[{\"id\":1,\"name\":\"示例\"}]}"
}
```

## 工作原理

1. Agent 完成任务后，按 [SKILL.md](SKILL.md) 规范将成果总结为 `head` / Markdown `body`。
2. 发送前对 `body`、`url` 执行 URL 反引号清洗，避免接收端 showdown 转换后链接失效。
3. 通过 MCP 协议调用 `send_push_message`，由一封传话服务端投递到通道。
4. 服务端返回 `{"code":0, ...}` 且包含 `messageIdList` 即发送成功；非 `code:0` 时 Agent 必须如实告知失败原因，不得降级为 HTTP 直调或谎报成功。

## License

MIT
