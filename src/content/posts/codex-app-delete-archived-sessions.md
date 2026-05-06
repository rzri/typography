---
title: Windows Codex App 永久删除归档会话
pubDate: 2026-05-06
categories: ['AI']
description: '通过 PowerShell 命令彻底清除 Windows 版 Codex App 的归档会话数据'
slug: codex-app-delete-archived-sessions
---

Codex App 的会话被归档后，并不会从磁盘上消失。它们安静地躺在 `%USERPROFILE%\.codex\archived_sessions` 目录里，随着时间推移越积越多。

如果想彻底清除这些历史会话，一条 PowerShell 命令就够了：

```powershell
Remove-Item -LiteralPath "$env:USERPROFILE\.codex\archived_sessions" -Recurse -Force
```

各参数的含义：

- **`-LiteralPath`**：按字面路径处理，不会把 `[]` 等字符当作通配符解析，避免误删。
- **`-Recurse`**：递归删除目录及其所有子内容。
- **`-Force`**：强制删除隐藏文件和只读文件。

执行后，整个 `archived_sessions` 文件夹会被移除。Codex App 下次启动时如果需要该目录，会自动重建。

如果只想删除部分会话，可以先列出目录内容，再按需清理：

```powershell
# 查看归档会话列表
Get-ChildItem "$env:USERPROFILE\.codex\archived_sessions"

# 删除超过 30 天的会话
Get-ChildItem "$env:USERPROFILE\.codex\archived_sessions" |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } |
    Remove-Item -Recurse -Force
```

没有回收站、没有二次确认，删了就是删了。操作前确保不需要这些会话记录。
