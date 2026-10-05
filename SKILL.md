---
name: push-server
slug: push-server
displayName: 一封传话推送
display_name: 一封传话推送
display_name_en: Aggregated Push
summary: 通过 push server MCP 向用户推送通知（批量采集时可一次推多条webhook）并用短期令牌回查消息
homepage: https://push.phprm.com/mcp.html
tags: [push, notification, mcp, webhook]
license: MIT
description: Send push notifications to users via the push server MCP, push one message per source of a batch collection with send_multi_message, and query pushed messages through OAuth2 HTTP APIs. Use this skill when a task result must be reported to the user, an important Markdown/JSON notification must reach the user's IM client, browser or webhook API, a batch collection must fan out to one message per source, or pushed messages need to be listed, opened, or channels managed. (Head + Markdown/JSON body, optional jump link; emoji formatting rules for human-readable messages.)
description_zh: 通过 push server MCP 向用户推送消息通知或将抓取内容推送至业务服务器；支持 send_multi_message 在批量采集时按来源推送多条（可各带自己的通道码），并可用 OAuth2 HTTP 接口查询历史消息。当需要向用户IM客户端、浏览器或webhook API发送 Markdown/json 格式的重要通知、批量采集需要按来源逐条推送、或需要拉取/查看已推送消息时使用本技能。（标题 + Markdown/json 正文，可选跳转链接；人读消息用 emoji 提升扫读效率）
description_en: Push message notifications to users through the push server MCP, push scraped content to a business server, fan out several messages of a batch collection via send_multi_message, and read back message history over OAuth2 HTTP APIs. Use this skill when an important notification in Markdown or JSON format must reach the user's IM client, browser, or webhook API, when a batch collection needs one message per source, or when past pushed messages need to be listed or opened. (Head + Markdown/JSON body, optional jump link; emoji formatting rules for human-readable messages.)
category: utilities
version: 1.5.4
author: teakong
---

# 一封传话推送

> **优先级总则（最高）**：本文件里的所有规范——emoji 用法、单条还是多条、Markdown 与 JSON 的选择、body 结构、乃至全部示例——都只是**用户没表态时的缺省基线**。用户一旦明确说出自己的推送偏好（如「不要 emoji」「分开推送」「就要表格」「标题不用 emoji，正文必须有」「用 Node 风格一点」），**一律以用户偏好为准**，本文件中与之冲突的规则自动让位；用户没提的部分仍按缺省执行。同一条消息里多条偏好冲突时，向用户确认一句，不要自作主张合并。

通过 `push-server` MCP 向用户汇报任务成果、向通知通道（IM 客户端、浏览器、webhook API）发送重要通知，把抓取内容以 JSON 推到用户自己的业务服务器，或用短期令牌经 HTTP 接口回查历史消息。

三个 MCP 工具按用途分工，**不要串用**：

| 工具 | 用途 |
|---|---|
| `send_push_message` | 推送一条消息，绝大多数场景用它 |
| `send_multi_message` | **仅批量采集类**一次调用发完多条（1~32 条），每条可指定自己的通道码。表格 / 报告场景不要用 |
| `get_access_token` | 用同一个通道码换短期 `access_token`，仅供后面的 HTTP 业务接口使用 |

推送工具与 `get_access_token` 都只认 `X-Push-Channel-Code`（长期通道码）；HTTP 业务接口只认 `Authorization: Bearer <access_token>`。**通道码不能当 Bearer 用，令牌也不能拿去推送。**

## 何时调用

