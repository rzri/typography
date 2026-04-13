---
title: Ubuntu 安装 OpenClaw
pubDate: 2026-03-21
categories: ['笔记']
description: 'Ubuntu 系统下 OpenClaw 的安装流程，包括环境准备、安装步骤、模型配置、飞书通道配置及 Skills 安装。'
slug: openclaw-ubuntu-install
---

## 目标

在 Ubuntu 上完成 OpenClaw 的安装与可用性验证，并配置：

- Gateway 本地模式（token 鉴权）
- 模型 Provider（以 OpenAI 协议为例）
- 飞书通道（websocket）
- Skills（通过 ClawHub 安装）
- 常见故障排查：403（WAF/网关拦截 User-Agent）

本文按“可复制、可验证、可回滚”的思路整理。

---

## 0. 环境准备

### 0.1 基础依赖

确保：

- Ubuntu 可联网
- 已安装 Node.js（建议 LTS）与 npm/pnpm
- 当前用户对目标目录有写权限

常用自检：

```bash
node -v
npm -v
pnpm -v
```

### 0.2 建议的目录结构

OpenClaw 常见目录（以 root 用户为例）：

- 配置文件：`~/.openclaw/openclaw.json`
- 工作区：`~/.openclaw/workspace/`

提示：工作区和配置文件通常包含敏感信息（token、appSecret、apiKey），不要提交到公开仓库。

---

## 1. 安装 OpenClaw

> 具体安装方式取决于你的发布渠道（npm、二进制、脚本）。这里给出验证思路与关键命令。

安装完成后，优先确认 CLI 可用：

```bash
openclaw --help
openclaw gateway status
```

如果 gateway 还没启动：

```bash
openclaw gateway start
```

---

## 2. 核心配置：openclaw.json

配置文件路径：

```bash
nano ~/.openclaw/openclaw.json
```

### 2.1 一份“可运行”的模板（示意，已做脱敏）

说明：

- `apiKey`、`appSecret`、`token` 这类敏感字段必须替换成你自己的
- 示例中的 `headers.User-Agent` 是为了解决某些 Provider/WAF 403 拦截问题（见下文 4）

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/ubuntu/.openclaw/workspace",
      "model": {
        "primary": "newapi/gpt-5.2",
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
      "appId": "cli_xxx",
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
      "token": "***REDACTED***"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "nodes": {
      "denyCommands": [
        "camera.snap",
        "camera.clip",
        "screen.record",
        "contacts.add",
        "calendar.add",
        "reminders.add",
        "sms.send"
      ]
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "newapi": {
        "baseUrl": "https://YOUR_PROVIDER_DOMAIN/v1",
        "apiKey": "***REDACTED***",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          {
            "id": "gpt-5.2",
            "name": "gpt-5.2"
          }
        ],
        "headers": {
          "User-Agent": "OpenClaw/2026.3.22"
        }
      }
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.3.22",
    "lastTouchedAt": "2026-03-21T00:00:00.000Z"
  }
}
```

### 2.2 配置完成后的重启

```bash
openclaw gateway restart
```

---

## 3. 连通性验证（Gateway Token）

用 curl 验证 OpenClaw Gateway → Provider 的链路。

```bash
curl http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "model": "newapi/gpt-5.2",
    "messages": [{"role": "user", "content": "hello"}],
    "stream": true
  }'
```

预期：

- 能正常返回流式内容（或非流式内容，取决于你的请求）
- 不出现 401/403/5xx

---

## 4. 403 故障排查：不是模型不可用，而是请求头被拦

### 4.1 现象

- 同样的请求内容
- 通过某些 Provider 的网关/WAF 会直接返回 **403**
- 看起来像“模型不可用”，但实际上是 **请求头被拦截**

### 4.2 已验证的根因模式

某些 WAF/网关会拦截特定的 `User-Agent`。

典型触发点（示例）：

- `User-Agent: OpenAI/JS 6.26.0`

同样请求，换一个 User-Agent 可能直接变成 200。

### 4.3 修复方式（OpenClaw 配置）

在 `models.providers.<provider>` 下增加 `headers` 字段，覆盖 User-Agent：

```json
"headers": {
  "User-Agent": "OpenClaw/2026.3.22"
}
```

更“伪装浏览器”的版本也可用（可选）：

```json
"headers": {
  "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
}
```

建议：

- 先用最简版本（例如 `OpenClaw/版本号`）验证
- 若仍被拦再换浏览器 UA

---

## 5. Skills 安装（ClawHub）

### 5.1 安装 ClawHub CLI

```bash
npm i -g clawhub
clawhub --help
```

### 5.2 搜索并安装 skill

示例：安装 Git Essentials 与 SSH Essentials。

```bash
clawhub search "Git Essentials"
clawhub search "Ssh Essentials"

clawhub install git-essentials
clawhub install ssh-essentials

clawhub list
```

安装目录默认是当前目录下的 `./skills/`（OpenClaw 工作区通常是 `~/.openclaw/workspace`）。

---

## 6. 安全与运维建议（强烈建议）

1. **不要把密钥、token、appSecret、apiKey 写进公开仓库**
   - 文档可用 `***REDACTED***` 或 `YOUR_xxx` 占位
2. Provider 403 这类问题，优先：
   - 用 curl 直连验证
   - 再通过 OpenClaw Gateway 验证
   - 最后改 `headers.User-Agent`
3. 修改配置后固定做三件事：
   - `openclaw gateway restart`
   - `openclaw gateway status`
   - curl 验证

---

## 7. 快速检查清单（复制即用）

```bash
openclaw gateway status
openclaw gateway restart

curl http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "model": "newapi/gpt-5.2",
    "messages": [{"role": "user", "content": "ping"}],
    "stream": true
  }'
```
