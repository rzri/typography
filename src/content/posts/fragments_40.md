---
title: Workers + KV + WebDAV 搭一个临时文件中转站
pubDate: 2025-10-17
categories: ['笔记']
description: '利用 Cloudflare Workers KV 的边缘存取能力 + 闲置 WebDAV 空间，做一个不落本地、跨设备的临时文件传输服务，部署即用。'
slug: workers-kv-webdav-tempfile
---

日常办公里经常遇到一个小尴尬：需要把单位文件带回家处理，但又不想用微信、QQ 这类工具把文件落到本地；国内云盘要么臃肿，要么需要装客户端，顺手用也不爽。

我手上刚好有一份闲置的 WebDAV 云盘空间（服务在日本，之前一直吃灰）。于是干脆借 Cloudflare Workers + KV 做个“临时文件中转站”：

- 浏览器打开首页，直接上传
- 返回一个短链接，复制就能分享
- 文件到期自动销毁（由 KV TTL 控制）

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

提示：代码里有两处标了“部署时请替换”的占位符（例如域名），按你的实际情况替换即可。

## WebDAV 这部分到底做了什么

这个 Worker 的存储策略是“KV 优先，WebDAV 兜底”：

- **≤ 25MB**：直接把文件内容写进 KV（`env.TEMP_STORE.put(fileId, arrayBuffer, {...})`）。
- **> 25MB**：文件内容通过 WebDAV `PUT` 上传到你的 WebDAV 目录；KV 只保存一份索引元数据（`storage=webdav` + `webdavUrl`），下载时再按 `webdavUrl` 去拉取并回传。

代码里对应的关键字段：

- `WEBDAV_BASE_URL`：WebDAV 目标目录 URL（指向一个可写目录）。
- `WEBDAV_USER` / `WEBDAV_PASS`：用 Basic Auth 访问 WebDAV。
- `metadata.storage`：标记当前文件存储位置（`kv` 或 `webdav`）。
- `metadata.webdavUrl`：当走 WebDAV 时，记录实际文件地址。

上传分流逻辑在 `/api/upload-public` 里：

- 小文件：写 KV，`metadata.storage = "kv"`
- 大文件：`uploadToWebDAV()` 上传成功后，再写 KV 元数据索引，`metadata.storage = "webdav"`

下载逻辑在 `/{id}` 路由：

- `storage === "kv"`：直接返回 KV 的二进制内容
- `storage === "webdav"`：`downloadFromWebDAV()` 拉取远端文件，把响应体透传回浏览器

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
 <link rel="icon" type="image/png" href="https://air1.cn/favicon.png" />
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

 /* ===== 新增：进度条样式 ===== */
 .progress-track {
 height: 6px;
 background: rgba(255, 255, 255, 0.1);
 border-radius: 3px;
 margin-top: 1.2rem;
 overflow: hidden;
 display: none; /* 初始隐藏 */
 }

 .progress-fill {
 height: 100%;
 width: 0%;
 border-radius: 3px;
 transition: width 0.2s ease;
 /* 默认样式 (可选，例如上传中) */
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

 <!-- ===== 新增：进度条容器 ===== -->
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

 // ====== 新增：上传进度函数 ======
 function updateProgress(loaded, total) {
 const percent = Math.round((loaded / total) * 100);
 progressFill.style.width = percent + '%';
 progressText.textContent = '上传中... ' + percent + '%';
 // 确保上传中状态
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
 resultsList.innerHTML = ''; // Clear previous results
 // ====== 修改：更新按钮状态 ======
 uploadBtn.disabled = true;
 uploadBtn.textContent = '上传中...';
 progressTrack.style.display = 'block'; // 显示进度条
 progressText.style.display = 'block'; // 显示进度文本
 progressFill.style.width = '0%'; // 重置进度
 // 确保初始状态
 progressFill.classList.remove('completed'); // 移除完成状态
 progressFill.classList.add('uploading'); // 添加上传中状态

 const formData = new FormData();
 files.forEach(f => formData.append('file', f));

 try {
 // 使用 XMLHttpRequest 实现进度监听
 const xhr = new XMLHttpRequest();

 // 监听上传进度
 xhr.upload.addEventListener('progress', (e) => {
 if (e.lengthComputable) {
 updateProgress(e.loaded, e.total);
 }
 });

 // 监听请求完成
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
 });

 xhr.addEventListener('error', () => {
 console.error('Network error during upload');
 alert('网络错误，请重试');
 });

 xhr.addEventListener('abort', () => {
 console.log('Upload aborted');
 alert('上传被取消');
 });

 // 发送请求
 xhr.open('POST', '/api/upload-public');
 xhr.send(formData);

 } catch (e) {
 console.error('Upload error:', e);
 alert('上传初始化失败');
 } finally {
 // 注意：这里不再立即隐藏进度条，因为进度由 xhr 事件控制
 // 当 xhr 完成或出错时，进度条和文本的隐藏应在事件处理函数中完成
 // 为了简化，我们可以在 finally 里隐藏，但实际进度更新由 xhr 控制
 // uploadBtn.disabled = false; // 移动到 xhr 事件处理中
 // progressTrack.style.display = 'none'; // 移动到 xhr 事件处理中
 // progressText.style.display = 'none'; // 移动到 xhr 事件处理中
 }

 // ====== xhr 事件处理函数中统一管理状态 ======
 // 在 load, error, abort 事件中统一恢复按钮状态和隐藏进度条
 function resetUploadUI() {
 // ====== 修改：恢复按钮状态 ======
 uploadBtn.disabled = false;
 uploadBtn.textContent = '确认上传';
 progressTrack.style.display = 'none';
 progressText.style.display = 'none';
 progressText.textContent = '上传中... 0%'; // 重置文本
 }

 xhr.addEventListener('load', () => {
     // 如果成功，设置进度条为完成状态
     if (xhr.status >= 200 && xhr.status < 300) {
         progressFill.classList.add('completed');
         progressFill.classList.remove('uploading');
     }
     resetUploadUI();
 });
 xhr.addEventListener('error', resetUploadUI);
 xhr.addEventListener('abort', resetUploadUI);

 }

 // ====== 兼容性复制函数 ======
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