- 当用户明确要求发送通知或汇报执行结果时。
- 用户**明确要求分开推**时（「分别推送」「分开推送」「每个 X 单独推一条」「一条一条发」「这个单独发我」）→ `send_multi_message`，照做。**不要替用户合并成一条。**
- **批量采集**时（一批并列结果每条各推一条，如「批量采集这 20 个页面」），或需要把结果同时推送到不同通道（不同通道码）时 → `send_multi_message`。
- 用户要求「整理成表格」「做成报告」「汇总给我」时 → **单条 `send_push_message`，body 写 Markdown 表格**。表格是用来横向对比的，拆成多条就废了。
- 需要把采集到的结构化内容以 JSON 推送到用户自己的服务器（webhook 通道）时。
- 需要列出或打开已推送过消息（分页、详情）时 → 先取令牌，再用 HTTP 接口。
- 需要管理通道本身时（看下有哪些通道、改成员昵称、新增/修改/删除/重置通道）→ 先取令牌，再用 HTTP 接口，规则详见 [references/api.md](references/api.md)。

## 随附参考文件（按需读取）

本文件只保留「推送 + 判定 + 红线 + 一个最小示例」。其余内容分裂成两份参考，**只在要做这些事时才读，不要预加载**：

| 要什么 | 读哪个 |
|---|---|
| 要把采集到的**结构化数据落到你自己的业务系统**（商品库 / 线索库 / 工单 / 时序指标，`body` = JSON，零 emoji） | [references/example.md](references/example.md) **第 1 节** |
| 组装推送内容时找场景参考（签到、采集、监控告警、CI、订单、行情、安全事件、日报周报等 11 类） | [references/example.md](references/example.md) 第 **2~12** 节（第 13 节是反例对照） |
| 通道管理、消息回查、字段 / 枚举 / 错误码的完整说明 | [references/api.md](references/api.md) |

**组装内容前先看 example.md 里最接近的场景照着改**，比从零写快，也不容易漏掉「head 结论先行」和「📌 下一步」。

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

> **配好了 ≠ 能推送，连上了也不等于凭证生效。** MCP 端点是**匿名可连接**的：即使完全没配 `X-Push-Channel-Code`（或占位符没被客户端解析），客户端照样能连上并列出三个工具。**工具列表出现不能作为凭证有效的证据**，只有真正推送成功才算。这正是「MCP 显示已连接、工具也出来了，一推就报通道码缺失 / 无效」的来源——排查时直接核对 Header 名与通道码是否被真正替换，不要去反复重连服务器。

## 凭证模型

| 场景 | 入口 | 凭证 |
|---|---|---|
| 推送单条 | MCP `send_push_message` | Header `X-Push-Channel-Code` |
| 推送多条 | MCP `send_multi_message` | Header `X-Push-Channel-Code` |
| 换取令牌 | MCP `get_access_token` | Header `X-Push-Channel-Code` |
| 查询消息等业务 API | HTTP `https://www.phprm.com/oauth2/push/*` | `Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}` |
| 健康自检 | HTTP `https://www.phprm.com/services/public/ping` | 无需凭证 |

- **通道码**：32 位长期凭证，只出现在 MCP 的两个推送工具与 `get_access_token` 的 Header / 入参中；仅在用户在控制台重置后才需要替换。
- **`access_token`**：短期动态凭证，由 `get_access_token` 签发，有效期 `expires_in` 秒（服务端默认 **7200 秒 = 2 小时**）。下文 curl 用 `${CHANNEL_ACCESS_TOKEN}` 代指。它的本体是一段 **RS256 签名的 JWT**（`eyJ` 开头的三段式）；直接当不透明字符串用即可，别去校验前缀、长度或自己验签。

**令牌应该在一次任务里复用，不要每条接口都重取**：同一个 `access_token` 可以连续调用任意多个 `/oauth2/push/*` 接口（9 个接口都认同一份 Bearer），重复调用 `get_access_token` 也不会让之前那张失效。所以一轮「取令牌 → 查分页 → 查详情 → 改通道」的任务里，只在开头取一次。

