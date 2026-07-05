# 出院後照表操作 Runbook

> 搭配 [opc-workflow-plan.md](./opc-workflow-plan.md)(定位與整體規劃)使用,本文件是**動手執行的順序表**。
> 依「站」為單位進行,每一站結尾有驗收清單,全部打勾再進下一站。
> 指令中 `<...>` 為佔位符,請換成自己的值。

---

## 事前準備(可在住院期間先備妥)

- [ ] 確認記得/找回以下憑證,建議裝一套密碼管理器(Bitwarden 免費或 1Password):
  - 各機器的登入帳號密碼
  - Tailscale 帳號
  - GitHub 帳號
  - NAS 管理員帳號
  - Apple ID
- [ ] Tailscale 管理頁(https://login.tailscale.com/admin)確認:
  - [ ] MagicDNS 已開啟
  - [ ] 所有機器都在線上、名稱清楚(hero4090 / local4090 / macbook / imac / nas)
  - [ ] 各機器金鑰設為不過期(Machines → 該機器 → Disable key expiry),避免哪天突然斷線

---

## 第 0 站:MacBook 駕駛艙(出院當天,約 1 小時)

之後所有站都是坐在 MacBook 前遠端完成,所以先武裝這台。

### 0.1 基礎工具

```bash
# Homebrew(若尚未安裝)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 基本工具
brew install git gh lazydocker
brew install --cask docker   # Docker Desktop(本機偶爾需要,主要用 context 連遠端)
```

### 0.2 Claude Code

```bash
npm install -g @anthropic-ai/claude-code   # 或 brew install claude-code
claude   # 首次執行會引導登入
```

### 0.3 Clone 中樞 repo

```bash
mkdir -p ~/dev && cd ~/dev
gh auth login          # 登入 GitHub
gh repo clone clickour/PIHub
cd PIHub && claude     # 之後任何基礎設施調整,直接在這裡對 Claude 下指令
```

### 0.4 SSH 金鑰(一次設定,終身免密碼)

```bash
ssh-keygen -t ed25519 -C "macbook"        # 一路 Enter 即可
ssh-copy-id <user>@hero4090               # 輸入一次 hero4090 密碼
ssh hero4090                              # 測試:應直接登入不問密碼
```

### 0.5 遠端 Docker context

```bash
docker context create hero4090 --docker "host=ssh://<user>@hero4090"
docker context use hero4090
docker ps    # 此後 MacBook 上的 docker 指令都作用在 hero4090
```

### ✅ 第 0 站驗收

- [ ] `claude` 可啟動並登入
- [ ] `ssh hero4090` 免密碼直接登入
- [ ] `docker ps` 顯示的是 hero4090 上的容器(就算是空的也算通過)

---

## 第 1 站:hero4090 AI 服務中心(半天,全程從 MacBook 遠端)

### 1.1 系統基礎

```bash
ssh hero4090

# 更新系統
sudo apt update && sudo apt upgrade -y

# 確認 NVIDIA 驅動正常
nvidia-smi    # 應顯示 RTX 4090

# Tailscale SSH + 開機自啟
sudo tailscale set --ssh
sudo systemctl enable --now tailscaled

# 關閉睡眠(伺服器不准睡)
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### 1.2 Docker + GPU 支援

```bash
# Docker(若已裝可跳過)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

# NVIDIA Container Toolkit(讓容器吃到 GPU)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 驗證 GPU 進得了容器
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

### 1.3 部署服務堆疊

在 hero4090 上 `mkdir -p ~/stack && cd ~/stack`,建立 `docker-compose.yml`:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    ports: ["11434:11434"]
    volumes: ["ollama:/root/.ollama"]
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: all, capabilities: [gpu]}]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports: ["3000:8080"]
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes: ["open-webui:/app/backend/data"]
    depends_on: [ollama]

  comfyui:
    image: ghcr.io/lecode-official/comfyui-docker:latest
    restart: unless-stopped
    ports: ["8188:8188"]
    volumes: ["comfyui-models:/opt/comfyui/models", "comfyui-output:/opt/comfyui/output"]
    deploy:
      resources:
        reservations:
          devices: [{driver: nvidia, count: all, capabilities: [gpu]}]

  uptime-kuma:
    image: louislam/uptime-kuma:1
    restart: unless-stopped
    ports: ["3001:3001"]
    volumes: ["uptime-kuma:/app/data"]

  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    restart: unless-stopped
    ports: ["7575:3000"]
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=*
    volumes: ["./homepage:/app/config"]

  portainer:
    image: portainer/portainer-ce:latest
    restart: unless-stopped
    ports: ["9443:9443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer:/data

volumes:
  ollama: {}
  open-webui: {}
  comfyui-models: {}
  comfyui-output: {}
  uptime-kuma: {}
  homepage: {}
  portainer: {}
```

```bash
docker compose up -d
docker compose ps          # 全部應為 running

# 拉第一批模型
docker exec -it stack-ollama-1 ollama pull qwen2.5-coder:32b
docker exec -it stack-ollama-1 ollama pull llama3.3:70b-instruct-q4_K_M
```

> 完成後把這個 compose 檔複製一份進 PIHub repo(`infra/hero4090/docker-compose.yml`)並 commit,
> 之後機器重灌或搬遷,一個 `git clone + docker compose up -d` 就還原。

### 1.4 設定服務入口與監控

- [ ] 手機/MacBook 開 `http://hero4090:3000` → Open WebUI 建管理員帳號,能與模型對話
- [ ] 開 `http://hero4090:8188` → ComfyUI 載入正常
- [ ] 開 `http://hero4090:3001` → Uptime Kuma 建帳號,把以下都加入監控:
  - hero4090 各服務(HTTP)、local4090(Ping)、NAS(Ping)、iMac(Ping)
  - 通知管道設定推播(建議加 ntfy 或 Telegram,推到 iPhone)
- [ ] 開 `http://hero4090:7575` → Homepage 把所有服務連結排上去,**iPhone/iPad 加入主畫面書籤**

### ✅ 第 1 站驗收

- [ ] `docker compose ps` 六個服務全部 running
- [ ] 重開機測試:`sudo reboot` 後 5 分鐘內所有服務自動回復
- [ ] iPhone 開 Homepage 書籤能進所有服務
- [ ] 關掉某個服務,Uptime Kuma 有推播到手機

---

## 第 2 站:NAS 資料中樞(約 1-2 小時)

### 2.1 建立共享結構

在 NAS 管理介面建立共享資料夾:

```
/volume1/
├── models/      # AI 模型庫(全機共用)
├── assets/      # 素材:ComfyUI 輸出、影片素材、遊戲美術
├── projects/    # 大型專案檔(影片工程檔等,源碼仍走 GitHub)
└── backup/      # 各機備份 + Time Machine
```

- [ ] 開啟 NFS(給 hero4090)與 SMB(給 Mac/Windows)服務
- [ ] Time Machine 目標資料夾設定完成(給兩台 Mac)

### 2.2 各機掛載

hero4090(NFS,開機自動掛載):

```bash
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nas
echo "<nas的tailscale-ip>:/volume1 /mnt/nas nfs defaults,_netdev,nofail 0 0" | sudo tee -a /etc/fstab
sudo mount -a && ls /mnt/nas
```

MacBook / iMac:Finder → 前往 → 連接伺服器 → `smb://nas`,登入後在「登入項目」加入自動掛載。

- [ ] 兩台 Mac 的 Time Machine 指向 NAS,完成第一次完整備份

### 2.3 3-2-1 備份補完

- [ ] hero4090 排程備份 Docker volumes 到 NAS(cron + rsync,或請 Claude Code 寫好放進 repo)
- [ ] 準備一顆外接碟,設定 NAS 的 USB 備份任務(每月手動接上跑一次,平時離線保存)

### ✅ 第 2 站驗收

- [ ] 三台電腦都能讀寫 NAS 共享
- [ ] Time Machine 至少完成一次完整備份
- [ ] hero4090 重開機後 `/mnt/nas` 自動掛載

---

## 第 3 站:local4090 重灌純 Win11 Pro(半天,需在機器前)

> 這是唯一需要人在機器前面的一站,安排在出院後體力恢復、其他站都完成之後即可。

### 3.1 重灌前

- [ ] 確認機器上沒有未備份的個人檔案(有就先丟 NAS `backup/`)
- [ ] 微軟官網做 Win11 安裝 USB;記下主機板 WiFi/LAN 驅動下載位置

### 3.2 重灌後基礎(依序)

- [ ] Windows Update 全部跑完 + NVIDIA 驅動(GeForce 官網版)
- [ ] 裝 Tailscale 並登入,確認 MacBook `ping local4090` 通
- [ ] 開啟遠端桌面(設定 → 系統 → 遠端桌面),用 iPad/MacBook 的 Windows App 測連線
- [ ] BIOS 開啟 Wake-on-LAN + Windows 網卡內容勾選「允許此裝置喚醒電腦」「僅允許 Magic Packet」
- [ ] 電源計畫:睡眠可以,休眠關閉;Windows Update 使用時段設好避免自動重開
- [ ] 掛載 NAS:檔案總管 → 連線網路磁碟機 → `\\nas\assets` 等

### 3.3 工作軟體

- [ ] **Sunshine**(串流主機端)+ iPad 裝 **Moonlight** 測試串流
- [ ] **DaVinci Resolve**(免費版)
- [ ] **Godot 4.x**(先做 2D;之後要 3D 再裝 Unreal)
- [ ] **Steam / itch.io** 客戶端(測試自己的成品用)

### ✅ 第 3 站驗收

- [ ] 機器睡眠後,能從 MacBook 用 WoL 喚醒(`brew install wakeonlan` → `wakeonlan <MAC位址>`)
- [ ] iPad Moonlight 串流可玩、延遲可接受
- [ ] Resolve 開啟專案、NAS 磁碟機可讀寫

---

## 第 4 站:iMac 輔助機(1 小時內)

- [ ] Tailscale 登入、開啟螢幕共享 + 遠端登入(系統設定 → 一般 → 共享)
- [ ] 瀏覽器全螢幕常駐 Uptime Kuma / Homepage,當公司的「狀態牆」
- [ ] Time Machine → NAS
- [ ] 只放行政類軟體(信件、行事曆、文件、視訊會議),不裝開發環境

### ✅ 第 4 站驗收

- [ ] MacBook 可透過「螢幕共享」遠端操作 iMac
- [ ] 狀態牆開機自動顯示

---

## 第 5 站:行動端收尾(30 分鐘)

iPhone + iPad Pro 各自確認:

- [ ] Tailscale App 開啟 VPN On Demand
- [ ] 主畫面書籤:Homepage 儀表板、Open WebUI、ComfyUI、NAS 管理介面
- [ ] iPad 裝:Termius(SSH)、Moonlight(串流)、Windows App(遠端桌面)
- [ ] Uptime Kuma 的推播通知在 iPhone 實測收得到
- [ ] iPad Sidecar 與 MacBook 配對測試(當第二螢幕)

---

## 完工總驗收(全部打勾 = 系統上線)

- [ ] 在 MacBook 用 Claude Code 開發,`docker context` 一鍵管理 hero4090
- [ ] 任何裝置開一個網址(Homepage)可達所有服務
- [ ] 任一台機器斷線,5 分鐘內手機收到通知
- [ ] 所有源碼在 GitHub、所有素材在 NAS、所有服務設定在 PIHub repo
- [ ] 拔掉任何一台執行機的電源再開機,服務自動回復、不需手動介入

---

## 日常操作速查表

| 我想… | 指令 / 動作(在 MacBook) |
|---|---|
| 看家裡服務狀態 | 開 Homepage 書籤,或 `docker ps` |
| 重啟某服務 | `docker compose -f ~/dev/PIHub/infra/hero4090/docker-compose.yml restart <服務名>`(或 Portainer 點一下) |
| 進 hero4090 終端機 | `ssh hero4090` |
| 叫醒 local4090 | `wakeonlan <MAC位址>` |
| 用 local4090 桌面 | Windows App(RDP)或 Moonlight(要 GPU 畫面時) |
| 換一個新模型 | `docker exec -it stack-ollama-1 ollama pull <模型名>` |
| 改基礎設施 | `cd ~/dev/PIHub && claude`,用說的 |

## 常見問題排除

| 症狀 | 先檢查 |
|---|---|
| 某台機器 Tailscale 離線 | 管理頁看最後上線時間;多半是機器睡眠/斷電 → WoL 或請家人重開 |
| `ssh hero4090` 名稱解析失敗 | MagicDNS 是否開啟;改用 `100.x.x.x` IP 測試 |
| 容器起不來且訊息含 GPU | `nvidia-smi` 是否正常 → 驅動更新後需 `sudo systemctl restart docker` |
| ComfyUI/Ollama 變超慢 | 兩個 GPU 服務同時重載模型會互擠;錯開使用或限制單邊 VRAM |
| NAS 掛載消失 | NAS 是否休眠;fstab 有 `nofail` 不會卡開機,`sudo mount -a` 重掛 |
| Windows 遠端連不上 | 是否睡著(先 WoL);Windows Update 是否重開機中 |
