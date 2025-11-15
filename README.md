# DorisWeb
for my wife Doris web surfing

---

## 个人专属网络环境搭建指南 (Mac + Cloudflare 方案)

本文档为 Doris 提供一个详细的、安全的、稳定的个人专属网络环境搭建方案。

### 方案优势与问题解答

1.  **伪装性如何？**
    本方案采用 `VLESS + WebSocket + TLS` 核心，并通过 Cloudflare 全球网络进行流量中转和加密。在外部看来，所有的网络活动都表现为对一个普通安全网站（你的个人域名）的访问，**伪装性极强，与常规的HTTPS流量没有区别**，能有效避免被检测和干扰。

2.  **域名会被封吗？**
    对于个人自用、流量不大的情况，**域名被封锁的风险极低**。因为流量特征与正常网页访问无异，且真实服务器IP被完美隐藏在Cloudflare之后。大规模的封锁策略通常不会针对这类个人目标。

3.  **这是什么方案？**
    这是一种服务器部署策略，我们将你位于国外的Mac电脑作为7x24小时运行的服务器。在其上运行 `Xray` 核心程序（提供VLESS+WS服务），并使用 `Cloudflare Tunnel` 技术，将服务安全地发布到互联网上，无需公网IP，也无需配置路由器。

### 一、 核心诉求 (Objective)

为Doris创建一个长期稳定、安全私密、低成本的个人上网环境，利用已有的国外Mac电脑作为服务器，避免额外的VPS服务器开销。

### 二、 整体方案 (Solution)

**架构图:**
`用户设备 (Client) -> Cloudflare 网络 -> Cloudflare Tunnel -> 国外Mac电脑 (Server) -> 互联网`

**核心组件:**
*   **服务器**: 一台位于国外的Mac电脑，要求7x24小时开机并联网。
*   **域名**: 一个你自己的域名，用于对接Cloudflare并启用TLS加密。
*   **Cloudflare账号**: 使用免费套餐即可。
*   **核心软件**:
    1.  `Xray`: 在Mac上运行，作为VLESS+WS协议的服务端。
    2.  `Cloudflared`: 在Mac上运行，创建与Cloudflare之间的反向隧道。

---

### 三、 服务器端 (Mac) 配置流程

#### 1. 安装必要软件 (通过 Homebrew)

首先，确保你的Mac上安装了 `Homebrew`。如果未安装，请在终端执行以下命令安装：
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

然后，通过 Homebrew 安装 `xray` 和 `cloudflared`：
```bash
brew install xray
brew install cloudflared
```

#### 2. 配置并启动 Xray 服务

1.  **确定并创建Xray配置目录**:
    根据你Mac芯片架构的不同，Homebrew的配置路径也不同：
    *   **Apple Silicon (M1/M2/M3等芯片)**: 路径为 `/opt/homebrew/etc/xray/`
    *   **Intel 芯片**: 路径为 `/usr/local/etc/xray/`

    请根据你的设备，执行对应的命令创建目录（如果目录已存在，则无需操作）：
    ```bash
    # Apple Silicon Mac 用户执行:
    mkdir -p /opt/homebrew/etc/xray

    # Intel Mac 用户执行:
    mkdir -p /usr/local/etc/xray
    ```

2.  **创建Xray配置文件**:
    在上一步创建的目录中，新建一个 `config.json` 文件。例如，对于Apple Silicon Mac，文件路径为 `/opt/homebrew/etc/xray/config.json`。

    将以下内容写入 `config.json` 文件：
    ```json
    {
      "inbounds": [
        {
          "port": 10000,
          "listen": "127.0.0.1",
          "protocol": "vless",
          "settings": {
            "clients": [
              {
                "id": "在此处替换为你自己的UUID"
              }
            ],
            "decryption": "none"
          },
          "streamSettings": {
            "network": "ws",
            "wsSettings": {
              "path": "/your_secret_path"
            }
          }
        }
      ],
      "outbounds": [
        {
          "protocol": "freedom"
        }
      ]
    }
    ```