- **按 `expires_in` 而不是凭感觉缓存**：用它可以直用到过期前；稳妥做法是留 60~120 秒余量，快到期了再重取。
- **别把它持久化**：跨会话复用旧令牌会撞上说不清的 401（人和控制台随时可能轮换凭证）。会话内记住即可，不要写进配置文件、长期记忆或推送内容。
- **401 / 403 才是失效信号**：业务错误（`code` 非 0）跟令牌无关，别看到报错就去重取令牌。
- **有一种 401 假象**：`access_token不正确请重新获取` 这条文案也会在「当前令牌主体不是浏览器子通道」时出现，重取一次仍然一样，属于通道结构问题而非过期，别循环重试。

> `/services/public/ping` 除了「活着」还会带回一小段示例通道，见下一节。

## ping 里的示例通道（可借用，但要打招呼）

健康自检 `/services/public/ping` 是**匿名**端点，除了存活结论还会返回一段 `rows`——服务端维护的**公开示例通道**（字段含义同 `channel/list` 的 `rows[]`，同样带 `pushUrl` / `qrCodeUrl` / `channelMemberRelId`；响应样例见 [references/api.md](references/api.md) 第 3 节）。列表可能为空。

其中两个地址字段可直接用：`pushUrl` 用浏览器打开就是**该通道的分享/管理页面**（share.html）；`qrCodeUrl` 是**推送记录页面**的二维码，扫码即可在手机上查看推送记录。想给用户一个"在哪看消息"的入口时优先用这两个字段，不要自己拼 URL。

如需测试推送，**不要**把返回的 `channelCode` 写进 MCP 配置，只允许通过 `send_push_message` / `send_multi_message` 的 `channelCode` 参数传它。

用它的规矩：

- **它是别人的、公开的示范通道**——任何拿到这串码的人都能往里推，所以 `ping` 里出现的码**绝不能当成用户自己的默认通道**，更不要拿去填 `${PHPRM_CHANNEL_CODE}`。
- 唯一允许的用途：**用户明确同意后**用它做一次链路自测（例如用户还没给自己的通道码、但想确认 MCP 是否可用）。自测前说明「这是服务端示例通道，任何人都能往里推」，成功即结束，**不要把正式通知发到这里**。
- 常规推送依然要用**用户本人**的通道码；本节不改动「只能使用用户本人提供的通道码」这条红线。

> 逐个接口对接 / 排错前先看 [references/api.md](references/api.md) 第 0 节（9 个接口的总览与调用约定，尤其是「只认 form / query，JSON body 会被忽略」）和 6.1（按接口预期的失败响应文案）。

## 业务 API（HTTP）

除健康自检外，全部是 `/oauth2/push/*`，流程两步：**先用 MCP `get_access_token` 换短期令牌 → 再带 `Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}` 调用**。注意**换令牌这一步一轮任务只做一次**（原因与复用规则见上一节），后续接口共用同一张令牌。

调用约定（照错这几条会白试很多次）：

- **参数只认 query 或 `application/x-www-form-urlencoded`**。`application/json` 的 body 会被完全忽略，表现为「XXX 参数缺失或非法」。
- **错误一律 HTTP 200 + 非 0 业务码**，靠 `code` 判定，不看 HTTP 状态码。只有 `401/403` 属于令牌失效，重取一次再试。
- **归属由令牌自身决定**，没有接口额外接受归属 / 租户类入参；目标通道一律用 32 位 `channelCode`。
- **别跟 `/services/push/*` 混用**：那是控制台登录态的另一套接口，参数名与返回值都不同。
- 写操作成功返回 `{"code":0,...,"data":true}`；**两个例外**：`resetChannel` 的 `data` 是新的 32 位通道码字符串，`channel/add` 的 `data` 是新建的通道对象（含它的 `channelCode`，可直接当 MCP 凭证，不必再拉一次列表）。

