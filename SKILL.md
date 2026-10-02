---
name: push-server
slug: push-server
displayName: 一封传话推送
display_name: 一封传话推送
display_name_en: Aggregated Push
summary: 通过 push server MCP 向用户推送通知、批量推送并用短期令牌回查消息
homepage: https://push.phprm.com/mcp.html
tags: [push, notification, mcp, webhook]
description: Send push notifications to users via the push server MCP, batch-push several messages in one call with send_multi_message, and query pushed messages through OAuth2 HTTP APIs. Use this skill when a task result must be reported to the user, an important Markdown/JSON notification must reach the user's IM client, browser or webhook API, several notifications need one call, or pushed messages need to be listed, opened, or channels managed. (Head + Markdown/JSON body, optional jump link.)
description_zh: 通过 push server MCP 向用户推送消息通知或将抓取内容推送至业务服务器；支持 send_multi_message 一次推送多条（可各带自己的通道码），并可用 OAuth2 HTTP 接口查询历史消息。当需要向用户IM客户端、浏览器或webhook API发送 Markdown/json 格式的重要通知、需要群发多条通知、或需要拉取/查看已推送消息时使用本技能。（标题 + Markdown/json 正文，可选跳转链接）
description_en: Push message notifications to users through the push server MCP, push scraped content to a business server, fan out several messages at once via send_multi_message, and read back message history over OAuth2 HTTP APIs. Use this skill when an important notification in Markdown or JSON format must reach the user's IM client, browser, or webhook API, when several notifications need one call, or when past pushed messages need to be listed or opened. (Head + Markdown/JSON body, optional jump link.)
category: utilities
version: 1.3.1
author: teakong
---

# 一封传话推送

通过 `push-server` MCP 向用户汇报任务成果、向通知通道（IM 客户端、浏览器、webhook API）发送重要通知，把抓取内容以 JSON 推到用户自己的业务服务器，或用短期令牌经 HTTP 接口回查历史消息。

三个 MCP 工具按用途分工，**不要串用**：

| 工具 | 用途 |
|---|---|
| `send_push_message` | 推送一条消息，绝大多数场景用它 |
| `send_multi_message` | 一次调用推送多条（1~32），每条可指定自己的通道码 |
| `get_access_token` | 用同一个通道码换短期 `access_token`，仅供后面的 HTTP 业务接口使用 |

推送工具与 `get_access_token` 都只认 `X-Push-Channel-Code`（长期通道码）；HTTP 业务接口只认 `Authorization: Bearer <access_token>`。**通道码不能当 Bearer 用，token 也不能拿去推送。**

## 何时调用

- 当用户明确要求发送通知或汇报执行结果时。
- 需要把结果同时推送到多个通道（多个通道码）时 → `send_multi_message`。
- 需要把采集到的结构化内容以 JSON 推送到用户自己的服务器（webhook 通道）时。
- 需要列出或打开已推送过消息（分页、详情）时 → 先取 access_token，再用 HTTP 接口。
- 需要管理通道本身时（看下有哪些通道、改成员昵称、新增/修改/删除/重置通道）→ 先取 access_token，再用 HTTP 接口，规则详见 [references/api.md](references/api.md)。

## 随附参考文件（按需读取）

本文件只保留「推送 + 判定 + 红线」。通道管理、消息回查、字段 / 枚举 / 错误码的完整说明都在 [references/api.md](references/api.md)——**只在要做这些事时才读它**，不要预加载。

## 配置说明（用户必读）

**安装 skill ≠ 配置通道**，必须先完成两步，否则无法推送：

