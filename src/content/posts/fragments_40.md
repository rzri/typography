---
title: Workers + KV + WebDAV 搭一个临时文件中转站
pubDate: 2025-10-17
updatedDate: 2025-06-11
categories: ['笔记']
description: '用 Cloudflare Workers + KV 做临时文件中转：单文件最大 99MB，多文件自动打包 ZIP（上限 50MB），7 天自动销毁，支持企业微信通知。'
slug: workers-kv-webdav-tempfile
---

日常办公里经常遇到一个小尴尬：需要把单位文件带回家处理，但又不想用微信、QQ 这类工具把文件落到本地；国内云盘要么臃肿，要么需要装客户端，顺手用也不爽。

我手上刚好有一份闲置的 WebDAV 云盘空间（服务在日本，之前一直吃灰）。于是干脆借 Cloudflare Workers + KV 做个"临时文件中转站"：

- 浏览器打开首页，拖拽或选择文件上传（单文件最大 99MB，多文件自动打包 ZIP 总上限 50MB）
- 返回一个短链接，复制就能分享
- 文件 7 天后自动销毁（由 KV TTL 控制）
- 可选：上传成功后自动推送企业微信通知

下面把从 0 到可用的部署过程和完整代码记录下来，方便以后复制粘贴。

## 第一步：创建 KV 命名空间

目标：创建一个名为 `TEMP_STORE` 的 KV 存储空间。

操作路径：
Dashboard 首页 → 左侧边栏「账户和主页」→「存储和数据库」→「Workers KV」

操作步骤：

1. 点击右上角「Create instance / 创建命名空间」
2. Name 填：`TEMP_STORE`
3. 点击「Create / 创建」

## 第二步：创建 Worker 并粘贴代码

目标：部署处理上传/下载逻辑的 Worker。

操作路径：
Dashboard 首页 → 左侧边栏「账户和主页」→「计算和 AI」→「Workers 和 Pages」

操作步骤：

1. 点击「创建应用」→ 选择「从 Hello World! 开始」
2. 应用名称输入：`tmp-worker`（随便起，别和现有冲突就行）
3. 进入代码编辑器后，全选并删除默认代码
4. 将下方完整 JS 代码**原样**粘贴到编辑区

提示：代码里的 WEBDAV_BASE 常量需要替换为你的 WebDAV 目录地址。其余配置（KV 绑定、环境变量）在后续步骤完成。

## 存储策略：KV 优先，WebDAV 兜底

这个 Worker 的核心逻辑是根据文件大小自动分流存储：

- **单文件 ≤ 25MB**：直接写进 KV（`env.TEMP_STORE.put(fileId, arrayBuffer, {...})`）。
- **单文件 > 25MB**：通过 WebDAV `PUT` 上传到远端目录；KV 只保存一份索引元数据（`storage="webdav"` + `webdavFilename`），下载时再根据元数据去 WebDAV 拉取并透传回浏览器。
- **多文件**：先用 `fflate` 库打包成 ZIP，再按上面的大小规则走 KV 或 WebDAV。

代码里对应的关键字段：

- `WEBDAV_BASE`：WebDAV 目标目录 URL（硬编码或通过环境变量配置）。**此处需替换为你自己的 WebDAV 地址**。
- `env.WEBDAV_ACCOUNT` / `env.WEBDAV_PASSWORD`：用 Basic Auth 访问 WebDAV。
- `metadata.storage`：标记当前文件存储位置（`kv` 或 `webdav`）。
- `metadata.webdavFilename`：当走 WebDAV 时，记录远端的实际文件名（由 `file_` + ID + 原文件名拼接而成）。

上传分流逻辑在 `/api/upload-public` 路由里：

- 单文件小文件：写 KV，`metadata.storage = "kv"`
- 单文件大文件：WebDAV 上传后，KV 只存元数据索引，`metadata.storage = "webdav"`
- 多文件：`fflate.zipSync()` 打包成 ZIP，再按大小走 KV 或 WebDAV

