---
title: Uptime Kuma 监控与 OpenClaw 技能集成
pubDate: 2026-03-11
categories: ['笔记']
description: '记录 Uptime Kuma 监控数据读取脚本开发全过程：从 API 限制到直接读库，从路径错误到格式确认，最终集成 OpenClaw 技能的完整经验。'
slug: uptime-kuma-deploy-openclaw-integration
---

# Uptime Kuma 监控与 OpenClaw 技能集成实战

##  核心目标

### 监控对象
- **Uptime Kuma** 自托管监控服务
- 监控类型：HTTP、TCP、Ping、DNS、Push 等
- 数据源：SQLite 数据库（`kuma.db`）

### 监控工具
- **Uptime Kuma** - 开源监控工具
- **SQLite3** - 数据库查询工具
- **Python3** - 脚本编写语言
- **OpenClaw** - 自动化框架集成

### 自动化工具
- Python 脚本自动读取数据库
- 智能适配监控项名称
- 自动生成可视化报告
- 支持命令行和对话两种调用方式

### 需求
1. ✅ 读取 Kuma 数据库中的监控数据
2. ✅ 统计指定周期内的可用性
3. ✅ 计算响应时间指标（平均、最小、最大）
4. ✅ 生成格式化报告（文本 + JSON）
5. ✅ 集成到 OpenClaw 技能系统
6. ✅ 支持命令行和对话调用

---

##  遇到的困难

### 1. API 功能限制
**问题**：Kuma 的 API 功能有限，无法直接获取完整的历史统计数据

**影响**：
- 无法通过 REST API 获取完整的历史心跳数据
- 需要手动导出或寻找其他方案

**尝试方案**：
- 使用 Kuma API → 数据不完整 ❌
- 导出 CSV → 需要手动处理 ❌
- 直接读数据库 → ✅ 最终方案

---

### 2. 路径错误
**问题**：数据库文件路径不确定

**排查过程**：
```bash
# 可能的路径
~/.kuma/kuma.db          # Docker 默认挂载
/app/data/kuma.db        # Docker 内部路径
/root/.kuma/kuma.db      # 直接安装
./kuma.db                # 当前目录
```

**解决方案**：
- 脚本中实现智能路径查找
- 按优先级尝试多个可能路径
- 找到第一个存在的数据库文件

---

### 3. 数据库格式不对
**问题**：数据库结构不明确，字段名与预期不符

**排查过程**：
```bash
# 使用 sqlite3 确认数据库格式
sqlite3 kuma.db ".tables"
sqlite3 kuma.db ".schema heartbeat"
sqlite3 kuma.db "SELECT * FROM heartbeat LIMIT 5;"
```

**发现的问题**：
- 字段名与预期不符
- 时间戳格式为 Unix 时间戳
- status 字段：1=正常，0=异常

**找到的可读解决方案**：
```sql
-- 正确的心跳查询语句
SELECT 
    COUNT(*) as total,
    SUM(CASE WHEN status = 1 THEN 1 ELSE 0 END) as up_count,
    AVG(response_time) as avg_response_time
FROM heartbeat 
WHERE monitor_id = ? AND time >= ?
```

**数据库名确认**：`kuma.db`

---

##  最终解决方案

### 脚本编写工作流程

#### 第一步：基础功能实现
```python
# 1. 数据库连接
conn = sqlite3.connect(db_path)

# 2. 获取监控列表
cursor.execute("SELECT id, name, type, url FROM monitor")

# 3. 获取心跳统计
cursor.execute("""
    SELECT COUNT(*), AVG(response_time) 
    FROM heartbeat 
    WHERE monitor_id = ? AND time >= ?
""")
```

#### 第二步：脚本优化
- ✅ 智能适配监控项名称
- ✅ 自动查找数据库路径
- ✅ 支持自定义统计周期
- ✅ 错误处理和用户提示

#### 第三步：输出报表格式
**包含图和表**：
```
================================================================================
KUMA 监控报告
================================================================================
生成时间：2026-03-12T09:00:00
统计周期：7 天
================================================================================

📊 监控项名称
   类型：HTTP
   URL: https://example.com
   可用率：99.95%
   总检测次数：1008
   正常次数：1007
   平均响应时间：125.34ms
   最小/最大响应时间：45.20ms / 890.50ms

================================================================================
```

---

## 🎯 最终脚本代码

### 文件结构
```
~/.openclaw/skills/kuma/
├── run.py          # 主脚本
├── SKILL.md        # 技能说明文档
└── skill.json      # 技能配置文件
```

