---
title: Google Chrome离线安装包下载方法
pubDate: 2025-08-26
categories: ['笔记']
description: 'Chrome官方默认提供的是在线安装程序，通过特定URL参数，可以直接获取离线安装包，适用于无网络环境。'
slug: chrome-offline-installation-package
---

# Google Chrome 离线安装包下载方法

## 基本原理

在 Chrome 官方下载页面 URL 后添加以下参数可触发离线安装包下载：

- `?standalone=1`：请求离线安装包（完整安装程序，无需联网）
- `&platform=xxx`：指定目标操作系统平台

> 注意：此方法仅适用于桌面版 Chrome，且需使用桌面浏览器访问。

## Windows 平台

### 64 位版本

```text
http://www.google.cn/chrome/browser/desktop/index.html?standalone=1&amp;platform=win64
```

### 32 位版本

```text
http://www.google.cn/chrome/browser/desktop/index.html?standalone=1&amp;platform=win32
```


> 如果只加 `?standalone=1`（不指定 platform），默认提供64位离线安装包。

## 补充说明

- 离线安装包包含完整浏览器文件，安装更快且不受网络影响。
- 适用于企业 IT 批量部署。
- 安装后 Chrome 默认仍会自动更新（可通过组策略禁用）。
- 该参数未被官方公开宣传，但在 `google.cn` 和 `google.com` 域名下长期有效（部分地区可能需切换地区）。