| 用途 | Method | Path | 参数                                                   | 成功时 `data` |
|---|---|---|------------------------------------------------------|---|
| 健康自检（匿名） | GET | `/services/public/ping` | 无                                                    | `{status,service,rows}`（`rows` 为示例通道，见「ping 里的示例通道」） |
| 消息分页 | GET | `/oauth2/push/message/page` | `page`、`limit`（缺省 1 / 10）                            | `current`/`total`/`totalPage`/`rows` |
| 消息详情 | GET | `/oauth2/push/message/detail` | `messageId`（取自分页）                                    | 单条消息 + `viewCount` |
| 通道列表 | GET | `/oauth2/push/channel/list` | 可选筛选：`channelName`（模糊）、`pushTypeName`（推送方式）          | 顶层父通道信息 + `rows`（浏览器子通道另带 `pushUrl` / `qrCodeUrl`） |
| 新增通道 | POST | `/oauth2/push/channel/add` | `channelName` + `pushType` 必填，`webhookUrl` / `signSecret` 按类型可选      | **新建的通道对象**（字段同通道列表 `rows[]`，仅浏览器类型带 `channelMemberRelId` / `nickname`） |
| 修改通道 | POST | `/oauth2/push/channel/edit` | `channelCode` + 要改的字段，含可选 **`status`（`1` 启用 / `0` 停用）** | `true` |
| 删除通道 | POST | `/oauth2/push/channel/deleteChannel` | `channelCode`                                        | `true` |
| 重置通道 | POST | `/oauth2/push/channel/resetChannel` | `channelCode`                                        | **新的通道码字符串** |
| 改成员昵称 | POST | `/oauth2/push/channel/member/edit` | `channelMemberRelId` + `nickname`（≤16 字符，空白算缺失）      | `true` |

下面这张表只是**路由索引**——字段、枚举、边界一律不在这里展开，通道管理细则只维护在 [references/api.md](references/api.md)，**做通道操作前先读它**：

| 要什么 | 去 api.md |
|---|---|
| 通道列表的字段含义、「当前通道怎么找」 | 5.1 |
| 新建通道的入参与「别重复建同一类型的通道」 | 5.2 |
| 改通道（含停用 `status=0` / 启用 `status=1`） | 5.3 |
| 删除的边界（当前通道不能删） | 5.4 |
| 重置的边界 + 重置后如何把新码写回 MCP | 5.5 |
| 改成员昵称（成员侧唯一接口） | 5.6 |
| `pushType` 取值表 / `status` 取值（只有 `0` / `1`） | 5.7 / 5.8 |
| 「推送到 xx 通道」用哪个通道码 | 5.9 |
| 各接口的失败文案与处理建议 | 6.1 / 6.2 |

## 执行步骤

1. 确认通道码：`${PHPRM_CHANNEL_CODE}` 占位符未解析（Header 未生效）且用户没给时，先向用户索取其本人的 32 位通道码，**不猜测、不用别人的通道码**。拿到后**默认显式传进 `channelCode`**——远端 schema 把 `channelCode` 列进了 `required`，无条件传可以避免客户端的严格校验卡住这一步（详见「工具参数」）。
2. 选工具：**默认单条 `send_push_message`**。升级为 `send_multi_message` 只有三种情况：**用户明确要求分开推**（「分别推送」「每个 X 单独一条」）、**批量采集**（并列结果每条各推一条）、**要发到不同通道**（每条可带自己的 `channelCode`）。反过来，**用户要表格 / 报告时不要拆成多条**，直接在单条 body 里写 Markdown 表格。**用户的显式措辞优先于这些缺省判断。**非常简单的结果可以只给 `head`。
3. 组装每条消息的 `head`（必填，≤200，纯文本）、`body`（可选，Markdown 或 JSON，≤50000）、`url`（可选，≤500）。**先分岔**：数据要落到用户的业务服务器（webhook）→ `body` 写合法 JSON 且整条零 emoji，字段名对齐对方表结构、带上批次号与采集时间；其余给人看的 → head 前置一个结论 emoji、body 按状态灯与结果符排版，规范见「内容排版：emoji 可读化」。
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