### run.py 核心内容
```python
import sys
import json
# 解析 OpenClaw 传入的参数
if __name__ == "__main__":
    time_dimension = "30天"  # 默认值
    try:
        # 方式1：OpenClaw 对话调用（JSON 字符串）
        if len(sys.argv) > 1 and sys.argv[1].startswith("{"):
            params = json.loads(sys.argv[1])
            time_dimension = params.get("time_dimension", "30天")
        # 方式2：命令行调用（直接传时间维度）
        elif len(sys.argv) > 1:
            time_dimension = sys.argv[1]
    except Exception as e:
        print(f"⚠️ 参数解析失败，使用默认值 30天：{e}")
    # 转换时间维度到天数
    time_mapping = {
        "7": "7天", "15": "15天", "30": "30天",
        "180": "6个月", "365": "12个月",
        "7天": "7天", "15天": "15天", "30天": "30天",
        "6个月": "6个月", "12个月": "12个月"
    }
    time_dimension = time_mapping.get(time_dimension, "30天")
    days_map = {"7天":7, "15天":15, "30天":30, "6个月":180, "12个月":365}
    days = days_map[time_dimension]
import sqlite3
import json
import sys
import os
import random
import requests
from bs4 import BeautifulSoup
import re
from urllib.parse import urlparse, urljoin
from datetime import datetime, timedelta
from docx import Document
from docx.shared import Inches, Pt, RGBColor
from docx.enum.text import WD_PARAGRAPH_ALIGNMENT
from docx.oxml.ns import qn
from docx.enum.table import WD_TABLE_ALIGNMENT
import matplotlib.pyplot as plt
import pandas as pd
from io import BytesIO
# -------------------------- 基础配置 --------------------------
KUMA_DB_PATH = "/root/uptime-kuma/data/kuma.db"
TIME_OPTIONS = {
    "7天": 7,
    "15天": 15,
    "30天": 30,
    "6个月": 180,
    "12个月": 365
}
REPORT_DIR = "/root/.openclaw/reports/"
# 恶意链接检测配置
TARGET_URL = "https://www.bjlot.com.cn"  # 目标网站（已保留你的配置）
MAX_LINK_COUNT = 50  # 随机爬取50个链接
# 修复matplotlib中文显示
plt.rcParams["font.family"] = ["WenQuanYi Micro Hei", "WenQuanYi Zen Hei", "AR PL UKai CN", "AR PL UMing CN"]
plt.rcParams["axes.unicode_minus"] = False
# 最近N次摘要展示数量
RECENT_CHECK_COUNT = 10
# -------------------------------------------------------------
if not os.path.exists(REPORT_DIR):
    os.makedirs(REPORT_DIR)
# ======================== 恶意链接检测相关函数 ========================
def check_malicious_links() -> dict:
    """
    随机爬取50个链接，检测恶意/钓鱼链接
    返回: 检测结果字典
    """
    result = {
        "total_links_crawled": 0,
        "random_links_count": 0,
        "safe_links": 0,
        "suspicious_links": [],
        "phishing_links": [],
        "error": ""
    }
    
    try:
        # 1. 爬取网页所有链接
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
        }
        resp = requests.get(TARGET_URL, headers=headers, timeout=15)
        resp.raise_for_status()
        soup = BeautifulSoup(resp.text, "html.parser")
        
        # 2. 提取所有有效链接（去重）
        base_url = f"{urlparse(TARGET_URL).scheme}://{urlparse(TARGET_URL).netloc}"
        all_links = set()
        for a in soup.find_all("a", href=True):
            href = a["href"]
            if not href or href.startswith("#") or href.startswith("javascript:"):
                continue
            full_url = urljoin(base_url, href)
            # 只保留HTTP/HTTPS链接
            if full_url.startswith(("http://", "https://")):
                all_links.add(full_url)
        result["total_links_crawled"] = len(all_links)
        
        # 3. 随机选择50个链接（不足50个则全部检测）
        link_list = list(all_links)
        random.shuffle(link_list)
        target_links = link_list[:MAX_LINK_COUNT]
        result["random_links_count"] = len(target_links)
        
        # 4. 恶意/钓鱼链接检测规则
        # 钓鱼关键词（可扩展）
        phishing_keywords = ["login", "signin", "verify", "secure", "bank", "pay", "wallet", "account", "update", "auth"]
        # 可疑域名后缀
        suspicious_suffixes = [".xyz", ".top", ".work", ".click", ".gq", ".tk", ".ml", ".cf", ".ga"]
        # 恶意域名黑名单（示例）
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
            
            # 检测异常字符/超长URL
            if len(link) > 200 or re.search(r"[^\w\./:_-]", link):
                is_suspicious = True
            
            # 分类统计
            if is_phishing:
                result["phishing_links"].append(link)
            elif is_suspicious:
                result["suspicious_links"].append(link)
            else:
                result["safe_links"] += 1
        
        return result
    except Exception as e:
        result["error"] = str(e)
        return result

def generate_malicious_detection_chart(detection_result):
    """生成恶意链接检测饼状图（中文标签）"""
    if detection_result.get("error"):
        return []
    
    # 统计数据
    safe = detection_result["safe_links"]
    suspicious = len(detection_result["suspicious_links"])
    phishing = len(detection_result["phishing_links"])
    labels = ["安全链接", "可疑链接", "钓鱼链接"]  # 中文标签
    sizes = [safe, suspicious, phishing]
    colors = ["#32CD32", "#FFD700", "#DC143C"]
    # 过滤数量为0的分类
    labels = [label for label, size in zip(labels, sizes) if size > 0]
    sizes = [size for size in sizes if size > 0]
    colors = [color for color, size in zip(colors, sizes) if size > 0]
    
    if not sizes:
        # 全是安全链接时强制生成100%安全的饼图
        labels = ["安全链接"]
        sizes = [detection_result["random_links_count"]]
        colors = ["#32CD32"]
    
    # 生成饼图（中文标题）
    fig, ax = plt.subplots(figsize=(6, 6))
    ax.pie(sizes, labels=labels, autopct="%1.1f%%", colors=colors, startangle=90, textprops={"fontsize": 10})
    ax.set_title(f"BJLOT 恶意链接检测结果（随机{MAX_LINK_COUNT}个链接）", fontsize=12)
    buf = BytesIO()
    fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
    buf.seek(0)
    plt.close(fig)
    return [("恶意链接检测饼状图", buf)]  # 中文图表标题

def add_malicious_detection_section(doc, detection_result):
    """添加恶意链接检测章节到报表（中文提示+指定排序）"""
    # 章节标题（保持BJLOT前缀风格）
    doc.add_heading("BJLOT Malicious Link Detection Report", level=2)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 13)
    
    # 1. 显示检测基本信息（中文标签）
    basic_info = doc.add_paragraph()
    basic_info.add_run(f"目标网址: {TARGET_URL}\n")
    basic_info.add_run(f"总链接数: {detection_result['total_links_crawled']}\n")
    basic_info.add_run(f"随机检测链接数: {detection_result['random_links_count']}\n")
    set_chinese_font(basic_info.runs[0], "宋体", 11)
    
    # 2. 显示错误信息（中文提示）
    if detection_result.get("error"):
        error_para = doc.add_paragraph(f"⚠️ 检测失败: {detection_result['error']}")
        set_chinese_font(error_para.runs[0], "宋体", 11, RGBColor(220, 20, 60))
        # 中文标题：检测结果可视化
        doc.add_heading("检测结果可视化", level=3)
        no_data_para = doc.add_paragraph("无可视化数据。")
        set_chinese_font(no_data_para.runs[0], "宋体", 10)
        return
    
    # 3. 显示安全状态提示（中文）
    if not detection_result["suspicious_links"] and not detection_result["phishing_links"]:
        safe_para = doc.add_paragraph("✅ 随机检测的链接中未发现可疑或钓鱼链接，所有链接均安全！")
        set_chinese_font(safe_para.runs[0], "宋体", 11, RGBColor(0, 128, 0))
    else:
        # 显示异常链接（中文标题：可疑链接列表）
        if detection_result["suspicious_links"]:
            doc.add_heading("可疑链接列表", level=3)
            for link in detection_result["suspicious_links"]:
                para = doc.add_paragraph(f"- {link}")
                set_chinese_font(para.runs[0], "宋体", 10)
        
        if detection_result["phishing_links"]:
            doc.add_heading("钓鱼链接列表（高危）", level=3)
            for link in detection_result["phishing_links"]:
                para = doc.add_paragraph(f"- {link}")
                set_chinese_font(para.runs[0], "宋体", 10, RGBColor(220, 20, 60))
    
    # 4. 强制添加饼状图（中文标题：检测结果可视化）
    doc.add_heading("检测结果可视化", level=3)
    charts = generate_malicious_detection_chart(detection_result)
    if charts:
        for chart_title, chart_buf in charts:
            chart_para = doc.add_paragraph(f"{chart_title}:")
            set_chinese_font(chart_para.runs[0], "宋体", 10)
            doc.add_picture(chart_buf, width=Inches(5))
    else:
        # 中文提示
        no_chart_para = doc.add_paragraph("无数据可生成图表。")
        set_chinese_font(no_chart_para.runs[0], "宋体", 10)

# ======================== 原有函数保持不变 ========================
def get_all_monitors():
    """获取所有监控项（保留原始名称）"""
    try:
        conn = sqlite3.connect(KUMA_DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT id, name FROM monitor;")
        monitors = cursor.fetchall()
        conn.close()
        return monitors
    except Exception as e:
        return {"code": -1, "msg": f"获取监控项失败：{str(e)}"}

def get_monitor_data(monitor_id, monitor_name, days):
    """获取监控数据（保留原始名称）"""
    start_time = (datetime.now() - timedelta(days=days)).strftime("%Y-%m-%d %H:%M:%S")
    time_dimension = [k for k, v in TIME_OPTIONS.items() if v == days][0]
    try:
        conn = sqlite3.connect(KUMA_DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()
        cursor.execute("""
            SELECT time, status, ping, msg 
            FROM heartbeat 
            WHERE monitor_id=? AND time >= ?
            ORDER BY time DESC;
        """, (monitor_id, start_time))
        raw_data = cursor.fetchall()
        conn.close()
        if not raw_data:
            return {
                "code": 0,
                "msg": f"【{monitor_name}】近{time_dimension}无数据",
                "monitor_name": monitor_name,
                "time_dimension": time_dimension,
                "data": [],
                "stats": {"总检测次数":0, "在线次数":0, "离线次数":0, "可用性":"0.00%", "平均响应时间(ms)":"0.00"}
            }
        parsed_data = []
        for row in raw_data:
            parsed_data.append({
                "检测时间": row["time"],
                "状态": "在线" if row["status"] == 1 else "离线",
                "响应时间(ms)": row["ping"] if row["ping"] else 0,
                "输出信息": row["msg"] if row["msg"] else "无"
            })
        total = len(parsed_data)
        up = len([d for d in parsed_data if d["状态"] == "在线"])
        down = total - up
        availability = (up / total) * 100 if total > 0 else 0
        avg_ping = sum([d["响应时间(ms)"] for d in parsed_data]) / total if total > 0 else 0
        return {
            "code": 1,
            "msg": f"【{monitor_name}】近{time_dimension}数据查询成功",
            "monitor_name": monitor_name,
            "time_dimension": time_dimension,
            "data": parsed_data,
            "stats": {
                "总检测次数": total,
                "在线次数": up,
                "离线次数": down,
                "可用性": f"{availability:.2f}%",
                "平均响应时间(ms)": f"{avg_ping:.2f}"
            }
        }
    except Exception as e:
        return {
            "code": -1,
            "msg": f"【{monitor_name}】查询失败：{str(e)}",
            "monitor_name": monitor_name,
            "stats": {}
        }

def generate_charts_by_type(monitor_name, detail_data, time_dimension):
    """按监控类型生成定制化中文图表（大小写不敏感匹配）"""
    if not detail_data:
        return []
    df = pd.DataFrame(detail_data)
    df["检测时间"] = pd.to_datetime(df["检测时间"], errors="coerce")
    df = df.dropna(subset=["检测时间"])
    if df.empty:
        return []
    df = df.sort_values("检测时间")
    df["状态码"] = df["状态"].map({"在线": 1, "离线": 0})
    charts = []
    monitor_name_lower = monitor_name.lower()  # 转为小写，大小写不敏感匹配
    # 1. DNSv4/DNSv6监控：解析状态折线图
    if "dnsv4" in monitor_name_lower or "dnsv6" in monitor_name_lower:
        # 解析状态折线图
        fig, ax = plt.subplots(figsize=(10, 4))
        ax.plot(df["检测时间"], df["状态码"], color="#1E90FF", marker="o", markersize=2, linewidth=1.5)
        ax.set_title(f"{monitor_name} - 解析状态趋势（近{time_dimension}）", fontsize=12)
        ax.set_xlabel("检测时间", fontsize=10)
        ax.set_ylabel("解析状态（1=成功，0=失败）", fontsize=10)
        ax.set_ylim(-0.1, 1.1)
        ax.set_yticks([0, 1])
        ax.set_yticklabels(["失败", "成功"])
        ax.grid(alpha=0.3)
        fig.autofmt_xdate()
        buf = BytesIO()
        fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
        buf.seek(0)
        plt.close(fig)
        charts.append(("解析状态趋势图", buf))
    # 2. HTTPS监控：可用性面积图（最佳展示方式）
    elif "https" in monitor_name_lower:
        # 可用性面积图
        fig, ax = plt.subplots(figsize=(10, 4))
        ax.fill_between(df["检测时间"], df["状态码"], color="#32CD32", alpha=0.6)
        ax.plot(df["检测时间"], df["状态码"], color="#228B22", linewidth=1)
        ax.set_title(f"{monitor_name} - HTTPS可用性趋势（近{time_dimension}）", fontsize=12)
        ax.set_xlabel("检测时间", fontsize=10)
        ax.set_ylabel("服务状态（1=可用，0=不可用）", fontsize=10)
        ax.set_ylim(-0.1, 1.1)
        ax.grid(alpha=0.3)
        fig.autofmt_xdate()
        buf = BytesIO()
        fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
        buf.seek(0)
        plt.close(fig)
        charts.append(("HTTPS可用性面积图", buf))
    # 3. PING/PORT监控：双折线图（响应时间+连通状态）
    elif "ping" in monitor_name_lower or "port" in monitor_name_lower:
        # 双Y轴折线图：响应时间 + 连通状态
        fig, ax1 = plt.subplots(figsize=(10, 4))
        
        # 左Y轴：响应时间
        color1 = "#FF6347"
        ax1.set_xlabel("检测时间", fontsize=10)
        ax1.set_ylabel("响应时间(ms)", color=color1, fontsize=10)
        ax1.plot(df["检测时间"], df["响应时间(ms)"], color=color1, marker="o", markersize=2, linewidth=1.5, label="响应时间")
        ax1.tick_params(axis="y", labelcolor=color1)
        ax1.grid(alpha=0.3)
        
        # 右Y轴：连通状态
        ax2 = ax1.twinx()
        color2 = "#1E90FF"
        ax2.set_ylabel("连通状态（1=通，0=不通）", color=color2, fontsize=10)
        ax2.plot(df["检测时间"], df["状态码"], color=color2, marker="s", markersize=2, linewidth=1.5, label="连通状态")
        ax2.tick_params(axis="y", labelcolor=color2)
        ax2.set_ylim(-0.1, 1.1)
        ax2.set_yticks([0, 1])
        ax2.set_yticklabels(["不通", "通"])
        
        # 标题和格式
        fig.suptitle(f"{monitor_name} - 响应时间&连通状态趋势（近{time_dimension}）", fontsize=12)
        fig.autofmt_xdate()
        
        # 保存图片
        buf = BytesIO()
        fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
        buf.seek(0)
        plt.close(fig)
        charts.append(("响应时间&连通状态双折线图", buf))
    # 4. Keyword关键字监控：异常占比扇形图
    elif "keyword" in monitor_name_lower or "关键字" in monitor_name_lower:
        # 异常占比扇形图
        status_counts = df["状态"].value_counts()
        labels = [f"{status}({count}次)" for status, count in status_counts.items()]
        colors = ["#32CD32" if status == "在线" else "#DC143C" for status in status_counts.index]
        fig, ax = plt.subplots(figsize=(6, 6))
        ax.pie(status_counts.values, labels=labels, autopct="%1.1f%%", colors=colors, startangle=90)
        ax.set_title(f"{monitor_name} - 关键字监控异常占比（近{time_dimension}）", fontsize=12)
        buf = BytesIO()
        fig.savefig(buf, format="png", dpi=300, bbox_inches="tight")
        buf.seek(0)
        plt.close(fig)
        charts.append(("关键字异常占比图", buf))
    return charts

def set_chinese_font(run, font_name="宋体", size=10, color=RGBColor(0, 0, 0)):
    """强制设置Word中文显示字体"""
    run.font.name = font_name
    run._element.rPr.rFonts.set(qn("w:eastAsia"), font_name)
    run.font.size = Pt(size)
    run.font.color.rgb = color

def add_core_stats_table(doc, stats):
    """添加核心统计信息表格（替代文本展示）"""
    doc.add_heading("核心统计信息", level=3)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 12)
    
    # 创建2列4行的统计表格
    table = doc.add_table(rows=4, cols=2)
    table.style = "Table Grid"
    table.alignment = WD_TABLE_ALIGNMENT.CENTER
    table.width = Inches(6)  # 固定表格宽度
    
    # 表格内容
    stats_data = [
        ["总检测次数", f"{stats['总检测次数']} 次"],
        ["在线次数", f"{stats['在线次数']} 次"],
        ["离线次数", f"{stats['离线次数']} 次"],
        ["可用性", stats['可用性']],
        ["平均响应时间", f"{stats['平均响应时间(ms)']} ms"]
    ]
    
    # 填充表格（合并最后一行的两个单元格）
    for i in range(4):
        row_cells = table.rows[i].cells
        # 第一列（指标名称）
        run1 = row_cells[0].paragraphs[0].add_run(stats_data[i][0])
        set_chinese_font(run1, "黑体", 10)
        row_cells[0].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
        # 第二列（指标值）
        run2 = row_cells[1].paragraphs[0].add_run(stats_data[i][1])
        set_chinese_font(run2, "宋体", 10)
        row_cells[1].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
    
    # 处理第五行（平均响应时间）
    new_row = table.add_row().cells
    # 合并单元格
    new_row[0].merge(new_row[1])
    para = new_row[0].paragraphs[0]
    para.add_run(f"{stats_data[4][0]}：{stats_data[4][1]}")
    set_chinese_font(para.runs[0], "宋体", 10)
    para.alignment = WD_PARAGRAPH_ALIGNMENT.CENTER

def add_recent_summary(monitor_name, detail_data, doc):
    """添加最近10次监控摘要（按类型定制）"""
    if not detail_data:
        return
    
    # 取最近10条数据
    recent_data = detail_data[:RECENT_CHECK_COUNT]
    monitor_name_lower = monitor_name.lower()
    
    # 1. DNSv4/DNSv6：最近10次解析情况
    if "dnsv4" in monitor_name_lower or "dnsv6" in monitor_name_lower:
        doc.add_heading("最近10次解析情况", level=3)
        for run in doc.paragraphs[-1].runs:
            set_chinese_font(run, "黑体", 12)
        table = doc.add_table(rows=1, cols=3)
        table.style = "Table Grid"
        headers = ["检测时间", "解析状态", "输出信息"]
        hdr_cells = table.rows[0].cells
        for i, header in enumerate(headers):
            run = hdr_cells[i].paragraphs[0].add_run(header)
            set_chinese_font(run, "黑体", 10)
            hdr_cells[i].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
        for item in recent_data:
            row_cells = table.add_row().cells
            run1 = row_cells[0].paragraphs[0].add_run(item["检测时间"])
            set_chinese_font(run1, "宋体", 9)
            run2 = row_cells[1].paragraphs[0].add_run(item["状态"])
            set_chinese_font(run2, "宋体", 9)
            run3 = row_cells[2].paragraphs[0].add_run(item["输出信息"])
            set_chinese_font(run3, "宋体", 9)
    
    # 2. HTTPS：最近10次监控内容
    elif "https" in monitor_name_lower:
        doc.add_heading("最近10次HTTPS监控情况", level=3)
        for run in doc.paragraphs[-1].runs:
            set_chinese_font(run, "黑体", 12)
        table = doc.add_table(rows=1, cols=4)
        table.style = "Table Grid"
        headers = ["检测时间", "服务状态", "响应时间(ms)", "输出信息"]
        hdr_cells = table.rows[0].cells
        for i, header in enumerate(headers):
            run = hdr_cells[i].paragraphs[0].add_run(header)
            set_chinese_font(run, "黑体", 10)
            hdr_cells[i].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
        for item in recent_data:
            row_cells = table.add_row().cells
            run1 = row_cells[0].paragraphs[0].add_run(item["检测时间"])
            set_chinese_font(run1, "宋体", 9)
            run2 = row_cells[1].paragraphs[0].add_run(item["状态"])
            set_chinese_font(run2, "宋体", 9)
            run3 = row_cells[2].paragraphs[0].add_run(str(item["响应时间(ms)"]))
            set_chinese_font(run3, "宋体", 9)
            run4 = row_cells[3].paragraphs[0].add_run(item["输出信息"])
            set_chinese_font(run4, "宋体", 9)
    
    # 3. PING/PORT：最近10次监控情况
    elif "ping" in monitor_name_lower or "port" in monitor_name_lower:
        doc.add_heading("最近10次连通性监控情况", level=3)
        for run in doc.paragraphs[-1].runs:
            set_chinese_font(run, "黑体", 12)
        table = doc.add_table(rows=1, cols=4)
        table.style = "Table Grid"
        headers = ["检测时间", "连通状态", "响应时间(ms)", "输出信息"]
        hdr_cells = table.rows[0].cells
        for i, header in enumerate(headers):
            run = hdr_cells[i].paragraphs[0].add_run(header)
            set_chinese_font(run, "黑体", 10)
            hdr_cells[i].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
        for item in recent_data:
            row_cells = table.add_row().cells
            run1 = row_cells[0].paragraphs[0].add_run(item["检测时间"])
            set_chinese_font(run1, "宋体", 9)
            run2 = row_cells[1].paragraphs[0].add_run(item["状态"])
            set_chinese_font(run2, "宋体", 9)
            run3 = row_cells[2].paragraphs[0].add_run(str(item["响应时间(ms)"]))
            set_chinese_font(run3, "宋体", 9)
            run4 = row_cells[3].paragraphs[0].add_run(item["输出信息"])
            set_chinese_font(run4, "宋体", 9)
    
    # 4. Keyword关键字：异常状态摘要
    elif "keyword" in monitor_name_lower or "关键字" in monitor_name_lower:
        doc.add_heading("关键字监控异常摘要", level=3)
        for run in doc.paragraphs[-1].runs:
            set_chinese_font(run, "黑体", 12)
        abnormal_count = len([d for d in recent_data if d["状态"] == "离线"])
        if abnormal_count > 0:
            para = doc.add_paragraph(f"⚠️ 最近{RECENT_CHECK_COUNT}次监控中，共检测到{abnormal_count}次异常！")
            set_chinese_font(para.runs[0], "宋体", 11, RGBColor(220, 20, 60))
        else:
            para = doc.add_paragraph(f"✅ 最近{RECENT_CHECK_COUNT}次监控中，未检测到任何异常！")
            set_chinese_font(para.runs[0], "宋体", 11, RGBColor(0, 128, 0))

def add_summary_table(doc, all_results, time_dimension):
    """添加所有监控项中文总览表"""
    doc.add_heading(f"监控总览（近{time_dimension}）", level=1)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 14)
    headers = ["监控项名称", "总检测次数", "在线次数", "离线次数", "可用性", "平均响应时间(ms)"]
    table = doc.add_table(rows=1, cols=len(headers))
    table.style = "Table Grid"
    table.alignment = WD_TABLE_ALIGNMENT.CENTER
    hdr_cells = table.rows[0].cells
    for i, header in enumerate(headers):
        run = hdr_cells[i].paragraphs[0].add_run(header)
        set_chinese_font(run, "黑体", 11)
        hdr_cells[i].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
    for res in all_results:
        if res["code"] != 1:
            continue
        row_cells = table.add_row().cells
        row_data = [
            res["monitor_name"],  # 保留原始监控项名称
            str(res["stats"]["总检测次数"]),
            str(res["stats"]["在线次数"]),
            str(res["stats"]["离线次数"]),
            res["stats"]["可用性"],
            res["stats"]["平均响应时间(ms)"]
        ]
        for i, cell_val in enumerate(row_data):
            run = row_cells[i].paragraphs[0].add_run(cell_val)
            set_chinese_font(run, "宋体", 10)
            row_cells[i].paragraphs[0].alignment = WD_PARAGRAPH_ALIGNMENT.CENTER

def add_monitor_section(doc, monitor_result):
    """添加单个监控项章节（核心统计表格化）"""
    # 自动读取监控项原始名称作为标题
    monitor_name = monitor_result["monitor_name"]
    time_dimension = monitor_result["time_dimension"]
    
    # 1. 监控项标题（完全匹配数据库名称）
    doc.add_heading(f"{monitor_name}", level=2)
    for run in doc.paragraphs[-1].runs:
        set_chinese_font(run, "黑体", 13)
    
    # 2. 核心统计信息（表格化展示）
    if monitor_result["code"] == 1:
        add_core_stats_table(doc, monitor_result["stats"])
    
    # 3. 可视化图表
    if monitor_result["code"] == 1 and monitor_result["data"]:
        doc.add_heading("可视化图表", level=3)
        for run in doc.paragraphs[-1].runs:
            set_chinese_font(run, "黑体", 12)
        charts = generate_charts_by_type(monitor_name, monitor_result["data"], time_dimension)
        for chart_title, chart_buf in charts:
            chart_para = doc.add_paragraph(f"{chart_title}：")
            set_chinese_font(chart_para.runs[0], "宋体", 10)
            doc.add_picture(chart_buf, width=Inches(6))
    
    # 4. 最近10次监控摘要（按类型定制）
    if monitor_result["code"] == 1 and monitor_result["data"]:
        add_recent_summary(monitor_name, monitor_result["data"], doc)

def generate_combined_report(all_results, time_dimension):
    """生成全中文汇总报表（调整恶意链接章节到最底部）"""
    doc = Document()
    # 1. 报表标题
    title = doc.add_heading(f"全监控项数据总报表（近{time_dimension}）", 0)
    title.alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
    for run in title.runs:
        set_chinese_font(run, "黑体", 20)
    
    # 2. 生成时间
    gen_time_para = doc.add_paragraph()
    gen_time_para.alignment = WD_PARAGRAPH_ALIGNMENT.CENTER
    run = gen_time_para.add_run(f"报表生成时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    set_chinese_font(run, "宋体", 12)
    doc.add_page_break()
    
    # 3. 总览表
    add_summary_table(doc, all_results, time_dimension)
    doc.add_page_break()
    
    # 4. 逐个监控项章节（移到恶意链接检测之前）
    for res in all_results:
        if res["code"] == 1:
            add_monitor_section(doc, res)
            doc.add_page_break()
    
    # 5. 恶意链接检测章节（移到最底部）
    print("正在执行恶意链接检测...")
    detection_result = check_malicious_links()
    add_malicious_detection_section(doc, detection_result)
    doc.add_page_break()
    
    # 6. 保存报表（中文名称）
    report_filename = f"监控总报表_近{time_dimension}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.docx"
    report_path = os.path.join(REPORT_DIR, report_filename)
    doc.save(report_path)
    return report_path

def interactive_select():
    """中文交互式选择"""
    print("===== 监控数据查询 =====")
    print("支持的时间维度：")
    for idx, (k, v) in enumerate(TIME_OPTIONS.items(), 1):
        print(f"{idx}. {k}")
    while True:
        try:
            choice = int(input("\n请输入序号（1-5）："))
            if 1 <= choice <= 5:
                return list(TIME_OPTIONS.values())[choice-1]
            else:
                print("请输入1-5之间的序号！")
        except ValueError:
            print("请输入有效数字！")

if __name__ == "__main__":
    # 处理命令行参数
    if len(sys.argv) > 1:
        arg = sys.argv[1]
        if arg in TIME_OPTIONS.keys():
            days = TIME_OPTIONS[arg]
        elif arg.isdigit():
            arg_int = int(arg)
            if arg_int in TIME_OPTIONS.values():
                days = arg_int
            else:
                print(f"错误：仅支持以下天数：{list(TIME_OPTIONS.values())}")
                sys.exit(1)
        else:
            print("错误：参数格式错误，支持：7/15/30/180/365 或 7天/15天/30天/6个月/12个月")
            sys.exit(1)
    else:
        days = interactive_select()
    time_dimension = [k for k, v in TIME_OPTIONS.items() if v == days][0]
    
    # 获取所有监控项
    all_monitors = get_all_monitors()
    if isinstance(all_monitors, dict) and all_monitors["code"] == -1:
        print(json.dumps(all_monitors, ensure_ascii=False, indent=2))
        sys.exit(1)
    if not all_monitors:
        print(json.dumps({"code": 0, "msg": "未找到任何监控项！"}, ensure_ascii=False, indent=2))
        sys.exit(0)
    
    # 处理每个监控项
    all_results = []
    for monitor_id, monitor_name in all_monitors:
        print(f"正在处理：{monitor_name}")
        res = get_monitor_data(monitor_id, monitor_name, days)
        all_results.append(res)
    
    # 生成汇总报表
    print("正在生成汇总总报表...")
    combined_report_path = generate_combined_report(all_results, time_dimension)
    print(f"\n✅ 汇总总报表已生成：{combined_report_path}")
    print("\n===== 处理结果汇总 =====")
    for res in all_results:
        print(f"{res['monitor_name']}：{res['msg']}")
```
---

