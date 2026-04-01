# OpenClaw Gateway Configuration Format

## Overview

The OpenClaw Gateway expects a specific configuration format that differs from the v2.0.0 "collective" reference config. This document explains the correct format and location.

## Config File Location

**Correct location:** `~/.openclaw/config/openclaw.json`

**NOT:** `~/.openclaw/openclaw.json` (this is a different config used by the CLI)

## Key Differences: Gateway Format vs v2.0.0 Collective Format

| Aspect | Gateway Format (Correct) | v2.0.0 Collective Format (Wrong for Gateway) |
|--------|-------------------------|---------------------------------------------|
| `models.providers.litellm.baseUrl` | `baseUrl` (camelCase) | `base_url` (snake_case) |
| `models.providers.litellm.enabled` | Not present | `true` |
| `agents` | Object with `defaults` | Array of agent definitions |
| `gateway.mode` | Required (`"local"` or `"remote"`) | Not present |
| Top-level keys | `meta`, `wizard`, `auth`, `models`, `agents`, `messages`, `commands`, `gateway`, `model_routing` | `version`, `collective`, `models`, `agents`, `model_routing`, `a2a_protocol`, `embedding`, `memory`, `skills_repository` |

## Gateway-Compatible Config Structure

```json
{
  "meta": {
    "lastTouchedVersion": "2.0.0",
    "lastTouchedAt": "2026-04-01T00:00:00.000Z"
  },
  "wizard": {
    "lastRunAt": "2026-04-01T00:00:00.000Z",
    "lastRunVersion": "2.0.0",
    "lastRunCommand": "configure",
    "lastRunMode": "local"
  },
  "auth": {
    "profiles": {
      "litellm:default": {
        "provider": "litellm",
        "mode": "api_key"
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "litellm": {
        "baseUrl": "http://localhost:4000",
        "api": "openai-completions",
        "models": [...]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "litellm/claude-opus-4-6"
      },
      "compaction": {
        "mode": "safeguard"
      },
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    }
  },
  "messages": {
    "ackReactionScope": "group-mentions"
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "your-token-here"
    }
  },
  "model_routing": {
    "default": "minimax/MiniMax-M2.7",
    "aliases": {...},
    "passthrough_endpoints": {...}
  }
}
```

## Common Validation Errors and Fixes

### Error: `models.providers.litellm.baseUrl: Invalid input: expected string, received undefined`

**Cause:** Using `base_url` (snake_case) instead of `baseUrl` (camelCase)

**Fix:** Change `"base_url": "..."` to `"baseUrl": "..."`

### Error: `models.providers.litellm: Unrecognized keys: "enabled", "base_url"`

**Cause:** Gateway doesn't recognize `enabled` field and expects `baseUrl` not `base_url`

**Fix:** Remove `"enabled": true` and rename `base_url` to `baseUrl`

### Error: `agents: Invalid input: expected object, received array`

**Cause:** Gateway expects `agents` as an object with `defaults`, not an array of agent definitions

**Fix:** Change from:
```json
"agents": [
  {"id": "steward", "name": "Steward", ...},
  {"id": "alpha", "name": "Alpha", ...}
]
```
To:
```json
"agents": {
  "defaults": {
    "model": {"primary": "litellm/claude-opus-4-6"},
    "maxConcurrent": 4
  }
}
```

### Error: `<root>: Unrecognized keys: "version", "collective", "a2a_protocol", ...`

**Cause:** Gateway schema doesn't recognize these top-level keys

**Fix:** Remove `version`, `collective`, `a2a_protocol`, `embedding`, `memory`, `skills_repository` from the gateway config. These can remain in the collective reference config at `heretek-openclaw-core/openclaw.json`.

### Error: `Gateway mode is unset`

**Cause:** Missing `gateway.mode` configuration

**Fix:** Add `"gateway": {"mode": "local"}` to your config

## Reference Config Files

- **Gateway config (runtime):** `~/.openclaw/config/openclaw.json`
- **Collective reference config:** `heretek-openclaw-core/openclaw.json` (updated to gateway-compatible format)

## Verification

Run `openclaw doctor` to verify your configuration is valid:

```bash
openclaw doctor
```

If the config is valid, you should see doctor complete without schema validation errors.

## Related Documentation

- [`AGENT_MODEL_CONFIG.md`](./AGENT_MODEL_CONFIG.md) - Per-agent model configuration
- [`PROVIDER_SETUP.md`](./PROVIDER_SETUP.md) - Provider setup instructions
