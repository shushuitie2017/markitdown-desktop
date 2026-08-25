# 技术架构

## 技术栈

| 层 | 技术 | 版本 | 说明 |
|----|------|------|------|
| 转换引擎 | `markitdown[all]` | requirements.txt 未锁版本 ⚠️ | Microsoft 官方库，`[all]` extra 装全部格式解析器（PDF/DOCX/PPTX/音频等） |
| Web 框架 | FastAPI | 未锁版本 ⚠️ | 单文件应用 `server.py` |
| ASGI 服务器 | uvicorn | 未锁版本 ⚠️ | 由 `main.py` 以 `uvicorn.run("server:app", ...)` 启动 |
| 表单解析 | python-multipart | 未锁版本 ⚠️ | FastAPI `UploadFile` 依赖 |
| 前端 | 原生 HTML/CSS/JS | — | 无框架、无构建步骤、无外部 CDN 依赖 |
| Python | 3.12+（workspace 约定） | — | 代码本身无版本硬性检查 |

⚠️ 待确认：`requirements.txt` 四个依赖全部未 pin 版本，重装可能漂移；接手后建议 `pip freeze` 锁一份。

## 分层结构

```
浏览器（static/ 单页应用）
    │  fetch POST /api/convert (multipart/form-data)
    ▼
main.py ── 进程入口：读环境变量 → 启动 uvicorn →（本机监听时）延迟 1.5s 开浏览器
    ▼
server.py ── FastAPI 应用（全部后端逻辑，约 100 行）
    │   ├─ 静态文件挂载 /static
    │   ├─ 扩展名白名单 + 50MB 大小闸门
    │   └─ MarkItDown 转换器（模块级单例 converter = MarkItDown()）
    ▼
markitdown 库 ── 实际的格式解析与 Markdown 生成
```

前后端各自维护一份**相同的**校验常量（`SUPPORTED_EXTENSIONS` / `ALLOWED_EXTENSIONS`、50MB 上限）：前端先拦一道给即时反馈，后端再拦一道做真正的安全闸。**改支持格式时四处要同步**：`server.py` 的 `SUPPORTED_EXTENSIONS`、`script.js` 的 `ALLOWED_EXTENSIONS`、`index.html` 的 `<input accept>`、`server.py` `/api/formats` 的 `categories` 分类。

## 转换管线（POST /api/convert）

1. 取 `UploadFile.filename` 后缀（小写）→ 不在 `SUPPORTED_EXTENSIONS` 白名单 → 400
2. `await file.read()` 全量读入内存 → 超过 `MAX_FILE_SIZE`（50MB）→ 400
3. `tempfile.mkdtemp(prefix="markitdown_")` 建临时目录，按原文件名落盘
4. `await asyncio.to_thread(converter.convert, path)` —— 转换是同步阻塞调用，放线程池避免卡住事件循环
5. 成功返回 `{"success": true, "filename", "markdown": result.text_content, "title"}`
6. 任何异常 → 500，detail 为 `Conversion failed (<异常类名>): <消息>`
7. `finally` 中 `shutil.rmtree(tmp_dir, ignore_errors=True)` 清临时目录——**无残留、无历史记录、无状态**

## API 端点

| 方法 | 路径 | 请求 | 响应 | 说明 |
|------|------|------|------|------|
| GET | `/` | — | `static/index.html` | 前端单页 |
| GET | `/download` | — | zip 文件流 | 分发根目录 `MarkItDown-1.0.0-win-x64.zip`（markitdown-electron 的构建产物）；文件不存在 → 404 |
| GET | `/api/formats` | — | `{extensions: [...], categories: {...}}` | 支持格式清单（documents/web/data/media/archive/email 六类） |
| POST | `/api/convert` | multipart `file` 字段 | 见上节 | 核心转换端点 |
| GET | `/static/*` | — | 静态文件 | StaticFiles 挂载 |

注意：错误响应走 FastAPI 默认的 `{"detail": "..."}` 结构，**不是** workspace Flask 规范的 `{"error": "..."}`；前端 `script.js` 按 `err.detail` 解析，二者已配套，勿单边改。

## 支持的文件格式（21 种扩展名）

`.pdf .docx .pptx .xlsx .xls .html .htm .csv .json .xml .jpg .jpeg .png .wav .mp3 .m4a .zip .epub .ipynb .msg`

## 前端要点（static/script.js，IIFE 单文件）

- 拖拽/点击选文件 → 双重校验 → `FormData` POST → 结果双视图（Preview 渲染 / Raw 原文）
- **自带迷你 Markdown 渲染器**（`renderMarkdown`，约 180 行）：支持标题/表格/列表/引用/代码块/行内格式；代码块与行内代码先用 `\x00` 占位符抽出、escapeHtml 后回填，防 XSS
- 操作：Copy（clipboard API）、Save .md（Blob + a[download]）、Convert Another（重置状态）
- 无 i18n——界面纯英文（workspace 三语规范在此项目未实施 ⚠️ 待确认是否需要补）

## 安全与限制

- 文件大小上限 50MB（前后端各一道）
- 扩展名白名单（仅按后缀判断，不做 magic-number 嗅探）
- 全量读入内存再落盘——超大并发上传时内存压力点在 `await file.read()`
- 无鉴权：任何能访问端口的人都能用转换与下载端点
- 无 CORS 中间件：默认同源使用（前端由同一服务托管，无跨域需求）
