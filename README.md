# Windows 指定程式使用 VPN Gate

使用 Docker VPN Gate Proxy 與 Clash Verge Rev，讓指定程式使用 VPN Gate，其他程式直連。以下以 Chrome、ChatGPT、Codex 為例。

## 1. 安裝所需軟體

安裝並開啟：

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)：使用 Linux 容器。
- [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev/releases)：本文使用 Rev 版本。

初次設定先關閉 Clash 的 TUN 模式。

## 2. 啟動 Docker VPN

建立一個資料夾，在裡面新增 `docker-compose.yml`，貼上以下完整內容。檔案副檔名必須是 `.yml`，不是 `.yml.txt`。

```yaml
services:
  vpn-proxy:
    image: ghcr.io/xiaowen-king/vpngate-proxy:latest
    container_name: vpn-proxy
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:1080:1080"
    dns:
      - 1.1.1.1
    volumes:
      - ./data:/data
    restart: unless-stopped
```

在這個資料夾開啟 PowerShell，執行：

```powershell
docker compose up -d
```

首次啟動需要等待 Docker 下載映像。

## 3. 設定 VPN 管理面板

### 3.1 打開面板並登入

在瀏覽器網址列輸入：

```text
http://127.0.0.1:8080
```

首次登入的預設密碼是：

```text
admin
```

### 3.2 填入 VPN Gate API

進入面板的「設定／设置」頁面，依照下表填寫：

| 設定項目 | 填寫內容 |
|---|---|
| API 地址 | `https://www.vpngate.net/api/iphone/` |
| 面板密碼 | 將預設 `admin` 改成自己的密碼 |
| VPN 使用者名稱 | vpn |
| VPN 密碼 | vpn |
| 預設連接地區 | 先保留 `all`，也可選需要的地區 |
| SOCKS 端口 | 保留 `1080` |
| 面板端口 | 保留 `8080` |

其他設定先保留預設值，按「儲存／保存」。

### 3.3 等待節點連線

保存後，系統會取得節點並自動嘗試連線。

返回儀表板，確認顯示已連線，且日誌出現健康檢查成功，例如：

```text
健康检测成功 ... HTTP 200
```

想手動選節點時，到節點列表選擇節點並執行連線；介面文字可能隨版本不同。第一次先確定 Docker 能正常連線，再進行 Clash 設定。

如果面板沒有節點，先檢查 API 地址是否可訪問，不要直接開啟 TUN 反覆嘗試。

## 4. 查詢電腦的網卡名稱

在 PowerShell 執行：

```powershell
Get-NetAdapter | Format-Table Name, InterfaceDescription, Status -AutoSize
```

記下目前真正上網的實體網卡在 `Name` 欄顯示的名稱：

- 網路線：可能是 `Ethernet`。
- Wi-Fi：可能是 `Wi-Fi`。

以你的實際名稱為準，不要選 `Mihomo`、`vEthernet` 或其他 VPN 虛擬網卡。

## 5. 建立 Clash 配置

新增檔案 `mihomo.yaml`，貼上以下完整內容。

**先修改 `interface-name: Ethernet`：** 把 `Ethernet` 換成上一步查到的實體網卡名稱。例如 `interface-name: "Wi-Fi"`。

```yaml
mixed-port: 7897
allow-lan: false
bind-address: 127.0.0.1
mode: rule
log-level: info
ipv6: false
find-process-mode: always

# 改成你目前上網的實體網卡名稱
interface-name: Ethernet

tun:
  enable: true
  stack: mixed
  auto-route: true
  auto-detect-interface: false
  strict-route: false
  dns-hijack:
    - any:53
    - tcp://any:53

dns:
  enable: true
  listen: 127.0.0.1:1053
  ipv6: false
  enhanced-mode: redir-host
  nameserver:
    - "https://1.1.1.1/dns-query#DIRECT"

proxies:
  - name: Docker-VPN
    type: socks5
    server: 127.0.0.1
    port: 1080
    udp: false

proxy-groups:
  - name: VPN-Apps
    type: select
    proxies:
      - Docker-VPN

rules:
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR6,::1/128,DIRECT,no-resolve
  - AND,((PROCESS-NAME,chrome.exe),(NETWORK,UDP)),REJECT
  - AND,((PROCESS-NAME,chatgpt.exe),(NETWORK,UDP)),REJECT
  - AND,((PROCESS-NAME,codex.exe),(NETWORK,UDP)),REJECT
  - PROCESS-NAME,chrome.exe,VPN-Apps
  - PROCESS-NAME,chatgpt.exe,VPN-Apps
  - PROCESS-NAME,codex.exe,VPN-Apps
  - MATCH,DIRECT
```

