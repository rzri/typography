---
title: NewAPI 对接 OpenClaw
pubDate: 2026-04-03
categories: ['笔记']
description: '记录一次 NewAPI(OpenAI 兼容接口) 对接 OpenClaw 时遇到 403 的排查过程：根因是 WAF/网关拦截特定 User-Agent，通过 providers.headers 覆盖 UA 即可恢复 200。'
slug: newapi-openclaw-403-user-agent
---

## 背景与结论

现象：

- OpenClaw 通过某个 NewAPI（OpenAI 兼容）站点请求模型时返回 **403**
- 表面看像“模型不可用 / Key 不对 / 没权限”

结论（根因）：

- **403 并不一定是模型不可用**
- 在某些站点前置的 **WAF/网关** 场景里，可能是 **请求头被拦截**
- 高概率触发点：`User-Agent: OpenAI/JS 6.26.0`
- 同样请求内容，换一个 `User-Agent` 可能立刻变 **200**

> 这类问题的特点是：换模型仍然 403，但修改 UA 后立刻恢复。

---

## 1. 复现与定位思路

### 1.1 复现症状

- 通过 OpenClaw Gateway 或 agent 调用接口：返回 403
- 请求内容看起来完全正常

### 1.2 定位路径（推荐顺序）

1. 先用 `curl` 直连 OpenClaw Gateway（排除通道/机器人问题）
2. 若仍 403：怀疑 provider 侧拦截（WAF/反爬/风控）
3. 重点检查/覆盖 `User-Agent`

---

## 2. 修复方案：在 provider 上加 headers 覆盖 User-Agent

核心做法：在 `openclaw.json` 的 `models.providers.<provider>` 里加 `headers` 字段。

### 2.1 脱敏后的参考配置（骨架）

说明：

- 下方配置是“结构示例”，**不包含任何真实站点、Key、Token、Secret**
- 你需要替换所有 `YOUR_*` 占位符

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/ubuntu/.openclaw/workspace",
      "model": {
        "primary": "newapi/model-y",
        "fallbacks": []
      }
    }
  },
  "tools": {
    "profile": "coding"
  },
  "commands": {
    "native": "auto",
    "nativeSkills": "auto",
    "restart": true,
    "ownerDisplay": "raw"
  },
  "session": {
    "dmScope": "per-channel-peer"
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_***REDACTED***",
      "appSecret": "***REDACTED***",
      "connectionMode": "websocket",
      "domain": "feishu",
      "groupPolicy": "open"
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "YOUR_GATEWAY_TOKEN"
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "newapi": {
        "baseUrl": "https://YOUR_NEWAPI_DOMAIN/v1",
        "apiKey": "YOUR_NEWAPI_API_KEY",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          {
            "id": "model-y",
            "name": "model-y"
          }
        ],
        "headers": {
          "User-Agent": "OpenClaw/2026.3"
        }
      }
    }
  }
}
```

### 2.2 为什么这个方法有效

如果 provider/WAF 针对某些 UA 做了策略（拦截、挑战、降权），那就会出现：

- 请求内容完全一样
- 但只要 UA 不同，返回码就不同

这时在 OpenClaw provider 侧显式设置 `headers.User-Agent`，等价于“让请求看起来不像被拦的那类客户端”。

---

## 3. 执行步骤（最短闭环）

### 3.1 修改配置

```bash
nano ~/.openclaw/openclaw.json
```

在你的 `models.providers.newapi` 下增加/修改：

```json
"headers": {
  "User-Agent": "OpenClaw/2026.3"
}
```

### 3.2 重启 Gateway

```bash
openclaw gateway restart
```

### 3.3 用 curl 验证（建议永远保留这一步）

```bash
curl http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "model": "newapi/model-y",
    "messages": [{"role": "user", "content": "hello"}],
    "stream": true
  }'
```

预期：

- 从 403 变成 200（或至少不再是“被拦截”的 403）

---

## 4. 可选：更像浏览器的 User-Agent

如果 `OpenClaw/2026.3` 仍被拦，可以试更“浏览器化”的 UA：

```json
"headers": {
  "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
}
```

---
