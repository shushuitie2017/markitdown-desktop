# 本地开发指南

## 环境要求

- **Python 3.12+**（workspace 约定；代码无版本硬检查）
- Windows / Linux / macOS 均可跑（本机开发环境为 Windows + Git Bash）
- 无 Node.js 要求——前端零构建
- 无数据库、无 `.env` 必需项——克隆即跑

## 安装与启动

```bash
cd server-projects/markitdown-desktop

# 建虚拟环境 + 装依赖
python -m venv venv
venv/Scripts/pip install -r requirements.txt    # Windows
# Linux/macOS: venv/bin/pip install -r requirements.txt

# 启动（默认 127.0.0.1:8877，会自动打开浏览器）
venv/Scripts/python main.py
# Linux/macOS: venv/bin/python main.py
```

启动成功的标志：控制台打出 `MarkItDown Desktop` banner + uvicorn 日志，约 1.5 秒后浏览器自动打开 `http://127.0.0.1:8877`。

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MARKITDOWN_HOST` | `127.0.0.1` | 监听地址；设为 `0.0.0.0`（部署）时不再自动开浏览器 |
| `MARKITDOWN_PORT` | `8877` | 监听端口（workspace 端口分配表中本项目专属 8877） |

## 手工验证

```bash
# 格式清单
curl http://127.0.0.1:8877/api/formats

# 转换一个文件
curl -X POST http://127.0.0.1:8877/api/convert -F "file=@test.pdf"

# 下载端点（根目录无 zip 时预期 404 "Download not available"，属正常）
curl -i http://127.0.0.1:8877/download
```

项目**没有自动化测试**（无 tests/ 目录）——验证靠上面的 curl + 浏览器实操（拖文件 → Convert → Preview/Raw → Copy/Save）。

## 常见开发任务

### 加一种支持格式
四处同步（缺一处会出现「前端拦了后端其实支持」或反之的错位）：
1. `server.py` → `SUPPORTED_EXTENSIONS` 集合
2. `server.py` → `/api/formats` 的 `categories` 分类
3. `static/script.js` → `ALLOWED_EXTENSIONS` 集合
4. `static/index.html` → `<input accept=...>` 列表

另外确认 markitdown 库真的支持该格式（`markitdown[all]` 已含大多数解析器）。

### 改前端
`static/` 下直接改，刷新浏览器即生效（无热重载需求）。注意 uvicorn 默认不带 `--reload`，改 **Python** 文件后要手动重启进程。

## 已知坑

- **依赖未 pin 版本**：`requirements.txt` 四个包全裸写，`markitdown` 迭代较快，重装后行为可能变化。出现「以前能转现在报错」优先怀疑依赖漂移。
- **转换失败常见原因**：markitdown 对图片/音频类格式的深度提取（OCR、语音转写）依赖可选组件与外部条件，某些文件即使扩展名合法也会 500，错误信息里带异常类名（`Conversion failed (XxxError): ...`），照类名去 markitdown 上游查。
- **50MB 上限**是全量读入内存后才判断的（`await file.read()`），恶意/超大上传的内存峰值在此；改上限时留意机器内存。
- **`DOWNLOAD_FILE` 文件名写死** `MarkItDown-1.0.0-win-x64.zip`：markitdown-electron 发新版后，要么保持产物同名，要么同步改 `server.py` 常量。
- **端口冲突**：8877 是本项目在 workspace 端口分配表中的专属端口；若被占用（上次进程没杀干净），Windows 下用 PowerShell `Get-NetTCPConnection -LocalPort 8877` 查 PID 后 `Stop-Process`（勿用 Git Bash 管道跑 taskkill，会留卡死壳——workspace 已知坑）。
- **Windows 编码**：本项目源码/界面全英文，暂无中文输出编码问题；若日后加中文日志，按 workspace 规范强制 UTF-8 stdout。