### SKILL.md 内容
```markdown
{
  "name": "kuma",
  "description": "生成体彩网 Kuma 监控数据报表（支持7/15/30天/6/12个月）",
  "type": "python",
  "entry": "run.py",
  "parameters": [
    {
      "name": "time_dimension",
      "type": "string",
      "required": true,
      "description": "时间维度，支持：7/15/30/180/365 或 7天/15天/30天/6个月/12个月",
      "default": "30天"
    }
  ],
  "author": "admin",
  "version": "1.0.0"
}
(venv) root@aliyunopenclaw:~/.openclaw/skills/kuma# 
(venv) root@aliyunopenclaw:~/.openclaw/skills/kuma# 
(venv) root@aliyunopenclaw:~/.openclaw/skills/kuma# 
(venv) root@aliyunopenclaw:~/.openclaw/skills/kuma# cat SKILL.md 
---
name: kuma
description: 生成体彩网 Kuma 监控数据报表并发送 Word 文件到飞书。
metadata:
  {
    "openclaw": {
      "emoji": "📊",
      "requires": { "bins": ["python3"], "env": [] },
      "user-invocable": true,
      "command-dispatch": "tool",
      "command-tool": "bash"
    }
  }
---

# 体彩网 Kuma 监控报表生成器

## 技能意图
当用户询问「体彩网监控」、「Kuma 数据」、「生成报表」、「发送文件」时，**必须**执行本地 Python 脚本生成 Word 报表。

## 触发规则（强匹配）
用户输入包含以下任意关键词时，**禁止使用其他技能，仅执行本技能**：
- 体彩网
- BJLOT
- Kuma
- 监控报表
- 生成 word
- 发送文件
- 生成监控数据

## 执行逻辑
1. 解析用户参数（7天/15天/30天等）。
2. 执行 Python 脚本生成 Word 报表。
3. **重要**：如果脚本生成成功，调用飞书 API 将文件发送给用户。
4. 如果用户要求忽略时间节点，直接输出最新数据。

## 命令使用
/kuma 7天
/kuma 30天

```
---