下载逻辑在 `/{id}` 路由（要求 ID 恰好 6 位）：

- `storage === "kv"`：直接返回 KV 的二进制内容
- `storage === "webdav"`：根据 `webdavFilename` 拼接 WebDAV URL 并拉取，响应体透传回浏览器
- `Content-Disposition` 使用 RFC 5987 编码（`filename*=UTF-8''...`），确保中文文件名在各浏览器都能正确显示

<details>
<summary><strong>▶ 完整 Worker 代码</strong></summary>

```js
// ====== HTML 界面 ======
const HTML = `<!DOCTYPE html>
<html lang="zh-CN">
<head>
 <meta charset="utf-8" />
 <meta name="viewport" content="width=device-width, initial-scale=1" />
 <title>✨ Air1 TempFile</title>
 <link rel="icon" type="image/png" href="https://YOUR_DOMAIN/favicon.png" />
 <style>
 :root {
 --bg: #0f0f12;
 --card-bg: rgba(255, 255, 255, 0.06);
 --card-border: rgba(255, 255, 255, 0.1);
 --text: #f0f0f5;
 --muted: #a0a0b0;
 --primary: #4d9fff;
 --success: #4ade80;
 --danger: #f87171;
 --shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
 --radius: 18px;
 }

 @media (prefers-color-scheme: light) {
 :root {
 --bg: #f8f9ff;
 --card-bg: rgba(255, 255, 255, 0.7);
 --card-border: rgba(0, 0, 0, 0.08);
 --text: #1a1a25;
 --muted: #666;
 --primary: #2563eb;
 --success: #16a34a;
 --danger: #dc2626;
 --shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
 }
 }

 * {
 margin: 0;
 padding: 0;
 box-sizing: border-box;
 }

 body {
 background: var(--bg);
 color: var(--text);
 font-family: 'SF Pro Display', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
 min-height: 100vh;
 display: flex;
 flex-direction: column;
 align-items: center;
 justify-content: center;
 padding: 2rem 1.5rem;
 position: relative;
 overflow-x: hidden;
 }

 body::before {
 content: "";
 position: absolute;
 top: 0;
 left: 0;
 width: 100%;
 height: 100%;
 background:
 radial-gradient(circle at 20% 30%, rgba(77, 159, 255, 0.06) 0%, transparent 40%),
 radial-gradient(circle at 80% 70%, rgba(77, 159, 255, 0.04) 0%, transparent 50%);
 pointer-events: none;
 z-index: -1;
 }

 .container {
 width: 100%;
 max-width: 520px;
 text-align: center;
 }

 h1 {
 font-weight: 700;
 font-size: 2.2rem;
 margin-bottom: 0.4rem;
 background: linear-gradient(135deg, #ffffff, #a0a0ff);
 -webkit-background-clip: text;
 background-clip: text;
 color: transparent;
 background-size: 200% 200%;
 animation: gradientShift 8s ease infinite;
 }

 @keyframes gradientShift {
 0% { background-position: 0% 50%; }
 50% { background-position: 100% 50%; }
 100% { background-position: 0% 50%; }
 }

 .subtitle {
 color: var(--muted);
 font-size: 0.95rem;
 margin-bottom: 2.2rem;
 }

 .upload-card {
 background: var(--card-bg);
 border: 1px solid var(--card-border);
 backdrop-filter: blur(12px);
 -webkit-backdrop-filter: blur(12px);
 border-radius: var(--radius);
 padding: 2.2rem 1.5rem;
 margin-bottom: 1.8rem;
 cursor: pointer;
 transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
 position: relative;
 overflow: hidden;
 }

 .upload-card:hover {
 transform: translateY(-4px);
 box-shadow: var(--shadow);
 border-color: rgba(77, 159, 255, 0.3);
 }

 .upload-card.dragover {
 border-color: var(--primary);
 background: rgba(77, 159, 255, 0.08);
 }

 .upload-icon {
 font-size: 3.2rem;
 margin-bottom: 1.2rem;
 display: block;
 transition: transform 0.3s;
 }

 .upload-card:hover .upload-icon {
 transform: scale(1.1) rotate(3deg);
 }

 .upload-text {
 font-size: 1.1rem;
 font-weight: 500;
 margin-bottom: 0.4rem;
 }

 .upload-hint {
 font-size: 0.85rem;
 color: var(--muted);
 }

 .selected-file {
 margin-top: 0.6rem;
 font-size: 0.85rem;
 color: var(--primary);
 display: none;
 }

 #fileInput {
 display: none;
 }

 .btn {
 width: 100%;
 padding: 0.95rem;
 background: var(--primary);
 color: white;
 border: none;
 border-radius: 14px;
 font-size: 1.05rem;
 font-weight: 600;
 cursor: pointer;
 transition: all 0.25s;
 letter-spacing: 0.3px;
 }

 .btn:hover:not(:disabled) {
 background: #3a8bff;
 transform: translateY(-2px);
 box-shadow: 0 6px 16px rgba(77, 159, 255, 0.3);
 }

 .btn:disabled {
 opacity: 0.7;
 cursor: not-allowed;
 transform: none;
 box-shadow: none;
 }

 .result-card {
 background: var(--card-bg);
 border: 1px solid var(--card-border);
 backdrop-filter: blur(12px);
 -webkit-backdrop-filter: blur(12px);
 border-radius: var(--radius);
 padding: 1.6rem;
 margin-top: 1.5rem;
 display: none;
 }

 .result-card.show {
 display: block;
 animation: fadeIn 0.4s ease;
 }

 @keyframes fadeIn {
 from { opacity: 0; transform: translateY(10px); }
 to { opacity: 1; transform: translateY(0); }
 }

 .result-title {
 font-size: 1.1rem;
 margin-bottom: 1rem;
 color: var(--success);
 font-weight: 600;
 }

 .result-item {
 margin-bottom: 1rem;
 text-align: center;
 padding: 0.8rem;
 background: rgba(77, 159, 255, 0.05);
 border-radius: 10px;
 display: flex;
 flex-direction: column;
 align-items: center;
 }
 .result-filename {
 font-weight: bold;
 margin-bottom: 0.3rem;
 color: var(--text);
 }

 .result-link {
 display: block;
 word-break: break-all;
 color: var(--primary);
 text-decoration: none;
 font-size: 0.9rem;
 font-family: monospace;
 margin: 0.2rem 0;
 }

 .copy-btn {
 background: rgba(255, 255, 255, 0.12);
 color: var(--text);
 border: none;
 padding: 0.4rem 0.8rem;
 border-radius: 8px;
 font-weight: 600;
 cursor: pointer;
 transition: all 0.2s;
 font-size: 0.85rem;
 }

 .copy-btn:hover {
 background: rgba(255, 255, 255, 0.2);
 }

 .copy-btn.copied {
 background: var(--success);
 color: white;
 }

 .error-msg {
 color: var(--danger);
 margin-top: 0.5rem;
 font-size: 0.85rem;
 }

 /* ===== 进度条样式 ===== */
 .progress-track {
 height: 6px;
 background: rgba(255, 255, 255, 0.1);
 border-radius: 3px;
 margin-top: 1.2rem;
 overflow: hidden;
 display: none;
 }

 .progress-fill {
 height: 100%;
 width: 0%;
 border-radius: 3px;
 transition: width 0.2s ease;
 background: var(--primary);
 }
 .progress-fill.uploading {
 background: linear-gradient(to right, var(--primary), #80c4ff);
 }
 .progress-fill.completed {
 background: var(--success);
 }

 .progress-text {
 margin-top: 0.4rem;
 font-size: 0.85rem;
 color: var(--muted);
 }

 .footer {
 margin-top: 2.5rem;
 color: var(--muted);
 font-size: 0.8rem;
 opacity: 0.8;
 }

 @media (max-width: 480px) {
 h1 { font-size: 1.8rem; }
 .upload-card { padding: 1.8rem 1rem; }
 }
 </style>
</head>
<body>
 <div class="container">
 <h1>Air1 TempFile</h1>
 <p class="subtitle">安全上传 · 多文件自动打包 · 7天自动销毁</p>

 <div class="upload-card" id="dropArea">
 <span class="upload-icon">📤</span>
 <p class="upload-text">拖拽文件或点击上传</p>
 <p class="upload-hint">支持任意格式</p>
 <p class="selected-file" id="selectedFile"></p>
 <input type="file" id="fileInput" multiple />
 </div>

 <button class="btn" id="uploadBtn" onclick="uploadFiles()">确认上传</button>

 <div class="progress-track" id="progressTrack">
 <div class="progress-fill" id="progressFill"></div>
 </div>
 <p class="progress-text" id="progressText" style="display:none;">上传中...</p>

 <div class="result-card" id="resultCard">
 <div class="result-title">✅ 上传完成</div>
 <div id="resultsList"></div>
 </div>

 <p class="footer">Part of <strong>Air1 Quick Tools</strong> · Powered by Cloudflare</p>
 </div>

 <script>
 const dropArea = document.getElementById('dropArea');
 const fileInput = document.getElementById('fileInput');
 const uploadBtn = document.getElementById('uploadBtn');
 const resultCard = document.getElementById('resultCard');
 const resultsList = document.getElementById('resultsList');
 const progressTrack = document.getElementById('progressTrack');
 const progressFill = document.getElementById('progressFill');
 const progressText = document.getElementById('progressText');
 const selectedFileEl = document.getElementById('selectedFile');

 ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(e => {
 dropArea.addEventListener(e, preventDefaults, false);
 });

 function preventDefaults(e) {
 e.preventDefault();
 e.stopPropagation();
 }

 ['dragenter', 'dragover'].forEach(e => {
 dropArea.addEventListener(e, () => dropArea.classList.add('dragover'), false);
 });

 ['dragleave', 'drop'].forEach(e => {
 dropArea.addEventListener(e, () => dropArea.classList.remove('dragover'), false);
 });

 dropArea.addEventListener('drop', e => {
 fileInput.files = e.dataTransfer.files;
 fileInput.dispatchEvent(new Event('change'));
 });

 dropArea.addEventListener('click', () => fileInput.click());

 fileInput.addEventListener('change', () => {
 const files = fileInput.files;
 if (files.length > 0) {
 let names = Array.from(files).map(f => f.name).join(', ');
 if (names.length > 60) names = names.substring(0, 60) + '...';
 selectedFileEl.textContent = '已选择 ' + files.length + ' 个文件：' + names;
 selectedFileEl.style.display = 'block';
 } else {
 selectedFileEl.style.display = 'none';
 }
 });

 function updateProgress(loaded, total) {
 const percent = Math.round((loaded / total) * 100);
 progressFill.style.width = percent + '%';
 progressText.textContent = '上传中... ' + percent + '%';
 progressFill.classList.add('uploading');
 progressFill.classList.remove('completed');
 }

 async function uploadFiles() {
 const files = Array.from(fileInput.files);
 if (!files.length) return alert('请选择至少一个文件');

 for (const file of files) {
 if (file.size > 99 * 1024 * 1024) {
 return alert('「' + file.name + '」不能超过 99MB');
 }
 }

 resultCard.classList.remove('show');
 resultsList.innerHTML = '';
 uploadBtn.disabled = true;
 uploadBtn.textContent = '上传中...';
 progressTrack.style.display = 'block';
 progressText.style.display = 'block';
 progressFill.style.width = '0%';
 progressFill.classList.remove('completed');
 progressFill.classList.add('uploading');

 const formData = new FormData();
 files.forEach(f => formData.append('file', f));

 function resetUploadUI() {
 uploadBtn.disabled = false;
 uploadBtn.textContent = '确认上传';
 progressTrack.style.display = 'none';
 progressText.style.display = 'none';
 progressText.textContent = '上传中... 0%';
 }

 try {
 const xhr = new XMLHttpRequest();

 xhr.upload.addEventListener('progress', (e) => {
 if (e.lengthComputable) {
 updateProgress(e.loaded, e.total);
 }
 });

 xhr.addEventListener('load', () => {
 if (xhr.status >= 200 && xhr.status < 300) {
 try {
 const res = JSON.parse(xhr.responseText);
 if (res.downloadUrl) {
 const div = document.createElement('div');
 div.className = 'result-item';

 const filenameEl = document.createElement('div');
 filenameEl.className = 'result-filename';
 let filename = files.length === 1 ? files[0].name : 'upload_' + res.fileId + '.zip';
 filenameEl.textContent = '📁 ' + filename;
 div.appendChild(filenameEl);

 const linkEl = document.createElement('a');
 linkEl.className = 'result-link';
 linkEl.href = res.downloadUrl;
 linkEl.target = '_blank';
 linkEl.textContent = res.downloadUrl;
 div.appendChild(linkEl);

 const copyBtn = document.createElement('button');
 copyBtn.className = 'copy-btn';
 copyBtn.textContent = '📋 复制';
 copyBtn.onclick = function() {
 copyText({ target: copyBtn }, res.downloadUrl);
 };
 div.appendChild(copyBtn);

 if (res.notifyError) {
 const errorMsg = document.createElement('div');
 errorMsg.className = 'error-msg';
 errorMsg.textContent = '⚠️ 通知失败：' + res.notifyError;
 div.appendChild(errorMsg);
 }

 resultsList.appendChild(div);
 resultCard.classList.add('show');
 } else {
 alert(res.error || '上传失败');
 }
 } catch (parseError) {
 console.error('JSON parse error:', parseError);
 alert('服务器返回格式错误');
 }
 } else {
 console.error('Upload error:', xhr.statusText);
 alert('上传失败: ' + xhr.statusText);
 }
 resetUploadUI();
 });

 xhr.addEventListener('error', () => {
 console.error('Network error during upload');
 alert('网络错误，请重试');
 resetUploadUI();
 });

 xhr.addEventListener('abort', () => {
 console.log('Upload aborted');
 alert('上传被取消');
 resetUploadUI();
 });

 xhr.open('POST', '/api/upload-public');
 xhr.send(formData);

 } catch (e) {
 console.error('Upload error:', e);
 alert('上传初始化失败');
 resetUploadUI();
 }

 }

 function copyText(event, text) {
 const btn = event.target;
 if (navigator.clipboard && window.isSecureContext) {
 navigator.clipboard.writeText(text).then(() => {
 showCopySuccess(btn);
 }).catch(() => {
 fallbackCopyText(btn, text);
 });
 } else {
 fallbackCopyText(btn, text);
 }
 }

 function fallbackCopyText(btn, text) {
 const textarea = document.createElement('textarea');
 textarea.value = text;
 textarea.style.position = 'fixed';
 textarea.style.opacity = '0';
 document.body.appendChild(textarea);
 textarea.select();
 try {
 const ok = document.execCommand('copy');
 if (ok) {
 showCopySuccess(btn);
 } else {
 alert('复制失败，请手动长按链接复制');
 }
 } catch (e) {
 alert('复制失败，请手动复制');
 } finally {
 document.body.removeChild(textarea);
 }
 }

 function showCopySuccess(btn) {
 btn.classList.add('copied');
 btn.textContent = '✅ 已复制';
 setTimeout(() => {
 btn.classList.remove('copied');
 btn.textContent = '📋 复制';
 }, 2000);
 }
 </script>
</body>
</html>`;

// ====== 导入 fflate（Worker 顶层） ======
import { zipSync } from 'fflate';

// ====== 常量 ======
const MAX_TOTAL_SIZE = 50 * 1024 * 1024; // 多文件打包总上限 50MB
const EXPIRATION_TTL = 7 * 24 * 3600; // 7天

// ====== 生成随机 6 位 ID ======
async function generateFileId(env) {
 for (let i = 0; i < 5; i++) {
 const id = Math.random().toString(36).substring(2, 8).padStart(6, '0');
 if (!(await env.TEMP_STORE.get(id))) {
 return id;
 }
 }
 return (Math.random().toString(36).substring(2, 8) + Date.now().toString(36).slice(-2)).padStart(6, '0');
}

// ====== ZIP 打包（使用 fflate）======
function zipFiles(files) {
 const input = Object.fromEntries(files.map(f => [f.name, f.data]));
 return zipSync(input, { level: 6 });
}

// ====== 企业微信通知函数 ======
async function sendWeComWebhookNotification(env, fileData) {
 const WEBHOOK_URL = env.WECOM_WEBHOOK_URL;
 if (!WEBHOOK_URL) return;
 const { filename, size, downloadUrl } = fileData;
 let sizeText;
 if (size < 1024) sizeText = size + " B";
 else if (size < 1024 * 1024) sizeText = (size / 1024).toFixed(2) + " KB";
 else sizeText = (size / (1024 * 1024)).toFixed(2) + " MB";
 const now = new Date().toLocaleString("zh-CN", { timeZone: "Asia/Shanghai" });
 const content = `📁 新文件上传\n文件名：${filename}\n大小：${sizeText}\n时间：${now}\n🔗 下载地址：${downloadUrl}`;
 const payload = { msgtype: "text", text: { content } };
 const resp = await fetch(WEBHOOK_URL, {
 method: "POST",
 headers: { "Content-Type": "application/json" },
 body: JSON.stringify(payload)
 });
 const result = await resp.json();
 if (result.errcode !== 0) {
 throw new Error("Webhook 发送失败: " + result.errmsg);
 }
}

// ====== Content-Disposition 构造（RFC 5987）======
function buildContentDisposition(filename) {
 const fallback = filename.replace(/[\r\n"]/g, "_").replace(/[\u0080-\uFFFF]/g, "_") || "download";
 return `attachment; filename="${fallback}"; filename*=UTF-8''${encodeURIComponent(filename)}`;
}

// ====== 主入口 ======
export default {
 async fetch(request, env) {
 const url = new URL(request.url);
 const { pathname } = url;

 // 1. 首页
 if (pathname === "/") {
 return new Response(HTML, {
 headers: { "Content-Type": "text/html; charset=utf-8" }
 });
 }

 // 2. CORS 预检
 if (request.method === "OPTIONS") {
 return new Response(null, {
 headers: {
 "Access-Control-Allow-Origin": "*",
 "Access-Control-Allow-Methods": "POST, OPTIONS",
 "Access-Control-Allow-Headers": "Content-Type"
 }
 });
 }

 // 3. 上传（公开接口）
 if (pathname === "/api/upload-public" && request.method === "POST") {
 const contentType = request.headers.get("content-type") || "";
 if (!contentType.includes("multipart/form-data")) {
 return new Response(JSON.stringify({ error: "必须使用 multipart/form-data 上传" }), {
 status: 400,
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }
 try {
 const formData = await request.formData();
 const files = formData.getAll("file").filter(f => f instanceof File);
 if (!files.length) {
 return new Response(JSON.stringify({ error: "未提供有效文件" }), {
 status: 400,
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }

 // 单文件大小限制 99MB
 for (const file of files) {
 if (file.size > 99 * 1024 * 1024) {
 return new Response(JSON.stringify({ error: "文件「" + file.name + "」超过 99MB" }), {
 status: 400,
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }
 }

 const fileId = await generateFileId(env);
 let notifyError = null;

 if (files.length === 1) {
 // ── 单文件 ──
 const file = files[0];
 const buffer = await file.arrayBuffer();
 const filename = file.name;
 const fileContentType = file.type || "application/octet-stream";

 if (file.size <= 25 * 1024 * 1024) {
 await env.TEMP_STORE.put(fileId, buffer, {
 metadata: { filename, contentType: fileContentType, storage: "kv", isZip: false },
 expirationTtl: EXPIRATION_TTL
 });
 } else {
 const WEBDAV_BASE = "https://YOUR_WEBDAV_HOST/dav/YOUR_PATH/";
 const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
 const webdavFilename = "file_" + fileId + "_" + filename;
 const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);
 const resp = await fetch(webdavUrl, {
 method: "PUT",
 headers: { "Authorization": "Basic " + credentials, "Content-Type": fileContentType },
 body: buffer
 });
 if (!resp.ok) throw new Error("WebDAV upload failed: " + resp.status);
 await env.TEMP_STORE.put(fileId, "", {
 metadata: { filename, contentType: fileContentType, storage: "webdav", webdavFilename, isZip: false },
 expirationTtl: EXPIRATION_TTL
 });
 }

 try {
 await sendWeComWebhookNotification(env, {
 filename, size: file.size,
 downloadUrl: new URL("/" + fileId, request.url).toString()
 });
 } catch (e) { notifyError = e.message; }

 return new Response(JSON.stringify({
 downloadUrl: new URL("/" + fileId, request.url).toString(),
 fileId, notifyError
 }), {
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 } else {
 // ── 多文件：打包 ZIP ──
 let totalSize = 0;
 const fileBuffers = [];
 for (const file of files) {
 totalSize += file.size;
 const buffer = await file.arrayBuffer();
 fileBuffers.push({ name: file.name, data: new Uint8Array(buffer) });
 }
 if (totalSize > MAX_TOTAL_SIZE) {
 return new Response(JSON.stringify({ error: "总大小不能超过 " + (MAX_TOTAL_SIZE / (1024 * 1024)).toFixed(1) + "MB" }), {
 status: 400,
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }

 const zipBuffer = zipFiles(fileBuffers);
 const zipSize = zipBuffer.byteLength;
 const zipName = "upload_" + fileId + ".zip";

 if (zipSize <= 25 * 1024 * 1024) {
 await env.TEMP_STORE.put(fileId, zipBuffer, {
 metadata: { filename: zipName, contentType: "application/zip", storage: "kv", isZip: true },
 expirationTtl: EXPIRATION_TTL
 });
 } else {
 const WEBDAV_BASE = "https://YOUR_WEBDAV_HOST/dav/YOUR_PATH/";
 const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
 const webdavFilename = "zip_" + fileId + ".zip";
 const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);
 const resp = await fetch(webdavUrl, {
 method: "PUT",
 headers: { "Authorization": "Basic " + credentials, "Content-Type": "application/zip" },
 body: zipBuffer
 });
 if (!resp.ok) throw new Error("WebDAV upload failed: " + resp.status);
 await env.TEMP_STORE.put(fileId, "", {
 metadata: { filename: zipName, contentType: "application/zip", storage: "webdav", webdavFilename, isZip: true },
 expirationTtl: EXPIRATION_TTL
 });
 }

 try {
 await sendWeComWebhookNotification(env, {
 filename: zipName, size: zipSize,
 downloadUrl: new URL("/" + fileId, request.url).toString()
 });
 } catch (e) { notifyError = e.message; }

 return new Response(JSON.stringify({
 downloadUrl: new URL("/" + fileId, request.url).toString(),
 fileId, notifyError
 }), {
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }
 } catch (e) {
 console.error("Upload error:", e);
 return new Response(JSON.stringify({ error: e.message || "服务器内部错误" }), {
 status: 500,
 headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
 });
 }
 }

 // 4. 文件下载：通过 /{id} 访问（要求 ID 恰好 6 位）
 const segments = pathname.split("/").filter(Boolean);
 if (segments.length === 1 && segments[0].length === 6) {
 const id = segments[0];
 const reservedPaths = new Set(["api", "upload", "f", "favicon.ico", "robots.txt", "about", "s", "webdav"]);
 if (reservedPaths.has(id)) {
 return new Response("Reserved path", { status: 400 });
 }
 const entry = await env.TEMP_STORE.getWithMetadata(id, "arrayBuffer");
 if (!entry?.metadata) {
 return new Response("File not found", { status: 404 });
 }
 const { storage, webdavFilename, filename, contentType } = entry.metadata;

 if (storage === "kv") {
 return new Response(entry.value, {
 headers: {
 "Content-Type": contentType,
 "Content-Disposition": buildContentDisposition(filename),
 "Cache-Control": "no-store"
 }
 });
 }
 if (storage === "webdav" && webdavFilename) {
 const WEBDAV_BASE = "https://YOUR_WEBDAV_HOST/dav/YOUR_PATH/";
 const credentials = btoa(`${env.WEBDAV_ACCOUNT}:${env.WEBDAV_PASSWORD}`);
 const webdavUrl = WEBDAV_BASE + encodeURIComponent(webdavFilename);
 let resp;
 try {
 resp = await fetch(webdavUrl, {
 headers: { "Authorization": "Basic " + credentials }
 });
 } catch (e) {
 return new Response("Storage unavailable", { status: 502 });
 }
 if (!resp.ok) {
 return new Response("File not found", { status: 404 });
 }
 const headers = new Headers({
 "Content-Type": contentType,
 "Content-Disposition": buildContentDisposition(filename),
 "Cache-Control": "no-store",
 "Access-Control-Allow-Origin": "*"
 });
 return new Response(resp.body, { status: 200, headers });
 }
 return new Response("Invalid file record", { status: 500 });
 }

 // 5. 未匹配路由 → 404
 return new Response("Not Found", { status: 404 });
 }
};
```

</details>

## 第三步：绑定 KV 命名空间

目标：让 Worker 能读写你刚创建的 `TEMP_STORE`。

操作路径：Worker 编辑页面 → 顶部标签栏「绑定」

操作步骤：

1. 点击「添加绑定」→ 选择「KV 命名空间」
2. 变量名称（Variable name）：`TEMP_STORE`（必须与代码里的 `env.TEMP_STORE` 一致）
3. KV 命名空间：选择你刚创建的 `TEMP_STORE`
4. 点击「添加」

## 第四步：配置环境变量 / Secrets

这份代码用到了 WebDAV 与企业微信通知（可选），建议用 Secrets 配置：

- `WEBDAV_ACCOUNT`：WebDAV 用户名
- `WEBDAV_PASSWORD`：WebDAV 密码
- `WECOM_WEBHOOK_URL`：企业微信机器人 Webhook（不需要可以不填）

> WebDAV 目标目录在代码里以 `WEBDAV_BASE` 常量定义，按你的实际地址修改即可。

操作路径：Worker →「变量和机密」→「添加」

## 第五步：绑定自定义域名路由

如果你希望使用自己的域名访问（链接更短），可以给 Worker 绑定自定义域名。

操作路径：Worker 详情页 → 顶部标签栏「设置 / 域和路由」

操作步骤（简版）：

1. 添加自定义域名（例如 `tmp.yourdomain.com`）
2. 按提示在 DNS 里加记录
3. 生效后，用该域名访问 `/` 即可

## 第六步：验证功能

- 首页访问：浏览器打开 `https://你的域名/`，应显示上传页面
- 上传：选择文件点击上传（单文件最大 99MB，多文件打包 ZIP 总上限 50MB），返回下载链接
- 下载：打开下载链接，应以附件形式下载并保留原文件名
- 企业微信：上传成功后应收到通知（如已配置 Webhook）

API 也可以用 curl 快速测一下：

```bash
curl -X POST https://你的域名/api/upload-public \
 -F "file=@test.txt"
```

成功响应示例：

```json
{"downloadUrl":"https://你的域名/abcdef","fileId":"abcdef"}
```
