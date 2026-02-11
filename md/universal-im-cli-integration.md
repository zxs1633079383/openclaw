# Universal IM 插件 CLI 集成指南

本文档详细描述了如何将 Universal IM 插件集成到 OpenClaw CLI 中，包括配置机制、插件加载流程以及实现细节。

---

## 目录

1. [Mattermost 插件 CLI 配置分析](#1-mattermost-插件-cli-配置分析)
2. [OpenClaw CLI 架构概述](#2-openclaw-cli-架构概述)
3. [Universal IM CLI 集成方案](#3-universal-im-cli-集成方案)
4. [配置选项详解](#4-配置选项详解)
5. [实现代码示例](#5-实现代码示例)
6. [使用示例](#6-使用示例)
7. [配置文件结构](#7-配置文件结构)

---

## 1. Mattermost 插件 CLI 配置分析

### 1.1 配置入口点

Mattermost 插件通过 `openclaw channels add mattermost` 命令进行配置。核心配置项包括：

| CLI 选项 | 配置键 | 说明 |
|---------|-------|------|
| `--bot-token` | `botToken` | Mattermost Bot 令牌 |
| `--http-url` | `baseUrl` | Mattermost 服务器 URL |
| `--use-env` | - | 使用环境变量配置 |
| `--webhook-path` | `webhookPath` | Webhook 接收路径 |

### 1.2 CLI 命令定义位置

CLI 选项在 `src/cli/channels-cli.ts` 中定义：

```typescript
// src/cli/channels-cli.ts
const optionNamesAdd = [
  "botToken",    // --bot-token <token>
  "httpUrl",     // --http-url <url>
  "webhookUrl",  // --webhook-url <url>
  "webhookPath", // --webhook-path <path>
  "useEnv",      // --use-env
  // ... 其他选项
];
```

### 1.3 插件 Setup 适配器

Mattermost 在 `extensions/mattermost/src/channel.ts` 中定义了 `setup` 适配器：

```typescript
// extensions/mattermost/src/channel.ts
setup: {
  resolveAccountId: ({ accountId }) => normalizeAccountId(accountId),
  
  applyAccountName: ({ cfg, accountId, name }) =>
    applyAccountNameToChannelSection({
      cfg,
      channelKey: "mattermost",
      accountId,
      name,
    }),
    
  validateInput: ({ accountId, input }) => {
    // 验证 botToken 和 baseUrl 是否提供
    const token = input.botToken ?? input.token;
    const baseUrl = input.httpUrl;
    if (!input.useEnv && (!token || !baseUrl)) {
      return "Mattermost requires --bot-token and --http-url (or --use-env).";
    }
    return null;
  },
  
  applyAccountConfig: ({ cfg, accountId, input }) => {
    // 将 CLI 输入应用到配置文件
    const token = input.botToken ?? input.token;
    const baseUrl = input.httpUrl?.trim();
    // ... 返回更新后的配置
  },
},
```

### 1.4 插件加载流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLI 命令执行流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户输入命令                                                 │
│     └─> openclaw channels add mattermost --bot-token xxx       │
│                                                                 │
│  2. CLI 解析选项 (src/cli/channels-cli.ts)                      │
│     └─> 提取 botToken, httpUrl 等选项                           │
│                                                                 │
│  3. 调用 channelsAddCommand (src/commands/channels/add.ts)      │
│     └─> 加载插件: getChannelPlugin("mattermost")                │
│                                                                 │
│  4. 执行插件 setup 适配器                                        │
│     ├─> validateInput(): 验证输入                               │
│     └─> applyAccountConfig(): 应用配置                          │
│                                                                 │
│  5. 保存配置文件                                                 │
│     └─> ~/.openclaw/config.yaml                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. OpenClaw CLI 架构概述

### 2.1 核心文件结构

```
src/
├── cli/
│   ├── channels-cli.ts      # channels 命令定义 & 选项注册
│   ├── add-mutators.ts      # 配置应用辅助函数
│   └── index.ts             # CLI 入口
├── commands/
│   └── channels/
│       └── add.ts           # channels add 命令实现
└── channels/
    └── plugins/
        ├── index.ts         # 插件加载器
        └── types.ts         # 插件类型定义
```

### 2.2 ChannelSetupInput 类型

所有 CLI 选项映射到 `ChannelSetupInput` 类型：

```typescript
// src/channels/plugins/types.ts
export interface ChannelSetupInput {
  name?: string;
  botToken?: string;
  token?: string;
  httpUrl?: string;
  webhookUrl?: string;
  webhookPath?: string;
  useEnv?: boolean;
  // Universal IM 特有选项
  provider?: string;
  transport?: string;
  outboundUrl?: string;
  outboundSecret?: string;
  // ... 其他选项
}
```

### 2.3 插件注册机制

插件通过 `api.registerChannel()` 注册：

```typescript
// extensions/mattermost/index.ts
const plugin = {
  id: "mattermost",
  name: "Mattermost",
  description: "Mattermost channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setMattermostRuntime(api.runtime);
    api.registerChannel({ plugin: mattermostPlugin });
  },
};

export default plugin;
```

---

## 3. Universal IM CLI 集成方案

### 3.1 需要的 CLI 选项

Universal IM 插件需要以下 CLI 选项：

| 选项 | 类型 | 必需 | 说明 |
|-----|------|-----|------|
| `--provider` | string | 否 | Provider 类型 (custom, generic) |
| `--transport` | string | 否 | Transport 类型 (webhook, websocket, polling) |
| `--webhook-url` | string | 条件 | 外部回调 URL (Transport 需要) |
| `--webhook-path` | string | 否 | Webhook 接收路径 |
| `--outbound-url` | string | 否 | 发送消息的目标 URL |
| `--outbound-secret` | string | 否 | 出站消息签名密钥 |
| `--poll-interval` | number | 否 | 轮询间隔 (毫秒) |
| `--ws-url` | string | 条件 | WebSocket 连接 URL |

### 3.2 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                   Universal IM CLI 集成架构                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │   CLI       │───>│ channels/add │───>│ Universal IM     │   │
│  │   Options   │    │   Command    │    │ Plugin Setup     │   │
│  └─────────────┘    └──────────────┘    └──────────────────┘   │
│        │                   │                    │               │
│        v                   v                    v               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Config File                          │   │
│  │  channels:                                              │   │
│  │    universal-im:                                        │   │
│  │      enabled: true                                      │   │
│  │      provider: "custom"                                 │   │
│  │      transport: "webhook"                               │   │
│  │      webhookPath: "/webhook/universal-im"               │   │
│  │      outboundUrl: "https://your-im.example.com/api"    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 配置选项详解

### 4.1 Provider 选项

```bash
# 使用内置 custom provider
openclaw channels add universal-im --provider custom

# 使用 generic provider (如果已注册)
openclaw channels add universal-im --provider generic
```

Provider 决定了如何解析入站消息和构建出站消息。

### 4.2 Transport 选项

```bash
# Webhook Transport (推荐)
openclaw channels add universal-im \
  --transport webhook \
  --webhook-path /webhook/universal-im

# WebSocket Transport
openclaw channels add universal-im \
  --transport websocket \
  --ws-url wss://your-im.example.com/ws

# Polling Transport
openclaw channels add universal-im \
  --transport polling \
  --poll-interval 5000 \
  --http-url https://your-im.example.com/api/messages
```

### 4.3 回调地址配置

```bash
# 配置外部 IM 系统回调 Universal IM 的地址
openclaw channels add universal-im \
  --webhook-url https://your-gateway.example.com/webhook/universal-im

# 配置 Universal IM 发送消息到外部 IM 系统的地址
openclaw channels add universal-im \
  --outbound-url https://your-im.example.com/api/send \
  --outbound-secret your-secret-key
```

---

## 5. 实现代码示例

### 5.1 更新 CLI 选项定义

在 `src/cli/channels-cli.ts` 中添加 Universal IM 特有选项：

```typescript
// src/cli/channels-cli.ts
const optionNamesAdd = [
  // 现有选项
  "botToken",
  "httpUrl",
  "webhookUrl",
  "webhookPath",
  "useEnv",
  // Universal IM 新增选项
  "provider",      // --provider <type>
  "transport",     // --transport <type>
  "outboundUrl",   // --outbound-url <url>
  "outboundSecret",// --outbound-secret <secret>
  "pollInterval",  // --poll-interval <ms>
  "wsUrl",         // --ws-url <url>
];

// 在 add 子命令中注册选项
addCmd
  .option("--provider <type>", "Universal IM provider type (custom, generic)")
  .option("--transport <type>", "Universal IM transport type (webhook, websocket, polling)")
  .option("--outbound-url <url>", "URL to send outbound messages")
  .option("--outbound-secret <secret>", "Secret for signing outbound messages")
  .option("--poll-interval <ms>", "Polling interval in milliseconds", parseInt)
  .option("--ws-url <url>", "WebSocket connection URL");
```

### 5.2 Universal IM Setup 适配器

```typescript
// extensions/universal-im/src/channel.ts
export const universalImPlugin: ChannelPlugin = {
  meta: {
    id: "universal-im",
    name: "Universal IM",
    description: "Universal instant messaging integration",
  },
  
  // ... 其他适配器 ...
  
  setup: {
    resolveAccountId: ({ accountId }) => normalizeAccountId(accountId),
    
    applyAccountName: ({ cfg, accountId, name }) =>
      applyAccountNameToChannelSection({
        cfg,
        channelKey: "universal-im",
        accountId,
        name,
      }),
    
    validateInput: ({ accountId, input }) => {
      // 验证 transport 和相关配置的一致性
      const transport = input.transport ?? "webhook";
      
      if (transport === "websocket" && !input.wsUrl) {
        return "WebSocket transport requires --ws-url option.";
      }
      
      if (transport === "polling" && !input.httpUrl) {
        return "Polling transport requires --http-url option.";
      }
      
      return null;
    },
    
    applyAccountConfig: ({ cfg, accountId, input }) => {
      const namedConfig = applyAccountNameToChannelSection({
        cfg,
        channelKey: "universal-im",
        accountId,
        name: input.name,
      });
      
      const next =
        accountId !== DEFAULT_ACCOUNT_ID
          ? migrateBaseNameToDefaultAccount({
              cfg: namedConfig,
              channelKey: "universal-im",
            })
          : namedConfig;
      
      // 提取 Universal IM 特有配置
      const universalImConfig = {
        ...(input.provider ? { provider: input.provider } : {}),
        ...(input.transport ? { transport: input.transport } : {}),
        ...(input.webhookUrl ? { webhookUrl: input.webhookUrl.trim() } : {}),
        ...(input.webhookPath ? { webhookPath: input.webhookPath.trim() } : {}),
        ...(input.outboundUrl ? { outboundUrl: input.outboundUrl.trim() } : {}),
        ...(input.outboundSecret ? { outboundSecret: input.outboundSecret } : {}),
        ...(input.pollInterval ? { pollInterval: input.pollInterval } : {}),
        ...(input.wsUrl ? { wsUrl: input.wsUrl.trim() } : {}),
        ...(input.httpUrl ? { httpUrl: input.httpUrl.trim() } : {}),
      };
      
      if (accountId === DEFAULT_ACCOUNT_ID) {
        return {
          ...next,
          channels: {
            ...next.channels,
            "universal-im": {
              ...next.channels?.["universal-im"],
              enabled: true,
              ...universalImConfig,
            },
          },
        };
      }
      
      // 多账户配置
      return {
        ...next,
        channels: {
          ...next.channels,
          "universal-im": {
            ...next.channels?.["universal-im"],
            enabled: true,
            accounts: {
              ...next.channels?.["universal-im"]?.accounts,
              [accountId]: {
                ...next.channels?.["universal-im"]?.accounts?.[accountId],
                enabled: true,
                ...universalImConfig,
              },
            },
          },
        },
      };
    },
  },
};
```

### 5.3 插件入口点

```typescript
// extensions/universal-im/index.ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { emptyPluginConfigSchema } from "openclaw/plugin-sdk";

import { universalImPlugin } from "./src/channel.js";
import { setUniversalImRuntime } from "./src/runtime.js";

const plugin = {
  id: "universal-im",
  name: "Universal IM",
  description: "Universal instant messaging integration plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setUniversalImRuntime(api.runtime);
    api.registerChannel({ plugin: universalImPlugin });
  },
};

export default plugin;
```

---

## 6. 使用示例

### 6.1 基础配置 (Webhook Transport)

```bash
# 添加 Universal IM 渠道
openclaw channels add universal-im \
  --provider custom \
  --transport webhook \
  --webhook-path /webhook/universal-im \
  --outbound-url https://your-im.example.com/api/send
```

### 6.2 带签名的安全配置

```bash
openclaw channels add universal-im \
  --provider custom \
  --transport webhook \
  --webhook-path /webhook/universal-im \
  --outbound-url https://your-im.example.com/api/send \
  --outbound-secret "your-hmac-secret-key"
```

### 6.3 WebSocket 配置

```bash
openclaw channels add universal-im \
  --provider custom \
  --transport websocket \
  --ws-url wss://your-im.example.com/ws \
  --outbound-url https://your-im.example.com/api/send
```

### 6.4 轮询配置

```bash
openclaw channels add universal-im \
  --provider custom \
  --transport polling \
  --http-url https://your-im.example.com/api/messages \
  --poll-interval 3000 \
  --outbound-url https://your-im.example.com/api/send
```

### 6.5 多账户配置

```bash
# 添加第一个账户 (默认)
openclaw channels add universal-im \
  --provider custom \
  --transport webhook \
  --webhook-path /webhook/universal-im/main

# 添加第二个账户
openclaw channels add universal-im:secondary \
  --provider custom \
  --transport webhook \
  --webhook-path /webhook/universal-im/secondary \
  --outbound-url https://secondary-im.example.com/api/send
```

### 6.6 查看配置状态

```bash
# 查看 Universal IM 渠道状态
openclaw channels status universal-im

# 查看所有渠道状态
openclaw channels status --all

# 深度探测 (包括连接测试)
openclaw channels status universal-im --deep
```

### 6.7 启用/禁用

```bash
# 禁用 Universal IM
openclaw channels disable universal-im

# 重新启用
openclaw channels enable universal-im
```

---

## 7. 配置文件结构

### 7.1 单账户配置

```yaml
# ~/.openclaw/config.yaml
channels:
  universal-im:
    enabled: true
    provider: "custom"
    transport: "webhook"
    webhookPath: "/webhook/universal-im"
    outboundUrl: "https://your-im.example.com/api/send"
    outboundSecret: "your-secret-key"
```

### 7.2 多账户配置

```yaml
# ~/.openclaw/config.yaml
channels:
  universal-im:
    enabled: true
    # 默认账户配置
    provider: "custom"
    transport: "webhook"
    webhookPath: "/webhook/universal-im"
    outboundUrl: "https://main-im.example.com/api/send"
    
    # 额外账户
    accounts:
      secondary:
        enabled: true
        provider: "custom"
        transport: "webhook"
        webhookPath: "/webhook/universal-im/secondary"
        outboundUrl: "https://secondary-im.example.com/api/send"
      
      production:
        enabled: true
        provider: "custom"
        transport: "websocket"
        wsUrl: "wss://production-im.example.com/ws"
        outboundUrl: "https://production-im.example.com/api/send"
        outboundSecret: "production-secret"
```

### 7.3 完整配置示例

```yaml
# ~/.openclaw/config.yaml
gateway:
  mode: local
  host: localhost
  port: 18789

channels:
  universal-im:
    enabled: true
    provider: "custom"
    transport: "webhook"
    webhookPath: "/webhook/universal-im"
    webhookUrl: "https://gateway.example.com/webhook/universal-im"
    outboundUrl: "https://your-im.example.com/api/send"
    outboundSecret: "hmac-secret"
    
    # 高级配置
    pollInterval: 5000
    retryAttempts: 3
    timeout: 30000
    
    accounts:
      dev:
        enabled: true
        provider: "custom"
        transport: "webhook"
        webhookPath: "/webhook/universal-im/dev"
        outboundUrl: "http://localhost:8080/api/send"
```

---

## 8. 与外部 IM 系统集成

### 8.1 入站消息流程

```
┌──────────────┐    POST     ┌──────────────────┐    ┌───────────────┐
│  外部 IM      │ ─────────> │  OpenClaw        │───>│  AI 处理      │
│  系统         │            │  Webhook 端点     │    │               │
└──────────────┘            └──────────────────┘    └───────────────┘
                                                           │
                                                           v
┌──────────────┐    POST     ┌──────────────────┐    ┌───────────────┐
│  外部 IM      │ <───────── │  Universal IM     │<───│  AI 响应      │
│  系统         │            │  Outbound        │    │               │
└──────────────┘            └──────────────────┘    └───────────────┘
```

### 8.2 Webhook 消息格式

**入站消息 (IM → OpenClaw):**

```json
{
  "messageId": "msg-123",
  "chatId": "chat-456",
  "senderId": "user-789",
  "senderName": "张三",
  "text": "你好，请帮我写一段代码",
  "timestamp": 1706707200000,
  "attachments": [
    {
      "type": "image",
      "url": "https://example.com/image.png",
      "mimeType": "image/png"
    }
  ]
}
```

**出站消息 (OpenClaw → IM):**

```json
{
  "chatId": "chat-456",
  "text": "好的，这是您需要的代码...",
  "replyToMessageId": "msg-123",
  "timestamp": 1706707205000
}
```

### 8.3 签名验证

如果配置了 `outboundSecret`，出站消息会包含签名头：

```http
POST /api/send HTTP/1.1
Host: your-im.example.com
Content-Type: application/json
X-Signature: sha256=abc123...
X-Timestamp: 1706707205000

{"chatId": "chat-456", "text": "..."}
```

签名计算方式：

```javascript
const crypto = require('crypto');

function verifySignature(body, signature, secret, timestamp) {
  const payload = `${timestamp}.${JSON.stringify(body)}`;
  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  return `sha256=${expected}` === signature;
}
```

---

## 9. 故障排查

### 9.1 常用诊断命令

```bash
# 运行诊断
openclaw doctor

# 查看详细日志
openclaw gateway run --verbose

# 测试 Webhook 端点
curl -X POST http://localhost:18789/webhook/universal-im \
  -H "Content-Type: application/json" \
  -d '{"messageId":"test","chatId":"test","senderId":"test","text":"hello"}'
```

### 9.2 日志位置

```bash
# macOS
tail -f ~/Library/Logs/OpenClaw/gateway.log

# Linux
tail -f ~/.openclaw/logs/gateway.log

# 实时查看 Universal IM 日志
openclaw gateway run 2>&1 | grep "universal-im"
```

### 9.3 常见问题

| 问题 | 解决方案 |
|-----|---------|
| Webhook 无响应 | 检查 `--webhook-path` 是否正确配置 |
| 消息发送失败 | 验证 `--outbound-url` 是否可达 |
| 签名验证失败 | 确认 `--outbound-secret` 一致 |
| WebSocket 断开 | 检查 `--ws-url` 并确认网络连接 |

---

## 10. 总结

Universal IM 插件的 CLI 集成遵循 OpenClaw 的标准插件架构：

1. **CLI 选项** 在 `src/cli/channels-cli.ts` 中注册
2. **Setup 适配器** 在 `extensions/universal-im/src/channel.ts` 中处理配置
3. **插件入口** 在 `extensions/universal-im/index.ts` 中注册渠道
4. **配置持久化** 到 `~/.openclaw/config.yaml`

通过这种方式，Universal IM 可以像其他内置渠道 (Mattermost, Slack, Discord 等) 一样，通过统一的 CLI 接口进行配置和管理。
