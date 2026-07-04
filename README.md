# Check ProxyIP

基于 Cloudflare Workers 的 Socks5 代理 IP 连通性检测工具。支持批量检测代理是否可用、是否支持 IPv4/IPv6，并通过 GitHub Actions 每日自动巡检。

## 功能特性

- **代理连通性检测**：通过 TCP + TLS 直连代理，验证代理是否可达
- **IPv4/IPv6 双栈检测**：使用两个独立探针（`ipv4.090227.xyz` / `ipv6.090227.xyz`）精准判断出口协议栈
- **批量解析**：支持通过 DoH（DNS over HTTPS）解析域名代理的 A/AAAA/TXT 记录
- **Web 可视化面板**：内置暗色/亮色主题的前端界面，支持地图展示代理地理位置
- **GitHub Actions 自动化**：每天定时检测，自动生成可用代理列表和详细检测日报

## 工作原理

```
用户输入代理 IP:PORT → Worker TCP 直连代理 → TLS 握手 → 通过代理请求探针服务器
                                                              │
                                              ┌─────────────────┴──────────────────┐
                                              ↓                                    ↓
                                     ipv4.090227.xyz                       ipv6.090227.xyz
                                     (检测 IPv4 出口)                      (检测 IPv6 出口)
```

Worker 内置了一套完整的 TLS 1.2/1.3 客户端实现（基于 Web Crypto API），不依赖任何第三方库，直接通过 `cloudflare:sockets` 发起 TCP 连接。

## 快速开始

### 1. 部署 Cloudflare Worker

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages** → **创建应用程序** → **创建 Worker**
3. 将 [`worker.js`](worker.js) 的内容粘贴到代码编辑器中
4. 点击 **部署**

> 部署成功后，你的 Worker 地址类似：`https://check-proxy.你的用户名.workers.dev`

### 2. 配置 GitHub Actions 自动检测

1. **Fork 或克隆**本仓库到你的 GitHub 账号

2. **设置 Secrets**：进入仓库 **Settings → Secrets and variables → Actions**，添加以下两个 Secrets：

   | Secret 名称 | 说明 | 示例值 |
   |-------------|------|--------|
   | `WORKER_URL` | 你部署的 Worker 地址 | `https://check-proxy.xxx.workers.dev` |
   | `PROXY_IPS` | 待检测的代理 IP 列表 | 见下方格式说明 |

3. **`PROXY_IPS` 格式**：每行一个代理，支持 `#` 后跟备注信息（地区、运营商等）：

   ```
   221.124.53.3:40001#HK Hong Kong AS9304 HGC Global Communications Limited
   43.255.156.6:444#HK Tung Chung AS932 VH Global Limited
   43.255.156.57:444#HK Tung Chung AS932 VH Global Limited
   38.207.160.48:9443#HK Harmony Garden AS967 VMISS Inc.
   43.255.156.244:444#HK Tung Chung AS932 VH Global Limited
   ```

   也可以用逗号分隔（单行）：`1.2.3.4:443,5.6.7.8:8080#备注`

4. **手动触发测试**：进入 **Actions** → **Daily Proxy Check** → **Run workflow**

## 文件结构

```
.
├── worker.js                          # Cloudflare Worker 主程序
├── .github/workflows/check-proxy.yml  # GitHub Actions 自动检测工作流
├── proxies.txt                        # [自动生成] 当前可用的代理列表（原始格式）
├── report.md                          # [自动生成] 每日详细检测报告
└── README.md                          # 本文件
```

## 手动使用

### Web 界面

直接访问你的 Worker 地址，在输入框中粘贴代理 IP 列表，点击「检测」按钮即可。

### API 接口

**单个/批量检测代理**：
```bash
# GET 方式
curl "https://你的Worker地址/check?proxyip=1.2.3.4:443"

# POST 批量（推荐）
curl -X POST "https://你的Worker地址/check" \
  -H "Content-Type: application/json" \
  -d '{"proxyips": ["1.2.3.4:443", "5.6.7.8:8080"]}'
```

