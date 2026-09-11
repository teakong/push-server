---
name: "push-skill"
description: "通过 pushServer MCP 向用户推送消息通知。当需要向用户汇报任务执行结果、里程碑完成情况，或发送 Markdown 格式的重要通知时使用本技能。"
---

# Push 消息推送技能

本技能让 AI 通过 `pushServer` MCP 向用户汇报任务成果，或向用户的通知通道发送重要通知。

## 何时调用
- 完成重要的编码任务或里程碑之后。
- 当用户明确要求发送通知或汇报执行结果时。
- 需要将 Markdown 格式的结果推送到用户的通知通道时。

## 配置说明 (用户必读)
使用此技能前，请先在 AI 助手中添加以下远程 MCP 服务器：
- **名称**: `pushServer`
- **URL**: `https://www.phprm.com/services/push/mcp`
- **Headers**:
  - `X-Push-Channel-Code`: `您的32位通道码` (必须配置，否则无法接收推送)
  - `X-Mcp-Source`: `客户端标识` (可选，仅用于服务端日志区分来源，可填 trae、cursor、claude-code 等任意值)

## 安全红线（务必遵守）
- 通道码是推送凭证，只允许配置在 MCP 服务器的 `X-Push-Channel-Code` Header 中；**不得**硬编码进代码、配置文件并提交 git，也不得写入 `head`/`body`/`url` 推送内容。
- 推送内容中**禁止**包含密码、token、私钥、Cookie、完整环境变量等敏感信息原文；如需提及，只做脱敏描述（如「令牌已刷新」而非令牌值）。
- `channelCode` 只能填写用户本人提供的通道码，禁止向其他通道发送消息；不确定时不填，使用 Header 预设值。

## 使用规范
1. **内容要求**:
   - **标题 (`head`)**: 必填，必须为纯文本，支持 Unicode，长度限制 200 字符以内。
   - **内容 (`body`)**: (可选) 仅支持 **Markdown** 格式（不支持 HTML，请勿使用 `<br>`、`<table>` 等 HTML 标签），长度限制 50,000 字符以内。
   - **跳转 (`url`)**: (可选) 如果有相关的网页链接（如 PR 地址、构建日志等），请提供 URL 供用户点击跳转，长度限制 500 字符以内。
   - **指定通道码 (`channelCode`)**: (可选) 覆盖 Header 中的通道码，不指定时将以 Header 中预设的 X-Push-Channel-Code 进行推送。

2. **识别结果**: 任务完成后，自动将核心成果总结为符合上述规范的推送内容。如果任务非常简单，可以只提供 `head`。
3. **格式化**: 如果提供 `body`，请确保 Markdown 语法正确，以提供最佳的阅读体验。
4. **URL 与反引号规范（强制，发送前必须清洗）**:
   - 消息接收端（H5 聊天页）会使用 showdown 将 `body` 转换为 HTML，**URL 外层的反引号**（Markdown 行内代码语法）经转换后会残留进 `href`/`src`，导致链接不可点击、头像/图片裂图。
   - **禁止**用反引号包裹任何 URL（`http://`、`https://`、协议相对 `//` 或以域名开头的链接）；文件名、命令、代码标识等非 URL 内容仍可正常使用反引号。
   - 调用工具前，必须对 `body` 和 `url` 两个字段按顺序执行以下清洗：

     body = body.replace(/\]\(`(https?:\/\/[^`]+)`\)/g, ']($1)')   // ① 链接语法内 URL 的反引号: ](`URL`) → ](URL)
     body = body.replace(/`(https?:\/\/[^`\s)]+)`/g, '$1')          // ② 裸 URL 外层反引号: `URL` → URL
     url  = url.replace(/`(https?:\/\/[^`\s]+)`/g, '$1')            // ③ url 字段同样清洗

   - 反例（禁止）: 点[「详情」](`https://push.phprm.com/message/x`)、裸写 `https://a.com/x.png` 外层再包一层反引号
   - 正例: 点[「详情」](https://push.phprm.com/message/x)、裸写 https://a.com/x.png
5. **调用工具**: 必须调用已注册的 `pushServer` MCP 服务器提供的 `send_push_message` 工具（注意：本 skill 本身不是 MCP 服务器，不能把 skill 名称当作 server 调用）：
   - `head`: 任务简述。
   - `body`: (可选) Markdown 格式的详细汇报。
   - `url`: (可选) 目标链接。
   - `channelCode`: (可选) 指定通道码进行推送，不指定时将以 Header 中预设的 X-Push-Channel-Code 进行推送。
6. **成功判定与失败处理**:
   - 服务端返回 `{"code":0, ...}` 且包含 `messageIdList` 即表示发送成功。
   - 若 `pushServer` MCP 不可用（未配置/未连接）或返回非 `code:0`，必须明确告知用户推送失败及原因（如未配置通道码、通道码无效等）；**不得改用 HTTP 直接调用接口，也不得在未收到成功响应时谎称已发送**。

## 示例
**任务汇报（含跳转链接）的工具调用参数：**
```json
{
  "head": "代码优化完成",
  "body": "**优化清单**\n- **性能**: 减少了冗余的会话查找\n- **稳定性**: 增加了异常捕获",
  "url": "https://example.com/pr/123"
}
```
**简单通知（仅标题）：**
```json
{
  "head": "构建已完成"
}
```
**成功响应示例：**
```json
{"code": 0, "message": "请求成功", "data": {"messageIdList": ["1680030476695388161"]}}
```