1. 在 [一封传话](https://push.phprm.com/mcp.html) 注册账号并创建推送通道，获取 32 位通道码。
2. 在 AI 助手中手动添加远程 MCP 服务器（以 Trae CN 为例：AI 侧边对话框右上角「设置 → MCP → + 添加 → 手动添加」，选择 **Streamable HTTP** 类型），填入以下 JSON 并替换通道码：

```json
{
  "mcpServers": {
    "push-server": {
      "url": "https://www.phprm.com/services/push/mcp",
      "headers": {
        "X-Push-Channel-Code": "${PHPRM_CHANNEL_CODE}",
        "X-Mcp-Source": "trae"
      }
    }
  }
}
```

- **名称**：`push-server`（固定，必须与本技能调用的服务器名一致）
- **URL**：`https://www.phprm.com/services/push/mcp`
- **Headers**：
  - `X-Push-Channel-Code`：32 位通道码（必须配置，否则无法接收推送）
  - `X-Mcp-Source`：客户端标识（可选，仅用于服务端日志区分来源，可填 `trae`、`workbuddy`、`coze` 等任意值，按自己的客户端改）
- **传输类型**：Streamable HTTP（不是 SSE）。部分客户端要求 JSON 里显式声明 `type`，两种写法按客户端二选一、不要同时写：`"type": "streamable-http"`（Trae / Cursor / Cline 等多数客户端）或 `"type": "streamableHttp"`（WorkBuddy 等驼峰写法）；不认该字段的客户端整行删掉即可。

Claude Code、Cursor 等其他客户端同理，在各自的 MCP 配置中添加同名 HTTP 服务器即可。配置保存在用户本地客户端，不会进入任何代码仓库。

## 凭证模型

| 场景 | 入口 | 凭证 |
|---|---|---|
| 推送单条 | MCP `send_push_message` | Header `X-Push-Channel-Code` |
| 推送多条 | MCP `send_multi_message` | Header `X-Push-Channel-Code` |
| 换取令牌 | MCP `get_access_token` | Header `X-Push-Channel-Code` |
| 查询消息等业务 API | HTTP `https://www.phprm.com/oauth2/push/*` | `Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}` |
| 健康自检 | HTTP `https://www.phprm.com/services/public/ping` | 无需凭证 |

- **通道码**：32 位长期凭证，只出现在 MCP 的两个推送工具与 `get_access_token` 的 Header / 入参中；仅在用户在控制台重置后才需要替换。
- **access_token**：短期动态凭证，由 `get_access_token` 签发，只有几分钟有效期；过期重新获取即可，**不要缓存复用**。下文 curl 用 `${CHANNEL_ACCESS_TOKEN}` 代指。

> `/services/public/ping` 除了「活着」还会带回一小段示例通道，见下一节。

## ping 里的示例通道（可借用，但要打招呼）

健康自检 `/services/public/ping` 是**匿名**端点，除了存活结论还会返回一段 `rows`——服务端维护的**公开示例通道**（字段含义同 `channel/list` 的 `rows[]`，响应样例见 [references/api.md](references/api.md) 第 3 节）。列表可能为空。

如需测试推送，**不要**把返回的 `channelCode` 写进 MCP 配置，只允许通过 `send_push_message` / `send_multi_message` 的 `channelCode` 参数传它。

用它的规矩：

- **它是别人的、公开的示范通道**——任何拿到这串码的人都能往里推，所以 `ping` 里出现的码**绝不能当成用户自己的默认通道**，更不要拿去填 `${PHPRM_CHANNEL_CODE}`。
- 唯一允许的用途：**用户明确同意后**用它做一次链路自测（例如用户还没给自己的通道码、但想确认 MCP 是否可用）。自测前说明「这是服务端示例通道，任何人都能往里推」，成功即结束，**不要把正式通知发到这里**。
- 常规推送依然要用**用户本人**的通道码；本节不改动「只能使用用户本人提供的通道码」这条红线。

> 逐个接口对接 / 排错前先看 [references/api.md](references/api.md) 第 0 节（9 个接口的总览与调用约定，尤其是「只认 form / query，JSON body 会被忽略」）和 6.1（按接口预期的失败响应文案）。

## 业务 API（HTTP）

除健康自检外，全部是 `/oauth2/push/*`，流程固定两步：**先用 MCP `get_access_token` 换短期令牌 → 再带 `Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}` 调用**。

调用约定（照错这几条会白试很多次）：

- **参数只认 query 或 `application/x-www-form-urlencoded`**。`application/json` 的 body 会被完全忽略，表现为「XXX 参数缺失或非法」。
- **错误一律 HTTP 200 + 非 0 业务码**，靠 `code` 判定，不看 HTTP 状态码。只有 `401/403` 属于令牌失效，重取一次再试。
- **归属只认 token 的 `sub`**，没有任何接口接受 `userId` / `tenantId`；目标通道一律用 32 位 `channelCode` 。
- **别跟 `/services/push/*` 混用**：那是控制台登录态的另一套接口，参数名与返回值都不同。
- 写操作成功返回 `{"code":0,...,"data":true}`；**两个例外**：`resetChannel` 的 `data` 是新的 32 位通道码字符串，`channel/add` 的 `data` 是新建的通道对象（含它的 `channelCode`，可直接当 MCP 凭证，不必再拉一次列表）。

| 用途 | Method | Path | 参数                                                       | 成功时 `data` |
|---|---|---|----------------------------------------------------------|---|
| 健康自检（匿名） | GET | `/services/public/ping` | 无                                                        | `{status,service,rows}`（`rows` 为示例通道，见「ping 里的示例通道」） |
| 消息分页 | GET | `/oauth2/push/message/page` | `page`、`limit`（缺省 1 / 10）                                | `current`/`total`/`totalPage`/`rows` |
| 消息详情 | GET | `/oauth2/push/message/detail` | `messageId`（取自分页）                                        | 单条消息 + `viewCount` |
| 通道列表 | GET | `/oauth2/push/channel/list` | 可选筛选：`channelName`（模糊）、`pushTypeName`（不分页） | 顶层父通道信息 + `rows` |
| 新增通道 | POST | `/oauth2/push/channel/add` | `channelName` + `pushType` 必填，其余按类型                      | **新建的通道对象**（字段同通道列表 `rows[]`，`channelMemberRelId` 可能为空） |
| 修改通道 | POST | `/oauth2/push/channel/edit` | `channelCode` + 要改的字段，含可选 **`status`（`1` 启用 / `0` 停用）**  | `true` |
| 删除通道 | POST | `/oauth2/push/channel/deleteChannel` | `channelCode`                                            | `true` |
| 重置通道 | POST | `/oauth2/push/channel/resetChannel` | `channelCode`                                            | **新的通道码字符串** |
| 改成员昵称 | POST | `/oauth2/push/channel/member/edit` | `channelMemberRelId` + `nickname`（≤16 字符，空白算缺失） | `true` |

下面这张表只是**路由索引**——字段、枚举、边界一律不在这里展开，通道管理细则只维护在 [references/api.md](references/api.md)，**做通道操作前先读它**：

| 要什么 | 去 api.md |
|---|---|
| 通道列表的字段含义、`channelMemberRelId` 从哪来、「当前通道怎么找」 | 5.1 |
| 新建通道的入参与「别重复建同一类型的通道」 | 5.2 |
| 改通道（含停用 `status=0` / 启用 `status=1`） | 5.3 |
| 删除的边界（当前通道不能删） | 5.4 |
| 重置的边界 + 重置后如何把新码写回 MCP | 5.5 |
| 改成员昵称（成员侧唯一接口） | 5.6 |
| `pushType` 取值表 / `status` 取值（只有 `0` / `1`） | 5.7 / 5.8 |
| 「推送到 xx 通道」用哪个通道码 | 5.9 |
| 各接口的失败文案与处理建议 | 6.1 / 6.2 |

## 执行步骤

1. 确认通道码：`${PHPRM_CHANNEL_CODE}` 占位符未解析（Header 未生效）且用户没给时，先向用户索取其本人的 32 位通道码，**不猜测、不用别人的通道码**。Header 生效时通常不必传 `channelCode`，但若客户端按 schema 严格校验并报缺参，就把同一串码显式传进 `channelCode`（见「工具参数」）。
2. 选工具：一条 → `send_push_message`；多条不同内容或要发到多个通道 → `send_multi_message`（每条可带自己的 `channelCode`）。非常简单的结果可以只给 `head`。
3. 组装每条消息的 `head`（必填，≤200，纯文本）、`body`（可选，Markdown 或 JSON，≤50000）、`url`（可选，≤500）。
4. **发送前清洗（强制）**：接收端 H5 用 showdown 渲染 `body`，URL 外层一旦包了反引号，转换后会污染 `href`/`src`，导致链接点不动、图片裂图。对每条消息的 `body` 和 `url` 依次执行：

   ```
   body = body.replace(/\]\(`(https?:\/\/[^`]+)`\)/g, ']($1)')   // ① 链接语法内: ](`URL`) → ](URL)
   body = body.replace(/`(https?:\/\/[^`\s)]+)`/g, '$1')          // ② 裸 URL 外层反引号: `URL` → URL
   url  = url.replace(/`(https?:\/\/[^`\s]+)`/g, '$1')            // ③ url 字段同样清洗
   ```

   反例：点[「详情」](`https://push.phprm.com/message/x`)。正例：点[「详情」](https://push.phprm.com/message/x)。反引号只用于文件名、命令、代码标识等非 URL 内容。
5. 调用对应工具，**必须调用已注册的 `push-server` MCP 服务器提供的工具**（本 skill 不是 MCP 服务器，不能把 skill 名当 server 调用）。
6. 判定成功：`{"code":0, ...}` 且 `data.messageIdList` 非空即推送成功；`send_multi_message` 返回的是所有消息 ID 的并集，任一条失败即整次调用返回失败。
7. 需要回看已推送内容时：先 `get_access_token`，再按 [references/api.md](references/api.md) 调用消息分页/详情接口。

## 工具参数

三个工具的 `channelCode` 都被远端 schema 列进了 `required`，但服务端描述为「Header 已预置时可省」——**两种语义并存**，严格按 schema 校验的客户端可能在 Header 已生效时仍报缺参。处理办法：

- Header 未生效（没配 `${PHPRM_CHANNEL_CODE}`）：**必须显式传 `channelCode`**。
- Header 已生效但客户端报缺 `channelCode`：把同一串通道码显式填进 `channelCode`（值与 Header 一致），不要为此改配置或换工具。

### send_push_message

| 字段 | 必填 | 说明 |
|---|---|---|
| `head` | 是 | 标题，纯文本，支持 Unicode，≤200 字符 |
| `body` | 否 | 正文，Markdown/JSON（不支持 HTML，勿用 `<br>`、`<table>`）；JSON 用于 webhook 推送到业务服务器，接收方按原文解析、不渲染。≤50000 字符 |
| `url` | 否 | 跳转链接（如 PR 地址、构建日志），≤500 字符 |
| `channelCode` | 条件必填 | 覆盖 Header 中的通道码；不传则用 `X-Push-Channel-Code`。Header 预置时服务端可省，但 schema 标了必填（见本节开头） |

### send_multi_message

| 字段 | 必填 | 说明 |
|---|---|---|
| `messages` | 是 | 数组，1~32 项，每项 `{ head 必填, body, url, channelCode }` |
| `messages[].head` | 是 | 同上，逐项必填 |
| `messages[].channelCode` | 否 | 该项单独指定通道；不传则用工具级 `channelCode` / Header 通道码 |
| `channelCode` | 条件必填 | 工具级通道码，作为所有未单独指定项的默认通道；不传则用 Header（同样 schema 必填、Header 预置时可省） |

多条并发推送，**任一条失败整次即失败**并返回第一条失败原因；超过 32 条需拆成多次调用。

### get_access_token

| 字段 | 必填 | 说明                                                     |
|---|---|--------------------------------------------------------|
| `channelCode` | 条件必填 | 要换令牌的通道码；不传则取 Header `X-Push-Channel-Code`（schema 同样标了必填，见本节开头） |
| `scope` | 否 | 必须是该通道客户端已登记的 scope 的子集；不传则使用服务端默认值 `basic` |

## 安全红线（务必遵守）

- 通道码即推送凭证：只允许出现在 MCP Header、工具入参，以及**给用户的对话回复**里（轮换后必须让用户拿到新码，步骤见 references/api.md 5.5）；不得硬编码进代码/仓库配置并提交 git，也不得写进 `head`/`body`/`url`。
- 推送内容中禁止出现密码、token、私钥、Cookie、完整环境变量等敏感信息原文；需要提及时只做脱敏描述（如「令牌已刷新」而不是令牌值）。
- 只能使用用户本人提供的通道码，禁止向其他通道发送；`send_multi_message` 里每项都要复核目标通道是否确为用户自己的。
- 不得把 `access_token` 写进推送内容或日志；短期令牌同样不外泄。
- **不删除 `${PHPRM_CHANNEL_CODE}` 对应的通道**（服务端也会拒绝）。「删除」只能用于同组里真正确认废弃的子通道，执行前再向用户核对一次 `channelCode` 是否确认删除。
- **重置 `${PHPRM_CHANNEL_CODE}` 通道前先确认用户是要轮换凭证**；重置不可逆，旧码立即失效，且必须按 [references/api.md](references/api.md) 5.5 的 5 步把新码写回 MCP 的 `X-Push-Channel-Code`，漏一步就断推。
- **不重复创建相同类型的推送通道**：同一 `pushType` 尽量复用已有通道，新建前先说明原因并取得用户确认。

## 成功判定与失败处理

- 成功：返回 `{"code":0, ...}` 且含 `messageIdList`。
- MCP 不可用（未配置/未连接）或返回非 `code:0` 时，必须如实告知用户推送失败及原因（未配置通道码、通道码无效、条数超限等）；**不得降级为 HTTP 直调接口推送，更不得谎称已发送**。
- 排查按错误驱动，正常流程不要先 ping：

  ```
  正常流程: 直接调用 send_push_message → 成功即结束
  失败分支: 调用失败(连接层错误) → curl https://www.phprm.com/services/public/ping
            ├─ ping 返回 alive → 端点活着, 问题在凭证/配置: 核对通道码是否替换、Header 名是否一致
            └─ ping 失败      → 端点或网络问题: 核对 URL、网络连通性、传输类型(Streamable HTTP 而非 SSE)
  ```

- ping 与消息查询接口的请求/响应格式见 [references/api.md](references/api.md)。健康自检端点只用于排查，请勿高频轮询。

## 示例

**任务汇报（含跳转链接）**

```json
{
  "head": "代码优化完成",
  "body": "**优化清单**\n- **性能**: 减少了冗余的会话查找\n- **稳定性**: 增加了异常捕获",
  "url": "https://example.com/pr/123"
}
```

**简单通知（仅标题）**

```json
{ "head": "构建已完成" }
```

**抓取内容推送到业务服务器（webhook JSON）**

```json
{
  "head": "采集结果已就绪",
  "body": "{\"title\":\"xxx\",\"items\":[{\"id\":1,\"name\":\"示例\"}]}"
}
```

**一次推送多条（不同内容 / 不同通道）**

```json
{
  "channelCode": "11111111111111111111111111aaaaaa",
  "messages": [
    { "head": "构建已完成", "body": "主分支构建 #128 通过", "url": "https://example.com/build/128" },
    { "head": "部署已开始", "body": "预发环境正在部署", "channelCode": "22222222222222222222222222bbbbbb" }
  ]
}
```

**成功响应**

```json
{"code": 0, "message": "请求成功", "data": {"messageIdList": ["1680030476695388161"]}}
```


