# LoRaWAN 遠端 Gateway MQTT 穿透方案 (Cloudflare Tunnel)

**建立日期:** 2026-02-12
**架構:** RAK Gateway (TCP) -> NAS (Docker) -> Cloudflare Tunnel -> Windows Proxy -> RAK Server

## 1. 背景與目標

由於遠端站點 (Remote Site) **沒有固定 IP (Fixed IP)**，且 RAK Gateway 的 Packet Forwarder **不支援 WebSocket (WSS)** 協議，僅支援傳統 MQTT (TCP)。因此，我們利用 Cloudflare Tunnel 的 **Arbitrary TCP** 功能，配合 Synology NAS 作為中間轉發 (Proxy)，實現內網穿透。

### 架構圖解

`[子機 RAK Gateway]` --(TCP 1883)--> `[子機 Synology NAS]` --(Cloudflare Tunnel)--> `[Internet]` --(Cloudflare Tunnel)--> `[母機 Windows PC (.100)]` --(TCP 1883)--> `[母機 RAK Server (.103)]`

---

## 2. 母機端設定 (Server Side - Office)

**角色：** 負責將來自 Cloudflare 的流量轉發給 MQTT Broker。
**設備：** Windows PC (IP: `192.168.168.100`)
**目標 Server：** RAK Gateway / Mosquitto (IP: `192.168.168.103`)

### 2.1 Cloudflare Tunnel 設定

1. 進入 **Cloudflare Zero Trust Dashboard** -> **Networks** -> **Tunnels**。
2. 選擇母機運行的 Tunnel -> **Configure** -> **Public Hostname**。
3. 新增一條 Hostname：
* **Subdomain:** `mqtt-tcp` (完整域名範例: `mqtt-tcp.carrierhkai.com`)
* **Path:** (留空)
* **Service Type:** `tcp`
* **URL:** `192.168.168.103:1883`


> **注意：** 必須指向 MQTT Broker 的 IP (.103)，而非 Tunnel 本機 (.100)。



### 2.2 Cloudflare Access Policy (關鍵！)

由於機器對機器 (M2M) 通訊無法處理 Cloudflare 的登入驗證頁面，必須設定 **Bypass**。

1. 進入 **Access** -> **Applications**。
2. **Add an application** -> **Self-hosted**。
3. **Application Name:** `MQTT TCP Bypass`
4. **Application Domain:** `mqtt-tcp.carrierhkai.com` (必須與 2.1 設定一致)。
5. **Policy 設定:**
* **Policy Name:** `Allow All`
* **Action:** `Bypass` (繞過驗證)
* **Assign To:** Selector 選擇 `Everyone`。


6. **Save**。

### 2.3 母機 Mosquitto 設定 (.103)

確保 Mosquitto 允許外部連接，而非只監聽 localhost。

* 檢查設定檔 (`/etc/mosquitto/mosquitto.conf`)：
```conf
listener 1883 0.0.0.0
allow_anonymous true  # 視乎是否需要密碼

```



---

## 3. 子機端設定 (Client Side - Remote Site)

**角色：** 負責接收 Gateway 的 TCP 流量，並封裝進 Cloudflare Tunnel。
**設備：** Synology NAS (IP: `192.168.68.58`)
**來源：** RAK Gateway (IP: `192.168.68.xx`)

### 3.1 啟動 Docker 中轉 (Listener)

利用 Synology NAS 的 Docker 運行 `cloudflared` 來監聽 Port 1883。

1. 開啟 SSH 連接至 NAS。
2. 執行以下指令 (需 Root 權限)：
```bash
sudo docker run -d \
  --name cloudflared-mqtt-listener \
  --restart always \
  --net=host \
  cloudflare/cloudflared:latest \
  access tcp --hostname mqtt-tcp.carrierhkai.com --url 0.0.0.0:1883

```


* `--net=host`: 直接使用 NAS 網絡，避免 Docker Bridge 端口映射問題。
* `--hostname`: 填寫 2.1 設定的完整域名。
* `--url 0.0.0.0:1883`: 在 NAS 本機開放 1883 Port。



### 3.2 設定 NAS 防火牆

必須允許內網設備連接 NAS 的 Port 1883。

1. 進入 **DSM Control Panel** -> **Security** -> **Firewall**。
2. 編輯規則 (Edit Rules)：
* **Ports:** `1883` (TCP)
* **Source IP:** `Specific IP` -> `Subnet` (填寫子機網段，如 `192.168.68.0/24`) 或 `All`。
* **Action:** `Allow`


3. 確保規則置頂，並保存。

---

## 4. 最終設備設定 (RAK Gateway)

**設備：** RAK Gateway (Packet Forwarder / Gateway Bridge)

1. 登入 RAK Web UI。
2. 進入 **LoRa Network** -> **Packet Forwarder** / **General Setup**。
3. 設定如下：
* **Protocol:** `MQTT` (或 TCP)
* **Server Address:** `192.168.68.58` (**填寫 NAS 的 IP**，切勿填寫 Domain 或 母機 IP)
* **Server Port:** `1883`
* **Authentication:** 填寫母機 Mosquitto 的帳號密碼。


4. 點擊 **Save & Apply**。

---

## 5. 故障排除 (Troubleshooting)

如果連接失敗，請依序檢查：

1. **檢查 NAS Docker 狀態:**
```bash
sudo docker logs -f cloudflared-mqtt-listener

```


* 正常：顯示 `Start Websocket listener` 及 `Websocket request: GET /`。
* 異常：無反應 (Gateway 未連上 NAS) 或 `failed to connect` (Cloudflare 阻擋)。


2. **測試 NAS 端口 (由同一網絡電腦):**
```powershell
Test-NetConnection 192.168.68.58 -Port 1883

```


* 必須顯示 `True`。若是 `False`，檢查 NAS 防火牆。


3. **檢查 Cloudflare Policy:**
* 確保 `mqtt-tcp` 的 Policy Action 是 **Bypass**，否則 NAS 無法通過驗證。


4. **母機防火牆:**
* Windows PC (.100) 不需要開 Inbound Port。
* 但需確保 Windows PC 能 Ping 通 RAK Server (.103)。



---

### 6. 成功驗證

* 進入 **ChirpStack** -> **Gateways**。
* 該 Gateway 狀態應顯示為 **"Last seen: a few seconds ago"**。
* 在 **LoRaWAN Frames** 分頁能看到 `Uplink` 數據包。

---

### 建議後續行動：

* 你可以將上述內容 Copy 落一個 `README.md` 檔案。
* 如果有相關嘅圖片（例如你剛才俾我睇嗰啲），可以改名做 `assets/config-screenshot.png` 放埋入去 GitHub Repo，咁同事睇就更加一目了然。