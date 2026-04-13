---
title: Ubuntu 安装 OpenClaw（含 Gateway、通道、模型与 403 排查）
pubDate: 2026-03-21
categories: ['笔记']
description: '在 Ubuntu 上安装并跑通 OpenClaw：Gateway 启动与鉴权、飞书通道接入、模型 Provider 配置、curl 自检，以及 403（User-Agent 被 WAF/网关拦截）的定位与修复方法。'
slug: openclaw-ubuntu-install-guide
---

## 写在前面：这篇笔记解决什么

目标是把 OpenClaw 在 Ubuntu 上从“装上”变成“能稳定跑起来”，并能快速定位常见问题。

你最终会得到：

- OpenClaw CLI 可用
- Gateway 可启动、可重启、可用 token 鉴权
- 通道（以飞书为例）能连上
- 模型 Provider（OpenAI 兼容接口）可用
- 遇到 403 时，能用一套固定套路定位到“被 WAF/网关拦了请求头”，并通过配置修复

注意：本文**不包含任何真实站点域名、API Key、Token、AppSecret**，统一用占位符表示。

---

## 1. 基础环境准备

### 1.1 依赖自检

```bash
node -v
npm -v
pnpm -v
```

如果你只想验证 OpenClaw 本身，Node 版本的细节不重要；但如果你要装一些 CLI 工具或技能生态，建议用稳定的 LTS。

### 1.2 关键目录（建议记住）

- 配置文件：`~/.openclaw/openclaw.json`
- 工作区：`~/.openclaw/workspace/`

提醒：

- `openclaw.json` 里通常会出现密钥/Token/Secret，一旦写错位置或泄露，排查会很痛苦。
- 写教程或贴日志时，优先做脱敏：`***REDACTED***` / `YOUR_XXX`。

---

## 2. 安装与启动验证

安装方式取决于你自己的发布渠道（npm / 二进制 / 安装脚本）。安装完成后，先做这两步验证：

```bash
openclaw --help
openclaw gateway status
```

如果 gateway 未运行：

```bash
openclaw gateway start
```

改完配置后，常用重启：

```bash
openclaw gateway restart
```

---

## 3. openclaw.json：一份可运行的“骨架配置”（已脱敏）

编辑：

```bash
nano ~/.openclaw/openclaw.json
```

下面是一份**最小可运行骨架**（示意）。你需要替换占位符：

- `YOUR_PROVIDER_BASE_URL`（例如 `https://example.com/v1`）
- `YOUR_PROVIDER_API_KEY`
- `YOUR_GATEWAY_TOKEN`
- 飞书相关 `appId/appSecret`

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/ubuntu/.openclaw/workspace",
      "model": {
        "primary": "provider-x/model-y",
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
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "provider-x": {
        "baseUrl": "YOUR_PROVIDER_BASE_URL",
        "apiKey": "YOUR_PROVIDER_API_KEY",
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

改完之后重启：

```bash
openclaw gateway restart
openclaw gateway status
```

---

## 4. 用 curl 做链路自检（强烈建议）

当你不确定“是模型问题、网关问题、还是通道问题”时，最靠谱的办法是：**先用 curl 从本机打到 Gateway**。

```bash
curl http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "model": "provider-x/model-y",
    "messages": [{"role": "user", "content": "hello"}],
    "stream": true
  }'
```

预期：

- 返回 200
- body 能持续输出（stream=true）或正常返回 JSON

如果 curl 都不通，就先别看通道/机器人侧的表现，先把网关到 provider 的链路打通。

---

## 5. 403 排查：常见根因是请求头被 WAF/网关拦

### 5.1 典型误判

很多 403 看起来像“模型不可用/账号没权限”，但实际是：

- Provider 前面有 WAF/网关
- 拦截策略命中（尤其是某些 `User-Agent`）

### 5.2 一个高概率触发点：User-Agent

已知有些场景会对类似下面的 UA 做拦截（示例）：

- `User-Agent: OpenAI/JS 6.26.0`

同样的请求内容，如果你把 UA 换掉，就可能直接从 403 变 200。

### 5.3 修复方式：在 provider 配置里显式覆盖 headers

在 `models.providers.<你的provider>` 下设置：

```json
"headers": {
  "User-Agent": "OpenClaw/2026.3"
}
```

如果仍被拦，再换成更像浏览器的 UA（可选）：

```json
"headers": {
  "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
}
```

然后：

1. `openclaw gateway restart`
2. 重新 curl 验证

---

## 6. Skills：用 ClawHub 安装

### 6.1 安装 ClawHub CLI

```bash
npm i -g clawhub
clawhub --help
```

### 6.2 搜索、安装、查看

```bash
clawhub search "Git Essentials"
clawhub search "Ssh Essentials"

clawhub install git-essentials
clawhub install ssh-essentials

clawhub list
```

说明：

- 默认安装到当前目录的 `./skills/`
- 建议在 OpenClaw 工作区执行（例如 `~/.openclaw/workspace`）

---

## 7. 最后给自己留一份排障清单

```bash
# 1) Gateway 状态
openclaw gateway status

# 2) 重启（改完配置必做）
openclaw gateway restart

# 3) curl 自检（最快定位链路问题）
curl http://localhost:18789/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_GATEWAY_TOKEN" \
  -d '{
    "model": "provider-x/model-y",
    "messages": [{"role": "user", "content": "ping"}],
    "stream": true
  }'
```

如果返回 403：

- 优先怀疑 WAF/网关拦截
- 在 provider 配置加 `headers.User-Agent` 后再试