三个工具的 `channelCode` 都被远端 schema 列进了 `required`，但服务端描述为「Header 已预置时可省」——**两种语义并存**，严格按 schema 校验的客户端可能在 Header 已生效时仍报缺参。**稳妥做法：始终显式传 `channelCode`**（值与 Header 同源），一次写死就不用再拿模式去猜：

- Header 未生效（没配 `${PHPRM_CHANNEL_CODE}`）：**必须显式传 `channelCode`**。
- Header 已生效但客户端报缺 `channelCode`：把同一串通道码显式填进 `channelCode`（值与 Header 一致），不要为此改配置或换工具。

> **`body` 支持 Markdown / 纯文本 / JSON，已是远端 schema 的正式口径**：两个推送工具（单条的 `body`、多条的 `messages[].body`）的 schema 描述都已写明「Accepts Markdown, plain text, or a JSON document」，并注明内容**按字符串原文转发、不在服务端解析或渲染**——webhook 入库时把 JSON 文本塞进 `body` 会被原样投递到接收端，这是结构化数据入库链路的基础。个别尚未同步的实例返回的仍是更早的描述（只写了 Markdown），那只是文案滞后，**JSON 一样能推，别被旧描述拦住**。同理「不支持 HTML」是渲染侧（接收端 showdown）的约束，不是传输侧限制。

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

**用之前先确认是否真的需要多条**——默认应该是单条：

| 用户的要求 | 用哪个 |
|---|---|
| 「分别推送」「分开推送」「一条一条发」「这个单独发我」 | `send_multi_message` ——**用户点了名，照做** |
| 「批量采集这 N 个 X」「每个 X 各推一条」 | `send_multi_message` |
| 同一份内容要发到多个不同通道 | `send_multi_message` |
| 「整理成表格」「做成报告」「汇总给我」「做个对比」 | **单条** `send_push_message`，body 写 Markdown 表格 + 分区标题 |
| 「跑完了，告诉我结果」「帮我推一下」 | **单条** |

用户说「分开 / 分别」时不要替他合并——哪怕你觉得合成一条表格更好看。想优化可以推完之后问一句要不要合并。

**表格 / 报告类是最常见的误用点**：这类内容的价值在于行列之间的横向对比，拆成多条推送后，用户拿到的是 N 条孤立数据，要自己按顺序回忆才能比。除非用户明确说「每个 X 单独推一条」，否则一律走单条 + Markdown 表格。

### get_access_token

| 字段 | 必填 | 说明                                                     |
|---|---|--------------------------------------------------------|
| `channelCode` | 条件必填 | 要换令牌的通道码；不传则取 Header `X-Push-Channel-Code`（schema 同样标了必填，见本节开头） |
| `scope` | 否 | 必须是该通道客户端已登记的 scope 的子集；不传则使用服务端默认值 `basic` |

返回里的 `expires_in` 是**秒数**（默认 7200，即 2 小时），据此决定何时需要重取，不要臆测成几分钟。同一张令牌可以在本次任务里连续调用多个业务接口，详见上一节的复用规则。

## 内容排版：emoji 可读化

纯文本的 `head`/`body` 推到通知栏是一片灰色，跟日志没区别。emoji 的作用是在文字前面放一个**视觉锚点**——人扫一眼先看图形再读文字，判断「好不好、严不严重」的成本从 1 秒降到 0.2 秒。这是排版手段，不是装饰。

> 本节是**缺省偏好**。用户说过「不要 emoji」「只用 ✅❌」「标题别加表情」时，按用户的来（见文首「优先级总则」）。

### 第一步：判定这条消息加不加

**先判去向再谈排版**——分岔做错，后面全白做：

