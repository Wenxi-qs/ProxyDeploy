# ProxyDeploy

服务器安装代理的快速项目，使用 Docker Compose 容器化快速部署基于 **Xray + VLESS + REALITY** 的代理服务。

## 架构概述

```
          ┌──────────────────────────────────────────────────┐
          │                    服务器                          │
          │  ┌──────────┐    ┌──────────────┐                │
          │  │  Nginx   │ ──▶ │   Xray       │                │
          │  │  :443    │    │  (VLESS+      │                │
          │  │  stream  │    │   REALITY)    │                │
          │  │  (SNI    │    │              │                │
          │  │  分流)   │    │  20000(internal)              │
          │  └──────────┘    └──────────────┘                │
          │                                                   │
          │                  Docker 网络                        │
          │              192.168.1.0/24                       │
          └──────────────────────────────────────────────────┘
                          ▲
                          │ TLS (REALITY)
                          ▼
          ┌──────────────────────────────────────────────────┐
          │             客户端                                 │
          │  ┌────────────┐                                   │
          │  │ Xray       │                                   │
          │  │ (client)   │ 本机 SOCKS5 :1001                  │
          │  │            │ 或 TUN 虚拟网卡                    │
          │  └────────────┘                                   │
          └──────────────────────────────────────────────────┘
```

---

## 服务端配置

### 1. 环境要求

- Docker：[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
- Docker Compose：[https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
- 服务器开放端口：**80/tcp**、**443/tcp**（用于 REALITY + Nginx 回落）

### 2. 生成 x25519 密钥对

Xray REALITY 需要一套公私钥对，用于 TLS 指纹伪装：

```bash
docker compose run --rm xray x25519
```

输出示例：

```
Private key: YJ4tgqfTMqWkuZtsLhWOpFf86xxxxxxxxxxxxxx   # xray 私钥
Public key: Q_4tHLpuPiggjn-Rrb73ZVZLEYxxxxxxxxxxxxx    # xray 公钥
```

> **注意**：私钥填在**服务端**配置中，公钥填在**客户端**配置中。

### 3. 配置服务端 xray

编辑 [server/xray/config.json](server/xray/config.json)，修改以下字段：

#### 3.1 填入私钥

```json
"privateKey": "上一步生成的 Private key"
```

#### 3.2 填入 shortIds

```json
"shortIds": ["6ba85179e30d4fc2"]
```

> shortId 是 REALITY 的 short ID，可以任意填写（16 进制字符串，最多 16 字符），**必须与服务端和客户端一致**。不填或留空数组 `[]` 则匹配任意 shortId。

#### 3.3 配置用户 ID

```json
"clients": [
  {
    "id": "UUID格式的用户ID"
  }
]
```

> `id` 是 VLESS 协议的用户标识，需符合 UUID 格式，例如 `3a2f1c8e-5b7d-4f6a-9c1e-2d3f4a5b6c7d`。**服务端和客户端必须一致**。

#### 3.4 SNI 回落（伪装域名）

```json
"serverNames": ["www.apple.com"],
"target": "www.apple.com:443"
```

- `serverNames`：客户端发起 TLS 握手时的 SNI，Nginx 通过 `ssl_preread` 按 SNI 分流到 xray
- `target`：REALITY 回落的真实目标网站，推荐大厂 CDN 域名如 `www.apple.com`、`www.microsoft.com`、`aws.amazon.com`

### 4. 配置 Nginx

编辑 [server/nginx/conf/nginx.conf](server/nginx/conf/nginx.conf)：

#### 4.1 SNI 映射

```nginx
map $ssl_preread_server_name $backend_name {
    www.apple.com xray;
}
```

如果修改了 `serverNames`，此处需同步修改。

#### 4.2 xray 上游地址

```nginx
upstream xray {
    server 192.168.1.70:20000;
}
```

确保端口与 [server/xray/config.json](server/xray/config.json) 中 xray 的监听端口一致。

### 5. 启动服务端

```bash
cd server
docker compose up -d
```

验证服务是否运行：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f
```

---

## 客户端配置

### 1. 配置客户端 xray

编辑 [client/xray.json](client/xray.json)，修改以下字段：

#### 1.1 服务端 IP 地址

```json
"address": "你的服务器公网IP"
```

将 `address` 改为服务器的公网 IP 或域名。

#### 1.2 用户 ID（与服务端一致）

```json
"id": "与服务端 clients[0].id 相同的 UUID"
```

#### 1.3 服务端公钥（注意这里是公钥，不是私钥）

```json
"publicKey": "生成密钥对时的 Public key"
```

> 客户端配置的是 `publicKey`（对应服务端的 `privateKey`），不要填反。

#### 1.4 shortId（与服务端一致）

```json
"shortId": "与服务端 shortIds 数组中相同的值"
```

#### 1.5 伪装域名

```json
"serverName": "www.apple.com"
```

> 必须与服务端 `serverNames` 中的域名一致。

#### 1.6 其他可选配置

| 字段 | 说明 |
|------|------|
| `fingerprint` | TLS 指纹，推荐 `chrome` 或 `ios`（模拟真实浏览器 TLS 握手特征） |
| `network` | 传输协议，当前使用 `xhttp`（XHTTP） |
| `path` | 请求路径，与服务端 xhttpSettings.path 一致 |
| `port` | 服务器端口，与服务端 Nginx 监听端口一致（默认 443） |

### 2. 使用 v2rayN 导入配置

适用于 Windows 图形化客户端。

#### 2.1 下载 v2rayN

前往 [v2rayN  releases](https://github.com/2dust/v2rayN/releases) 下载最新版本。

#### 2.2 导入客户端配置

1. 按上方说明修改好 [client/xray.json](client/xray.json) 中的参数（服务器 IP、UUID、publicKey、shortId）
2. 打开 v2rayN，点击 **服务器 → 添加自定义配置服务器（Socks）**
3. 选择修改好的 `client/xray.json` 文件导入
4. 在服务器列表中右键导入的节点，点击 **设为活动服务器**

#### 2.3 启用代理

- 在 v2rayN 系统托盘中右键图标
- 选择 **系统代理 → 全局模式**（或 PAC 模式）
- 或手动设置浏览器 SOCKS5 代理到 `127.0.0.1:1001`

---

## 流量转发流程

```
客户端 Xray ──TLS(SNI: www.apple.com)──▶ Nginx :443
                                                │
                                         ssl_preread 检测 SNI
                                                │
                                         匹配 → xray upstream
                                                │
                                         回落不匹配 → 拒绝连接
                                                │
                                         Xray 解 VLESS 协议
                                                │
                                         转发到目标网站
```

## 数据流简图

```
浏览器/SOCKS客户端
      │ socks5 :1001
      ▼
Xray 客户端 ─── REALITY/TLS ───▶ Nginx (SNI 分流) ──▶ Xray 服务端 ──▶ 目标网站
                                   │ 443/tcp                20000
```

## 注意事项

1. **UUID 格式**：`id` 字段必须是标准的 UUID 格式，可以使用 `uuidgen` 或在线工具生成
2. **密钥对应关系**：服务端配置 `privateKey`，客户端配置对应的 `publicKey`，不要混淆
3. **shortId 一致性**：服务端用数组 `shortIds`，客户端用单个 `shortId`，值必须匹配
4. **防火墙**：确认服务器防火墙已放行 80/tcp 和 443/tcp 端口
5. **重启服务**：修改配置后需要重启容器 `docker compose restart`
