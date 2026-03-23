采用 **Rule Providers (规则集外挂)** 是最符合生产环境标准的做法。它可以让你通过订阅现成的规则库（如 GitHub 上每日更新的全球域名列表），或者指向你自己的远程/本地文件，实现“配置一次，自动更新”。

以下是基于 **Windows 11 + Docker + Gluetun + Clash Meta (TUN)** 的全量配置。

---

### 1. 目录结构准备

在 `C:\Network` 下建立如下结构：
```text
C:\Network\
├── docker-compose.yaml
├── config.yaml
├── gluetun/
│   ├── client.conf        <-- 你的 .ovpn 文件改名于此
│   └── auth.txt           <-- 第一行账号，第二行密码
└── ruleset/               <-- 存放本地自定义规则（可选）
    └── my-private.yaml
```

---

### 2. `docker-compose.yaml` (全量)

```yaml
version: '3.8'
services:
  # VPN 隧道：负责拨号 OpenVPN 并转为本地 SOCKS5 代理
  gluetun:
    image: qmcgaw/gluetun
    container_name: vpn-tunnel
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    volumes:
      - ./gluetun:/gluetun
    environment:
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=openvpn
      - OPENVPN_CUSTOM_CONFIG=/gluetun/client.conf
      - OPENVPN_USERFILE=/gluetun/auth.txt
      - PROXY_SOCKS5_LISTEN_ADDRESS=:1080
    ports:
      - "1080:1080" # 映射 SOCKS5 端口供 Clash 调用
    restart: always

  # 分流大脑：负责 TUN 模式透明网关和动态分流
  clash:
    image: metacubex/mihomo:latest
    container_name: clash-gateway
    privileged: true
    network_mode: "host" # 必须使用 host 模式以支持透明网关
    volumes:
      - ./config.yaml:/root/.config/mihomo/config.yaml
      - ./ruleset:/root/.config/mihomo/ruleset # 映射规则文件夹
    depends_on:
      - gluetun
    restart: always
```

---

### 3. `config.yaml` (Rule Providers 增强版)

这份配置引入了 **Loyalsoldier** 维护的规则集（目前社区最流行、更新最快），并展示了如何添加你自己的动态列表。

```yaml
# =========================================================
# 基础设置
# =========================================================
mixed-port: 7890
allow-left: true
mode: rule
log-level: info
ipv6: false

# TUN 模式配置
tun:
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - any:53

# DNS 策略
dns:
  enable: true
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.1/16
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  # 动态 DNS 路由：针对特定的规则集走特定的 DNS
  nameserver-policy:
    "rule-set:proxy_list,google_list": [https://8.8.8.8/dns-query]
    "rule-set:direct_list": [223.5.5.5, 119.29.29.29]

# =========================================================
# 代理节点定义 (指向 Gluetun 容器)
# =========================================================
proxies:
  - name: "VPN-Out"
    type: socks5
    server: 127.0.0.1
    port: 1080

proxy-groups:
  - name: 🚀 跨越国界
    type: select
    proxies:
      - "VPN-Out"
      - DIRECT

# =========================================================
# Rule Providers (动态规则集外挂)
# =========================================================
rule-providers:
  # 1. 自动更新的国外主流网站列表
  proxy_list:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400

  # 2. 自动更新的国内直连列表
  direct_list:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400

  # 3. 自动更新的 Google 服务列表
  google_list:
    type: http
    behavior: domain
    url: "https://raw.githubusercontent.com/Loyalsoldier/clash-rules/release/google.txt"
    path: ./ruleset/google.yaml
    interval: 86400

  # 4. 你的私有动态列表 (例如你放在 GitHub Gist 上的公司域名)
  my_special_list:
    type: http
    behavior: domain
    url: "https://your-gist-link.com/custom.txt" # 替换为你自己的链接
    path: ./ruleset/custom.yaml
    interval: 3600

# =========================================================
# 分流规则
# =========================================================
rules:
  # 1. 优先匹配你的私有列表
  - RULE-SET,my_special_list,🚀 跨越国界
  
  # 2. 匹配 Google/Github 等知名国外服务
  - RULE-SET,google_list,🚀 跨越国界
  - RULE-SET,proxy_list,🚀 跨越国界
  
  # 3. 匹配国内直连列表
  - RULE-SET,direct_list,DIRECT
  
  # 4. 基于 IP 的分流（防止漏网之鱼）
  - GEOIP,CN,DIRECT
  
  # 5. 兜底
  - MATCH,🚀 跨越国界
```

---

### 4. 关键操作步骤

#### A. 启动服务
在 `C:\Network` 目录下打开管理员终端：
```powershell
docker-compose up -d
```

#### B. 开启 Windows 转发 (必须执行)
这是让你的网卡 2 (LAN) 能够把流量送入 Docker 的“桥梁”。
```powershell
Set-NetIPInterface -Forwarding Enabled
```

#### C. 如何动态维护规则？
1.  **方法一 (远程)**：修改 `my_special_list` 对应的 URL 文件内容（比如你放在 Gist 或服务器上的 txt）。Clash 会按照 `interval` 设定的秒数自动下载。
2.  **方法二 (本地)**：你也可以直接修改 `C:\Network\ruleset\custom.yaml`（如果是 `type: file`），Clash 检测到文件变动会立即重载。

---

### 5. 网络专家给你的 1000M 环境调优建议

1.  **WSL2 网络优化**：
    由于 Docker Desktop 运行在 WSL2 之上，在 1000M 环境下建议在 `C:\Users\<你的用户名>\.wslconfig` 中添加以下内容，以获得接近原生的网络性能：
    ```ini
    [wsl2]
    networkingMode=mirrored
    dnsTunneling=true
    ```

2.  **防止 DNS 泄漏**：
    目前的配置使用了 `fake-ip`。这意味着当你的设备请求 `google.com` 时，Clash 会立即返回一个 `198.18.x.x` 的假地址。这种方式在 1000M 宽带下能极大地提升首屏加载速度，因为它跳过了漫长的真实 DNS 查询。

3.  **ASUS ACRH17 配置**：
    *   将物理线从迷你主机的网卡 2 接到华硕路由器的 **LAN 口**（注意不是 WAN 口）。
    *   在华硕后台**关闭 DHCP 服务**。
    *   在华硕后台将 LAN IP 设为 `192.168.10.2`（假设 Win11 LAN 是 `192.168.10.1`）。
    *   这样华硕就变成了一个纯粹的 **WiFi 发射器**，所有流量由 6600H 处理。
    *   高级做法（DHCP 下发）：
      - 如果你不想手动设置每个设备，可以在路由器 DHCP 服务器设置里：
      -  默认网关 填入：192.168.10.1
      -  DNS 服务器 填入：192.168.10.1

4.  **OpenVPN 性能预警**：
    再次提醒，OpenVPN 很难单线程跑满 1000M。如果测速只有 200-400Mbps，那是 OpenVPN 的瓶颈。你可以通过在 `gluetun` 配置文件中尝试开启 `MSSFIX` 来微调 MTU，以获取最佳吞吐量。

现在，你拥有了一个**全自动化、规则自动更新、基于强力 CPU 6600H** 的顶级软路由系统。
