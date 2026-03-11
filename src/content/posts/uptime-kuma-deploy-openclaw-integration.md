---
title: Uptime Kuma 监控部署与 OpenClaw 数据集成指南
pubDate: 2026-03-11
categories: ['笔记', '监控', 'OpenClaw', 'Python']
description: 'Uptime Kuma 监控工具的非 Docker 安装、PM2 进程管理、阿里云安全组配置，以及通过 OpenClaw 直接读取 SQLite 数据库生成 Word 巡检报告（含中文字体设置、恶意链接检测）的完整流程。'
slug: uptime-kuma-deploy-openclaw-integration
---

# Uptime Kuma 监控部署与 OpenClaw 数据集成指南

本文档详细记录 Uptime Kuma 监控工具从安装部署到与 OpenClaw 集成的完整流程，包括非 Docker 环境安装、PM2 进程管理、防火墙配置、**直接读取 SQLite 数据库**及**生成 Word 巡检报告**，并包含**恶意链接判断功能**和**中文字体设置**。

## 目录

1. [Uptime Kuma 简介](#uptime-kuma-简介)
2. [环境准备](#环境准备)
3. [安装 Uptime Kuma](#安装-uptime-kuma)
4. [PM2 进程管理](#pm2-进程管理)
5. [端口配置与安全组](#端口配置与安全组)
6. [配置网站监控](#配置网站监控)
7. [OpenClaw 集成 - 读取 SQLite 数据库](#openclaw-集成---读取-sqlite-数据库)
8. [Word 报表生成（含中文字体设置）](#word-报表生成含中文字体设置)
9. [恶意链接检测功能](#恶意链接检测功能)
10. [按监控类型生成定制图表](#按监控类型生成定制图表)
11. [常见问题排查](#常见问题排查)

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
/root/uptime-kuma/data/kuma.db
```

### 数据库表结构

Uptime Kuma 使用 SQLite 存储数据，主要表：

| 表名 | 说明 |
|------|------|
| `monitor` | 监控项配置（名称、URL、类型等） |
| `heartbeat` | 心跳记录（每次检测的结果） |

### 核心查询 SQL

```sql
-- 获取所有监控项
SELECT id, name FROM monitor;

-- 获取指定监控项的历史心跳数据
SELECT time, status, ping, msg 
FROM heartbeat 
WHERE monitor_id=? AND time >= ?
ORDER BY time DESC;
```

---

## Word 报表生成（含中文字体设置）

### 完整功能脚本说明

提供的完整脚本（111.txt）包含以下核心功能：

**1. 参数解析**
- 支持 OpenClaw 对话调用（JSON 字符串）
- 支持命令行直接调用
- 支持时间维度：7 天、15 天、30 天、6 个月、12 个月

**2. 中文字体设置（关键！）**

```python
from docx.shared import Pt, RGBColor
from docx.oxml.ns import qn

def set_chinese_font(run, font_name="宋体", size=10, color=RGBColor(0, 0, 0)):
    """强制设置 Word 中文显示字体"""
    run.font.name = font_name
    run._element.rPr.rFonts.set(qn("w:eastAsia"), font_name)  # 关键：设置东亚字体
    run.font.size = Pt(size)
    run.font.color.rgb = color
```

**使用示例：**
```python
# 设置标题字体（黑体，14 号）
doc.add_heading(f"监控总览（近{time_dimension}）", level=1)
for run in doc.paragraphs[-1].runs:
    set_chinese_font(run, "黑体", 14)

# 设置表格字体（宋体，10 号）
run = row_cells[0].paragraphs[0].add_run(cell_val)
set_chinese_font(run, "宋体", 10)

# 设置错误提示字体（红色）
set_chinese_font(para.runs[0], "宋体", 11, RGBColor(220, 20, 60))
```

**3. 核心统计信息表格化**

```python
def add_core_stats_table(doc, stats):
    """添加核心统计信息表格（替代文本展示）"""
    doc.add_heading("核心统计信息", level=3)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 12)
    
    # 创建 2 列 4 行的统计表格
    table = doc.add_table(rows=4, cols=2)
    table.style = "Table Grid"
    table.alignment = WD_TABLE_ALIGNMENT.CENTER
    table.width = Inches(6)  # 固定表格宽度
    
    # 表格内容
    stats_data = [
        ["总检测次数", f"{stats['总检测次数']} 次"],
        ["在线次数", f"{stats['在线次数']} 次"],
        ["离线次数", f"{stats['离线次数']} 次"],
        ["可用性", stats['可用性']]
    ]
    
    # 填充表格并设置字体
    for i in range(4):
        row_cells = table.rows[i].cells
        run1 = row_cells[0].paragraphs[0].add_run(stats_data[i][0])
        set_chinese_font(run1, "黑体", 10)  # 指标名称用黑体
        run2 = row_cells[1].paragraphs[0].add_run(stats_data[i][1])
        set_chinese_font(run2, "宋体", 10)  # 指标值用宋体
```

**4. 报表结构**

```
监控总报表_近 30 天_20260311_172200.docx
├── 报表标题（黑体，20 号，居中）
├── 生成时间（宋体，12 号，居中）
├── 监控总览表（所有监控项汇总）
├── 各监控项详细章节
│   ├── 监控项名称（黑体，13 号）
│   ├── 核心统计信息表格
│   ├── 可视化图表
│   └── 最近 10 次监控摘要
└── 恶意链接检测章节（最底部）
```

---

## 恶意链接检测功能

### 检测逻辑

```python
def check_malicious_links() -> dict:
    """
    随机爬取 50 个链接，检测恶意/钓鱼链接
    """
    # 1. 爬取网页所有链接
    resp = requests.get(TARGET_URL, headers=headers, timeout=15)
    soup = BeautifulSoup(resp.text, "html.parser")
    
    # 2. 提取所有有效链接（去重）
    base_url = f"{urlparse(TARGET_URL).scheme}://{urlparse(TARGET_URL).netloc}"
    all_links = set()
    for a in soup.find_all("a", href=True):
        href = a["href"]
        if not href or href.startswith("#") or href.startswith("javascript:"):
            continue
        full_url = urljoin(base_url, href)
        if full_url.startswith(("http://", "https://")):
            all_links.add(full_url)
    
    # 3. 随机选择 50 个链接检测
    link_list = list(all_links)
    random.shuffle(link_list)
    target_links = link_list[:MAX_LINK_COUNT]
    
    # 4. 检测规则
    phishing_keywords = ["login", "signin", "verify", "secure", "bank", "pay", 
                         "wallet", "account", "update", "auth"]
    suspicious_suffixes = [".xyz", ".top", ".work", ".click", ".gq", 
                          ".tk", ".ml", ".cf", ".ga"]
    malicious_domains = {"evil.com", "malicious.com", "phish.com", "hack.com"}
    
    for link in target_links:
        parsed = urlparse(link)
        is_suspicious = False
        is_phishing = False
        
        # 检测钓鱼关键词
        link_lower = link.lower()
        for keyword in phishing_keywords:
            if keyword in link_lower:
                is_phishing = True
                break
        
        # 检测可疑域名后缀
        if any(parsed.netloc.endswith(suffix) for suffix in suspicious_suffixes):
            is_suspicious = True
        
        # 检测恶意域名
        if parsed.netloc in malicious_domains:
            is_suspicious = True
        
        # 检测异常字符/超长 URL
        if len(link) > 200 or re.search(r"[^\w\./:_-]", link):
            is_suspicious = True
        
        # 分类统计
        if is_phishing:
            result["phishing_links"].append(link)
        elif is_suspicious:
            result["suspicious_links"].append(link)
        else:
            result["safe_links"] += 1
```

### 生成饼状图可视化

```python
def generate_malicious_detection_chart(detection_result):
    """生成恶意链接检测饼状图（中文标签）"""
    safe = detection_result["safe_links"]
    suspicious = len(detection_result["suspicious_links"])
    phishing = len(detection_result["phishing_links"])
    
    labels = ["安全链接", "可疑链接", "钓鱼链接"]  # 中文标签
    sizes = [safe, suspicious, phishing]
    colors = ["#32CD32", "#FFD700", "#DC143C"]
    
    # 生成饼图（中文标题）
    fig, ax = plt.subplots(figsize=(6, 6))
    ax.pie(sizes, labels=labels, autopct="%1.1f%%", colors=colors, startangle=90)
    ax.set_title(f"BJLOT 恶意链接检测结果（随机{MAX_LINK_COUNT}个链接）", fontsize=12)
    
    buf = BytesIO()
    fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
    buf.seek(0)
    plt.close(fig)
    return [("恶意链接检测饼状图", buf)]
```

### 添加到 Word 报表

```python
def add_malicious_detection_section(doc, detection_result):
    """添加恶意链接检测章节到报表"""
    # 章节标题
    doc.add_heading("BJLOT Malicious Link Detection Report", level=2)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 13)
    
    # 显示检测基本信息
    basic_info = doc.add_paragraph()
    basic_info.add_run(f"目标网址：{TARGET_URL}\n")
    basic_info.add_run(f"总链接数：{detection_result['total_links_crawled']}\n")
    basic_info.add_run(f"随机检测链接数：{detection_result['random_links_count']}\n")
    set_chinese_font(basic_info.runs[0], "宋体", 11)
    
    # 显示安全状态提示
    if not detection_result["suspicious_links"] and not detection_result["phishing_links"]:
        safe_para = doc.add_paragraph("✅ 随机检测的链接中未发现可疑或钓鱼链接，所有链接均安全！")
        set_chinese_font(safe_para.runs[0], "宋体", 11, RGBColor(0, 128, 0))
    
    # 强制添加饼状图
    doc.add_heading("检测结果可视化", level=3)
    charts = generate_malicious_detection_chart(detection_result)
    for chart_title, chart_buf in charts:
        chart_para = doc.add_paragraph(f"{chart_title}:")
        set_chinese_font(chart_para.runs[0], "宋体", 10)
        doc.add_picture(chart_buf, width=Inches(5))
```

---

## 按监控类型生成定制图表

### Matplotlib 中文支持配置

```python
import matplotlib.pyplot as plt

# 修复 matplotlib 中文显示
plt.rcParams["font.family"] = ["WenQuanYi Micro Hei", "WenQuanYi Zen Hei", 
                               "AR PL UKai CN", "AR PL UMing CN"]
plt.rcParams["axes.unicode_minus"] = False
```

### 1. DNSv4/DNSv6 监控：解析状态折线图

```python
if "dnsv4" in monitor_name_lower or "dnsv6" in monitor_name_lower:
    fig, ax = plt.subplots(figsize=(10, 4))
    ax.plot(df["检测时间"], df["状态码"], color="#1E90FF", marker="o", markersize=2, linewidth=1.5)
    ax.set_title(f"{monitor_name} - 解析状态趋势（近{time_dimension}）", fontsize=12)
    ax.set_xlabel("检测时间", fontsize=10)
    ax.set_ylabel("解析状态（1=成功，0=失败）", fontsize=10)
    ax.set_ylim(-0.1, 1.1)
    ax.set_yticks([0, 1])
    ax.set_yticklabels(["失败", "成功"])
```

### 2. HTTPS 监控：可用性面积图

```python
elif "https" in monitor_name_lower:
    fig, ax = plt.subplots(figsize=(10, 4))
    ax.fill_between(df["检测时间"], df["状态码"], color="#32CD32", alpha=0.6)
    ax.plot(df["检测时间"], df["状态码"], color="#228B22", linewidth=1)
    ax.set_title(f"{monitor_name} - HTTPS 可用性趋势（近{time_dimension}）", fontsize=12)
    ax.set_xlabel("检测时间", fontsize=10)
    ax.set_ylabel("服务状态（1=可用，0=不可用）", fontsize=10)
```

### 3. PING/PORT 监控：双 Y 轴折线图

```python
elif "ping" in monitor_name_lower or "port" in monitor_name_lower:
    fig, ax1 = plt.subplots(figsize=(10, 4))
    
    # 左 Y 轴：响应时间
    color1 = "#FF6347"
    ax1.set_xlabel("检测时间", fontsize=10)
    ax1.set_ylabel("响应时间 (ms)", color=color1, fontsize=10)
    ax1.plot(df["检测时间"], df["响应时间 (ms)"], color=color1, marker="o", markersize=2)
    
    # 右 Y 轴：连通状态
    ax2 = ax1.twinx()
    color2 = "#1E90FF"
    ax2.set_ylabel("连通状态（1=通，0=不通）", color=color2, fontsize=10)
    ax2.plot(df["检测时间"], df["状态码"], color=color2, marker="s", markersize=2)
    ax2.set_ylim(-0.1, 1.1)
    ax2.set_yticks([0, 1])
    ax2.set_yticklabels(["不通", "通"])
    
    fig.suptitle(f"{monitor_name} - 响应时间&连通状态趋势（近{time_dimension}）", fontsize=12)
```

### 4. Keyword 关键字监控：异常占比扇形图

```python
elif "keyword" in monitor_name_lower or "关键字" in monitor_name_lower:
    status_counts = df["状态"].value_counts()
    labels = [f"{status}({count}次)" for status, count in status_counts.items()]
    colors = ["#32CD32" if status == "在线" else "#DC143C" for status in status_counts.index]
    
    fig, ax = plt.subplots(figsize=(6, 6))
    ax.pie(status_counts.values, labels=labels, autopct="%1.1f%%", colors=colors, startangle=90)
    ax.set_title(f"{monitor_name} - 关键字监控异常占比（近{time_dimension}）", fontsize=12)
```

---

## 依赖安装

```bash
# 基础依赖
pip install requests beautifulsoup4 python-docx matplotlib pandas

# 安装中文字体（Linux）
apt install fonts-wqy-microhei fonts-wqy-zenhei fonts-arphic-ukai fonts-arphic-uming
```

---

## 使用方法

```bash
# 交互式选择时间维度
python3 111.txt

# 命令行指定时间维度
python3 111.txt 30 天
python3 111.txt 7 天
python3 111.txt 6 个月

# OpenClaw 调用（JSON 参数）
python3 111.txt '{"time_dimension":"30 天"}'
```

**输出示例：**
```
✅ 汇总总报表已生成：/root/.openclaw/reports/监控总报表_近 30 天_20260311_172200.docx
```

---

## 常见问题排查

### 问题 1：Word 中文显示为方框

**原因**：未正确设置东亚字体

**解决方案**：
```python
# 必须使用 set_chinese_font 函数
run._element.rPr.rFonts.set(qn("w:eastAsia"), "宋体")  # 关键！
```

### 问题 2：Matplotlib 中文显示为方框

**原因**：未配置中文字体

**解决方案**：
```python
plt.rcParams["font.family"] = ["WenQuanYi Micro Hei", "WenQuanYi Zen Hei"]
plt.rcParams["axes.unicode_minus"] = False
```

### 问题 3：SQLite 数据库读取失败

**可能原因**：
1. 数据库路径错误 → 确认 `/root/uptime-kuma/data/kuma.db` 存在
2. 数据库被锁定 → 确保 Uptime Kuma 进程正常运行
3. 权限不足 → 使用 `chmod 644 /root/uptime-kuma/data/kuma.db`

**测试命令**：
```bash
# 检查数据库文件
ls -la /root/uptime-kuma/data/kuma.db

# 使用 sqlite3 命令行测试
sqlite3 /root/uptime-kuma/data/kuma.db "SELECT name FROM monitor LIMIT 5;"
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
- [python-docx 文档](https://python-docx.readthedocs.io/)
- [SQLite Python 文档](https://docs.python.org/zh-cn/3/library/sqlite3.html)

---

## 更新日志

- **2026-03-11**: 初始版本，完整记录 Uptime Kuma 安装、配置及 OpenClaw 集成流程
  - 采用 SQLite 直接读取方式（非 API）
  - 增加恶意链接判断功能
  - 完善 Word 报表生成（含中文字体设置）
  - 按监控类型生成定制图表