| 场景 | `head` | `body` |
|---|---|---|
| **①** `body` 是 JSON（**webhook 推到业务服务器**，接收方 `JSON.parse` 后入库） | **不加** | **不加** |
| **②** `body` 是 Markdown / 纯文本 / 不传（**给人看**） | 加 1 个 | 按下面规范加 |

**判定依据只有一条：接收方是程序还是人的眼睛。** 数据的终点是你的商品库、线索库、工单表、时序库 → 走 ①；终点是通知栏 → 走 ②。用户没明说时默认走 ②。

**webhook / JSON 链路整条消息保持纯文本**：那里的 `body` 是数据结构不是文档，`head` 也常被当索引或去重键入库，任何 emoji 都会污染字段值或破坏解析。只有用户明确说「接收方只是展示、不解析」时才可对 `head` 放行，默认一律不加。

### `head`：一个 emoji + 一个结论

固定格式 `<emoji> + 半角空格 + 结论短语`，四条硬规则：

1. **emoji 放最前面**。通知栏会截断长标题，只有前置才一定被看到。
2. **一条 head 只用一个 emoji**。禁止 `🎉🚀✅ 构建完成` 这种堆砌。
3. **结论先行**。不看 body 也要知道「发生了什么 + 结果好坏」。`🎉 构建完成` 优于 `构建任务`。
4. **长度克制**。多数 emoji 占 1~2 个字符，head 上限 200 但建议 ≤30 字符，留给通知栏完整显示。

| 语义 | emoji | head 示例 |
|---|---|---|
| 成功 / 完成 | 🎉 | `🎉 代码优化完成` |
| 失败 / 崩溃 / 阻断 | 💣 | `💣 构建失败：单测未通过 3 项` |
| 警告 / 阈值告警 | ⚠️ | `⚠️ 磁盘剩余不足 10%` |
| 发布 / 上线 / 部署 | 🚀 | `🚀 v2.4.0 已发布至生产` |
| 进行中 / 排队 | ⏳ | `⏳ 预发环境部署中` |
| 启动 / 开始 | 🏗️ | `🏗️ 数据迁移已启动` |
| 自动化 / 机器人产出 | 🤖 | `🤖 每日情报已生成` |
| 数据采集 / 抓取 | 📥 | `📥 采集完成：128 页 / 1204 条` |
| 报告 / 统计 / 周报 | 📊 📈 📉 | `📊 本周数据周报` |
| 定时 / 提醒 | ⏰ | `⏰ 定时任务执行完毕` |
| 需要人工确认 / 待办 | 👀 🙋 | `👀 需要你确认：合并 PR #42` |
| 缺陷 / 修复 | 🐛 🛠️ | `🐛 已修复 3 个缺陷` |
| 安全 / 鉴权 / 密钥 | 🔒 🔑 | `🔒 检测到异地异常登录` |
| 消息 / 会话 / 公告 | 💬 📣 | `💬 客户回复待跟进` |
| 配置 / 环境变更 | ⚙️ | `⚙️ 生产配置已更新` |

贴合不上就用 🎉 / 💣 / ⚠️ / 🤖 四个兜底，不要硬套不相关的图形。

### `body`：状态灯 + 结果符

emoji 分两组，**语义边界必须分清，同一条消息里不许混用**：

| 组 | 成员 | 表达什么 | 用在哪 |
|---|---|---|---|
| 状态灯 | 🟢 🟡 🔴 ⚪ | **程度**：健康 / 警戒 / 危险 / 未执行 | 指标、健康度、分级列表 |
| 结果符 | ✅ ❌ ⚠️ | **二元判定**：通过 / 不通过 / 有风险但不阻塞 | 清单项、用例、步骤结果 |

常见搭配：

- 清单项：`- ✅ 单元测试 128/128 通过`、`- ❌ 接口 /v2/list 超时`、`- ⚠️ 2 个用例跳过`
- 分区标题：`## ✅ 通过项`、`## ❌ 失败项`、`## ⚠️ 需关注`
- 指标行：`- 🟢 响应 120ms ｜ 🟡 队列积压 47 ｜ 🔴 错误率 3.2%`
- 强调键值：`**🟢 状态**：正常`
- 后续动作：`1. 🛠️ 重跑失败用例`、`2. 🔑 刷新 API 令牌`

