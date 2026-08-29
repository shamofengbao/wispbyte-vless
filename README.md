# Wispbyte Node.js + Sing-Box VLESS 节点部署教程（Named Tunnel 固定隧道版）

本教程介绍如何在 Wispbyte 面板中部署 VLESS 节点，通过 Cloudflare Named Tunnel 绑定固定域名，告别 Quick Tunnel 每次重启换域名的问题。

## 许可证与商业限制

开源协议：本项目采用 CC BY-NC 4.0（署名-非商业性使用） 协议发布。

商业限制：严禁任何个人或组织将本教程、相关配置文件及代码用于任何形式的商业盈利、收费机场搭建、商业售卖或引流牟利。仅限个人技术研究与交流使用。

## 部署准备工作

### 1. Wispbyte 环境

登录 Wispbyte 控制面板，进入对应服务器的 **Files（文件管理器）** 页面。确保容器支持 Node.js 运行环境。

**Wispbyte 免费层保活规则：**

- **定期登录**：需每 14-30 天登录一次 Wispbyte 面板（有两种说法，建议 14 天登录一次最保险）
- **一人一账号一服务器**：免费层限制每个账号只能创建一台服务器，多开小号刷免费服务器会被直接删服务器 + 封号
- **超时后果**：服务器被停止/归档后，**文件全部保留**，重新登录再点启动即可恢复
- **邮件通知**：官方会在停止前发警告邮件、停止后再发确认邮件，注意查收邮箱和垃圾箱

> 只要做到定期登录面板，服务器就不会被回收。即使超时被停，文件不丢，重新登录点启动即可恢复。

### 2. 生成 UUID

使用任意 UUID 生成器生成一个 UUID（如 `b4f7fd9e-a092-490f-8532-74a756cbbf89`），后续需要填入 `config.json` 和客户端配置中。

### 3. 申请 Cloudflare 隧道

1. 注册/登录 [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) 面板
2. 进入 **Networks → Tunnels → Create a tunnel**
3. 选择 **Cloudflared** 模式，创建隧道，获取 **Token**（保存好，后续填入 `index.js`）
4. 在隧道的 **Public Hostnames** 中添加一条记录：
   - **Subdomain**：自选（如 `proxy`）
   - **Domain**：你的域名（需托管在 Cloudflare）
   - **Service**：`HTTP`
   - **URL**：`localhost:3000`（与 `index.js` 中 `PORT` 默认值一致，如 Wispbyte 分配的端口不同请相应修改）

> Cloudflare Tunnel 完全免费，不限流量、不限隧道数量。只需一个域名托管在 Cloudflare（DNS 托管免费）。没有域名可在 [FreeDomains](https://domain.stackryze.com) 申请免费子域名。

## 核心步骤详解

### 第一步：创建服务端配置文件

在文件管理器根目录下新建或编辑名为 `config.json` 的文件，将节点监听配置写入其中。

**需要修改的参数：**

- `"uuid"`：替换为你生成的 UUID

关键参数说明：

- 协议类型：vless
- 本地监听端口：10086（由后端程序进行转发，无需修改）
- UUID：你自己生成的 UUID
- 传输路径：`/api/v2/telemetry/stream_8f91a`

### 第二步：创建主控与转发脚本

在根目录下新建或编辑名为 `index.js` 的文件。该脚本会自动在线下载适配的 sing-box 内核、自动解压赋予权限，并建立安全的双向 WebSocket 流量转发通道。同时会下载 cloudflared 并使用 Named Tunnel Token 启动固定隧道。

**需要修改的参数：**

- `CF_TUNNEL_TOKEN`：替换为你从 Cloudflare 获取的 Token（也可通过 Wispbyte 环境变量 `CF_TUNNEL_TOKEN` 注入）

脚本特点：
- 强制 `--protocol http2`，避免容器环境 QUIC（UDP）超时断连
- cloudflared 进程退出时打印退出码，便于排查
- sing-box 二进制伪装为 `npm-system-worker`

### 第三步：启动与状态检查

所有的配置文件准备就绪后，切换至控制台 **Console** 页面。

点击 **Restart（重启）** 按钮重新加载容器。

检查控制台日志，确认出现以下提示即代表正常运行：

```
[+] 主服务已监听端口 3000
[+] 启动后台 sing-box 进程...
[+] 启动 Cloudflare 固定隧道 (Named Tunnel)...
[+] 隧道域名: 请在 Cloudflare Zero Trust 面板查看
[cf] ... Registered tunnel connection ... protocol=http2
```

看到 `Registered tunnel connection` 和 `protocol=http2` 即表示隧道连接成功。

## 客户端导入与连接参数

在本地代理客户端（如 v2rayN 等）中新建 VLESS 节点，填入以下参数：

| 参数 | 值 |
|------|------|
| 别名备注 | 自定义 |
| 服务器地址 | 你在 CF 绑定的域名（如 `proxy.yourdomain.com`） |
| 服务端口 | 443 |
| 用户 ID (UUID) | 与 config.json 中一致 |
| 传输协议 | ws (WebSocket) |
| 伪装路径 | /api/v2/telemetry/stream_8f91a |
| 传输层安全 (TLS) | tls |
| SNI | 与服务器地址相同 |
| 允许不安全 | 关闭 |

### 节点快捷格式（替换 YOUR_UUID 和 YOUR_DOMAIN）

```
vless://YOUR_UUID@YOUR_DOMAIN:443?encryption=none&security=tls&sni=YOUR_DOMAIN&type=ws&path=%2Fapi%2Fv2%2Ftelemetry%2Fstream_8f91a#VLESS-NamedTunnel
```

## 注意事项

- **Wispbyte 保活**：免费层需每 14-30 天登录一次面板，否则服务器会被停止/归档（文件不丢，重新登录点启动即可恢复）。多开小号会被封号。详见上方“Wispbyte 免费层保活规则”。
- **端口匹配**：CF 隧道的 Public Hostname 回源地址（如 `localhost:3000`）必须与 `index.js` 中 Node.js 主服务监听的端口一致。Wispbyte 可能通过环境变量 `PORT` 或 `SERVER_PORT` 分配不同端口，请确保两端对齐。
- **Token 安全**：更安全的做法是通过 Wispbyte 面板的环境变量 `CF_TUNNEL_TOKEN` 注入 Token，脚本会优先读取环境变量，不写死在代码里。
- **协议选择**：脚本强制使用 `http2`（TCP），因为大部分容器平台对 QUIC（UDP）支持不稳定，容易每 2-3 分钟断连。如果确认你的环境 QUIC 稳定，可移除 `--protocol http2` 参数恢复 QUIC 以获得更低延迟。
- **端口说明**：`index.js` 中 Node.js 主服务监听端口（`PORT`，默认 3000）是 cloudflared 回源的目标端口；`config.json` 中 sing-box 监听的 `10086` 是内部端口，Node.js 会将 WebSocket 流量转发到这个端口。两个端口不要混淆。
