# push-server

一个 [Agent Skill](https://github.com/vercel-labs/agent-skills)，通过 [一封传话](https://push.phprm.com/mcp.html) 的 `push-server` MCP 服务器，让 AI 助手（Trae、Claude、Workbuddy、OpenClaw 等）主动向您汇报任务成果或发送重要通知。

- 纯 MCP 调用，无需脚本与任何运行时依赖。
- 支持 Markdown 正文与点击跳转链接，推送到您的浏览器、IM 客户端、Webhook 等通知通道。
- 支持 `send_multi_message` 一次调用推送多条消息（1~32 条，每条可指定自己的通道码），适合群发与多步骤汇报。
- 支持 `body` 传 JSON：把 AI 采集到的内容序列化成 JSON 文本放进 `body`，经 Webhook 类型的通道原样转发到您自己的业务服务接口（自建接收端），不参与 Markdown 渲染。
- 支持查询已推送消息：先用 `get_access_token` 换短期令牌，再调用 `/oauth2/push/message/*` 查分页与详情。
- 支持通道管理：用同一令牌调用 `/oauth2/push/channel/*`，查看下属通道列表、管理成员、**新增/修改/删除/重置通道**（`channel/edit` 可用 `status` 传 `1` 启用 / `0` 停用）；**相同类型（`pushType`）的通道只建一条、够用就复用**，不要把同一类推送拆成多条。成员侧只开放 1 个接口（改昵称），其余成员功能（删除成员、成员列表分页、拉黑/恢复、邀请加入、批量处理）请在官网控制台完成。
- 内置发送前 URL 反引号清洗规则，保证接收端链接可点、图片不裂。

## 安装

通过 [`skills`](https://www.npmjs.com/package/skills) CLI 安装：

```bash
# 安装到当前项目
npx skills add teakong/push-server

# 或安装到全局（所有 agent 可用）
npx skills add teakong/push-server -g
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
  - `X-Mcp-Source`：客户端标识（可选，仅用于服务端日志区分来源，可填 `trae`、`workbuddy`、`coze` 等任意值，按自己的客户端改）
- **传输类型**：Streamable HTTP（不是 SSE）。部分客户端要求 JSON 里显式声明 `type`，两种写法按客户端二选一、不要同时写：`"type": "streamable-http"`（Trae / Cursor / Cline 等多数客户端）或 `"type": "streamableHttp"`（WorkBuddy 等驼峰写法）；不认该字段的客户端整行删掉即可。

Claude Code、Cursor 等其他客户端同理，在各自的 MCP 配置中添加同名 HTTP 服务器即可。配置保存在本地客户端，不会进入任何代码仓库。

> ⚠️ 通道码等同推送凭证，请勿提交进代码库或打印到日志。

### 凭证分工（务必区分）

| 场景 | 入口 | 凭证 |
|---|---|---|
| 推送单条 | MCP `send_push_message` | Header `X-Push-Channel-Code` |
| 推送多条 | MCP `send_multi_message` | Header `X-Push-Channel-Code` |
| 换取令牌 | MCP `get_access_token` | Header `X-Push-Channel-Code` |
| 查询消息等业务 API | HTTP `https://www.phprm.com/oauth2/push/*` | `Authorization: Bearer <access_token>` |
| 健康自检 | HTTP `https://www.phprm.com/services/public/ping` | 无需凭证 |

通道码不能当 Bearer 用，`access_token` 也不能拿去推送。

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

安装并配置好 MCP 后，直接让 agent「任务完成后给我推送一条通知」即可。服务端提供三个工具，按用途分工：

| 工具 | 用途 |
|------|------|
| `send_push_message` | 推送一条消息，绝大多数场景用它 |
| `send_multi_message` | 一次调用推送多条（1~32），每条可指定自己的通道码 |
| `get_access_token` | 用通道码换短期 `access_token`，供 `/oauth2/push/*` 业务接口使用 |

> 三个工具的 `channelCode` 在远端 schema 里都标了必填，但服务端允许 Header 已预置时省略：Header 未配置时必须显式传；Header 已配置、客户端却仍报缺参时，把同一串通道码显式填进 `channelCode` 即可（值与 Header 一致，不要改配置或换工具）。

**推送单条 —— `send_push_message`**

| 字段 | 必填 | 说明 |
|------|------|------|
| `head` | 是 | 消息标题，纯文本，200 字符以内 |
| `body` | 否 | 正文，支持 Markdown/JSON（不支持 HTML），50,000 字符以内；JSON 用于 Webhook 通道推送，接收端按原文解析 |
| `url` | 否 | 点击跳转链接（如 PR 地址、构建日志），500 字符以内 |
| `channelCode` | 条件必填 | 指定通道码，覆盖 Header 中的 `X-Push-Channel-Code`；Header 预置时服务端可省，但远端 schema 标了必填（见下方提示） |

**一次推送多条 —— `send_multi_message`**

| 字段 | 必填 | 说明 |
|------|------|------|
| `messages` | 是 | 数组，1~32 项，每项 `{ head, body, url, channelCode }` |
| `messages[].head` | 是 | 逐项必填，规则同单条的 `head` |
| `messages[].channelCode` | 否 | 该条单独指定通道，覆盖工具级 `channelCode` / Header 通道码 |
| `channelCode` | 条件必填 | 工具级兜底通道码，作为所有未单独指定项的默认通道；不传则用 Header（同样 schema 标必填、Header 预置时可省） |

多条并发推送，**任一条失败则整次调用失败**并返回第一条失败原因；超过 32 条需拆成多次调用。

**换取令牌 —— `get_access_token`**

| 字段 | 必填 | 说明 |
|------|------|------|
| `channelCode` | 条件必填 | 要换令牌的通道码；不传则取 Header `X-Push-Channel-Code`（schema 同样标了必填） |
| `scope` | 否 | 必须是该通道客户端已登记 scope 的子集；不传则使用服务端默认值 `basic` |

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

一次推送多条（不同内容 / 不同通道）：

```json
{
  "channelCode": "换成你自己的32位通道码",
  "messages": [
    { "head": "构建已完成", "body": "主分支构建 #128 通过", "url": "https://example.com/build/128" },
    { "head": "部署已开始", "body": "预发环境正在部署", "channelCode": "换成你自己的另一个32位通道码" }
  ]
}
```

## 回查已推送的消息

1. 调用 MCP `get_access_token` 换取短期 `access_token`（`scope` 必须是该通道客户端已登记范围的子集，不传则用服务端默认值 `basic`；`channelCode` 不传则取 Header `X-Push-Channel-Code`）。
2. 用它请求 OAuth2 业务接口：

```bash
curl --location "https://www.phprm.com/oauth2/push/message/page?page=1&limit=10" \
  --header "Accept: application/json" \
  --header "Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}"
```

消息详情只需传 `messageId`（来自分页返回的 `rows`）。完整的参数、响应字段与错误码见 [references/api.md](references/api.md)。

## 管理通道

同一个令牌还能管理当前凭证所属的通道：

```bash
# 下属通道列表（不分页；可用 channelName 模糊匹配、pushTypeName 筛选）
curl --location "https://www.phprm.com/oauth2/push/channel/list" \
  --header "Accept: application/json" \
  --header "Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}"
```

返回的 `data.rows[].channelCode` 可直接当新的 MCP 推送凭证；同一份响应里 `pushType=1`（浏览器）那几行还带 `channelMemberRelId`，即**该子通道创建人的成员 ID**，是改昵称这个唯一成员接口的**入参来源**（没有成员分页接口，也没有删除成员接口）。此外还有通道新增/修改/删除/重置与改昵称共 5 个接口，参数与约束见 [references/api.md](references/api.md)。其中「新增通道」是最后手段：先在上面的列表里找同 `pushType` 的现有通道复用，确实没有再新建。

> ⚠️ 删除通道不可恢复（服务端禁止删除当前凭证绑定的通道，`${PHPRM_CHANNEL_CODE}` 对应的那条永远别删）；重置通道会**立即作废旧通道码**。
> 重置自己的通道是允许的（这也是轮换凭证的唯一途径），但重置后必须：把新码告诉用户 → 自己尝试替换 MCP 配置里的 `X-Push-Channel-Code` → 提醒重连 → 发一条推送验证。重置别人（同组子通道）不用动 Header。

推送失败时不要先猜原因，用匿名健康端点二分定位：

```bash
curl -s https://www.phprm.com/services/public/ping
```

返回 `{"code":0,"message":"ok","data":{"status":"alive","service":"push-server"}}` 说明端点活着，问题在凭证或客户端配置。

该端点还会带回一小段服务端维护的**公开示例通道**（`rows`）：想先验证链路时，可在用户同意后把它的 `channelCode` 通过工具的 `channelCode` 参数传一次做自测，**不要写进 MCP 配置、也不要拿它收正式通知**。

## 工作原理

1. Agent 完成任务后，按 [SKILL.md](SKILL.md) 规范将成果总结为 `head` / Markdown `body`；多条内容改用 `send_multi_message`。
2. 发送前对每条消息的 `body`、`url` 执行 URL 反引号清洗，避免接收端 showdown 转换后链接失效。
3. 通过 MCP 协议调用 `send_push_message` / `send_multi_message`，由一封传话服务端投递到通道。
4. 服务端返回 `{"code":0, ...}` 且包含 `messageIdList` 即发送成功（多条时是所有消息 ID 的并集）；非 `code:0` 时 Agent 必须如实告知失败原因，不得降级为 HTTP 直调推送或谎报成功。

## License

MIT