补充表（按需取用，别贪多）：

| emoji | 语义 | emoji | 语义 |
|---|---|---|---|
| 📌 重点 / 下一步 | 📎 附件 | 🗂️ 分类 / 归档 | 🧾 明细 / 账单 |
| 🕐 耗时 / 时间 | 💰 金额 / 成本 | 👤 负责人 / 用户 | 🏷️ 标签 |
| 🔍 检索 / 排查 | 🧪 测试 | 📦 产物 / 依赖 | 🌐 网络 / 站点 |
| 💾 存储 / 备份 | 🧹 清理 | ♻️ 重试 / 回滚 | 🔗 链接 / 跳转 |

### emoji 红线（别做过头）

1. **不承载唯一信息**。文字必须能自解释：写 `❌ 失败`，不能只写 `❌`。纯文本客户端和读屏软件看不到图形。
2. **同类语义全篇统一**。别在一条消息里用 ✅ 和 🟢 同时表示「正常」。
3. **密度克制**。一个列表项 / 一行**最多一个** emoji，塞三个只会更吵。
4. **表格里少用**。列宽会被撑歪；要用就放单元格开头。
5. **不进代码 / 命令 / URL / 文件路径 / JSON 字段值**。
6. **不做装饰、不替代标点**。`head` 里尤其不许连排。
7. 若用户反馈客户端显示成方框 □（老系统缺字形），直接停用 emoji 改纯文本，不要反复试。

> 场景模板不在这里展开，**组装内容前先看 [references/example.md](references/example.md)**。

## 安全红线（务必遵守）

- 通道码即推送凭证：只允许出现在 MCP Header、工具入参，以及**给用户的对话回复**里（轮换后必须让用户拿到新码，步骤见 references/api.md 5.5）；不得硬编码进代码/仓库配置并提交 git，也不得写进 `head`/`body`/`url`。
- 推送内容中禁止出现密码、令牌、私钥、Cookie、完整环境变量等敏感信息原文；需要提及时只做脱敏描述（如「令牌已刷新」而不是令牌值）。
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

**示例的顺序就是判断顺序：先看这条消息给谁。** 第一个是 webhook 入库（推到业务服务器，`body` 是 JSON、零 emoji）——这是 push-server 商业价值最高的一类用法，AI 负责采集清洗、push-server 负责投递、业务侧直接落库。

下面只是**最小骨架**，用来确认字段怎么填。真实推送前先到 [references/example.md](references/example.md) 找最接近的场景：第 1 节是这类入库示例的完整版，第 2~12 节是 11 类人读场景（第 13 节为反例对照），含成功 / 失败 / 告警三种分支写法。

> 示例里的 emoji 与排版只是缺省基线，**用户表达了偏好时以其偏好为准**。

**结构化数据推送到业务服务器（webhook JSON，零 emoji，商业入库链路）**

```json
{
  "head": "采集批次 2026-10-03T10:32",
  "body": "{\"batchId\":\"20261003-1032\",\"source\":\"shop.example.com/new\",\"collectedAt\":\"2026-10-03T10:32+08:00\",\"total\":48,\"items\":[{\"sku\":\"SKU1001\",\"name\":\"降噪耳机\",\"price\":899,\"stock\":132},{\"sku\":\"SKU1002\",\"name\":\"机械键盘\",\"price\":459,\"stock\":3}]}"
}
```

> `head` 也是纯文本——接收端常拿它当批次号或去重键入库；`body` 必须是裸 JSON，不能包 Markdown 围栏。同款数据的多源批量入库、线索入库、监控与舆情批次写法见 [references/example.md](references/example.md) 第 1 节。

