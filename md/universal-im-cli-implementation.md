# Universal IM 插件 CLI 配置实现文档

本文档描述了如何通过 CLI 命令配置 Universal IM 插件，以及实现过程中的代码修改。

---

## 1. 验收测试结果

```bash
curl -X POST http://localhost:8080/api/v1/local/message \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "test-session-001",
    "userId": "test-user-001",
    "text": "你好，请介绍一下你自己 我是zlc"
  }'
# 返回: {"success":true}
```

✅ **测试通过**

---

## 2. CLI 配置命令

### 2.1 完整配置命令

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

### 2.2 CLI 选项说明

| 选项 | 说明 | 示例值 |
|-----|------|--------|
| `--channel` | 渠道类型 | `universal-im` |
| `--name` | 渠道显示名称 | `"UIP Gateway"` |
| `--provider` | Provider 类型 | `custom` |
| `--transport` | 传输方式 | `webhook` / `websocket` / `polling` |
| `--webhook-path` | Webhook 接收路径 | `/universal-im/default/webhook` |
| `--outbound-url` | AI 响应发送地址 | `http://localhost:8080/api/v1/openclaw/outbound` |
| `--dm-policy` | 私聊策略 | `open` / `pairing` / `allowlist` / `disabled` |

### 2.3 生成的配置文件

执行命令后，`~/.openclaw/openclaw.json` 中的配置：

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

---

## 3. 代码实现详解

### 3.1 修改的文件

| 文件 | 修改内容 |
|-----|---------|
| `src/channels/plugins/types.core.ts` | 添加 `ChannelSetupInput` 新字段 |
| `src/cli/channels-cli.ts` | 添加 CLI 选项定义 |
| `src/commands/channels/add.ts` | 传递新选项到 setup adapter |
| `src/commands/channels/add-mutators.ts` | 添加新参数传递 |
| `extensions/universal-im/src/channel.ts` | 实现 `applyAccountConfig` 处理新选项 |

### 3.2 ChannelSetupInput 类型扩展

```typescript
// src/channels/plugins/types.core.ts
export type ChannelSetupInput = {
  // ... 现有字段 ...
  
  // Universal IM specific options
  outboundUrl?: string;
  provider?: string;
  transport?: string;
  dmPolicy?: string;
};
```

### 3.3 CLI 选项定义

```typescript
// src/cli/channels-cli.ts
channels
  .command("add")
  // ... 现有选项 ...
  // Universal IM specific options
  .option("--outbound-url <url>", "Outbound URL for sending AI responses (Universal IM)")
  .option("--provider <provider>", "Provider type (Universal IM: custom)")
  .option("--transport <transport>", "Transport type (Universal IM: webhook/websocket/polling)")
  .option("--dm-policy <policy>", "DM policy (pairing/allowlist/open/disabled)")
```

### 3.4 Setup Adapter 实现

```typescript
// extensions/universal-im/src/channel.ts
setup: {
  validateInput: ({ accountId, input }) => {
    // 验证 transport 类型
    const transport = input.transport;
    if (transport && !["webhook", "websocket", "polling"].includes(transport)) {
      return `Invalid transport "${transport}". Must be one of: webhook, websocket, polling.`;
    }
    // 验证 dmPolicy
    const dmPolicy = input.dmPolicy;
    if (dmPolicy && !["pairing", "allowlist", "open", "disabled"].includes(dmPolicy)) {
      return `Invalid dmPolicy "${dmPolicy}". Must be one of: pairing, allowlist, open, disabled.`;
    }
    return null;
  },
  
  applyAccountConfig: ({ cfg, accountId, input }) => {
    // 支持 --webhook-url 和 --outbound-url 两种方式设置出站 URL
    const outboundUrl = (input.outboundUrl ?? input.webhookUrl)?.trim();
    const webhookPath = input.webhookPath?.trim();
    const provider = input.provider?.trim();
    const transport = input.transport?.trim();
    const dmPolicy = input.dmPolicy?.trim();
    
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        "universal-im": {
          ...cfg.channels?.["universal-im"],
          enabled: true,
          ...(provider ? { provider } : {}),
          ...(transport ? { transport } : {}),
          ...(dmPolicy ? { dmPolicy } : {}),
          ...(dmPolicy === "open" ? { allowFrom: ["*"] } : {}),
          ...(webhookPath ? { webhook: { path: webhookPath } } : {}),
          ...(outboundUrl ? { outbound: { url: outboundUrl } } : {}),
        },
      },
    };
  },
},
```

