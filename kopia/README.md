# Kopia Standalone Server

獨立的 Kopia 備份伺服器，供多台主機集中備份使用。

## 架構

```
┌─────────────────┐     ┌─────────────────┐
│  主機 A         │     │  備份主機       │
│  Kopia Client   │────→│  Kopia Server   │──→ Google Drive
└─────────────────┘     └─────────────────┘
         ↑                      ↑
┌─────────────────┐             │
│  主機 B         │─────────────┘
│  Kopia Client   │
└─────────────────┘
```

## 備份主機設定

### 1. 設定環境變數

```bash
cp .env.example .env

# 編輯 .env，設定密碼
# 或使用自動產生：
sed -i "s|KOPIA_REPO_PASSWORD=.*|KOPIA_REPO_PASSWORD=$(openssl rand -base64 32)|" .env
sed -i "s|KOPIA_SERVER_PASSWORD=.*|KOPIA_SERVER_PASSWORD=$(openssl rand -base64 16)|" .env
```

### 2. 設定 rclone（Google Drive）

```bash
docker compose --profile setup run --rm rclone
# 依照提示設定 Google Drive
```

### 3. 啟動服務

```bash
docker compose up -d
```

### 4. 初始化 Repository

```bash
docker exec -it kopia-server kopia repository create rclone \
  --remote-path="gdrive:/kopia-backup"
```

## 其他主機連接

### 方式 1：透過 Kopia Server API

在其他主機上安裝 Kopia client：

```bash
# Ubuntu/Debian
curl -s https://kopia.io/signing-key | sudo gpg --dearmor -o /usr/share/keyrings/kopia-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kopia-keyring.gpg] http://packages.kopia.io/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/kopia.list
sudo apt update && sudo apt install kopia

# 連接到 Kopia Server
kopia repository connect server \
  --url=http://備份主機IP:51515 \
  --override-username=主機名稱 \
  --override-hostname=主機名稱 \
  --password=你的KOPIA_SERVER_PASSWORD

# 設定要備份的目錄
kopia snapshot create /path/to/backup

# 設定自動備份
kopia policy set /path/to/backup \
  --snapshot-interval=6h \
  --keep-daily=7
```

### 方式 2：使用 Docker（推薦）

在其他主機上用 Docker 運行 Kopia client：

```yaml
# docker-compose.yml（放在要備份的主機上）
services:
  kopia-client:
    image: kopia/kopia:latest
    container_name: kopia-client
    restart: unless-stopped
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        kopia repository connect server \
          --url=http://備份主機IP:51515 \
          --override-username=$${HOSTNAME} \
          --override-hostname=$${HOSTNAME} \
          --password=$${KOPIA_SERVER_PASSWORD} && \
        kopia snapshot create /source && \
        sleep infinity
    environment:
      KOPIA_SERVER_PASSWORD: ${KOPIA_SERVER_PASSWORD}
      HOSTNAME: ${HOSTNAME:-client1}
    volumes:
      - ./data/kopia:/app/config
      - /要備份的目錄:/source:ro
```

## 查看所有備份

在備份主機上：

```bash
docker exec kopia-server kopia snapshot list --all
```

或透過 Web UI：`http://備份主機IP:51515`

## 注意事項

- 確保備份主機的 51515 port 可被其他主機存取
- 建議使用內網 IP，或透過 VPN/Tailscale 連接
- 如需對外開放，建議加上 TLS（設定 `--tls-cert-file` 和 `--tls-key-file`）