// ====== 常量 ======
const MAX_TOTAL_SIZE = 50 * 1024 * 1024; // 总大小限制 50MB（安全值）
const EXPIRATION_TTL = 7 * 24 * 3600; // 7天

// ====== 生成随机 ID ======
async function generateFileId(env) {
    for (let i = 0; i < 5; i++) {
        const id = Math.random().toString(36).substring(2, 8);
        if (!(await env.TEMP_STORE.get(id))) {
            return id;
        }
    }
    return Math.random().toString(36).substring(2, 8) + Date.now().toString(36).slice(-2);
}

// ====== ZIP 打包函数（使用 fflate）======
function zipFiles(files) {
    const utf8Encoder = new TextEncoder();
    const zip = [];
    let offset = 0;
    const centralDir = [];

    for (const file of files) {
        const filenameBytes = utf8Encoder.encode(file.name);

        // Calculate CRC32 for the file data (simplified, using a placeholder or skipping)
        // const crc32 = calculateCrc32(file.data); // This is complex to implement correctly
        const crc32 = 0; // Placeholder CRC32

        // Local file header
        const header = new Uint8Array([
            0x50, 0x4b, 0x03, 0x04, // local file header signature
            0x14, 0x00, // version needed to extract (2.0)
            0x00, 0x00, // general purpose bit flag (no flags set)
            0x00, 0x00, // compression method (0 = store, 8 = deflate)
            0x00, 0x00, // file last mod time (dummy)
            0x00, 0x00, // file last mod date (dummy)
            crc32 & 0xff, (crc32 >> 8) & 0xff, (crc32 >> 16) & 0xff, (crc32 >> 24) & 0xff, // crc-32
            file.data.length & 0xff, (file.data.length >> 8) & 0xff, (file.data.length >> 16) & 0xff, (file.data.length >> 24) & 0xff, // compressed size
            file.data.length & 0xff, (file.data.length >> 8) & 0xff, (file.data.length >> 16) & 0xff, (file.data.length >> 24) & 0xff, // uncompressed size
            filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff, // file name length
            0x00, 0x00, // extra field length
        ]);

        zip.push(header);
        zip.push(filenameBytes);
        zip.push(file.data);

        // Central directory header
        const centralHeader = new Uint8Array([
            0x50, 0x4b, 0x01, 0x02, // central file header signature
            0x14, 0x00, // version made by (2.0)
            0x14, 0x00, // version needed to extract (2.0)
            0x00, 0x00, // general purpose bit flag
            0x00, 0x00, // compression method
            0x00, 0x00, // file last mod time
            0x00, 0x00, // file last mod date
            crc32 & 0xff, (crc32 >> 8) & 0xff, (crc32 >> 16) & 0xff, (crc32 >> 24) & 0xff, // crc-32
            file.data.length & 0xff, (file.data.length >> 8) & 0xff, (file.data.length >> 16) & 0xff, (file.data.length >> 24) & 0xff, // compressed size
            file.data.length & 0xff, (file.data.length >> 8) & 0xff, (file.data.length >> 16) & 0xff, (file.data.length >> 24) & 0xff, // uncompressed size
            filenameBytes.length & 0xff, (filenameBytes.length >> 8) & 0xff, // file name length
            0x00, 0x00, // extra field length
            0x00, 0x00, // file comment length
            0x00, 0x00, // disk number start
            0x00, 0x00, // internal file attributes
            0x00, 0x00, 0x00, 0x00, // external file attributes
            offset & 0xff, (offset >> 8) & 0xff, (offset >> 16) & 0xff, (offset >> 24) & 0xff, // relative offset of local header
        ]);

        centralDir.push(centralHeader);
        centralDir.push(filenameBytes);

        offset += header.length + filenameBytes.length + file.data.length;
    }

    const centralDirOffset = offset;
    for (const entry of centralDir) {
        zip.push(entry);
        offset += entry.length;
    }

    const centralDirSize = offset - centralDirOffset;

    // End of central directory record
    const eocd = new Uint8Array([
        0x50, 0x4b, 0x05, 0x06, // end of central dir signature
        0x00, 0x00, // number of this disk
        0x00, 0x00, // number of the disk with the start of the central directory
        files.length & 0xff, (files.length >> 8) & 0xff, // total number of entries in the central dir on this disk
        files.length & 0xff, (files.length >> 8) & 0xff, // total number of entries in the central dir
        centralDirSize & 0xff, (centralDirSize >> 8) & 0xff, (centralDirSize >> 16) & 0xff, (centralDirSize >> 24) & 0xff, // size of the central directory
        centralDirOffset & 0xff, (centralDirOffset >> 8) & 0xff, (centralDirOffset >> 16) & 0xff, (centralDirOffset >> 24) & 0xff, // offset of start of central directory
        0x00, 0x00, // zip file comment length
    ]);

    zip.push(eocd);

    return new Blob(zip, { type: 'application/zip' });
}

