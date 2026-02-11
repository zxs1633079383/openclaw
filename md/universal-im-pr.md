# Pull Request

## Title

```
feat(channels): add Universal IM plugin CLI configuration support
```

---

## Description

### Summary

This PR adds CLI configuration support for the Universal IM plugin, enabling any instant messaging system to integrate with OpenClaw through a standardized webhook-based protocol.

### Motivation

Currently, integrating a new IM platform requires building a dedicated plugin. The Universal IM plugin provides a standardized interface that any messaging system can implement via simple HTTP webhooks, making OpenClaw accessible to:

- Custom enterprise IM systems
- Regional messaging platforms (WeChat, LINE, KakaoTalk, etc.)
- IoT/embedded chat interfaces
- Custom chatbot frontends
- Legacy systems requiring AI enhancement

### Changes

#### New CLI Options

Added Universal IM specific options to `openclaw channels add`:

| Option | Description |
|--------|-------------|
| `--outbound-url <url>` | URL for sending AI responses back to the IM system |
| `--provider <type>` | Provider type (default: `custom`) |
| `--transport <type>` | Transport type: `webhook`, `websocket`, or `polling` |
| `--dm-policy <policy>` | DM policy: `open`, `pairing`, `allowlist`, or `disabled` |

#### Example Usage

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

#### Generated Configuration

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

### Files Changed

| File | Change |
|------|--------|
| `src/channels/plugins/types.core.ts` | Added `outboundUrl`, `provider`, `transport`, `dmPolicy` to `ChannelSetupInput` |
| `src/cli/channels-cli.ts` | Added CLI option definitions for Universal IM |
| `src/commands/channels/add.ts` | Pass new options to plugin setup adapter |
| `src/commands/channels/add-mutators.ts` | Added new parameters to config mutator |
| `extensions/universal-im/src/channel.ts` | Implemented setup adapter to handle new CLI options |

### Integration

The Universal IM plugin supports two integration approaches:

**1. Direct Integration** - Call OpenClaw webhook directly:
```bash
curl -X POST http://localhost:18789/universal-im/default/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "messageId": "msg-001",
    "sender": { "id": "user-001", "name": "John" },
    "conversation": { "type": "direct", "id": "session-001" },
    "text": "Hello"
  }'
```

**2. Via Intermediate Gateway** - For complex production scenarios with custom business logic.

### Testing

- [x] CLI options display correctly in `--help`
- [x] Configuration is correctly written to `~/.openclaw/openclaw.json`
- [x] End-to-end test with Go gateway returns `{"success":true}`
- [x] Lint passes: `pnpm lint`
- [x] Build passes: `pnpm build`

### AI-Assisted Development 🤖

This PR was developed with AI assistance (Claude). The implementation:
- [x] Follows existing OpenClaw patterns (similar to Mattermost, Matrix plugins)
- [x] Has been fully tested with end-to-end integration
- [x] Code reviewed and linted
- [x] Documentation created

---

## Checklist

- [x] Tested locally with OpenClaw instance
- [x] Ran linter: `pnpm lint`
- [x] PR is focused (one feature)
- [x] Described what & why
- [x] Marked as AI-assisted

---

## Related

- Uses the same `ChannelPlugin` interface and setup adapters as other channels
- Compatible with existing OpenClaw Gateway infrastructure
- No breaking changes to existing functionality
