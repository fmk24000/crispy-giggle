# Synology NAS 部署 Cloudflare Tunnel 完整指南

這份文檔記錄了如何在 Synology NAS (使用 Container Manager) 上透過 Docker 部署 Cloudflare Tunnel (cloudflared)，實現無需開 Port (Port Forwarding) 即可安全從外網存取 NAS 內的各種服務 (DSM, Synology Drive, Nextcloud 等)。

## 📋 前置準備

1. **Synology NAS** (支援 Docker / Container Manager 的型號)
2. **Cloudflare 帳號** (已綁定你擁有的 Domain)
3. **Container Manager** (於套件中心安裝)

---

## 🚀 第一步：獲取 Tunnel Token

1. 登入 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)。
2. 進入 **Networks** > **Tunnels**。
3. 點擊 **Create a tunnel**。
4. 選擇 **Cloudflared** 作為 Connector。
5. 輸入 Tunnel 名稱 (例如 `nas-tunnel`)，點擊 **Save tunnel**。
6. 在 "Install and run a connector" 頁面，選擇 **Docker**。
7. 在下方的指令框中，複製 `--token` 後面那串長長的字元 (由 `ey` 開頭的字串)。
> **注意：** 我們只需要那個 **Token**，不需要複製整行 `docker run` 指令。



---

## 📂 第二步：NAS 資料夾設定

為了方便管理，建議在 NAS 建立專用資料夾 (雖然 Cloudflared 其實不太需要掛載 Volume，但保持良好習慣)：

1. 開啟 **File Station**。
2. 在 `/docker` 資料夾下，建立一個新資料夾命名為 `cloudflared`。
* 路徑範例：`/volume1/docker/cloudflared`



---

## 🐳 第三步：Docker Compose 設定 (核心代碼)

開啟 Synology **Container Manager**，建立一個新專案 (Project)：

* **專案名稱**：`cloudflare-tunnel`
* **路徑**：選擇剛才建立的 `/volume1/docker/cloudflared`
* **來源**：選擇「建立 docker-compose.yml」

將以下代碼貼入編輯器：

```yaml
version: '3.9'

services:
  cloudflared:
    container_name: cloudflared-tunnel
    image: cloudflare/cloudflared:latest
    restart: always
    # 這是 Cloudflare Tunnel 的標準啟動指令
    command: tunnel run
    environment:
      # 👇 請將下面這行換成你在第一步複製的 Token
      - TUNNEL_TOKEN=eyJhIjoiM... (貼上你既Token) ...
    
    # Cloudflared 極度輕量，通常不需要特別限制資源，但為了安全可加上
    deploy:
      resources:
        limits:
          memory: 256M

```

點擊 **「下一步」** 直至完成並啟動專案。
當看到容器狀態顯示為 **「綠色 (Running)」**，代表 Tunnel 已成功打通。

---

## 🔗 第四步：設定 Public Hostname (服務對接)

回到 **Cloudflare Zero Trust Dashboard** 剛才的 Tunnel 頁面，點擊 **Next** 進入 **Public Hostname** 設定。

在這裡，你可以將網域 (Domain) 對接到 NAS 內網的特定 Port。

### 常用服務對接範例：

| 服務名稱 | Public Hostname (網域) | Service Type | URL (內網 IP:Port) | 備註 |
| --- | --- | --- | --- | --- |
| **DSM 管理介面** | `nas.yourdomain.com` | HTTP | `192.168.1.58:5000` | 建議配合 WAF 加強防護 |
| **Synology Drive** | `drive.yourdomain.com` | HTTP | `192.168.1.58:10002` | *需在「登入入口」設定專用 Port* |
| **File Browser** | `files.yourdomain.com` | HTTP | `192.168.1.58:10101` | Docker 容器服務 |
| **Nextcloud** | `office.yourdomain.com` | HTTP | `192.168.1.58:11000` | Docker 容器服務 |

> **提示：** 雖然 Docker 容器之間可以用 `hostname` 通訊，但在 Cloudflare Tunnel 設定中，直接填寫 NAS 的 **內網 IP (192.168.x.x)** 通常是最穩定且不容易出錯的方法。

---

## 🛠️ 進階技巧：Synology Drive 專用入口

為了讓同事直接進入 Drive 而不經過 DSM 登入畫面：

1. 在 DSM **控制台** > **登入入口 (Login Portal)** > **應用程式**。
2. 編輯 **Synology Drive**。
3. 在 **自訂連接埠 (HTTP)** 輸入 `10002` (或自選號碼)。
4. 在 Cloudflare Tunnel 將 `drive.yourdomain.com` 指向 `HTTP://192.168.xx.xx:10002`。

---

## ⚠️ 注意事項

1. **安全性**：雖然 Tunnel 隱藏了真實 IP，但建議在 Cloudflare Zero Trust 的 **Access** 頁面設定額外驗證 (如 Email OTP)，特別是對於 SSH 或 DSM 管理介面。
2. **HTTPS**：Cloudflare 會自動為你的 Public Hostname 提供 SSL 證書 (HTTPS)。在 NAS 端或 Docker 端通常只需要設定 **HTTP** 即可，Cloudflare 會處理加密連線。
3. **上傳限制**：Cloudflare 免費版預設有 100MB 上傳限制。如果 Synology Drive 上傳大檔失敗，這通常是原因。可透過 Cloudflare WAF 規則嘗試放寬，或在 Client 端切片上傳。