**任务汇报（含跳转链接）**

```json
{
  "head": "🎉 代码优化完成",
  "body": "## ✅ 优化清单\n- ✅ **性能**：减少冗余会话查找，P95 下降 38%\n- ✅ **稳定性**：补充异常捕获与重试\n- ⚠️ **兼容性**：旧版客户端需灰度验证",
  "url": "https://example.com/pr/123"
}
```

**失败汇报**

```json
{
  "head": "💣 构建失败：单测未通过 3 项",
  "body": "## ❌ 失败用例\n- ❌ `UserServiceTest#login` 断言超时\n- ❌ `OrderServiceTest#refund` 空指针\n- ❌ `PayServiceTest#callback` 签名校验失败\n\n## 📌 下一步\n1. 🔍 查看完整日志\n2. 🛠️ 修复后重跑 CI",
  "url": "https://example.com/build/128"
}
```

**简单通知（仅标题）**

```json
{ "head": "🎉 构建已完成" }
```

**告警通知（状态灯分级）**

```json
{
  "head": "⚠️ 磁盘剩余不足 10%",
  "body": "- 🔴 根分区已用 91%（剩余 18G）\n- 🟡 日志目录 7 天增长 12G\n- 🟢 数据盘正常\n\n## 📌 建议\n1. 🧹 清理 30 天前归档日志\n2. 💾 评估扩容",
  "url": "https://example.com/monitor/disk"
}
```

**同一批采集结果给人看时（Markdown 版，对照上面 webhook 版）**

```json
{
  "head": "📥 每日采集完成：128 页 / 1204 条",
  "body": "## 🟢 概览\n- ✅ 页面 **128** ｜ ❌ 失败 **3**\n- 🗂️ 入库 **1,204** 条 ｜ 🕐 6m10s\n\n## ⚠️ 需关注\n- 🟡 docs.example.com 1.8s → 4.2s\n- 🔴 api.example.com/v2 连续 3 次 403\n\n## 📌 下一步\n1. 🛠️ 重跑失败的 3 个页面\n2. 🔑 检查 API 令牌",
  "url": "https://example.com/report/2026-10-03"
}
```

**表格报告（单条 + Markdown 表格，不要拆成多条）**

```json
{
  "head": "📊 竞品功能对比：3 款产品",
  "body": "## 🟢 结论\n- 🏆 **综合最强**：产品 C，覆盖 18/20 项\n- 💰 **性价比**：产品 E，¥99 覆盖 15/20 项\n\n## 📋 对比明细\n| 产品 | 单价 | 覆盖 | 推荐度 |\n|---|---|---|---|\n| 产品 C | ¥399 | 18/20 | 🟢 |\n| 产品 E | ¥99 | 15/20 | 🟢 |\n| 产品 F | ¥599 | 9/20 | 🔴 |\n\n## 📌 下一步\n1. 🧪 出 C / E 的详细试用对比",
  "url": "https://example.com/report/competitors"
}
```

**批量采集（`send_multi_message`，并列结果每条一条）**

```json
{
  "channelCode": "11111111111111111111111111aaaaaa",
  "messages": [
    { "head": "📥 站点A 采集完成：48 条", "body": "- ✅ 成功 48 ｜ ❌ 失败 0\n- 🕐 耗时 12.4s", "url": "https://example.com/collect/a" },
    { "head": "📥 站点B 采集完成：12 条", "body": "- ✅ 成功 10 ｜ ❌ 失败 2\n- 🕐 耗时 3.1s", "url": "https://example.com/collect/b" }
  ]
}
```

> 只有**批量采集**和**跨通道发送**才用多条。前面的「表格报告」示例虽然也是多条数据，但必须走单条 Markdown 表格。

**成功响应**

```json
{"code": 0, "message": "请求成功", "data": {"messageIdList": ["1680030476695388161"]}}
```


