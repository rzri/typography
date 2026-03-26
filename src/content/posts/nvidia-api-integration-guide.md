---
title: Nvidia API 接入 OpenClaw
pubDate: 2026-03-26
categories: ['笔记']
description: '记录 Nvidia 免费 API 接入 OpenClaw 的流程：API Key 申请、模型配置，支持 Kimi K2.5、Qwen3.5、GLM-5 及 Llama 3.3 等模型。'
slug: nvidia-api-integration-guide
---

# Nvidia API 接入 OpenClaw

本文档详细记录如何将 Nvidia 免费 API 接入 OpenClaw，配置多款主流开源模型，实现零成本的 AI Agent 运行环境。

> **Nvidia API 优势**：完全免费、模型丰富、支持 Qwen3.5、GLM-5、Kimi K2.5、Llama 3.3 等主流开源模型，适合日常轻度使用和临时应急场景。

## 目录

1. [Nvidia API Key 申请](#nvidia-api-key-申请)
2. [OpenClaw 模型配置](#openclaw-模型配置)
3. [模型列表](#模型列表)
4. [API 使用验证](#api-使用验证)
5. [常见问题排查](#常见问题排查)

---

## Nvidia API Key 申请

### 1. 访问 Nvidia 模型平台

访问 Nvidia 官方模型平台并注册登录：

- **平台地址**：[https://build.nvidia.com/models](https://build.nvidia.com/models)
- **账号要求**：需要完成邮箱验证，建议使用工作或常用邮箱

### 2. 创建 API Key

**操作步骤**：

1. 登录成功后，点击右上角 **头像图标**
2. 在下拉菜单中选择 **"API Keys"**
3. 点击 **"Generate API Key"** 按钮创建新密钥
4. 系统会生成一串密钥字符，**立即复制并妥善保存**

> ⚠️ **重要提示**：
> - API Key 只显示一次，关闭页面后无法再次查看
> - 不要将 API Key 提交到代码仓库或公开分享
> - 建议存储在环境变量或加密配置文件中

### 3. API Key 安全存储

**推荐方式**（环境变量）：

```bash
# 在 ~/.openclaw/.env 文件中存储
echo 'NVIDIA_API_KEY=your_api_key_here' >> ~/.openclaw/.env

# 或在系统环境变量中设置
export NVIDIA_API_KEY="your_api_key_here"

# 添加到 ~/.bashrc 永久生效
echo 'export NVIDIA_API_KEY="your_api_key_here"' >> ~/.bashrc
source ~/.bashrc
```

---

## OpenClaw 模型配置

### 1. 编辑配置文件

```bash
# 编辑 OpenClaw 配置文件
vim ~/.openclaw/openclaw.json
```

### 2. 添加 Nvidia 模型配置

在 `models` 节点下添加 Nvidia 配置（完整配置示例）：

```json5
{
  "models": {
    "mode": "merge",
    "providers": {
      "nvidia": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "apiKey": "${NVIDIA_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "moonshotai/kimi-k2.5",
            "name": "Kimi K2.5",
            "reasoning": true,
            "input": ["text", "image"],
            "contextWindow": 262144,
            "maxTokens": 32768
          },
          {
            "id": "minimaxai/minimax-m2.1",
            "name": "Minimax M2.1",
            "reasoning": true,
            "input": ["text", "image"],
            "contextWindow": 131072,
            "maxTokens": 16384
          },
          {
            "id": "qwen/qwen3.5-397b-a17b",
            "name": "Qwen3.5-397b-A17B",
            "reasoning": true,
            "input": ["text", "image"],
            "contextWindow": 400000,
            "maxTokens": 65536
          },
          {
            "id": "z-ai/glm-5",
            "name": "GLM-5",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 202752,
            "maxTokens": 16384
          },
          {
            "id": "meta/llama-3.3-70b-instruct",
            "name": "Llama 3.3 70B Instruct",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 131072,
            "maxTokens": 16384
          }
        ]
      }
    }
  }
}
```

### 3. 配置默认模型

在 `agents` 节点下设置默认模型：

```json5
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "nvidia/moonshotai/kimi-k2.5",
        "fallback": "nvidia/qwen/qwen3.5-397b-a17b"
      }
    }
  }
}
```

### 4. 重启服务生效

```bash
# 重启 OpenClaw 网关服务
openclaw gateway restart

# 验证服务状态
openclaw gateway status
```

---

## 模型列表

### 已配置模型参数

| 模型 ID | 模型名称 | 上下文窗口 | 最大 Token | 支持输入 | 推理能力 |
|---------|----------|------------|------------|----------|----------|
| `moonshotai/kimi-k2.5` | Kimi K2.5 | 262,144 | 32,768 | 文本、图片 | ✅ |
| `minimaxai/minimax-m2.1` | Minimax M2.1 | 131,072 | 16,384 | 文本、图片 | ✅ |
| `qwen/qwen3.5-397b-a17b` | Qwen3.5-397B-A17B | 400,000 | 65,536 | 文本、图片 | ✅ |
| `z-ai/glm-5` | GLM-5 | 202,752 | 16,384 | 文本 | ✅ |
| `meta/llama-3.3-70b-instruct` | Llama 3.3 70B Instruct | 131,072 | 16,384 | 文本 | ✅ |

### 模型选型建议

| 场景 | 推荐模型 | 理由 |
|------|----------|------|
| **超长文本处理** | Kimi K2.5 | 262K 上下文，擅长文档分析 |
| **编程任务** | Qwen3.5-397B | 400K 上下文，代码能力强 |
| **逻辑推理** | GLM-5 / Llama 3.3 | 推理优化，准确率高 |
| **多模态任务** | Kimi K2.5 / Qwen3.5 | 支持图片输入 |
| **日常对话** | Llama 3.3 70B | 响应快，通用能力强 |

---

## API 使用验证

### 1. 测试模型调用

```bash
# 执行测试命令，验证模型是否正常调用
openclaw chat --model "nvidia/moonshotai/kimi-k2.5" --message "测试 Nvidia API 接入，输出当前时间和模型名称"
```

### 2. 预期输出

成功输出以下内容即为配置生效：

```json
{
  "model": "nvidia/moonshotai/kimi-k2.5",
  "response": "当前时间：2026-03-26 11:30:00，使用模型：Kimi K2.5，Nvidia API 接入成功！",
  "status": "success"
}
```

### 3. 测试其他模型

```bash
# 测试 Qwen3.5 模型
openclaw chat --model "nvidia/qwen/qwen3.5-397b-a17b" --message "你好，请用中文自我介绍"

# 测试 GLM-5 模型
openclaw chat --model "nvidia/z-ai/glm-5" --message "解释一下什么是量子计算"

# 测试 Llama 3.3 模型
openclaw chat --model "nvidia/meta/llama-3.3-70b-instruct" --message "Write a short poem about coding"
```

### 4. 查看模型状态

```bash
# 查看当前模型配置
openclaw models status

# 测试模型连接
openclaw models status --probe
```

---

## 常见问题排查

### 1. API Key 配置失败

**症状**：模型调用返回 401 认证错误

**解决方案**：

```bash
# 检查 API Key 是否正确设置
echo $NVIDIA_API_KEY

# 验证 .env 文件内容
cat ~/.openclaw/.env

# 重新设置环境变量
export NVIDIA_API_KEY="your_correct_api_key"

# 重启服务
openclaw gateway restart
```

**检查清单**：
- [ ] API Key 字符完整，无多余空格
- [ ] Nvidia 账号已完成邮箱验证
- [ ] API Key 未过期或被禁用

### 2. 模型调用失败

**症状**：返回 500 错误或超时

**解决方案**：

```bash
# 测试 Nvidia API 连通性
curl -I https://integrate.api.nvidia.com/v1

# 检查网络代理设置
echo $HTTP_PROXY
echo $HTTPS_PROXY

# 查看 OpenClaw 日志
openclaw logs --follow | grep -i nvidia
```

**可能原因**：
- 网络连接问题
- 模型临时不可用
- 请求频率超限

### 3. 限流问题

**症状**：返回 429 Too Many Requests

**解决方案**：

```bash
# 配置调用频率限制
openclaw config set models.rateLimit 2

# 切换到备用模型
openclaw config set agents.defaults.model.primary "nvidia/qwen/qwen3.5-397b-a17b"

# 等待 30-60 秒后重试
```

**限流预防**：
- 避免短时间内高频调用
- 批量任务拆分执行
- 配置自动切换到备用模型

### 4. 服务重启失败

**症状**：`openclaw gateway restart` 报错

**解决方案**：

```bash
# 查看错误日志
journalctl -u openclaw

# 检查配置文件格式
cat ~/.openclaw/openclaw.json | jq .

# 备份并重新初始化配置
mv ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
openclaw config init
```

---

## 配置参数说明

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `mode` | 合并多模型配置 | `"merge"` |
| `baseUrl` | Nvidia API 基础地址 | `https://integrate.api.nvidia.com/v1` |
| `apiKey` | API 认证密钥 | `${NVIDIA_API_KEY}` |
| `api` | API 协议类型 | `"openai-completions"` |
| `reasoning` | 是否支持推理任务 | `true` |
| `contextWindow` | 上下文窗口大小（tokens） | `262144` |
| `maxTokens` | 单次生成最大 Token 数 | `32768` |
| `input` | 支持的输入类型 | `["text", "image"]` |

---

## 参考资源

- [Nvidia 模型平台](https://build.nvidia.com/models)
- [Nvidia API 文档](https://docs.api.nvidia.com)
- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [ClawHub 技能市场](https://clawhub.com)

---

## 更新日志

- **2026-03-26**: 初始版本，记录 Nvidia API 接入完整流程，新增 Llama 3.3 70B 模型支持
