# OPC 一人公司:硬體定位與工作流程最大化方案

> 目標:以現有設備為基礎,建立一套支撐「vibe coding 網站、行動 App、影片、影像、2D/3D 遊戲」
> 的一人公司工作系統。原則:**算力分工、資料集中、單一入口、先出貨再優化**。

## 一、整體架構

```
                 iPhone / iPad Pro (Tailscale)
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
  MacBook Pro M5     hero4090           local4090
  128GB              Ubuntu 24.04       Win11 Pro
  【主力開發機】      【24/7 AI 服務】    【遊戲/剪輯/測試機】
        │                 │                  │
        └────────┬────────┴───────┬──────────┘
                 ▼                ▼
               NAS【資料中樞】   iMac Intel【輔助/監控機】
```

## 二、各設備定位與該裝的東西

### MacBook Pro M5 128GB — 主力開發機(公司的大腦)

一人公司 90% 的時間在這台上。所有專案的源頭都在這裡,其他機器都是它的「外設」。

| 用途 | 工具 |
|---|---|
| Vibe coding | Claude Code + VS Code / Cursor |
| iOS/macOS App | Xcode(**只有 Mac 能做 iOS App,這台是唯一入口**) |
| 跨平台 App | Expo (React Native) 或 Flutter |
| 設計 | Figma(網頁版即可) |
| 超大模型本地推論 | LM Studio / Ollama(MLX 後端,128GB 可跑 70B~120B 量化模型) |
| 版本控制 | Git + GitHub,**所有專案一律進 GitHub** |

原則:MacBook 不跑常駐服務。它跟著人走,常駐的事交給 hero4090。

### hero4090 — 24/7 AI 服務中心(公司的工廠)

唯一的 Linux + CUDA 原生機,穩定性最好,扛所有不關機的服務。全部用 Docker Compose 管理,重開機自動復原。

| 服務 | 用途 |
|---|---|
| ComfyUI | 影像生成(Flux / SDXL)、遊戲素材、App icon、網站視覺 |
| 影片生成 | Wan 2.x / HunyuanVideo 等開源模型(24GB VRAM 跑量化版) |
| Ollama / vLLM | 常用 LLM 常駐(14B~32B 量化,回應快) |
| LiteLLM | 統一 API 閘道:手機/平板/任何裝置只記一個網址 |
| Open WebUI | 行動端聊天入口 |
| Uptime Kuma | 監控全部機器與服務,掛掉推播到 iPhone |
| Homepage/Homarr | 儀表板:一頁列出所有服務,設成 iPad/iPhone 書籤 |
| n8n(選配) | 自動化流程(社群發文、批次生圖等) |

搭配 `tailscale serve` 為各服務加上 HTTPS,行動端體驗更好。

### local4090 — 遊戲開發 / 剪輯 / Windows 測試機

重灌純 Win11 Pro 是對的:遊戲引擎工具鏈與玩家市場都以 Windows 為主。

| 用途 | 工具 |
|---|---|
| 遊戲引擎 | **先 Godot(免費、輕量、對新手最友善,2D 尤佳)**;3D 高保真再上 Unreal;要做手遊生態再考慮 Unity |
| 影片剪輯 | DaVinci Resolve(免費版功能已足,CUDA 加速) |
| Windows 測試 | 網站在 Windows 瀏覽器的相容性、遊戲成品測試 |
| 遠端串流 | Sunshine(主機端)+ Moonlight(iPad/MacBook 端),低延遲遠端使用完整 GPU 桌面 |

設定:開啟 Wake-on-LAN,平時可休眠,需要時從手機喚醒;關閉 Windows Update 自動重開機的作業時段。

### iMac Intel 64GB — 輔助工作機

Intel Mac 已接近 macOS 支援尾聲,**不要把關鍵工作押在它身上**。適合:

- 行政事務:信件、會議、文件、記帳
- 常開的儀表板螢幕(顯示 Uptime Kuma / Homepage)
- 第二工作位;跑不需要 GPU 的輕量工作

### NAS — 資料中樞(公司的金庫)

- 共享 `models/` 目錄(NFS/SMB):模型下載一次,全部機器掛載共用
- 共享 `assets/` 目錄:影片素材、遊戲美術、ComfyUI 輸出
- Time Machine(兩台 Mac)+ hero4090 Docker volume 定期備份
- **3-2-1 備份**:NAS 本身 + 一顆定期離線的外接碟 + 雲端(重要源碼在 GitHub 已算一份)
- 大型二進位素材不進 Git,放 NAS;源碼一律 GitHub

