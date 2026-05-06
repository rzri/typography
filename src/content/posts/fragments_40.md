---
title: Cloudflare Workers 临时文件中转站：从基础版到生产环境
pubDate: 2025-10-17
categories: ['笔记']
description: '在 Cloudflare Workers + KV 基础方案上，扩展 WebDAV 混合存储、多文件打包、企业微信通知等功能，搭建可用的临时文件分享服务。'
slug: cloudflare-workers-temp-file-transfer
---

日常需要把文件从单位带回家，微信/QQ 传文件会留痕，国内云盘臃肿，手里刚好有闲置的 InfiniCLOUD（日本 TeraCLOUD）40GB WebDAV 空间。索性用 Cloudflare Workers 搭了个临时文件中转服务。

为什么不直接用 R2？免费额度 10GB 已有他用，超了要收费，不想惦记这事。

## 基础方案

最简单的版本只需要一个 Cloudflare Worker + 一个 KV 命名空间，代码已开源在 [tempfiles-workers](https://github.com/haenlau/tempfiles-workers)。

核心逻辑：

```js
// 上传：写入 KV，设 12 小时过期
await env.TEMP_STORE.put(fileId, arrayBuffer, {
  metadata: { filename: file.name, contentType: file.type },
  expirationTtl: 43200 // 12小时
});

// 下载：从 KV 读取，强制附件下载
const entry = await env.TEMP_STORE.getWithMetadata(id, "arrayBuffer");
return new Response(entry.value, {
  headers: {
    "Content-Type": entry.metadata.contentType,
    "Content-Disposition": 'attachment; filename="' + encodeURIComponent(entry.metadata.filename) + '"',
    "Cache-Control": "no-store"
  }
});
```

单文件 ≤ 25MB，12 小时自动过期，6 位随机短链。够用，但有几个痛点：

- 25MB 上限太小，稍微大点的文件就传不了
- 只能单文件，多个文件得一个一个传
- 过期太快，12 小时不够"周末带回家处理"的场景
- 传完没提醒，得自己盯着

## 生产版改动

在基础版上做了几轮迭代，主要改动：

### 1. KV + WebDAV 混储

KV 单次写入上限 25MB，但 TeraCLOUD WebDAV 没这个限制。按文件大小自动分流：

```js
if (file.size <= 25 * 1024 * 1024) {
  // 小文件走 KV，速度快
  await env.TEMP_STORE.put(fileId, buffer, {
    metadata: { filename, storage: "kv", ... },
    expirationTtl: 7 * 24 * 3600
  });
} else {
  // 大文件走 WebDAV，KV 只存元数据做索引
  await fetch(webdavUrl, {
    method: 'PUT',
    headers: { 'Authorization': 'Basic ' + credentials },
    body: buffer
  });
  await env.TEMP_STORE.put(fileId, "", {
    metadata: { filename, storage: "webdav", webdavFilename, ... },
    expirationTtl: 7 * 24 * 3600
  });
}
```

下载时根据 `metadata.storage` 字段判断来源，KV 直接返回，WebDAV 则代理转发：

```js
if (storage === "kv") {
  return new Response(entry.value, { headers });
}
if (storage === "webdav") {
  const resp = await fetch(webdavUrl, {
    headers: { 'Authorization': 'Basic ' + credentials }
  });
  return new Response(resp.body, { headers });
}
```

单文件上限从 25MB 提到 99MB，覆盖绝大多数场景。

### 2. 多文件自动打包

多文件上传时，用纯 JS 实现了一个最小 ZIP 打包器（不依赖外部库），在 Worker 内完成：

```js
const formData = await request.formData();
const files = formData.getAll("file").filter(f => f instanceof File);

if (files.length === 1) {
  // 单文件直接存
} else {
  // 多文件打包成 zip
  const zipBuffer = zipFiles(fileBuffers);
  // 存储逻辑同上，按大小分流
}
```

ZIP 打包器手动构造 Local File Header、Central Directory 和 EOCD，不做压缩（`compression method = 0`），纯粹把多个文件塞进一个包里。虽然 CRC32 用了占位值，但主流解压工具都能正常处理。

### 3. 过期时间从 12 小时改到 7 天

```js
const EXPIRATION_TTL = 7 * 24 * 3600; // 7天
```

KV 的 `expirationTtl` 是被动过期，写入时设定 TTL，到期后 KV 自动清除。WebDAV 端的文件需要额外处理——可以写个定时 Cron 清理，或者就让 KV 元数据过期后下载返回 404，WebDAV 端的文件虽然还在但已经不可达了。

### 4. 企业微信 Webhook 通知

上传成功后推一条消息到企业微信群，不用自己盯着：

```js
async function sendWeComWebhookNotification(env, fileData) {
  const content = '📁 新文件上传\n文件名：' + filename
    + '\n大小：' + sizeText
    + '\n🔗 下载地址：' + downloadUrl;

  await fetch(env.WECOM_WEBHOOK_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ msgtype: "text", text: { content } })
  });
}
```

`WECOM_WEBHOOK_URL` 通过 Worker 的 Secrets 配置，不会出现在代码里。

### 5. ID 碰撞检查

基础版直接 `Math.random().toString(36).substring(2, 8)` 生成 6 位 ID，生产版加了一轮碰撞检查：

```js
async function generateFileId(env) {
  for (let i = 0; i < 5; i++) {
    const id = Math.random().toString(36).substring(2, 8);
    if (!(await env.TEMP_STORE.get(id))) return id;
  }
  // 5 次都碰撞了，追加时间戳后缀
  return Math.random().toString(36).substring(2, 8)
    + Date.now().toString(36).slice(-2);
}
```

KV 的 `get` 操作不计费（免费额度内），多几次检查成本可以忽略。

## 部署步骤

1. **创建 KV 命名空间**：Dashboard → 存储和数据库 → Workers KV → 创建 `TEMP_STORE`
2. **创建 Worker**：Dashboard → 计算和 AI → Workers 和 Pages → 创建应用 → 粘贴代码 → Save and Deploy
3. **绑定 KV**：Worker → 绑定 → 添加 KV 命名空间 → 变量名 `TEMP_STORE`
4. **配置 Secrets**：Worker → 变量和机密 → 添加密钥
   - `WECOM_WEBHOOK_URL`：企业微信机器人 Webhook 地址
   - `WEBDAV_ACCOUNT` / `WEBDAV_PASSWORD`：WebDAV 凭据
5. **绑定域名**：Worker → 域和路由 → 添加自定义域

## 前端

前端是内嵌在 Worker 代码里的单页 HTML，支持拖拽上传、暗色/亮色主题自适应、上传进度条。上传用 `XMLHttpRequest` 而不是 `fetch`，因为需要监听 `xhr.upload.progress` 事件：

```js
const xhr = new XMLHttpRequest();
xhr.upload.addEventListener('progress', (e) => {
  if (e.lengthComputable) updateProgress(e.loaded, e.total);
});
xhr.open('POST', '/api/upload-public');
xhr.send(formData);
```

## 局限

- KV 单次写入上限 25MB 不可突破，大文件只能走 WebDAV 中转
- WebDAV 代理下载会占用 Worker 的请求时间，大文件可能超时
- ZIP 打包器不做压缩，纯打包，适合已经压缩过的文件（图片、PDF），不适合大量小文本文件
- 免费 Worker 每天 10 万次请求，日常够用，但不适合公开分享

源码：[tempfiles-workers](https://github.com/haenlau/tempfiles-workers)（基础版，生产版代码量更大，按需取用）