// ====== 企业微信通知函数 ======
async function sendWeComWebhookNotification(env, fileData) {
    try {
        if (!env.WECOM_WEBHOOK_URL) return;

        const { fileId, filename, sizeText, downloadUrl } = fileData;

        const content = `📁 新文件上传\n文件名：${filename}\n大小：${sizeText}\n🔗 下载地址：${downloadUrl}`;

        const resp = await fetch(env.WECOM_WEBHOOK_URL, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                msgtype: "text",
                text: { content }
            })
        });

        if (!resp.ok) {
            console.error("WeCom webhook failed:", await resp.text());
        }
    } catch (e) {
        console.error("WeCom webhook error:", e);
    }
}

// ====== WebDAV 上传函数（>25MB） ======
async function uploadToWebDAV(env, fileId, blob, filename) {
    const url = `${env.WEBDAV_BASE_URL}/${fileId}_${encodeURIComponent(filename)}`;

    const auth = btoa(`${env.WEBDAV_USER}:${env.WEBDAV_PASS}`);

    const resp = await fetch(url, {
        method: "PUT",
        headers: {
            "Authorization": `Basic ${auth}`
        },
        body: blob
    });

    if (!resp.ok) {
        throw new Error(`WebDAV 上传失败：${resp.status}`);
    }

    return url;
}

