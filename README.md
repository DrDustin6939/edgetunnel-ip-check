# Check ProxyIP

基于 Cloudflare Workers 的 Socks5 代理 IP 连通性检测工具。支持批量检测代理是否可用、是否支持 IPv4/IPv6，并通过 GitHub Actions 每日自动巡检。

## 功能特性

- 代理连通性检测：通过 TCP + TLS 直连代理，验证代理是否可达
- IPv4/IPv6 双栈检测：使用两个独立探针精准判断出口协议栈
- 批量解析：支持通过 DoH 解析域名代理的 A/AAAA/TXT 记录
- Web 可视化面板：内置暗色/亮色主题的前端界面，支持地图展示代理地理位置
- GitHub Actions 自动化：每天定时检测，自动生成可用代理列表和详细检测日报

## 快速开始

### 1. 部署 Cloudflare Worker

1. 登录 Cloudflare Dashboard
2. 进入 Workers & Pages -> 创建应用程序 -> 创建 Worker
3. 将 worker.js 的内容粘贴到代码编辑器中
4. 点击部署

### 2. 配置 GitHub Actions 自动检测

1. Fork 或克隆本仓库到你的 GitHub 账号
2. 设置 Secrets：进入仓库 Settings -> Secrets and variables -> Actions，添加两个 Secrets：
   - WORKER_URL：你部署的 Worker 地址
   - PROXY_IPS：待检测的代理 IP 列表

## 文件结构

- worker.js - Cloudflare Worker 主程序
- .github/workflows/check-proxy.yml - GitHub Actions 自动检测工作流
- proxies.txt - [自动生成] 当前可用的代理列表
- report.md - [自动生成] 每日详细检测报告
- README.md - 本文件

## API 接口

GET /check?proxyip=IP:PORT - 单个检测
POST /check with JSON body - 批量检测

## GitHub Actions

- 定时运行：每天 UTC 0:00（北京时间 8:00）
- 手动触发：在 Actions 页面点击 Run workflow

## 致谢

基于 CF-Workers-CheckProxyIP 构建，探针服务由 090227.xyz 提供