# markitdown-desktop 接手文档导读

> 本目录是该项目的「接手文档」：让新接手者不靠口传即可了解全貌。
> 生成日期：2026-07-12。以当日代码为准；若代码已演进，以代码为准并更新本目录。

## 项目一句话

基于 Microsoft [markitdown](https://github.com/microsoft/markitdown) 库的**文档转 Markdown Web 服务**：FastAPI 后端 + 原生 JS 单页前端，上传 PDF/DOCX/PPTX/XLSX/图片/音频等文件，返回 Markdown 文本（可预览、复制、下载 .md）。本地开发端口 **8877**。

## 文档索引

| 文档 | 内容 |
|------|------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | 技术栈与版本、分层结构、转换管线、API 端点 |
| [DIRECTORY.md](DIRECTORY.md) | 目录树逐项说明 |
| [DEVELOPMENT.md](DEVELOPMENT.md) | 本地环境、依赖安装、启动命令、已知坑 |
| [DEPLOYMENT.md](DEPLOYMENT.md) | 部署形态（systemd + 8877）、线上现状、运维命令 |

## 快速上手（30 秒版）

```bash
cd server-projects/markitdown-desktop
python -m venv venv
venv/Scripts/pip install -r requirements.txt   # Windows；Linux 为 venv/bin/pip
venv/Scripts/python main.py                    # 启动后自动打开浏览器 http://127.0.0.1:8877
```

## 与 markitdown-electron 的关系

同级目录 `server-projects/markitdown-electron` 是**独立项目**：它负责构建本服务的桌面单机版下载包（Electron 客户端）。二者的交接点只有一个文件——构建产物 `MarkItDown-1.0.0-win-x64.zip` 放到**本项目根目录**后，由本服务的 `GET /download` 端点向用户分发（`server.py` 中 `DOWNLOAD_FILE` 常量写死此文件名；文件不存在时返回 404 "Download not available"）。该 zip 被 `.gitignore` 的 `*.zip` 规则排除，不入库。markitdown-electron 内部细节请看该项目自己的文档，此处不展开。

## 关键事实速查

- **入口**：`main.py`（读环境变量 `MARKITDOWN_HOST` / `MARKITDOWN_PORT`，默认 `127.0.0.1:8877`，本机监听时自动开浏览器）
- **后端**：`server.py`（FastAPI 单文件，4 个路由）
- **前端**：`static/`（无框架、无构建步骤，直接静态服务）
- **无数据库、无用户系统、无持久化**——每次转换用完即删临时文件，服务无状态
- **servers.json 存在**（gitignored）：部署连接信息以它为权威，绝不入库、绝不写进文档
- **注意**：项目名叫 "desktop"（历史沿革，README 自称 "MarkItDown Web"），但本仓库是 Web 服务本体；桌面版由 markitdown-electron 打包