保持 `enhanced-mode: redir-host`，不要改成 `fake-ip`。本次成功配置使用真實 IP DNS，避免 Docker 拿到 `198.18.x.x` 後把虛擬目的地送進自己的 VPN。

## 6. 在 Clash Verge Rev 匯入並啟用

### 6.1 匯入本機配置

1. 打開 Clash Verge Rev。
2. 進入左側的「訂閱／配置（Profiles）」頁面。
3. 把剛才的 `mihomo.yaml` 拖進該頁面，匯入為本機配置。
4. 點選新匯入的配置卡片，設為目前使用的配置。

如果你的版本沒有拖拽匯入，使用「新建」，類型選「本地／Local」，再選擇 `mihomo.yaml` 匯入。

這是完整本機配置，不需要購買或輸入代理訂閱地址。

### 6.2 啟用服務模式

1. 到「設定 → 服務模式」。
2. 依提示安裝／啟用服務，允許 Windows 管理員授權。
3. 確認服務模式正在運行。

若無法使用服務模式，可以先完全退出 Clash，再以管理員身分啟動，作為替代方式。

### 6.3 選擇規則模式並開啟 TUN

1. 將路由模式設為「規則／Rule」，不要選「全域／Global」或「直連／Direct」。
2. 初次測試可先關閉「系統代理」，減少額外變數。
3. 開啟「TUN／虛擬網卡模式」。
4. 在代理頁面確認 `VPN-Apps` 選擇的是 `Docker-VPN`。

如 GUI 的 TUN 設定覆寫了檔案，請確認實際生效值為：

| 設定 | 值 |
|---|---|
| 協定堆疊／Stack | `mixed` |
| 自動路由／Auto Route | 開啟 |
| 自動偵測出口介面 | 關閉 |
| 嚴格路由／Strict Route | 關閉 |
| DNS 模式 | `redir-host` |

不同版本的入口名稱可能不同；重點是套用的配置與上述值一致。

### 6.4 重新啟動應用和容器

完全退出 Chrome、ChatGPT、Codex，然後在 PowerShell 執行：

```powershell
Clear-DnsClientCache
docker restart vpn-proxy
```

等面板重新顯示連線成功及健康檢查正常，再開啟指定應用。

## 7. 確認設定完成

在 Clash 的連線／日誌頁面，查看指定應用是否命中：

```text
chrome.exe → VPN-Apps
```

未列入規則的程式應使用 `DIRECT`。

這份範例讓指定程式的 TCP 走 VPN；其 UDP 被阻擋。DNS 上游是直連，不是所有 DNS 都經 VPN的配置。

## 8. 改成自己的程式

在工作管理員「詳細資料」找到實際執行檔名稱，例如 `myapp.exe`。

在規則中加入下列兩行，且必須位於最後的 `MATCH,DIRECT` 之前：

```yaml
- AND,((PROCESS-NAME,myapp.exe),(NETWORK,UDP)),REJECT
- PROCESS-NAME,myapp.exe,VPN-Apps
```

不需要的範例程式，可以刪除它對應的 UDP 拒絕規則與代理規則。

修改的是 Clash 已匯入的配置時，保存並重新套用；如果修改的是原始 `mihomo.yaml`，需要重新匯入，因為本機匯入會複製一份檔案。

應用有其他子程序時，要查看日誌，按實際程序名稱補上規則。

---

Docker 服務使用 [xiaowen-king/vpngate-proxy](https://github.com/xiaowen-king/vpngate-proxy)。Clash 操作可參考 [本機配置匯入](https://www.clashverge.dev/guide/profile.html) 及 [Windows 服務模式說明](https://www.clashverge.dev/faq/windows.html)。
