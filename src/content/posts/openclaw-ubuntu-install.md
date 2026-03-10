---
title: Ubuntu 系统安装 OpenClaw 完全指南
pubDate: 2026-03-10
categories: ['笔记']
description: 'Ubuntu 系统下 OpenClaw 的完整安装流程，包括环境准备、OpenClaw 安装、大模型配置、飞书通道配置及 Skills 安装。'
slug: openclaw-ubuntu-install
---

# Ubuntu 系统安装 OpenClaw 完全指南

本文档详细记录在 Ubuntu 系统上从零开始安装和配置 OpenClaw 的完整流程，涵盖环境准备、OpenClaw 安装、大模型配置、飞书通道配置以及 Skills 安装。

## 目录

1. [环境准备](#环境准备)
2. [OpenClaw 安装](#openclaw-安装)
3. [大模型配置](#大模型配置)
4. [飞书通道配置](#飞书通道配置)
5. [Skills 安装](#skills-安装)
6. [验证与测试](#验证与测试)
7. [常见问题](#常见问题)

---

## 环境准备

### 系统要求

- **操作系统**: Ubuntu 20.04 LTS 或更高版本
- **Node.js**: 22 或更高版本
- **内存**: 建议 2GB 以上
- **磁盘空间**: 建议 5GB 以上可用空间

### 1. 更新系统包

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. 安装基础依赖

```bash
sudo apt install -y curl git build-essential
```

### 3. 安装 Node.js 22

OpenClaw 要求 Node.js 22 或更高版本。推荐使用 NodeSource 官方源安装：

```bash
# 下载并执行 NodeSource 安装脚本
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -

# 安装 Node.js
sudo apt-get install -y nodejs

# 验证安装
node -v
npm -v
```

预期输出：
```
v22.x.x
10.x.x
```

### 4. 配置 npm 全局包路径（可选）

如果遇到权限问题，可以配置 npm 使用用户目录：

```bash
# 创建全局包目录
mkdir -p ~/.npm-global

# 配置 npm 使用新目录
npm config set prefix ~/.npm-global

# 添加到环境变量
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## OpenClaw 安装

### 方法一：安装脚本（推荐）

这是最简单和最推荐的安装方式，会自动处理所有依赖：

```bash
# 执行官方安装脚本
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装脚本会自动：
- 检测并安装 Node.js（如果缺失）
- 安装 Git
- 通过 npm 安装 OpenClaw
- 运行初始化向导

### 方法二：npm 手动安装

如果你已经配置好 Node.js 环境，可以直接使用 npm 安装：

```bash
# 全局安装 OpenClaw
npm install -g openclaw@latest

# 如果遇到 sharp 构建错误，使用以下命令
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest

# 安装网关服务
openclaw onboard --install-daemon
```

### 验证安装

```bash
# 检查 OpenClaw 版本
openclaw --version

# 运行健康检查
openclaw doctor

# 查看状态
openclaw status
```

---

## 大模型配置

OpenClaw 支持多种大模型提供商，这里介绍几种常用的配置方式。

### 1. 运行配置向导

```bash
openclaw onboard
```

在向导中可以选择：
- 快速入门（Quickstart）
- 高级模式（Advanced）
- 手动配置（Manual）

### 2. 配置阿里云通义千问（Qwen）

Qwen 提供免费层 OAuth 访问（2000 次请求/天）：

```bash
# 启用 Qwen 插件
openclaw plugins enable qwen-portal-auth

# 重启网关
openclaw gateway restart

# 登录 Qwen
openclaw models auth login --provider qwen-portal --set-default
```

这会运行 Qwen 的设备代码 OAuth 流程，并在 `models.json` 中写入配置。

**可用模型 ID：**
- `qwen-portal/coder-model` - 代码模型
- `qwen-portal/vision-model` - 视觉模型

**切换模型：**
```bash
openclaw models set qwen-portal/coder-model
```

### 3. 配置其他模型提供商

#### Anthropic (Claude)

```bash
openclaw models auth setup-token --provider anthropic
```

#### OpenAI

```bash
openclaw models auth add
# 按提示输入 API Key
```

#### Moonshot (Kimi)

```bash
openclaw models auth add
# 选择 Moonshot 提供商并输入 API Key
```

### 4. 手动配置文件

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "qwen-portal/coder-model"
      }
    }
  },
  models: {
    providers: {
      "qwen-portal": {
        "authType": "oauth",
        "baseUrl": "https://portal.qwen.ai/v1"
      }
    }
  }
}
```

### 5. 验证模型配置

```bash
# 查看已配置的模型
openclaw models list

# 查看模型状态
openclaw models status

# 测试模型连接
openclaw models status --probe
```

---

## 飞书通道配置

### 1. 安装飞书插件

```bash
openclaw plugins install @openclaw/feishu
```

### 2. 创建飞书应用

#### 2.1 访问飞书开放平台

访问 [飞书开放平台](https://open.feishu.cn/app) 并登录。

> **注意**: 国际版（Lark）用户使用 https://open.larksuite.com/app

#### 2.2 创建企业应用

1. 点击「创建企业应用」
2. 填写应用名称和描述
3. 选择应用图标

#### 2.3 获取凭证

在「凭证与基础信息」页面，复制：
- **App ID**（格式：`cli_xxx`）
- **App Secret**

> ⚠️ **重要**: 妥善保管 App Secret，不要泄露

#### 2.4 配置权限

在「权限管理」页面，点击「批量导入」，粘贴以下权限配置：

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": ["aily:file:read", "aily:file:write", "im:chat.access_event.bot_p2p_chat:read"]
  }
}
```

#### 2.5 启用机器人能力

在「应用功能」>「机器人」中：
1. 启用机器人能力
2. 设置机器人名称

#### 2.6 配置事件订阅

> ⚠️ **重要**: 在配置事件订阅前，确保：
> 1. 已执行 `openclaw channels add` 添加飞书通道
> 2. 网关正在运行（`openclaw gateway status`）

在「事件订阅」页面：
1. 选择「使用长连接接收事件」（WebSocket）
2. 添加事件：`im.message.receive_v1`

#### 2.7 发布应用

1. 在「版本管理与发布」中创建版本
2. 提交审核并发布
3. 等待管理员审批（企业应用通常自动审批）

### 3. 配置 OpenClaw 飞书通道

#### 方法一：使用向导（推荐）

```bash
openclaw channels add
```

选择 **Feishu**，然后输入 App ID 和 App Secret。

#### 方法二：配置文件

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  channels: {
    feishu: {
      enabled: true,
      dmPolicy: "pairing",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          botName: "My AI assistant"
        }
      }
    }
  }
}
```

### 4. 启动并测试

```bash
# 启动网关
openclaw gateway

