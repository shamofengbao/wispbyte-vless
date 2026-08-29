# Wispbyte Node.js + Sing-Box VLESS 节点部署教程（Named Tunnel 固定隧道版）

本教程介绍如何在 Wispbyte 面板中部署 VLESS 节点，通过 Cloudflare Named Tunnel 绑定固定域名，告别 Quick Tunnel 每次重启换域名的问题。

## 许可证与商业限制

开源协议：本项目采用 CC BY-NC 4.0（署名-非商业性使用） 协议发布。

商业限制：严禁任何个人或组织将本教程、相关配置文件及代码用于任何形式的商业盈利、收费机场搭建、商业售卖或引流牟利。仅限个人技术研究与交流使用。

## 部署准备工作

1. 登录 Wispbyte 控制面板，进入对应服务器的 **Files（文件管理器）** 页面。
2. 确保容器支持 Node.js 运行环境。
3. 在 Cloudflare Zero Trust 面板创建 Named Tunnel，获取 Token，并绑定公开主机名 `sjgx.19821103.xyz` → `http://localhost:27116`。

## 核心步骤详解

### 第一步：创建服务端配置文件

在文件管理器根目录下新建或编辑名为 `config.json` 的文件，将节点监听配置写入其中。

关键参数说明：

- 协议类型：vless
- 本地监听端口：10086（由后端程序进行转发）
- UUID：`97be4055-fdd1-48d2-997b-b8d246ec5101`
- 传输路径：`/api/v2/telemetry/stream_8f91a`

### 第二步：创建主控与转发脚本

在根目录下新建或编辑名为 `index.js` 的文件。该脚本会自动在线下载适配的 sing-box 内核、自动解压赋予权限，并建立安全的双向 WebSocket 流量转发通道。同时会下载 cloudflared 并使用 Named Tunnel Token 启动固定隧道。

脚本内置配置：
- Cloudflare Named Tunnel Token（已写入脚本，也可通过环境变量 `CF_TUNNEL_TOKEN` 覆盖）
- 隧道固定域名：`sjgx.19821103.xyz`
- Cloudflare 隧道回源地址：`http://localhost:27116`（在 CF Zero Trust 面板的 Tunnel Public Hostname 中配置）

### 第三步：启动与状态检查

所有的配置文件准备就绪后，切换至控制台 **Console** 页面。

点击 **Restart（重启）** 按钮重新加载容器。

检查控制台日志，确认出现以下提示即代表正常运行：

```
[+] 主服务已监听端口 3000
[+] 正在下载 sing-box 内核...
[+] 启动后台 sing-box 进程...
[+] 启动 Cloudflare 固定隧道 (Named Tunnel)...
[+] 隧道域名: sjgx.19821103.xyz
```

## 客户端导入与连接参数

在本地代理客户端（如 v2rayN 等）中新建 VLESS 节点，填入以下参数：

| 参数 | 值 |
|------|------|
| 别名备注 | sjgx-VLESS |
| 服务器地址 | sjgx.19821103.xyz |
| 服务端口 | 443 |
| 用户 ID (UUID) | 97be4055-fdd1-48d2-997b-b8d246ec5101（需与 config.json 保持一致） |
| 传输协议 | ws (WebSocket) |
| 伪装路径 | /api/v2/telemetry/stream_8f91a |
| 传输层安全 (TLS) | tls |
| SNI | sjgx.19821103.xyz |
| 允许不安全 | 关闭 |

### 节点快捷格式

```
vless://97be4055-fdd1-48d2-997b-b8d246ec5101@sjgx.19821103.xyz:443?encryption=none&security=tls&sni=sjgx.19821103.xyz&type=ws&path=%2Fapi%2Fv2%2Ftelemetry%2Fstream_8f91a#sjgx-VLESS
```

## 注意事项

- **Cloudflare 隧道配置**：在 CF Zero Trust → Networks → Tunnels → 你的隧道 → Public Hostnames 中，需要添加一条记录：`sjgx.19821103.xyz` → `http://localhost:27116`。如果端口不是 27116，请在 CF 面板中修改回源地址，或在 `index.js` 中调整 `PORT` 变量。
- **Token 安全**：Token 已硬编码在 `index.js` 中作为默认值。更安全的做法是通过 Wispbyte 面板的环境变量 `CF_TUNNEL_TOKEN` 注入，脚本会优先读取环境变量。
- **端口说明**：`index.js` 中 Node.js 主服务监听端口（`PORT`，默认 3000）是 cloudflared 回源的目标端口；`config.json` 中 sing-box 监听的 `10086` 是内部端口，Node.js 会将 WebSocket 流量转发到这个端口。CF 隧道的回源地址应指向 Node.js 主服务端口。
