---
title: Uptime Kuma 监控部署与 OpenClaw 数据集成指南
pubDate: 2026-03-11
categories: ['笔记', '监控', 'OpenClaw']
description: 'Uptime Kuma 监控工具的非 Docker 安装、PM2 进程管理、阿里云安全组配置，以及通过 OpenClaw 读取监控数据生成巡检报告的完整流程。'
slug: uptime-kuma-deploy-openclaw-integration
---

# Uptime Kuma 监控部署与 OpenClaw 数据集成指南

本文档详细记录 Uptime Kuma 监控工具从安装部署到与 OpenClaw 集成的完整流程，包括非 Docker 环境安装、PM2 进程管理、防火墙配置、API 调用及自动化巡检报告生成。

## 目录

1. [Uptime Kuma 简介](#uptime-kuma-简介)
2. [环境准备](#环境准备)
3. [安装 Uptime Kuma](#安装-uptime-kuma)
4. [PM2 进程管理](#pm2-进程管理)
5. [端口配置与安全组](#端口配置与安全组)
6. [配置网站监控](#配置网站监控)
7. [OpenClaw 集成 - 读取监控数据](#openclaw-集成---读取监控数据)
8. [自动化月度巡检报告](#自动化月度巡检报告)
9. [常见问题排查](#常见问题排查)

---

## Uptime Kuma 简介

**Uptime Kuma** 是一款轻量易用的开源监控工具，支持：

- **监控类型**：HTTP/HTTPS、TCP、Ping、DNS、Docker、数据库等 20+ 种
- **告警渠道**：90+ 告警方式（钉钉/企业微信/飞书/邮件/Webhook）
- **核心能力**：HTTP 关键词校验、JSON 校验、SSL 证书预警、可视化仪表盘、状态页
- **优势**：零依赖、Docker 一键部署、界面简洁、社区活跃（GitHub 57k+ Star）
- **适用场景**：个人站点、中小项目、快速搭建状态监控

---

## 环境准备

### 系统要求

- **操作系统**：主流 Linux 发行版（Debian、Ubuntu、Fedora、ArchLinux）或 Windows 10/Server 2012 R2+
- **Node.js**：≥ 20.4
- **Git**：必需
- **PM2**：推荐（用于后台运行）

### 1. 检查 Node.js 版本

```bash
node -v
```

若版本低于 20.4，使用 nvm 升级：

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# 重新加载环境变量
source ~/.bashrc

# 安装 Node.js 20.4+
nvm install 20.4
nvm use 20.4

# 验证版本
node -v
```

### 2. 检查 Git 是否安装

```bash
git --version
```

未安装则根据系统安装：

```bash
# Debian/Ubuntu
apt install git -y

# Fedora
dnf install git -y

# Arch
pacman -S git
```

---

## 安装 Uptime Kuma

### 1. 克隆源码

```bash
git clone https://github.com/louislam/uptime-kuma.git
cd uptime-kuma
```

### 2. 执行初始化安装

```bash
npm run setup
```

此命令会安装所有依赖并构建前端。

---

## PM2 进程管理

### 1. 安装 PM2

```bash
# 全局安装 PM2 及日志轮转插件
npm install pm2 -g && pm2 install pm2-logrotate
```

### 2. 启动 Uptime Kuma

```bash
# 在 uptime-kuma 目录下执行
pm2 start server/server.js --name uptime-kuma
```

### 3. 验证启动状态

```bash
# 查看 PM2 进程列表
pm2 status

# 查看实时日志
pm2 logs uptime-kuma --lines 100

# 查看端口监听
ss -tuln | grep 3001
```

正常输出示例：
```
tcp LISTEN 0 511 0.0.0.0:3001 0.0.0.0:*
```

### 4. 设置开机自启

```bash
# 生成 PM2 开机自启配置
pm2 startup

# 保存当前进程列表
pm2 save
```

### 5. PM2 运维命令速查

| 操作场景 | 命令 |
|---------|------|
| 查看运行状态 | `pm2 status uptime-kuma` |
| 查看实时日志 | `pm2 logs uptime-kuma --lines 100` |
| 重启服务 | `pm2 restart uptime-kuma` |
| 停止服务 | `pm2 stop uptime-kuma` |
| 删除进程 | `pm2 delete uptime-kuma` |
| 验证端口监听 | `ss -tuln \| grep 3001` |
| 手动临时启动 | `cd ~/uptime-kuma && node server/server.js` |
| 开机自启生效 | `pm2 startup && pm2 save` |

---

## 端口配置与安全组

### 场景一：系统防火墙（ufw）

```bash
# 允许公网访问 3001 端口
sudo ufw allow 3001/tcp

# 仅允许内网访问（更安全）
sudo ufw allow from 127.0.0.1 to any port 3001 proto tcp

# 重新加载规则
sudo ufw reload

# 验证规则
sudo ufw status numbered
```

### 场景二：阿里云安全组（推荐）

云主机优先配置安全组，系统防火墙次之。

**配置步骤：**

1. 登录阿里云控制台 → ECS 服务器 → 安全组
2. 添加安全组规则：

| 配置项 | 取值/操作 |
|--------|----------|
| 规则方向 | 入方向 |
| 协议类型 | TCP |
| 端口范围 | 3001/3001 |
| 授权对象 | 本机公网 IP/32（最安全）或 0.0.0.0/0（公网访问） |
| 优先级 | 100（默认） |
| 规则名称 | Uptime Kuma 3001 端口 |

3. 保存生效（1-2 分钟内生效）

**验证访问：**
```bash
# 本地浏览器访问
http://云主机公网IP:3001
```

---

## 配置网站监控

### 方法一：网页界面配置（推荐）

1. **登录后台**：访问 `http://云主机 IP:3001`，首次需创建管理员账号

2. **添加监控项**：点击右下角 **+ Add Monitor**

3. **配置参数**（以北京体彩网为例）：

| 配置项 | 取值 |
|--------|------|
| Monitor Type | HTTP(s) |
| Friendly Name | 北京体彩网 |
| URL | https://www.bjlot.com.cn |
| Method | GET（默认） |
| Port | 留空（HTTPS 默认 443） |
| Check Interval | 1 minute |
| Retry Interval | 30 seconds |
| Timeout | 10 seconds |
| Keywords | 北京体彩网（可选，校验页面内容） |
| SSL Certificate Expiry Check | 勾选（SSL 证书过期预警） |

4. **配置告警**：在 Notifications 区域勾选告警渠道（邮件/钉钉/企业微信等）

5. **保存生效**：点击 Save

### 方法二：配置文件导入（批量添加）

1. 在网页添加一个测试监控项
2. Settings → Import / Export → Export 下载 JSON 模板
3. 修改 JSON 配置，添加新监控项：

```json
{
  "id": 2,
  "name": "北京体彩网",
  "url": "https://www.bjlot.com.cn",
  "method": "GET",
  "timeout": 10000,
  "interval": 60,
  "retryInterval": 30,
  "accepted_statuscodes": [200, 201, 202, 301, 302],
  "keyword": "北京体彩网",
  "expiryNotification": true,
  "monitorType": "http"
}
```

4. Settings → Import / Export → Import 上传修改后的 JSON

---

## OpenClaw 集成 - 读取监控数据

### 前置准备：获取 API Token

1. 登录 Uptime Kuma 后台
2. 点击右上角头像 → **API Tokens** → **Add Token**
3. 填写名称（如 `openclaw-access`），权限选择 `Read Only`
4. 保存并复制 Token（如 `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`）

### Python 脚本：调用 Uptime Kuma API

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Uptime Kuma 监控数据读取脚本
用于 OpenClaw 集成，获取监控项状态和历史心跳数据
"""

import requests
import json
from datetime import datetime, timedelta

# ==================== 配置项 ====================
KUMA_URL = "http://你的云主机 IP:3001"  # 替换为实际地址
KUMA_TOKEN = "你的 API Token"  # 替换为生成的 Token
MONITOR_NAME = "北京体彩网"  # 要读取的监控项名称
# =============================================

# 初始化请求头（带 Token 认证）
headers = {
    "Authorization": f"Bearer {KUMA_TOKEN}",
    "Content-Type": "application/json"
}


def get_all_monitors():
    """获取所有监控项列表（含 ID、名称、状态）"""
    url = f"{KUMA_URL}/api/monitor"
    try:
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()
        monitors = response.json()
        print("✅ 所有监控项列表：")
        for m in monitors:
            status = '在线' if m['status'] == 1 else '离线'
            print(f"  ID: {m['id']}, 名称：{m['name']}, 状态：{status}")
        return monitors
    except Exception as e:
        print(f"❌ 获取监控列表失败：{str(e)}")
        return None


def get_monitor_heartbeats(monitor_id, days=30):
    """获取指定监控项的历史心跳数据（默认近 30 天）"""
    end_time = datetime.now().timestamp() * 1000  # 毫秒时间戳
    start_time = (datetime.now() - timedelta(days=days)).timestamp() * 1000
    
    url = f"{KUMA_URL}/api/heartbeat/{monitor_id}"
    params = {
        "start": int(start_time),
        "end": int(end_time)
    }
    
    try:
        response = requests.get(url, headers=headers, params=params, timeout=15)
        response.raise_for_status()
        heartbeats = response.json()
        
        # 解析关键数据
        parsed_data = []
        for hb in heartbeats:
            parsed_data.append({
                "时间": datetime.fromtimestamp(hb["time"]/1000).strftime("%Y-%m-%d %H:%M:%S"),
                "状态": "成功" if hb["status"] == 1 else "失败",
                "响应时间 (ms)": hb["ping"] if hb["ping"] else 0,
                "错误信息": hb["msg"] if hb["msg"] else "无"
            })
        
        print(f"\n✅ 监控项【{MONITOR_NAME}】近{days}天历史数据（前 10 条）：")
        print(json.dumps(parsed_data[:10], ensure_ascii=False, indent=2))
        return parsed_data
    except Exception as e:
        print(f"❌ 获取历史心跳数据失败：{str(e)}")
        return None


def calculate_statistics(heartbeats):
    """计算监控统计数据"""
    if not heartbeats:
        return None
    
    total = len(heartbeats)
    success = sum(1 for h in heartbeats if h["状态"] == "成功")
    failed = total - success
    availability = (success / total * 100) if total > 0 else 0
    avg_response = sum(h["响应时间 (ms)"] for h in heartbeats) / total if total > 0 else 0
    
    return {
        "总检测次数": total,
        "成功次数": success,
        "失败次数": failed,
        "可用性 (%)": round(availability, 2),
        "平均响应时间 (ms)": round(avg_response, 2)
    }


def main():
    # 第一步：获取所有监控项，找到目标监控项的 ID
    monitors = get_all_monitors()
    if not monitors:
        return
    
    target_monitor = next((m for m in monitors if m["name"] == MONITOR_NAME), None)
    if not target_monitor:
        print(f"❌ 未找到监控项：{MONITOR_NAME}")
        return
    
    print(f"\n✅ 找到目标监控项：ID={target_monitor['id']}, 名称={target_monitor['name']}")
    
    # 第二步：获取历史心跳数据
    heartbeats = get_monitor_heartbeats(target_monitor['id'], days=30)
    if not heartbeats:
        return
    
    # 第三步：计算统计数据
    stats = calculate_statistics(heartbeats)
    print(f"\n📊 统计数据：")
    print(json.dumps(stats, ensure_ascii=False, indent=2))
    
    # 第四步：导出为 JSON 文件（供 OpenClaw 使用）
    output = {
        "监控项": MONITOR_NAME,
        "统计周期": "近 30 天",
        "生成时间": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "统计数据": stats,
        "详细数据": heartbeats
    }
    
    output_file = f"uptime_kuma_report_{datetime.now().strftime('%Y%m%d')}.json"
    with open(output_file, 'w', encoding='utf-8') as f:
        json.dump(output, f, ensure_ascii=False, indent=2)
    
    print(f"\n💾 数据已导出至：{output_file}")


if __name__ == "__main__":
    main()
```

### 使用方法

1. 修改脚本中的配置项（`KUMA_URL`、`KUMA_TOKEN`、`MONITOR_NAME`）
2. 安装依赖：`pip install requests`
3. 运行脚本：`python3 uptime_kuma_monitor.py`
4. 输出 JSON 文件可供 OpenClaw 读取并生成报告

---

## 自动化月度巡检报告

### OpenClaw 定时任务配置

在 OpenClaw 中创建 Skill，每月 1 号自动执行：

```bash
# Cron 表达式：每月 1 号 0 点执行
0 0 1 * *
```

### 报告生成流程

1. **拉取数据**：调用 Uptime Kuma API 获取上月监控数据
2. **清洗统计**：
   - 可用性百分比
   - 平均响应时间
   - 故障次数和时长
   - SSL 证书状态
3. **生成图表**：使用 ECharts 或 Matplotlib 生成趋势图
4. **填充模板**：按巡检报告模板生成 Word/PDF
5. **发送报告**：通过邮件/钉钉/企业微信发送

### 报告模板示例

```markdown
# 月度监控巡检报告

**报告周期**: 2026-02-01 ~ 2026-02-28
**生成时间**: 2026-03-01 00:00:00

## 监控概览

| 监控项 | 可用性 | 平均响应时间 | 故障次数 |
|--------|--------|-------------|----------|
| 北京体彩网 | 99.85% | 245ms | 3 |

## 可用性趋势图

[插入 ECharts 趋势图]

## 故障详情

| 时间 | 故障时长 | 错误信息 |
|------|---------|---------|
| 2026-02-05 14:23 | 2 分钟 | Connection timeout |

## SSL 证书状态

| 域名 | 过期时间 | 剩余天数 |
|------|---------|---------|
| www.bjlot.com.cn | 2026-12-15 | 289 |
```

---

## 常见问题排查

### 问题 1：3001 端口未监听

**症状**：`ss -tuln` 看不到 3001 端口

**排查步骤**：

```bash
# 1. 检查 PM2 进程状态
pm2 status

# 2. 查看日志定位错误
pm2 logs uptime-kuma --lines 100

# 3. 清理并重新启动
pm2 stop uptime-kuma && pm2 delete uptime-kuma
cd ~/uptime-kuma
pm2 start server/server.js --name uptime-kuma

# 4. 验证端口
ss -tuln | grep 3001
```

### 问题 2：Node.js 版本不足

```bash
# 检查版本
node -v

# 升级 Node.js
nvm install 20.4
nvm use 20.4

# 重新安装依赖
cd ~/uptime-kuma
rm -rf node_modules package-lock.json
npm run setup
pm2 restart uptime-kuma
```

### 问题 3：API 调用失败

**可能原因**：

1. Token 无效或过期 → 重新生成 Token
2. 网络不通 → 检查服务器能否访问 Kuma 地址
3. 权限不足 → 确认 Token 权限为 Read Only 或更高

**测试命令**：

```bash
curl -H "Authorization: Bearer 你的 Token" \
     http://云主机IP:3001/api/monitor
```

### 问题 4：云主机无法访问

**检查清单**：

- [ ] 阿里云安全组已放行 3001 端口（入方向，TCP）
- [ ] 授权对象配置正确（本机 IP/32 或 0.0.0.0/0）
- [ ] Uptime Kuma 进程正常运行（`pm2 status`）
- [ ] 端口监听正常（`ss -tuln | grep 3001`）

---

## 参考资源

- [Uptime Kuma GitHub](https://github.com/louislam/uptime-kuma)
- [Uptime Kuma 官方文档](https://github.com/louislam/uptime-kuma/wiki)
- [PM2 官方文档](https://pm2.keymetrics.io/docs/usage/quick-start/)
- [阿里云安全组配置](https://help.aliyun.com/document_detail/25471.html)
- [OpenClaw 文档](https://docs.openclaw.ai)

---

## 更新日志

- **2026-03-11**: 初始版本，完整记录 Uptime Kuma 安装、配置及 OpenClaw 集成流程
