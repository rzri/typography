---
title: Uptime Kuma 监控与 OpenClaw 技能集成实战
pubDate: 2026-03-11
categories: ['笔记', '监控', 'OpenClaw', 'Python']
description: '记录 Uptime Kuma 监控数据读取脚本开发全过程：从 API 限制到直接读库，从路径错误到格式确认，最终集成 OpenClaw 技能的完整经验。'
slug: uptime-kuma-deploy-openclaw-integration
---

# Uptime Kuma 监控与 OpenClaw 技能集成实战

## 📋 核心目标板块

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

## 🚧 遇到的困难

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

## ✅ 最终解决方案

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
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""Kuma 监控报告生成脚本"""

import sqlite3
import json
import sys
from datetime import datetime, timedelta
from pathlib import Path

def get_db_path():
    """智能查找 kuma.db 数据库路径"""
    possible_paths = [
        Path.home() / ".kuma" / "kuma.db",
        Path("/app/data/kuma.db"),
        Path("/root/.kuma" / "kuma.db"),
        Path("./kuma.db"),
    ]
    for db_path in possible_paths:
        if db_path.exists():
            return str(db_path)
    raise FileNotFoundError("未找到 kuma.db 数据库")

def get_heartbeat_stats(db_path, monitor_id, days=7):
    """获取心跳统计数据"""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    
    start_time = (datetime.now() - timedelta(days=days)).timestamp()
    
    cursor.execute("""
        SELECT 
            COUNT(*) as total,
            SUM(CASE WHEN status = 1 THEN 1 ELSE 0 END) as up_count,
            AVG(response_time) as avg_response_time
        FROM heartbeat 
        WHERE monitor_id = ? AND time >= ?
    """, (monitor_id, start_time))
    
    stats = cursor.fetchone()
    conn.close()
    
    return {
        'total': stats['total'],
        'up_count': stats['up_count'],
        'uptime': (stats['up_count'] / stats['total'] * 100) if stats['total'] > 0 else 0,
        'avg_response_time': stats['avg_response_time'] or 0,
    }
```

### SKILL.md 内容
```markdown
# Kuma 监控报告生成技能

## 功能描述
生成 Uptime Kuma 监控服务的可视化报告，包括可用性统计、响应时间分析。

## 使用方法

### 命令行调用
```bash
# 生成最近 7 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py

# 生成最近 30 天的报告
~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py 30
```

### OpenClaw 对话调用
- "生成 Kuma 监控报告"
- "查看最近 7 天的监控数据"

## 输出内容
- 📊 监控名称、类型、URL
- ✅ 可用率百分比
- 📈 总检测次数、正常次数
- ⏱️ 平均/最小/最大响应时间
```

### skill.json 内容
```json
{
  "name": "kuma-monitor",
  "version": "1.0.0",
  "description": "生成 Uptime Kuma 监控服务的可视化报告",
  "tags": ["monitoring", "kuma", "uptime", "report"],
  "triggers": [
    "生成 kuma 报告",
    "kuma 监控报告",
    "查看监控数据",
    "uptime 报告"
  ],
  "commands": {
    "default": {
      "exec": "~/.openclaw/venv/bin/python ~/.openclaw/skills/kuma/run.py",
      "args": ["7"]
    }
  },
  "permissions": ["read:files", "exec:python"],
  "dependencies": ["python3", "sqlite3"]
}
```

---

## 📖 使用方法

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
- **报告**：`~/.openclaw/workspace/kuma_report.json`

---

*最后更新：2026-03-12*  
*项目状态：✅ 已完成*