**DNS 解析**：
```bash
curl "https://你的Worker地址/resolve?proxyip=example.com:443"
```

**批量解析**（上限 15 个）：
```bash
curl -X POST "https://你的Worker地址/resolve-batch" \
  -H "Content-Type: application/json" \
  -d '{"targets": ["example.com:443", "test.com:8443"]}'
```

### API 响应示例

```json
{
  "candidate": "1.2.3.4:443",
  "success": true,
  "proxyIP": "1.2.3.4",
  "portRemote": 443,
  "inferred_stack": "dual_stack",
  "supports_ipv4": true,
  "supports_ipv6": true,
  "dual_stack": true,
  "responseTime": 245,
  "colo": "HKG",
  "timeStamp": "2026-07-04T00:00:00.000Z",
  "probe_results": {
    "ipv4": {
      "candidate": "1.2.3.4:443",
      "connect_ms": 120,
      "tls_ms": 80,
      "http_ms": 45,
      "status_code": 200,
      "ok": true,
      "exit": { "ip": "203.0.113.1", "ipType": "ipv4" }
    },
    "ipv6": {
      "candidate": "1.2.3.4:443",
      "connect_ms": 130,
      "tls_ms": 85,
      "http_ms": 50,
      "status_code": 200,
      "ok": true,
      "exit": { "ip": "2001:db8::1", "ipType": "ipv6" }
    }
  }
}
```

## GitHub Actions 自动检测

### 运行频率

- **定时运行**：每天 UTC 0:00（北京时间 8:00）
- **手动触发**：在 Actions 页面点击 `Run workflow`

### 输出文件

Action 每次运行后会自动提交两个文件：

| 文件 | 说明 |
|------|------|
| `proxies.txt` | 检测成功的代理列表，保留原始输入格式（含备注），每天覆盖 |
| `report.md` | 详细日报，包含概况统计、成功/失败代理详情表格、可用代理列表 |

### `report.md` 日报示例

```markdown
# 代理检测日报

**检测时间**：2026-07-04 08:00:00 (北京时间)

## 概况

| 项目 | 数值 |
|------|------|
| 检测总数 | 50 |
| 成功数量 | 42 |
| 失败数量 | 8 |
| 成功率 | 84.0% |

## 成功代理

| 代理地址 | 端口 | 协议栈 | IPv4 | IPv6 | 响应时间 |
|----------|------|--------|------|------|----------|
| 221.124.53.3 | 40001 | dual_stack | true | true | 245ms |
| 43.255.156.6 | 444 | ipv4_only | true | false | 180ms |

## 可用代理列表

\`\`\`
221.124.53.3:40001#HK Hong Kong AS9304 HGC Global Communications Limited
43.255.156.6:444#HK Tung Chung AS932 VH Global Limited
\`\`\`

## 失败代理

| 代理地址 | 失败原因 |
|----------|----------|
| 10.0.0.1:443 | tcp connect timeout after 9999ms |
```

## 自定义

- **修改检测频率**：编辑 `.github/workflows/check-proxy.yml` 中的 `cron` 表达式
- **修改探针目标**：编辑 `worker.js` 中的 `PROBE_TARGETS` 常量
- **修改超时时间**：编辑 `worker.js` 中的 `DEFAULT_TIMEOUT_MS` 常量

## 技术栈

- **运行时**：Cloudflare Workers
- **TLS 实现**：纯 JavaScript + Web Crypto API（TLS 1.2/1.3）
- **前端**：原生 HTML/CSS/JS + Leaflet 地图 + Plus Jakarta Sans 字体
- **自动化**：GitHub Actions

## 致谢

- 基于 [CF-Workers-CheckProxyIP](https://github.com/cmliu/CF-Workers-CheckProxyIP) 构建
- 探针服务由 [090227.xyz](https://090227.xyz) 提供