# 查看网关状态
openclaw gateway status

# 查看日志
openclaw logs --follow
```

### 5. 配对审批

默认情况下，机器人会回复配对码。在飞书中向机器人发送消息后，使用以下命令审批：

```bash
# 查看配对请求
openclaw pairing list feishu

# 审批配对
openclaw pairing approve feishu <CODE>
```

### 6. 获取用户和群组 ID

#### 用户 ID（open_id）

格式：`ou_xxx`

**方法一**（推荐）：
1. 启动网关并与机器人私聊
2. 运行 `openclaw logs --follow` 查看 `open_id`

**方法二**：
```bash
openclaw pairing list feishu
```

#### 群组 ID（chat_id）

格式：`oc_xxx`

**方法一**（推荐）：
1. 在群组中 @提及机器人
2. 运行 `openclaw logs --follow` 查看 `chat_id`

### 7. 群组配置示例

#### 允许所有群组，需要 @提及（默认）

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open"
    }
  }
}
```

#### 仅允许特定群组

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx", "oc_yyy"]
    }
  }
}
```

#### 配置群组内发送者白名单

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          allowFrom: ["ou_user1", "ou_user2"]
        }
      }
    }
  }
}
```

---

## Skills 安装

Skills 是 OpenClaw 的扩展功能模块，提供各种专用能力。

### 1. 查看可用 Skills

```bash
# 列出所有技能
openclaw skills list

# 查看技能详情
openclaw skills info <skill-name>

# 检查技能就绪状态
openclaw skills check
```

### 2. 使用 ClawHub 安装 Skills

ClawHub 是 OpenClaw 的技能市场：

