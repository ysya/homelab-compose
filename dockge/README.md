# Dockge Standalone

獨立的 Dockge 實例，用於管理 Docker Compose stacks。

## 快速開始

### 1. 設定環境變數

```bash
cp .env.example .env

# 設定 stacks 目錄（必須是完整路徑）
sed -i "s|STACKS_DIR=.*|STACKS_DIR=$(pwd)/stacks|" .env

# 建立 stacks 目錄
mkdir -p stacks
```

### 2. 啟動服務

```bash
docker compose up -d
```

### 3. 存取 Web UI

開啟 `http://localhost:5001`

## 連接到獨立 Kopia Server

如果你有獨立的 Kopia 備份主機，可以在這台主機上安裝 Kopia client 來備份：

```bash
# 安裝 Kopia
curl -s https://kopia.io/signing-key | sudo gpg --dearmor -o /usr/share/keyrings/kopia-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kopia-keyring.gpg] http://packages.kopia.io/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/kopia.list
sudo apt update && sudo apt install kopia

# 連接到 Kopia Server
kopia repository connect server \
  --url=http://備份主機IP:51515 \
  --override-hostname=$(hostname) \
  --password=你的密碼

# 設定備份（Dockge 資料 + stacks）
kopia policy set ./data --snapshot-interval=6h --keep-daily=7
kopia policy set ./stacks --snapshot-interval=6h --keep-daily=7

# 手動觸發第一次備份
kopia snapshot create ./data
kopia snapshot create ./stacks
```

## 目錄結構

```
dockge/
├── compose.yml      # Dockge 服務
├── .env.example     # 環境變數範本
├── .env             # 你的設定
├── data/            # Dockge 資料
│   └── dockge/
└── stacks/          # Docker Compose stacks
```
