# 部署说明

## 部署形态（按仓库内 README 的既定流程）

- **形态**：Ubuntu 服务器 + Python venv + **systemd 服务 `markitdown.service`**，监听 `0.0.0.0:8877`
- **无容器、无反代配置入库**：仓库内只有应用层；nginx/域名/证书按 workspace 通用做法在服务器侧维护
- **连接信息（主机 IP、密钥、用户等）一律见项目根目录 `servers.json`（gitignored），绝不写进文档或提交入库**

按 workspace 约定，`server-projects/` 下的服务部署在 chi 服务器、线上统一走 `<name>.bluecatbot.com` 子域 + nginx + certbot（完整上线流程见 `bluecat-deploy` skill）。

⚠️ 待确认：本项目**当前实际线上状态**（是否已上线、具体子域名、nginx 站点配置）无法从仓库文件核实——README 的部署手顺写的是通用 `YOUR_SERVER_IP` 占位符。接手后先看 `servers.json`，再上服务器 `systemctl status markitdown` 核实。

## 首次部署手顺（摘自根目录 README.md，权威版以其为准）

```bash
# 1. 上传（main.py / server.py / requirements.txt / static/，可选单机版 zip）
scp -i your-key.pem -r markitdown-desktop/ ubuntu@YOUR_SERVER_IP:/home/ubuntu/

# 2. 服务器上建 venv 装依赖
cd /home/ubuntu/markitdown-desktop
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 3. 手动试跑验证
MARKITDOWN_HOST=0.0.0.0 MARKITDOWN_PORT=8877 venv/bin/python main.py

# 4. systemd 单元（/etc/systemd/system/markitdown.service）
#    关键项：User=ubuntu, WorkingDirectory=/home/ubuntu/markitdown-desktop,
#    Environment=MARKITDOWN_HOST=0.0.0.0 / MARKITDOWN_PORT=8877,
#    ExecStart=<venv>/python main.py, Restart=on-failure

# 5. 启用
sudo systemctl daemon-reload && sudo systemctl enable markitdown && sudo systemctl start markitdown

# 6. 验证
curl http://localhost:8877/api/formats
```

完整单元文件文本与防火墙开口（ufw/安全组 8877）见根目录 `README.md` 第 4、7 节。

## 更新部署

```bash
# 只更新代码文件 + 重启（README「更新部署」节）
scp -i your-key.pem server.py ubuntu@YOUR_SERVER_IP:/home/ubuntu/markitdown-desktop/
scp -i your-key.pem static/* ubuntu@YOUR_SERVER_IP:/home/ubuntu/markitdown-desktop/static/
ssh -i your-key.pem ubuntu@YOUR_SERVER_IP "sudo systemctl restart markitdown"
```

依赖变更时（改了 requirements.txt）还需在服务器 venv 里重新 `pip install -r requirements.txt`。

## 运维命令

```bash
sudo journalctl -u markitdown -f     # 跟日志
sudo systemctl restart markitdown    # 重启
sudo systemctl status markitdown     # 状态
```

## 单机版下载包的发布链路

1. `markitdown-electron` 项目构建出 `MarkItDown-1.0.0-win-x64.zip`
2. 把 zip 放到服务器上本项目**根目录**（与 `server.py` 同级）
3. 线上 `GET /download` 即开始分发；不放则该端点 404（页面其余功能不受影响）
4. 版本升级时注意 `server.py` 里 `DOWNLOAD_FILE` 文件名写死 `1.0.0`，需同步

## 安全注意

- 服务本身**无鉴权**——公网暴露时任何人可用转换端点；50MB 上限与扩展名白名单是仅有的闸门
- `.gitignore` 已排除 `servers.json` / `.env` / `*.pem` / `*.zip`；`.claude/rules.json` 另有 project-guard 硬规则拦截 scp 外泄凭据类文件
- 转换临时文件用 `tempfile.mkdtemp` + 请求结束即删，服务器不留用户文件
