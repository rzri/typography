---
title: Uptime Kuma 监控部署与 OpenClaw 数据集成指南
pubDate: 2026-03-11
categories: ['笔记', '监控', 'OpenClaw', 'Python']
description: 'Uptime Kuma 监控工具的非 Docker 安装、PM2 进程管理、阿里云安全组配置，以及通过 OpenClaw 直接读取 SQLite 数据库生成巡检报告并判断恶意链接的完整流程。'
slug: uptime-kuma-deploy-openclaw-integration
---

# Uptime Kuma 监控部署与 OpenClaw 数据集成指南

本文档详细记录 Uptime Kuma 监控工具从安装部署到与 OpenClaw 集成的完整流程，包括非 Docker 环境安装、PM2 进程管理、防火墙配置、**直接读取 SQLite 数据库**及自动化巡检报告生成，并包含**恶意链接判断功能**。

## 目录

1. [Uptime Kuma 简介](#uptime-kuma-简介)
2. [环境准备](#环境准备)
3. [安装 Uptime Kuma](#安装-uptime-kuma)
4. [PM2 进程管理](#pm2-进程管理)
5. [端口配置与安全组](#端口配置与安全组)
6. [配置网站监控](#配置网站监控)
7. [OpenClaw 集成 - 读取 SQLite 数据库](#openclaw-集成---读取-sqlite-数据库)
8. [恶意链接判断功能](#恶意链接判断功能)
9. [自动化月度巡检报告](#自动化月度巡检报告)
10. [常见问题排查](#常见问题排查)

---

## Uptime Kuma 简介

**Uptime Kuma** 是一款轻量易用的开源监控工具，支持：

- **监控类型**：HTTP/HTTPS、TCP、Ping、DNS、Docker、数据库等 20+ 种
- **告警渠道**：90+ 告警方式（钉钉/企业微信/飞书/邮件/Webhook）
- **核心能力**：HTTP 关键词校验、JSON 校验、SSL 证书预警、可视化仪表盘、状态页
- **优势**：零依赖、Docker 一键部署、界面简洁、社区活跃（GitHub 57k+ Star）
- **适用场景**：个人站点、中小项目、快速搭建状态监控
- **数据存储**：SQLite 数据库（`~/uptime-kuma/data/db.sqlite`）

---

## 环境准备

### 系统要求

- **操作系统**：主流 Linux 发行版（Debian、Ubuntu、Fedora、ArchLinux）或 Windows 10/Server 2012 R2+
- **Node.js**：≥ 20.4
- **Git**：必需
- **PM2**：推荐（用于后台运行）
- **Python**：3.8+（用于数据读取脚本）

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
http://云主机公网 IP:3001
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

## OpenClaw 集成 - 读取 SQLite 数据库

> **重要说明**：通过 API 方式读取监控数据需要配置 Token 且功能受限，**推荐直接读取 Uptime Kuma 的 SQLite 数据库**，数据更完整、无需认证、查询更灵活。

### 数据库位置

```bash
~/uptime-kuma/data/db.sqlite
```

### 数据库表结构

Uptime Kuma 使用 SQLite 存储数据，主要表：

| 表名 | 说明 |
|------|------|
| `monitor` | 监控项配置（名称、URL、类型等） |
| `heartbeat` | 心跳记录（每次检测的结果） |
| `notification` | 告警渠道配置 |
| `monitor_notification` | 监控项与告警渠道关联 |

### Python 脚本：读取 SQLite 数据库

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Uptime Kuma 监控数据读取脚本（SQLite 数据库方式）
用于 OpenClaw 集成，直接读取 db.sqlite 获取监控数据
"""

import sqlite3
import json
from datetime import datetime, timedelta
from pathlib import Path

# ==================== 配置项 ====================
# Uptime Kuma 数据目录路径
KUMA_DATA_DIR = Path.home() / "uptime-kuma" / "data"
DB_PATH = KUMA_DATA_DIR / "db.sqlite"
# =============================================


def get_db_connection():
    """获取数据库连接"""
    if not DB_PATH.exists():
        raise FileNotFoundError(f"数据库文件不存在：{DB_PATH}")
    
    conn = sqlite3.connect(str(DB_PATH))
    conn.row_factory = sqlite3.Row  # 使结果可以按列名访问
    return conn


def get_all_monitors():
    """获取所有监控项列表"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT id, name, url, type, interval, hostname, port, path, 
               keyword, expiryNotification, status
        FROM monitor
        ORDER BY id
    """)
    
    monitors = []
    for row in cursor.fetchall():
        monitors.append({
            "id": row["id"],
            "name": row["name"],
            "url": row["url"],
            "type": row["type"],
            "interval": row["interval"],
            "hostname": row["hostname"],
            "port": row["port"],
            "path": row["path"],
            "keyword": row["keyword"],
            "expiry_notification": row["expiryNotification"],
            "status": "在线" if row["status"] == 1 else "离线"
        })
    
    conn.close()
    return monitors


def get_monitor_heartbeats(monitor_id, days=30):
    """获取指定监控项的历史心跳数据"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # 计算时间范围
    start_time = (datetime.now() - timedelta(days=days)).timestamp()
    
    cursor.execute("""
        SELECT time, status, ping, msg
        FROM heartbeat
        WHERE monitor_id = ? AND time >= ?
        ORDER BY time DESC
    """, (monitor_id, start_time))
    
    heartbeats = []
    for row in cursor.fetchall():
        heartbeats.append({
            "时间": datetime.fromtimestamp(row["time"]).strftime("%Y-%m-%d %H:%M:%S"),
            "状态": "成功" if row["status"] == 1 else "失败",
            "响应时间 (ms)": row["ping"] if row["ping"] else 0,
            "错误信息": row["msg"] if row["msg"] else "无"
        })
    
    conn.close()
    return heartbeats


def calculate_statistics(heartbeats):
    """计算监控统计数据"""
    if not heartbeats:
        return None
    
    total = len(heartbeats)
    success = sum(1 for h in heartbeats if h["状态"] == "成功")
    failed = total - success
    availability = (success / total * 100) if total > 0 else 0
    
    # 计算平均响应时间（排除失败请求）
    response_times = [h["响应时间 (ms)"] for h in heartbeats if h["状态"] == "成功" and h["响应时间 (ms)"] > 0]
    avg_response = sum(response_times) / len(response_times) if response_times else 0
    
    return {
        "总检测次数": total,
        "成功次数": success,
        "失败次数": failed,
        "可用性 (%)": round(availability, 2),
        "平均响应时间 (ms)": round(avg_response, 2)
    }


def main():
    # 第一步：获取所有监控项
    print("📊 获取所有监控项...")
    monitors = get_all_monitors()
    
    print(f"✅ 共找到 {len(monitors)} 个监控项：")
    for m in monitors:
        print(f"  ID: {m['id']}, 名称：{m['name']}, 状态：{m['status']}")
    
    # 第二步：选择目标监控项（示例：北京体彩网）
    target_name = "北京体彩网"
    target_monitor = next((m for m in monitors if m["name"] == target_name), None)
    
    if not target_monitor:
        print(f"❌ 未找到监控项：{target_name}")
        return
    
    print(f"\n✅ 选择监控项：{target_monitor['name']} (ID: {target_monitor['id']})")
    
    # 第三步：获取历史心跳数据
    print(f"\n📈 获取近 30 天历史数据...")
    heartbeats = get_monitor_heartbeats(target_monitor['id'], days=30)
    print(f"✅ 共获取 {len(heartbeats)} 条心跳记录")
    
    # 第四步：计算统计数据
    stats = calculate_statistics(heartbeats)
    print(f"\n📊 统计数据：")
    print(json.dumps(stats, ensure_ascii=False, indent=2))
    
    # 第五步：导出为 JSON 文件
    output = {
        "监控项": target_monitor['name'],
        "URL": target_monitor['url'],
        "统计周期": "近 30 天",
        "生成时间": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "统计数据": stats,
        "详细数据": heartbeats[:100]  # 只导出前 100 条详细数据
    }
    
    output_file = f"uptime_kuma_report_{datetime.now().strftime('%Y%m%d')}.json"
    with open(output_file, 'w', encoding='utf-8') as f:
        json.dump(output, f, ensure_ascii=False, indent=2)
    
    print(f"\n💾 数据已导出至：{output_file}")


if __name__ == "__main__":
    main()
```

### 使用方法

```bash
# 安装依赖（SQLite 是 Python 内置模块，无需额外安装）
# 运行脚本
python3 uptime_kuma_sqlite_reader.py
```

### 优势对比

| 方式 | API 读取 | SQLite 直接读取 |
|------|---------|----------------|
| 配置复杂度 | 需要生成 Token | 无需配置 |
| 数据完整性 | 部分字段 | 全部字段 |
| 查询灵活性 | 受限 | 完全灵活 |
| 性能 | 网络请求 | 本地读取 |
| 推荐度 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 恶意链接判断功能

在读取监控数据后，可以添加恶意链接判断功能，检测监控的 URL 是否存在安全风险。

### Python 脚本：恶意链接判断

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
恶意链接判断脚本
检测 URL 是否存在安全风险（钓鱼、恶意软件、异常重定向等）
"""

import re
import socket
import ssl
from urllib.parse import urlparse
from datetime import datetime
import whois


class MaliciousLinkDetector:
    """恶意链接检测器"""
    
    def __init__(self):
        # 常见钓鱼网站特征
        self.phishing_keywords = [
            'login', 'signin', 'account', 'verify', 'secure',
            'update', 'confirm', 'suspended', 'limited'
        ]
        
        # 可疑顶级域名
        self.suspicious_tlds = [
            '.tk', '.ml', '.ga', '.cf', '.gq',  # 免费域名
            '.xyz', '.top', '.club', '.work',   # 低价域名
            '.cc', '.pw', '.ws'                 # 隐私保护域名
        ]
    
    def check_url_structure(self, url):
        """检查 URL 结构"""
        parsed = urlparse(url)
        hostname = parsed.hostname or ""
        
        issues = []
        
        # 检查是否使用 HTTPS
        if parsed.scheme != 'https':
            issues.append("⚠️ 未使用 HTTPS 加密")
        
        # 检查域名长度（过长可能是钓鱼）
        if len(hostname) > 50:
            issues.append("⚠️ 域名过长，可能是钓鱼网站")
        
        # 检查是否包含 IP 地址
        if re.match(r'^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$', hostname):
            issues.append("⚠️ 直接使用 IP 地址，不推荐")
        
        # 检查是否包含可疑关键词
        for keyword in self.phishing_keywords:
            if keyword in hostname.lower():
                issues.append(f"⚠️ 域名包含可疑关键词：{keyword}")
                break
        
        # 检查可疑顶级域名
        for tld in self.suspicious_tlds:
            if hostname.endswith(tld):
                issues.append(f"⚠️ 使用可疑顶级域名：{tld}")
                break
        
        return issues
    
    def check_ssl_certificate(self, url):
        """检查 SSL 证书"""
        parsed = urlparse(url)
        hostname = parsed.hostname or ""
        
        if parsed.scheme != 'https':
            return ["⚠️ 未使用 HTTPS，无法检查 SSL 证书"]
        
        issues = []
        
        try:
            # 获取 SSL 证书信息
            context = ssl.create_default_context()
            with socket.create_connection((hostname, 443), timeout=5) as sock:
                with context.wrap_socket(sock, server_hostname=hostname) as ssock:
                    cert = ssock.getpeercert()
                    
                    # 检查证书有效期
                    not_after = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
                    days_remaining = (not_after - datetime.now()).days
                    
                    if days_remaining < 30:
                        issues.append(f"⚠️ SSL 证书即将过期（剩余{days_remaining}天）")
                    elif days_remaining < 0:
                        issues.append("❌ SSL 证书已过期")
                    
                    # 检查证书颁发机构
                    issuer = dict(x[0] for x in cert['issuer'])
                    if 'Let\'s Encrypt' in str(issuer):
                        pass  # Let's Encrypt 是合法的
                    elif not issuer:
                        issues.append("⚠️ 自签名证书，存在风险")
        
        except Exception as e:
            issues.append(f"❌ SSL 证书检查失败：{str(e)}")
        
        return issues
    
    def check_domain_age(self, url):
        """检查域名注册时长"""
        parsed = urlparse(url)
        hostname = parsed.hostname or ""
        
        # 移除 www.前缀
        if hostname.startswith('www.'):
            hostname = hostname[4:]
        
        try:
            w = whois.whois(hostname)
            creation_date = w.creation_date
            
            # whois 返回的可能是列表
            if isinstance(creation_date, list):
                creation_date = creation_date[0]
            
            if creation_date:
                age_days = (datetime.now() - creation_date).days
                
                if age_days < 30:
                    return [f"⚠️ 域名注册时间很短（{age_days}天），可能是新注册的恶意域名"]
                elif age_days < 90:
                    return [f"⚠️ 域名较新（{age_days}天），需保持警惕"]
        except Exception as e:
            return [f"ℹ️ 无法查询域名注册信息：{str(e)}"]
        
        return []
    
    def check_redirect(self, url):
        """检查 URL 重定向"""
        import requests
        
        try:
            response = requests.get(url, timeout=10, allow_redirects=False)
            
            if response.status_code in [301, 302, 307, 308]:
                redirect_url = response.headers.get('Location', '')
                return [f"⚠️ 存在重定向：{redirect_url}"]
        except Exception as e:
            return [f"❌ 重定向检查失败：{str(e)}"]
        
        return []
    
    def analyze(self, url):
        """综合分析 URL"""
        print(f"\n🔍 分析 URL: {url}")
        print("=" * 60)
        
        all_issues = []
        
        # 1. 检查 URL 结构
        print("\n📋 检查 URL 结构...")
        issues = self.check_url_structure(url)
        all_issues.extend(issues)
        for issue in issues:
            print(f"  {issue}")
        if not issues:
            print("  ✅ URL 结构正常")
        
        # 2. 检查 SSL 证书
        print("\n🔒 检查 SSL 证书...")
        issues = self.check_ssl_certificate(url)
        all_issues.extend(issues)
        for issue in issues:
            print(f"  {issue}")
        if not issues:
            print("  ✅ SSL 证书正常")
        
        # 3. 检查域名注册时长
        print("\n📅 检查域名注册时长...")
        issues = self.check_domain_age(url)
        all_issues.extend(issues)
        for issue in issues:
            print(f"  {issue}")
        if not issues:
            print("  ✅ 域名注册时长正常")
        
        # 4. 检查重定向
        print("\n🔄 检查重定向...")
        issues = self.check_redirect(url)
        all_issues.extend(issues)
        for issue in issues:
            print(f"  {issue}")
        if not issues:
            print("  ✅ 无重定向")
        
        # 综合评估
        print("\n" + "=" * 60)
        if len(all_issues) == 0:
            print("✅ 综合评估：安全")
        elif len(all_issues) <= 2:
            print("⚠️ 综合评估：低风险（发现少量问题）")
        else:
            print("❌ 综合评估：高风险（发现多个问题）")
        
        print(f"\n共发现 {len(all_issues)} 个问题：")
        for issue in all_issues:
            print(f"  - {issue}")
        
        return {
            "url": url,
            "issues": all_issues,
            "risk_level": "安全" if len(all_issues) == 0 else "低风险" if len(all_issues) <= 2 else "高风险"
        }


def main():
    """主函数"""
    detector = MaliciousLinkDetector()
    
    # 示例：检测北京体彩网
    urls = [
        "https://www.bjlot.com.cn",
        # 可以添加更多需要检测的 URL
    ]
    
    results = []
    for url in urls:
        result = detector.analyze(url)
        results.append(result)
    
    # 导出结果
    output_file = f"malicious_link_check_{datetime.now().strftime('%Y%m%d')}.json"
    with open(output_file, 'w', encoding='utf-8') as f:
        json.dump(results, f, ensure_ascii=False, indent=2)
    
    print(f"\n💾 检测结果已导出至：{output_file}")


if __name__ == "__main__":
    main()
```

### 安装依赖

```bash
pip install requests python-whois
```

### 使用方法

```bash
# 运行恶意链接检测
python3 malicious_link_detector.py
```

---

## 自动化月度巡检报告

### OpenClaw 定时任务配置

在 OpenClaw 中创建 Skill，每月 1 号自动执行：

```bash
# Cron 表达式：每月 1 号 0 点执行
0 0 1 * *
```

### 报告生成流程

1. **读取数据**：直接读取 Uptime Kuma 的 SQLite 数据库
2. **清洗统计**：
   - 可用性百分比
   - 平均响应时间
   - 故障次数和时长
   - SSL 证书状态
3. **恶意链接检测**：对监控的 URL 进行安全风险评估
4. **生成图表**：使用 Matplotlib 生成趋势图
5. **填充模板**：按巡检报告模板生成 Word/PDF
6. **发送报告**：通过邮件/钉钉/企业微信发送

### 报告模板示例

```markdown
# 月度监控巡检报告

**报告周期**: 2026-02-01 ~ 2026-02-28
**生成时间**: 2026-03-01 00:00:00

## 监控概览

| 监控项 | 可用性 | 平均响应时间 | 故障次数 | 安全风险 |
|--------|--------|-------------|----------|---------|
| 北京体彩网 | 99.85% | 245ms | 3 | 安全 |

## 可用性趋势图

[插入 Matplotlib 趋势图]

## 故障详情

| 时间 | 故障时长 | 错误信息 |
|------|---------|---------|
| 2026-02-05 14:23 | 2 分钟 | Connection timeout |

## SSL 证书状态

| 域名 | 过期时间 | 剩余天数 |
|------|---------|---------|
| www.bjlot.com.cn | 2026-12-15 | 289 |

## 恶意链接检测结果

| 监控项 | URL | 风险等级 | 问题数量 |
|--------|-----|---------|---------|
| 北京体彩网 | https://www.bjlot.com.cn | 安全 | 0 |
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

### 问题 3：SQLite 数据库读取失败

**可能原因**：

1. 数据库路径错误 → 确认 `~/uptime-kuma/data/db.sqlite` 存在
2. 数据库被锁定 → 确保 Uptime Kuma 进程正常运行
3. 权限不足 → 使用 `chmod 644 ~/uptime-kuma/data/db.sqlite`

**测试命令**：

```bash
# 检查数据库文件
ls -la ~/uptime-kuma/data/db.sqlite

# 使用 sqlite3 命令行测试
sqlite3 ~/uptime-kuma/data/db.sqlite "SELECT name FROM monitor LIMIT 5;"
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
- [SQLite Python 文档](https://docs.python.org/zh-cn/3/library/sqlite3.html)

---

## 更新日志

- **2026-03-11**: 初始版本，完整记录 Uptime Kuma 安装、配置及 OpenClaw 集成流程
  - 采用 SQLite 直接读取方式（非 API）
  - 增加恶意链接判断功能
  - 完善月度巡检报告生成方案