---

## 4. 数据流架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        端到端通信流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. 用户发送消息                                                            │
│      curl POST http://localhost:8080/api/v1/local/message                  │
│      {"sessionId":"...", "userId":"...", "text":"..."}                     │
│                           ↓                                                 │
│   2. Go Gateway 转发                                                        │
│      POST http://localhost:18789/universal-im/default/webhook              │
│                           ↓                                                 │
│   3. OpenClaw Gateway 接收                                                  │
│      Universal IM Plugin → custom provider → parseInbound()                │
│                           ↓                                                 │
│   4. AI 处理                                                                │
│      dispatchReplyFromConfig() → Qwen Portal → 生成响应                     │
│                           ↓                                                 │
│   5. 发送响应                                                                │
│      POST http://localhost:8080/api/v1/openclaw/outbound                   │
│      {"to":"user:...", "text":"AI响应...", "chatId":"..."}                 │
│                           ↓                                                 │
│   6. Go Gateway 接收响应                                                    │
│      返回给用户/IM客户端                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 常用命令

### 5.1 添加配置

```bash
# 基础配置
openclaw channels add --channel universal-im \
  --webhook-path "/universal-im/default/webhook" \
  --outbound-url "http://localhost:8080/api/v1/openclaw/outbound" \
  --dm-policy open

# 完整配置
openclaw channels add --channel universal-im \
  --name "UIP Gateway" \
  --provider custom \
  --transport webhook \
  --webhook-path "/universal-im/default/webhook" \
  --outbound-url "http://localhost:8080/api/v1/openclaw/outbound" \
  --dm-policy open
```

### 5.2 查看状态

```bash
openclaw channels status
```

输出示例：
```
Gateway reachable.
- Universal IM default (UIP Gateway): enabled, configured, running, connected, mode:custom/webhook
```

### 5.3 禁用/启用

```bash
# 禁用
openclaw channels remove --channel universal-im

# 重新启用 (重新运行 add 命令)
openclaw channels add --channel universal-im ...
```

### 5.4 查看帮助

```bash
openclaw channels add --help
```

---

## 6. 配置选项映射

| CLI 选项 | 配置路径 | 说明 |
|---------|---------|------|
| `--name` | `channels.universal-im.name` | 显示名称 |
| `--provider` | `channels.universal-im.provider` | Provider 类型 |
| `--transport` | `channels.universal-im.transport` | 传输方式 |
| `--webhook-path` | `channels.universal-im.webhook.path` | Webhook 路径 |
| `--outbound-url` | `channels.universal-im.outbound.url` | 出站 URL |
| `--dm-policy` | `channels.universal-im.dmPolicy` | 私聊策略 |

---

## 7. 测试验证

### 7.1 前置条件

1. OpenClaw Gateway 运行中 (`openclaw gateway run`)
2. Go Gateway 运行中 (`localhost:8080`)
3. Universal IM 配置完成

### 7.2 测试命令

```bash
# 发送测试消息
curl -X POST http://localhost:8080/api/v1/local/message \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "test-session-001",
    "userId": "test-user-001",
    "text": "你好，请介绍一下你自己"
  }'

# 期望响应
{"success":true}
```

### 7.3 验证步骤

1. 检查 curl 返回 `{"success":true}"`
2. 检查 OpenClaw 日志确认消息接收
3. 检查 Go Gateway 日志确认 AI 响应接收

---

## 8. 总结

Universal IM 插件现在支持通过标准的 `openclaw channels add` 命令进行配置：

- ✅ `--outbound-url` - 设置 AI 响应发送地址
- ✅ `--provider` - 设置 Provider 类型
- ✅ `--transport` - 设置传输方式
- ✅ `--dm-policy` - 设置私聊策略
- ✅ `--webhook-path` - 设置 Webhook 接收路径
- ✅ 自动设置 `allowFrom: ["*"]` 当 `dmPolicy=open`

所有配置选项与现有渠道 (Mattermost, Slack, Discord 等) 保持一致的 CLI 风格。