### skill.json 内容
```json
{
  "name": "kuma",
  "description": "生成体彩网 Kuma 监控数据报表（支持7/15/30天/6/12个月）",
  "type": "python",
  "entry": "run.py",
  "parameters": [
    {
      "name": "time_dimension",
      "type": "string",
      "required": true,
      "description": "时间维度，支持：7/15/30/180/365 或 7天/15天/30天/6个月/12个月",
      "default": "30天"
    }
  ],
  "author": "admin",
  "version": "1.0.0"
}

```

##  使用方法

### skills注册

让你的智能体”刷新 skills”或重启 Gateway 网关。OpenClaw 将发现新目录并索引 SKILL.md

### 命令行调用

```bash
# 生成最近 7 天的报告（默认）
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py

# 生成最近 30 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py 30

# 生成最近 1 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py 1
```

### 通过 OpenClaw 对话调用

直接对 OpenClaw 说：
- "生成 Kuma 监控报告"
- "查看最近 7 天的监控数据"
- "运行 kuma 监控技能"

## 输出内容

报告包含以下信息：

### 每个监控项
- 📊 监控名称
- 类型（HTTP、TCP、Ping 等）
- 监控 URL/地址
- 可用率百分比
- 总检测次数
- 正常次数
- 平均响应时间（ms）
- 最小/最大响应时间（ms）