### iPhone / iPad Pro — 行動指揮所

- iPad Pro:Termius/Blink SSH、Moonlight 串流 local4090、Safari 開 Open WebUI 與儀表板、claude.ai/code 遠端派工、Sidecar 當 MacBook 第二螢幕、**App 實機測試裝置**
- iPhone:監控通知接收端、App 實機測試、隨時用 Claude 遠端下指令

## 三、五大領域的工作流

| 領域 | 入門技術棧 | 主要機器 | 產出路徑 |
|---|---|---|---|
| 網站 | Next.js + Tailwind,部署 Vercel(免費) | MacBook | Claude Code 生成 → GitHub → Vercel 自動上線 |
| 行動 App | Expo(一套碼出 iOS+Android)或 SwiftUI | MacBook + iPhone/iPad 實機 | Claude Code → Xcode/EAS → TestFlight → App Store |
| 影像 | ComfyUI(Flux/SDXL) | hero4090 | 手機/平板遠端開 ComfyUI 網頁生成 → 存 NAS |
| 影片 | 生成:hero4090;剪輯:DaVinci Resolve | hero4090 + local4090 | 生成素材 → NAS → Resolve 剪輯輸出 |
| 遊戲 | **2D 先行:Godot**;3D:Unreal | local4090(MacBook 也可跑 Godot) | Claude Code 寫 GDScript → itch.io / Steam |

新手順序建議:**網站 → App → 影像/影片 → 2D 遊戲 → 3D 遊戲**,由簡入深,每一步的成果都能養下一步(例如 ComfyUI 生的素材直接餵遊戲和影片)。

## 四、需要增購的部分

### 必要(小錢、直接解鎖能力)

| 項目 | 費用 | 理由 |
|---|---|---|
| Claude Pro/Max 訂閱 | 月費 | vibe coding 的核心引擎,整套方案的槓桿所在 |
| Apple Developer Program | US$99/年 | 沒有它 App 無法上架、TestFlight 無法用 |
| Google Play 開發者帳號 | US$25 一次性 | 若要出 Android 版 |
| 網域 | ~US$10/年 | 作品集與產品門面 |
| UPS 不斷電系統 | 中低價位即可 | 保護 hero4090 + NAS,24/7 機器必備 |
| 離線備份外接碟 | 依 NAS 容量 | 補齊 3-2-1 備份的最後一環 |

### 不建議現在買

- **新 GPU / 新 Mac / 更多主機**:雙 4090 + M5 128GB 的算力已遠超一人所需,瓶頸是時間和經驗,不是硬體
- 付費遊戲引擎資產、課程包:先用免費資源做出第一個作品再說

## 五、執行路線圖

### Phase 0 — 基礎建設(住院期間即可全遠端完成)

1. 全機 Tailscale + MagicDNS,hero4090 開 Tailscale SSH
2. hero4090 服務 Docker 化:ComfyUI、Ollama、LiteLLM、Open WebUI、Uptime Kuma、Homepage
3. NAS 建 `models/`、`assets/`、`backup/` 共享,各機掛載
4. GitHub 帳號整理,PIHub 作為中樞 repo(基礎設施設定、文件、待辦都放這)

### Phase 1 — 第一個月:網站

- 用 Claude Code 做一個作品集網站/簡單 SaaS,部署 Vercel、綁網域
- 目標:**完整走一次「想法 → 上線」**,建立信心與流程

### Phase 2 — 第二個月:影像影片管線 + App 起步

- ComfyUI 建立自己常用的工作流(風格、尺寸模板)
- DaVinci Resolve 剪出第一支影片(素材來自 ComfyUI)
- 以 Expo 起一個小 App,跑通 TestFlight 到實機

### Phase 3 — 第三個月:遊戲

- Godot 做一個小 2D 遊戲(素材用自家 ComfyUI 生成),發佈 itch.io
- 之後再評估 3D(Unreal on local4090)

### 貫穿原則

- **一次只做一個專案**,做完上線再開下一個;半成品是一人公司最大的成本
- 所有源碼進 GitHub、所有素材進 NAS、所有服務進儀表板
- 每個專案開工前先讓 Claude 寫 `CLAUDE.md`(專案規範),vibe coding 品質會穩定很多
