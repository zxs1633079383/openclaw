# GitHub Discussion/Issue 文档

---

## 标题

**[Feature] Universal IM Plugin - Connect Any Instant Messaging System to OpenClaw**

---

## 正文内容 (复制以下内容到 GitHub)

### 🦞 Feature Request: Universal IM Plugin

#### Summary

I've implemented a **Universal IM Plugin** that enables **any instant messaging system** to integrate with OpenClaw through a standardized webhook-based protocol. This allows developers to connect their custom IM platforms, enterprise messaging systems, or any third-party chat applications to OpenClaw's AI capabilities.

#### Motivation

Currently, OpenClaw supports specific messaging platforms (Telegram, Discord, Slack, WhatsApp, etc.) with dedicated plugins. However, there are many scenarios where users need to integrate:

- **Custom enterprise IM systems** (internal chat apps)
- **Regional messaging platforms** (WeChat, LINE, KakaoTalk, etc.)
- **IoT/embedded chat interfaces**
- **Custom chatbot frontends**
- **Legacy systems** requiring AI enhancement

Building a dedicated plugin for each platform is time-consuming and not scalable. The Universal IM Plugin solves this by providing a **standardized interface** that any system can implement.

#### Implementation

I've implemented the Universal IM plugin with the following components:

**1. CLI Configuration Support**

```bash
openclaw channels add \
  --channel universal-im \
  --name "UIP Gateway" \
  --provider custom \
  --transport webhook \
  --webhook-path "/universal-im/default/webhook" \
  --outbound-url "http://localhost:8080/api/v1/openclaw/outbound" \
  --dm-policy open
```

**2. Generated Configuration**

```json
{
  "channels": {
    "universal-im": {
      "name": "UIP Gateway",
      "enabled": true,
      "transport": "webhook",
      "provider": "custom",
      "webhook": {
        "path": "/universal-im/default/webhook"
      },
      "outbound": {
        "url": "http://localhost:8080/api/v1/openclaw/outbound"
      },
      "dmPolicy": "open",
      "allowFrom": ["*"]
    }
  }
}
```

**3. New CLI Options Added**

| Option | Description |
|--------|-------------|
| `--outbound-url <url>` | URL for sending AI responses back to the IM system |
| `--provider <type>` | Provider type (default: `custom`) |
| `--transport <type>` | Transport type: `webhook`, `websocket`, or `polling` |
| `--dm-policy <policy>` | DM policy: `open`, `pairing`, `allowlist`, or `disabled` |

#### Architecture

**Direct Integration (No intermediate gateway needed):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Direct Integration Architecture                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐   POST webhook        ┌──────────────────────────┐  │
│   │   Any IM     │ ───────────────────>  │   OpenClaw Gateway       │  │
│   │   System     │                       │   localhost:18789        │  │
│   │              │                       │                          │  │
│   │  (WeChat,    │                       │  /universal-im/default/  │  │
│   │   LINE,      │                       │  webhook                 │  │
│   │   Custom,    │                       │                          │  │
│   │   Web App)   │                       │  ┌────────────────────┐  │  │
│   └──────────────┘                       │  │ Universal IM       │  │  │
│          ↑                               │  │ Plugin             │  │  │
│          │                               │  │ - parse message    │  │  │
│          │                               │  │ - AI processing    │  │  │
│          │   POST outbound               │  │ - send response    │  │  │
│          └────────────────────────────── │  └────────────────────┘  │  │
│                                          └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**With Intermediate Gateway (For complex production scenarios):**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 Intermediate Gateway Architecture                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────┐    ┌─────────────────┐    ┌──────────────────────────┐ │
│  │  Any IM    │    │  Your Gateway   │    │   OpenClaw Gateway       │ │
│  │  System    │───>│  (Go/Node/Py)   │───>│   localhost:18789        │ │
│  │            │    │  localhost:8080 │    │                          │ │
│  │            │    │                 │    │  /universal-im/.../      │ │
│  │            │    │  - Auth         │    │  webhook                 │ │
│  │            │<───│  - Transform    │<───│                          │ │
│  │            │    │  - Rate limit   │    │  Universal IM Plugin     │ │
│  └────────────┘    └─────────────────┘    └──────────────────────────┘ │
│                                                                         │
│  Benefits of intermediate gateway:                                      │
│  - Handle IM platform authentication (OAuth, tokens)                    │
│  - Transform proprietary message formats                                │
│  - Manage rate limiting and retries                                     │
│  - Aggregate multiple IM sources                                        │
│  - Add custom business logic                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Message Protocol

**Inbound Message (IM → OpenClaw):**

```json
{
  "messageId": "msg-123",
  "timestamp": 1706707200000,
  "sender": {
    "id": "user-001",
    "name": "John Doe"
  },
  "conversation": {
    "type": "direct",
    "id": "session-001"
  },
  "text": "Hello, can you help me?"
}
```

**Outbound Response (OpenClaw → IM):**

```json
{
  "to": "user:user-001",
  "text": "Of course! How can I assist you today?",
  "chatId": "session-001",
  "replyToId": "msg-123"
}
```

#### Testing

There are **two integration approaches** for Universal IM:

**Approach 1: Direct Integration (Recommended for simple use cases)**

Your IM system can directly call OpenClaw's webhook endpoint without any intermediate layer:

```bash
# Direct call to OpenClaw Gateway webhook
curl -X POST http://localhost:18789/universal-im/default/webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "messageId": "msg-001",
    "sender": { "id": "user-001", "name": "John" },
    "conversation": { "type": "direct", "id": "session-001" },
    "text": "Hello, can you help me?"
  }'

# Response: {"ok":true}
# AI response will be POSTed to your configured outbound URL
```

