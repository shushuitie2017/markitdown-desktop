# 目录结构说明

```
markitdown-desktop/
├── main.py                          # 进程入口
├── server.py                        # FastAPI 后端（全部服务端逻辑）
├── requirements.txt                 # Python 依赖（4 行，未 pin 版本）
├── README.md                        # 项目简介 + Ubuntu/systemd 部署手顺 + API 表
├── .gitignore                       # 排除 venv/.env/build/*.zip/servers.json/*.pem 等
├── servers.json                     # 部署连接信息（gitignored，权威来源，绝不入库/外传）
├── MarkItDown-1.0.0-win-x64.zip     # （可选，当前不在仓库）单机版下载包，来自 markitdown-electron 构建
├── docs/                            # 本接手文档目录
├── static/                          # 前端（无构建步骤，uvicorn 直接静态服务）
│   ├── index.html                   # 单页骨架：上传区/加载态/错误条/输出区（Preview·Raw 双 tab）
│   ├── style.css                    # 样式（浅色主题，BEM 命名）
│   └── script.js                    # 全部前端逻辑（IIFE）：拖拽上传/校验/转换/迷你 MD 渲染器/复制/保存
├── .git/                            # git 仓库
└── .claude/                         # Claude Code 项目配置（claude-flow 播的基线）
    ├── settings.json                # 项目级 Claude 设置
    ├── rules.json                   # project-guard 硬规则：禁 scp 外泄 .env/.db/.pem、禁 git add .env/config.json
    ├── hooks/                       # PreToolUse 守卫钩子（project-guard.py）
    ├── skills/                      # 项目内快照的 skill
    ├── memory/                      # 项目记忆
    └── flow.json                    # claude-flow 快照元数据
```

## 逐文件说明

### main.py（42 行）
唯一入口。读 `MARKITDOWN_HOST`（默认 `127.0.0.1`）/ `MARKITDOWN_PORT`（默认 `8877`）→ 打印 banner → 若监听本机则起 daemon 线程延迟 1.5 秒 `webbrowser.open`（桌面式体验）→ `uvicorn.run("server:app")`。部署时设 `MARKITDOWN_HOST=0.0.0.0` 即不开浏览器。

### server.py（99 行）
FastAPI 应用本体，四个路由 + 静态挂载，详见 [ARCHITECTURE.md](ARCHITECTURE.md)。关键常量：`MAX_FILE_SIZE`（50MB）、`SUPPORTED_EXTENSIONS`（21 个扩展名）、`DOWNLOAD_FILE`（单机版 zip 路径，文件名 1.0.0 写死——发新版单机包要同步改这里）。

### requirements.txt
`markitdown[all]` / `fastapi` / `uvicorn` / `python-multipart`。全部未锁版本。

### static/script.js（487 行）
前端全逻辑：常量（与后端镜像的扩展名白名单 + 50MB）、拖拽/文件选择、`/api/convert` 调用、输出双视图切换、Copy/Save/Clear 动作、自带迷你 Markdown → HTML 渲染器（含 escapeHtml 防注入）。**无任何 npm 依赖，无需构建**。

### static/index.html
英文界面单页。`<input accept>` 列的扩展名与 JS/后端白名单一致（改格式支持时须同步）。头部有 "Desktop App" 下载按钮指向 `/download`。

### servers.json
存在于本地但被 gitignore。部署主机/端口/凭据等连接信息以此为准，文档中一律只写「见 servers.json（gitignored）」。

### .claude/
claude-flow 铺开的项目守卫基线（rules.json 两条硬规则针对凭据外泄），与业务功能无关，接手开发时一般不需要动。
