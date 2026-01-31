# Homelab Compose

Dockge + Kopia 的 Homelab 基礎設施管理與備份方案。

## 架構概覽

```
┌─────────────────────────────────────────────────────────┐
│                    這個 Repo                            │
├─────────────────────────────────────────────────────────┤
│  Dockge (Port 5001)      │  Kopia (Port 51515)         │
│  - Docker Compose 管理    │  - 備份 Server UI           │
│  - 簡單直覺的 UI          │  - 增量備份 + 去重          │
│                          │  - 加密儲存 → Google Drive  │
└─────────────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
    ┌─────────┐     ┌─────────┐     ┌─────────┐
    │ Stack A │     │ Stack B │     │ Stack C │
    │ (nginx) │     │  (DB)   │     │ (app)   │
    └─────────┘     └─────────┘     └─────────┘
```

## 快速開始

### 1. 複製環境變數

```bash
cp .env.example .env
```

### 2. 編輯 `.env`

```bash
# 產生安全的密碼
openssl rand -base64 32 # 用於 KOPIA_REPO_PASSWORD
```

重要設定：
- `STACKS_DIR`: 你的 Docker Compose 檔案目錄（必須是完整路徑）

### 3. 設定 Google Drive（rclone）

```bash
# 啟動 rclone 互動式設定
docker compose --profile setup run --rm rclone

# 依照提示操作：
# 1. 輸入 n（新增 remote）
# 2. 輸入名稱，例如：gdrive
# 3. 選擇 Google Drive（輸入對應數字）
# 4. client_id 和 client_secret 可留空（使用 rclone 預設）
# 5. scope 選擇 1（完整存取）
# 6. 其他選項按 Enter 使用預設值
# 7. 會開啟瀏覽器進行 Google 授權
# 8. 完成後輸入 q 離開
```

### 4. 啟動服務

```bash
docker compose up -d
```

### 5. 初始化 Kopia Repository（使用 Google Drive）

```bash
# 使用 rclone 後端連接 Google Drive
docker exec -it kopia-server kopia repository create rclone \
  --remote-path="gdrive:/kopia-backup"

# gdrive 是你在 rclone 設定的 remote 名稱
# /kopia-backup 是 Google Drive 上的資料夾路徑
```

### 6. 設定備份 Policy

```bash
docker exec -it kopia-server sh

# 設定自動備份（每 6 小時）
kopia policy set /source/stacks \
  --snapshot-interval=6h \
  --keep-hourly=24 \
  --keep-daily=7 \
  --keep-weekly=4 \
  --keep-monthly=6

kopia policy set /source/homelab-data \
  --snapshot-interval=6h \
  --keep-daily=7

# 手動觸發第一次備份
kopia snapshot create /source/stacks
kopia snapshot create /source/homelab-data
```

### 7. 存取 Web UI

| 服務 | URL | 說明 |
|------|-----|------|
| Dockge | http://localhost:5001 | Docker 管理 |
| Kopia | http://localhost:51515 | 備份管理 |

## 目錄結構

```
homelab-compose/
├── docker-compose.yml     # 服務定義
├── .env.example           # 環境變數範本
├── .env                   # 你的設定（不進 Git）
└── data/                  # 服務資料（bind mount）
    ├── dockge/           # Dockge 資料
    ├── kopia/            # Kopia 設定
    └── rclone/           # rclone 設定（含 Google 授權）
```

## 備份策略

### Kopia 備份什麼？

| 路徑 | 內容 |
|------|------|
| `/source/homelab-data` | Dockge 資料 + Kopia 設定 |
| `/source/stacks` | 所有 Docker Compose stacks |

### 備份流程

```
本地資料 → Kopia（增量+去重+加密）→ rclone → Google Drive
```

所有上傳到 Google Drive 的資料都是**加密的**，即使 Google 也無法讀取內容。

## 災難恢復

### 從 Google Drive 恢復

```bash
# 1. 安裝 Docker
curl -sSL https://get.docker.com | sh

# 2. 安裝 rclone 並設定 Google Drive
curl https://rclone.org/install.sh | sudo bash
rclone config
# 重新設定 gdrive remote（同上面的步驟）

# 3. 安裝 Kopia CLI
curl -s https://kopia.io/signing-key | sudo gpg --dearmor -o /usr/share/keyrings/kopia-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kopia-keyring.gpg] http://packages.kopia.io/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/kopia.list
sudo apt update && sudo apt install kopia

# 4. 連接 Google Drive Repository
kopia repository connect rclone --remote-path="gdrive:/kopia-backup"

# 5. 列出可用備份
kopia snapshot list

# 6. 還原資料
kopia restore <snapshot-id> /home/user/stacks

# 7. 重新啟動服務
docker compose up -d
```

## 重要提醒

⚠️ **務必備份這些到密碼管理器：**
- `KOPIA_REPO_PASSWORD` - 遺失無法恢復備份
- `data/rclone/rclone.conf` - Google Drive 授權（或記得如何重新授權）

## 相關資源

- [Dockge GitHub](https://github.com/louislam/dockge)
- [Kopia 官方文檔](https://kopia.io/docs)
- [Kopia rclone Repository](https://kopia.io/docs/repositories/#rclone)
- [rclone Google Drive 設定](https://rclone.org/drive/)