```bash
# 搜索技能
npx clawhub search <keyword>

# 安装技能
npx clawhub install <skill-name>

# 更新已安装的技能
npx clawhub update

# 同步所有技能到最新版本
npx clawhub sync
```

### 3. 常用 Skills 推荐

#### 天气技能

```bash
npx clawhub install weather
```

提供当前天气和预报功能。

#### Git 技能

```bash
npx clawhub install git-essentials
```

提供 Git 版本控制命令和工作流。

#### SSH 技能

```bash
npx clawhub install ssh-essentials
```

提供 SSH 远程访问、密钥管理、隧道等功能。

#### 网络技能

```bash
npx clawhub install network
```

提供 TCP/IP、DNS、路由和网络诊断工具。

#### 图片生成技能

```bash
npx clawhub install image-generate
```

使用内置脚本生成图片。

#### 视频生成技能

```bash
npx clawhub install video-generate
```

使用脚本生成视频。

#### 自我改进技能

```bash
npx clawhub install self-improvement
```

捕获学习、错误和修正，实现持续改进。

### 4. 手动安装 Skills

从 ClawHub 下载技能包后：

```bash
# 解压到技能目录
mkdir -p ~/.openclaw/workspace/skills/<skill-name>
cd ~/.openclaw/workspace/skills/<skill-name>

# 按照技能的 README 配置
```

### 5. 验证 Skills 安装

```bash
# 查看已安装的技能
openclaw skills list --eligible

# 详细输出（包含缺失的依赖）
openclaw skills check --verbose
```

---

## 验证与测试

### 1. 系统健康检查

```bash
# 运行完整诊断
openclaw doctor

# 深度检查
openclaw doctor --deep
```

### 2. 网关状态

```bash
# 查看网关状态
openclaw gateway status

# 查看实时日志
openclaw logs --follow

# 重启网关
openclaw gateway restart
```

### 3. 测试消息发送

```bash
# 发送测试消息（需要配置通道）
openclaw message send --target <user_id> --message "Hello from OpenClaw"
```

### 4. 打开控制面板

```bash
# 在浏览器中打开控制面板
openclaw dashboard
```

访问地址：`http://127.0.0.1:18789/`

### 5. 测试对话

在飞书中向机器人发送消息，测试：
- 文本回复
- 命令响应（如 `/status`、`/reset`）
- 文件处理

---

## 常见问题

### 1. `openclaw: command not found`

**原因**: npm 全局 bin 目录不在 PATH 中

**解决方案**:

```bash
# 查找 npm 全局前缀
npm prefix -g

# 添加到 PATH（~/.bashrc 或 ~/.zshrc）
export PATH="$(npm prefix -g)/bin:$PATH"

# 重新加载配置
source ~/.bashrc
```

### 2. sharp 构建错误

**原因**: 系统安装了全局 libvips

**解决方案**:

```bash
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

### 3. 飞书机器人不响应

**检查清单**:
- [ ] 应用已发布并审批
- [ ] 事件订阅包含 `im.message.receive_v1`
- [ ] 已启用长连接
- [ ] 应用权限配置完整
- [ ] 网关正在运行：`openclaw gateway status`
- [ ] 查看日志：`openclaw logs --follow`

### 4. 群组中机器人无响应

**检查清单**:
- [ ] 机器人已添加到群组
- [ ] 使用 @提及机器人（默认行为）
- [ ] `groupPolicy` 未设置为 `"disabled"`
- [ ] 检查 `requireMention` 配置

### 5. 模型连接失败

**解决方案**:

```bash
# 检查模型配置
openclaw models status

# 测试连接
openclaw models status --probe

# 重新认证
openclaw models auth login --provider <provider-name>
```

### 6. 权限错误（EACCES）

**原因**: npm 全局目录权限问题

**解决方案**:

```bash
# 使用用户目录
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
export PATH=~/.npm-global/bin:$PATH

# 添加到 ~/.bashrc 永久生效
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## 参考资源

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [GitHub 仓库](https://github.com/openclaw/openclaw)
- [ClawHub 技能市场](https://clawhub.com)
- [Discord 社区](https://discord.com/invite/clawd)
- [飞书开放平台](https://open.feishu.cn)

---

## 更新日志

- **2026-03-10**: 初始版本，完整记录 Ubuntu 安装流程