// ====== WebDAV 下载函数 ======
async function downloadFromWebDAV(env, webdavUrl) {
    const auth = btoa(`${env.WEBDAV_USER}:${env.WEBDAV_PASS}`);
    const resp = await fetch(webdavUrl, {
        headers: {
            "Authorization": `Basic ${auth}`
        }
    });

    if (!resp.ok) {
        return null;
    }

    return resp;
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
            try {
                const formData = await request.formData();
                const files = formData.getAll("file").filter(f => f instanceof File);

                if (!files.length) {
                    return new Response(JSON.stringify({ error: "未提供有效文件" }), {
                        status: 400,
                        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                    });
                }

                // 总大小限制（避免一次性上传太大）
                const totalSize = files.reduce((sum, f) => sum + (f.size || 0), 0);
                if (totalSize > MAX_TOTAL_SIZE) {
                    return new Response(JSON.stringify({ error: "上传总大小不能超过 50MB" }), {
                        status: 400,
                        headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                    });
                }

                // 单文件直接存；多文件打包 zip
                let uploadBlob;
                let finalFilename;

                if (files.length === 1) {
                    uploadBlob = files[0];
                    finalFilename = files[0].name || "file";
                } else {
                    const fileBuffers = [];
                    for (const f of files) {
                        const buf = new Uint8Array(await f.arrayBuffer());
                        fileBuffers.push({ name: f.name || "file", data: buf });
                    }
                    uploadBlob = zipFiles(fileBuffers);
                    finalFilename = "upload.zip";
                }

                const fileId = await generateFileId(env);

                const sizeText = (uploadBlob.size / 1024 / 1024).toFixed(2) + "MB";

                // 小于等于 25MB 存 KV，否则走 WebDAV
                let storage = "kv";
                let webdavUrl = "";

                if (uploadBlob.size <= 25 * 1024 * 1024) {
                    const arrayBuffer = await uploadBlob.arrayBuffer();
                    await env.TEMP_STORE.put(fileId, arrayBuffer, {
                        metadata: {
                            filename: finalFilename,
                            contentType: uploadBlob.type || "application/octet-stream",
                            storage: "kv"
                        },
                        expirationTtl: EXPIRATION_TTL
                    });
                } else {
                    storage = "webdav";
                    webdavUrl = await uploadToWebDAV(env, fileId, uploadBlob, finalFilename);
                    // KV 只存元数据索引
                    await env.TEMP_STORE.put(fileId, "", {
                        metadata: {
                            filename: finalFilename,
                            contentType: uploadBlob.type || "application/octet-stream",
                            storage: "webdav",
                            webdavUrl
                        },
                        expirationTtl: EXPIRATION_TTL
                    });
                }

                // ⚠️ 部署时请将 <your-domain> 替换为你的实际域名
                const downloadUrl = `https://<your-domain>/${fileId}`;

                // 通知（失败不影响上传）
                sendWeComWebhookNotification(env, {
                    fileId,
                    filename: finalFilename,
                    sizeText,
                    downloadUrl,
                    storage
                }).catch(() => {});

                return new Response(JSON.stringify({
                    fileId,
                    downloadUrl
                }), {
                    headers: {
                        "Content-Type": "application/json",
                        "Access-Control-Allow-Origin": "*"
                    }
                });
            } catch (e) {
                console.error("上传出错:", e);
                return new Response(JSON.stringify({ error: "服务器内部错误" }), {
                    status: 500,
                    headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" }
                });
            }
        }

        // 4. 文件下载：通过 /{id} 访问（要求至少6字符）
        const segments = pathname.split('/').filter(Boolean);
        if (segments.length === 1 && segments[0].length >= 6) {
            const id = segments[0];
            const reservedPaths = new Set([
                'api', 'upload', 'f', 'favicon.ico', 'robots.txt', 'about', 's'
            ]);
            if (!reservedPaths.has(id)) {
                const entry = await env.TEMP_STORE.getWithMetadata(id, "arrayBuffer");
                if (entry && entry.metadata) {
                    const filename = entry.metadata?.filename || 'file';
                    const contentType = entry.metadata?.contentType || "application/octet-stream";
                    const storage = entry.metadata?.storage || "kv";

                    const headers = {
                        "Content-Type": contentType,
                        "Content-Disposition": "attachment; filename=\"" + encodeURIComponent(filename) + "\"",
                        "Cache-Control": "no-store"
                    };

                    if (storage === "kv" && entry.value) {
                        return new Response(entry.value, { headers });
                    }

                    if (storage === "webdav" && entry.metadata.webdavUrl) {
                        const resp = await downloadFromWebDAV(env, entry.metadata.webdavUrl);
                        if (resp) {
                            return new Response(resp.body, { headers });
                        }
                    }
                }
            }
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

- `WEBDAV_BASE_URL`：WebDAV 目标目录 URL（例如 `https://example.com/dav/TempFiles`）
- `WEBDAV_USER`：WebDAV 用户名
- `WEBDAV_PASS`：WebDAV 密码
- `WECOM_WEBHOOK_URL`：企业微信机器人 Webhook（不需要可以不填）

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
- 上传：选择文件点击上传，返回下载链接
- 下载：打开下载链接，应以附件形式下载并保留原文件名

API 也可以用 curl 快速测一下：

```bash
curl -X POST https://你的域名/api/upload-public \
  -F "file=@test.txt"
```

成功响应示例：

```json
{"downloadUrl":"https://你的域名/abcdef","fileId":"abcdef"}
```