### 报告文件
- 终端输出：格式化文本报告
- 文件保存：`~/.openclaw/report/*`

## 技术实现

- 直接读取 SQLite 数据库（kuma.db）
- 自动适配监控项名称
- 智能查找数据库路径
- 支持自定义统计周期

## 注意事项

1. 确保 Kuma 服务正在运行，数据库文件可访问
2. 需要 Python 3.6+ 环境
3. 报告生成不影响 Kuma 服务正常运行
4. JSON 报告可用于后续数据分析或集成

## 故障排查

### 问题：找不到数据库
**解决**：确认 Kuma 安装路径，检查 `kuma.db` 文件位置

### 问题：权限错误
**解决**：确保当前用户对数据库文件有读取权限

### 问题：报告数据为空
**解决**：检查 Kuma 服务是否正常运行，是否有监控项配置
```


### 命令行调用
```bash
# 生成最近 7 天的报告（默认）
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py

# 生成最近 30 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py 30

# 生成最近 1 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py 1
```

### OpenClaw 对话调用
```
用户：生成 Kuma 监控报告
助手：正在生成最近 7 天的 Kuma 监控报告...
（自动执行脚本并返回结果）

用户：查看最近 30 天的监控数据
助手：好的，为您生成 30 天的监控统计...
```

### 输出示例
```
✓ 数据库路径：/root/.kuma/kuma.db
================================================================================
KUMA 监控报告
================================================================================
生成时间：2026-03-12T09:00:00
统计周期：7 天
================================================================================

📊 百度首页
   类型：HTTP
   URL: https://www.baidu.com
   可用率：100.00%
   总检测次数：1008
   正常次数：1008
   平均响应时间：89.23ms

📊 Google 搜索
   类型：HTTP
   URL: https://www.google.com
   可用率：99.80%
   总检测次数：1008
   正常次数：1006
   平均响应时间：245.67ms

================================================================================
✓ 报告已保存至：~/.openclaw/workspace/kuma_report.json
```

---

## 📝 项目总结

### 技术栈
- Python 3.6+
- SQLite3
- OpenClaw 技能框架

### 关键成果
1. ✅ 解决了 API 限制问题（直接读库）
2. ✅ 实现了智能路径查找
3. ✅ 完成了数据库格式适配
4. ✅ 生成了可视化报告
5. ✅ 集成了 OpenClaw 技能系统

### 核心文件
- **脚本**：`~/.openclaw/skills/kuma/run.py`
- **文档**：`~/.openclaw/skills/kuma/SKILL.md`
- **配置**：`~/.openclaw/skills/kuma/skill.json`
- **报告**：`~/.openclaw/reports`

---

*最后更新：2026-03-12*  
*项目状态：✅ 已完成*