3.  **修改关键参数**:
    *   `id`: 你需要一个UUID作为连接密码。**推荐**在Mac终端中通过以下命令本地生成，它更安全、更快捷：
        ```bash
        uuidgen
        ```
        执行后，将返回的一长串字符（例如 `F8A8B8E0-5E3C-4F9B-8F1A-1B8E9D6C2A0B`）复制并替换掉配置文件中的 `"在此处替换为你自己的UUID"`。
        或者，你也可以访问 [UUID在线生成网站](https://www.uuidgenerator.net/) 来生成。
    *   `path`: 将 `"/your_secret_path"` 修改为一个自定义的、不易猜到的路径，例如 `"/doris-1234"`。**记住这个路径，客户端配置时需要**。

4.  **启动Xray服务**:
    使用 `brew services` 将Xray作为系统服务在后台运行，这样开机后会自动启动。
    ```bash
    brew services start xray
    ```
    你可以使用 `brew services list` 命令查看服务是否已成功启动。

#### 3. 配置并启动 Cloudflare Tunnel

1.  **登录Cloudflare**:
    在终端执行，这会打开浏览器让你登录并授权。
    ```bash
    cloudflared tunnel login
    ```

2.  **创建隧道**:
    给你的隧道起一个名字，例如 `doris-tunnel`。
    ```bash
    cloudflared tunnel create doris-tunnel
    ```
    执行后会返回一个隧道ID，并在 `~/.cloudflared/` 目录下生成一个 `隧道ID.json` 的凭证文件。

3.  **创建隧道配置文件**:
    在 `~/.cloudflared/` 目录下创建一个名为 `config.yml` 的文件。
    ```bash
    touch ~/.cloudflared/config.yml
    ```
    并将以下内容写入该文件：
    ```yaml
    tunnel: doris-tunnel
    credentials-file: /Users/你的Mac用户名/.cloudflared/你的隧道ID.json

    ingress:
      - hostname: sub.yourdomain.com
        service: http://localhost:10000
      - service: http_status:404
    ```

4.  **修改关键参数**:
    *   `credentials-file`: 将 `你的Mac用户名` 和 `你的隧道ID.json` 替换为你的实际信息。
    *   `hostname`: 将 `sub.yourdomain.com` 替换为你准备使用的域名（或子域名）。

5.  **关联域名与隧道**:
    执行以下命令，Cloudflare会自动帮你创建DNS记录。
    ```bash
    cloudflared tunnel route dns doris-tunnel sub.yourdomain.com
    ```
    将 `doris-tunnel` 和 `sub.yourdomain.com` 替换为你的隧道名称和域名。

6.  **将Tunnel作为服务运行**:
    为了让隧道也能开机自启，我们需要为它创建一个系统服务。
    *   确定 `cloudflared` 可执行文件路径。Apple Silicon Mac 通常在 `/opt/homebrew/bin/cloudflared`，Intel Mac 在 `/usr/local/bin/cloudflared`。
    *   创建 `launchd` 配置文件：
        ```bash
        touch ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
        ```
    *   将以下内容写入该文件，并确保修改了所有尖括号内的占位符：
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
        <plist version="1.0">
          <dict>
            <key>Label</key>
            <string>com.cloudflare.cloudflared</string>
            <key>ProgramArguments</key>
            <array>
              <string>/opt/homebrew/bin/cloudflared</string> <!-- Intel Mac 请改为 /usr/local/bin/cloudflared -->
              <string>tunnel</string>
              <string>--config</string>
              <string>/Users/<你的Mac用户名>/.cloudflared/config.yml</string>
              <string>run</string>
            </array>
            <key>RunAtLoad</key>
            <true/>
            <key>KeepAlive</key>
            <true/>
            <key>StandardOutPath</key>
            <string>/Users/<你的Mac用户名>/.cloudflared/cloudflared.log</string>
            <key>StandardErrorPath</key>
            <string>/Users/<你的Mac用户名>/.cloudflared/cloudflared.log</string>
          </dict>
        </plist>
        ```
    *   加载并启动服务：
        ```bash
        launchctl load ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
        launchctl start com.cloudflare.cloudflared
        ```

至此，服务器端（Mac）已全部配置完成。

---

### 四、 用户端 (Client) 配置流程

在需要上网的设备上（手机、电脑）下载对应的客户端软件。

*   **Windows**: `v2rayN`
*   **macOS**: `V2RayU`
*   **Android**: `V2RayNG`
*   **iOS**: `Shadowrocket` (小火箭, 付费) 或 `FoXray` (免费)

打开客户端，手动添加一个新的服务器配置，填入以下信息：

*   **别名/备注 (Alias)**: `Doris Home` (或任意你喜欢的名字)
*   **地址 (Address)**: `sub.yourdomain.com` (你在`config.yml`中设置的域名)
*   **端口 (Port)**: `443`
*   **用户ID (UUID)**: 你在Mac上`config.json`里设置的那个UUID
*   **传输协议 (Network)**: `ws` (或 `websocket`)
*   **伪装路径 (Path)**: 你在Mac上`config.json`里设置的路径 (例如 `/your_secret_path`)
*   **底层传输安全 (TLS/Security)**: 选择 `tls`

保存配置后，连接即可。

---

### 五、 补充说明：与现有Cloudflare Tunnel共存

如果你的Mac上已经为了其他服务（例如暴露本地开发环境）而运行了一个`cloudflared`隧道，你完全可以再为Doris的服务创建并运行一个新的隧道，它们之间不会产生冲突。

Cloudflare Tunnel通过**隧道名称/ID**和**配置文件**来区分不同的隧道，并通过**DNS记录**来区分不同域名的流量走向。

**推荐方案：使用单个`config.yml`管理多个服务**

这是最简洁高效的管理方式。你不需要创建新的隧道，而是可以在你现有的隧道配置文件中，增加一条指向Xray服务的`ingress`规则。

假设你已经有一个隧道`dev-tunnel`，用于暴露你的开发服务`dev.yourdomain.com`。你可以这样修改你的`config.yml`文件：

```yaml
# 你的隧道配置文件，例如 ~/.cloudflared/config.yml

tunnel: dev-tunnel
credentials-file: /Users/你的Mac用户名/.cloudflared/你的开发隧道ID.json

ingress:
  # 规则一：已有的开发服务
  - hostname: dev.yourdomain.com
    service: http://localhost:3000  # 假设你的开发服务运行在3000端口

  # 规则二：新增的Doris上网服务
  - hostname: sub.yourdomain.com   # 分配给Doris的域名
    service: http://localhost:10000 # 指向我们配置的Xray服务端口

  # 规则三：所有其他未匹配的请求都返回404
  - service: http_status:404
```

**操作步骤：**

1.  **修改配置**：按上述格式修改你现有的`config.yml`文件，加入新的`hostname`和`service`。
2.  **设置DNS**：在Cloudflare的DNS管理页面，确保为新的域名 `sub.yourdomain.com` 添加一条`CNAME`记录，指向你的隧道地址（例如 `你的开发隧道ID.cfargotunnel.com`）。如果你不确定，可以执行 `cloudflared tunnel route dns dev-tunnel sub.yourdomain.com` 来自动创建。
3.  **重启服务**：如果你之前已经将`cloudflared`设置为系统服务，需要重启一下来加载新的配置：
    ```bash
    launchctl stop com.cloudflare.cloudflared
    launchctl start com.cloudflare.cloudflared
    ```

通过这种方式，同一个`cloudflared`进程可以同时处理多个域名的流量，并将它们分别转发到你本地Mac上运行的不同服务，既高效又易于管理。