**Approach 2: Through an Intermediate Gateway (For complex scenarios)**

For production deployments, you may want an intermediate gateway (e.g., Go, Node.js, Python) to:
- Handle authentication with your IM platform
- Transform message formats
- Manage rate limiting
- Handle multiple IM sources

I tested with a Go-based gateway that forwards messages:

```bash
# This curl goes to the Go Gateway (localhost:8080)
# The Go Gateway then forwards to OpenClaw (localhost:18789)
curl -X POST http://localhost:8080/api/v1/local/message \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "test-session-001",
    "userId": "test-user-001",
    "text": "hello"
  }'

# Response: {"success":true}
```

**Integration Flow Comparison:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Approach 1: Direct Integration                                             │
│  ───────────────────────────────                                            │
│  Your IM App  ──POST──>  OpenClaw Gateway (:18789)  ──POST──>  Your IM App │
│               webhook    /universal-im/.../webhook   outbound               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Approach 2: With Intermediate Gateway                                      │
│  ─────────────────────────────────────                                      │
│  Your IM App  ──>  Go/Node Gateway (:8080)  ──>  OpenClaw (:18789)         │
│                          │                              │                   │
│                          └──────────< outbound <────────┘                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

Both approaches are fully supported. Choose based on your architecture needs.

#### Files Changed

| File | Change |
|------|--------|
| `src/channels/plugins/types.core.ts` | Added `outboundUrl`, `provider`, `transport`, `dmPolicy` to `ChannelSetupInput` |
| `src/cli/channels-cli.ts` | Added CLI option definitions |
| `src/commands/channels/add.ts` | Pass new options to setup adapter |
| `src/commands/channels/add-mutators.ts` | Pass new parameters |
| `extensions/universal-im/src/channel.ts` | Implement setup adapter for new options |

#### Use Cases

1. **Enterprise Integration**: Connect internal chat systems to OpenClaw AI
2. **Regional Platforms**: Integrate LINE, KakaoTalk, or other regional messengers
3. **IoT Devices**: Add AI chat capabilities to smart devices
4. **Custom Frontends**: Build web/mobile chat UIs backed by OpenClaw
5. **Legacy System Enhancement**: Add AI to existing messaging infrastructure

#### Future Enhancements

- [ ] WebSocket transport for real-time bidirectional communication
- [ ] Polling transport for systems that can't receive webhooks
- [ ] Provider registry for custom message format adapters
- [ ] Built-in signature verification for secure webhooks
- [ ] Rate limiting and quota management

#### AI-Assisted Development 🤖

This feature was developed with AI assistance (Claude). The implementation has been:
- [x] Fully tested with end-to-end integration
- [x] Code reviewed and linted
- [x] Documentation created
- [x] Follows existing OpenClaw patterns (similar to Mattermost plugin)

---

### Questions for Maintainers

1. Should the Universal IM plugin be included in the core `extensions/` directory or as a separate package?
2. Are there any security considerations I should address for the webhook endpoint?
3. Would you like me to add additional transport types (WebSocket/Polling) in this PR or as follow-ups?

---

### Related

- Follows the plugin architecture established by Mattermost, Matrix, and other channel plugins
- Uses the same `ChannelPlugin` interface and setup adapters
- Compatible with existing OpenClaw Gateway infrastructure

---

**Labels suggested**: `enhancement`, `new-feature`, `universal-im`, `channels`

---

## 如何提交

1. 访问 https://github.com/openclaw/openclaw/discussions/new
2. 选择 "Feature Request" 或 "Ideas" 类别
3. 复制上述 "正文内容" 部分
4. 提交讨论

或者如果要提交 Issue：
1. 访问 https://github.com/openclaw/openclaw/issues/new
2. 复制上述内容
3. 添加标签: `enhancement`, `new-feature`

---

## 中文版摘要 (供参考)

### 功能：Universal IM 插件 - 让任何即时通信系统接入 OpenClaw

**为什么开发这个功能？**

目前 OpenClaw 仅支持特定的消息平台（Telegram、Discord、Slack 等）。但是：
- 企业有自己的内部 IM 系统需要接入 AI
- 区域性平台（微信、LINE 等）需要集成
- 自定义聊天前端需要 AI 能力
- 为每个平台开发专用插件不可扩展

**解决方案：**

Universal IM 插件提供标准化接口，任何 IM 系统只需实现简单的 webhook 协议即可接入 OpenClaw：

```bash
# 一条命令完成配置
openclaw channels add --channel universal-im \
  --webhook-path "/universal-im/default/webhook" \
  --outbound-url "http://your-im-gateway/api/outbound" \
  --dm-policy open
```

**测试验证：**

方式一：直接调用 OpenClaw Webhook（无需中间层）
```bash
# 直接调用 OpenClaw Gateway
curl -X POST http://localhost:18789/universal-im/default/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "sender": {"id": "user1", "name": "测试用户"},
    "conversation": {"type": "direct", "id": "session-001"},
    "text": "你好"
  }'
# 返回: {"ok":true}
# AI 响应会 POST 到配置的 outbound URL
```

方式二：通过中间 Gateway（适合复杂场景）
```bash
# 通过 Go Gateway 转发（localhost:8080 → localhost:18789）
curl -X POST http://localhost:8080/api/v1/local/message \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test","userId":"user1","text":"你好"}'
# 返回: {"success":true}
```

两种方式都完全支持，根据架构需求选择。
