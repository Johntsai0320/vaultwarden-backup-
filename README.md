# NAS 私人密碼庫 — Vaultwarden + Vaultwarden-Backup（本機 + Google Drive 雙重備份）

專為 **Synology NAS（群暉 Container Manager）** 設計的私人密碼管理平台（相容 Bitwarden 用戶端）與自動化加密熱備份解決方案，支援 **NAS 本機 + Google Drive 異地雙重備份**。

部署架構參考 [`Johntsai0320/LibreTV`](https://github.com/Johntsai0320/LibreTV)（`.env` 集中管理參數 + GHCR 官方多架構映像檔）與 [`Johntsai0320/novel`](https://github.com/Johntsai0320/novel)（NAS 本機目錄持久化掛載與 Synology Container Manager 專案流程）。

---

## ✨ 特色設計

1. **雙容器協同運作（同專案一鍵啟動）**
   - **`vaultwarden`**：官方輕量級 Bitwarden 相容後端（[`ghcr.io/dani-garcia/vaultwarden:latest`](https://github.com/dani-garcia/vaultwarden/pkgs/container/vaultwarden)，內建 `/healthcheck.sh` 與 WebSocket Port 80 即時同步）。
   - **`vaultwarden-backup`**：採用 [`ghcr.io/ttionya/vaultwarden-backup:latest`](https://github.com/ttionya/vaultwarden-backup/pkgs/container/vaultwarden-backup)，透過 SQLite 官方線上 `.backup` API 熱備份（相容 WAL 模式，無需停機），並使用 **7z 加密（含檔名標頭加密）**打包。
2. **NAS 本機 + Google Drive 3-2-1 雙重備份**
   - 第一備份目標 **`[BitwardenBackup]`**：儲存於 NAS 本機 `./backups`（可快速原地還原）。
   - 第二備份目標 **`[GoogleDrive]`**：自動同步至您的 Google Drive `/VaultwardenBackup` 資料夾。
   - 預設於**每日台灣時間凌晨 03:00** 自動備份，保留最近 **30 天** 備份檔（可於 `.env` 調整 `BACKUP_CRON` 與 `BACKUP_KEEP_DAYS`）。
3. **敏感憑證 Git 隔離設計**
   - `.env`（含 `ZIP_PASSWORD`、`ADMIN_TOKEN`）與 `rclone/rclone.conf`（含 Google Drive OAuth `refresh_token`）皆已列入 `.gitignore`，不會誤推到 GitHub。
4. **避開現有服務 Port 衝突**
   - 預設映射到 NAS **`8200`** 連接埠（避開 `LibreTV` 的 `8080`、`novel` 的 `3000` 與 DSM 的 `5000/5001`）。

---

## 🚀 部署步驟（目標路徑：`/volume1/docker/vaultwarden`）

### 步驟 1：下載專案至 NAS 資料夾

透過 SSH 連線到 Synology NAS，將專案 Clone 至 `/volume1/docker/vaultwarden`：

```bash
cd /volume1/docker
git clone https://github.com/Johntsai0320/vaultwarden-backup-.git vaultwarden
cd /volume1/docker/vaultwarden
```

### 步驟 2：建立 `.env` 與 `rclone/rclone.conf`

```bash
cp .env.example .env
cp rclone/rclone.conf.example rclone/rclone.conf
```

編輯 `.env`（可用 DSM **Text Editor** 或 SSH `vi .env`），至少設定強密碼：

```env
ZIP_PASSWORD='請改為您的高強度備份加密密碼'
```

> [!TIP]
> 若備份密碼或 `ADMIN_TOKEN` 含有 `$` 或空白等特殊字元，請務必使用**單引號 `'...'`** 包住，避免被 Docker Compose 誤判為變數替換。

### 步驟 3：設定 Google Drive 遠端授權（無頭 NAS 教學）

由於 `.env.example` 預設啟用雙重備份（`RCLONE_REMOTE_NAME_1=GoogleDrive`），**啟動專案前請先完成 Google Drive 授權**（否則 `vaultwarden-backup` 啟動檢查連線時會停止並等待授權完成）。

> [!NOTE]
> 若想先以「僅 NAS 本機備份」啟動測試，可先將 `.env` 內的 `RCLONE_REMOTE_NAME_1=` 與 `RCLONE_REMOTE_DIR_1=` 留空，之後再回來設定。

1. 在 NAS SSH 執行互動式 `rclone config`：
   ```bash
   docker run --rm -it \
     -v /volume1/docker/vaultwarden/rclone:/config/rclone \
     ghcr.io/ttionya/vaultwarden-backup:latest \
     rclone config
   ```
2. 依序輸入以下選項：
   - `e/n/d/r/c/s/q>` 輸入 `n`（新增遠端）
   - `name>` 輸入 **`GoogleDrive`**（須與 `.env` 的 `RCLONE_REMOTE_NAME_1` 完全一致）
   - `Storage>` 輸入 `drive`（Google Drive）
   - `client_id>` 與 `client_secret>` 直接按 `Enter` 跳過（或填入自建 GCP OAuth Client）
   - `scope>` 建議輸入 **`3`**（`drive.file`：僅授權存取 `rclone` 自行建立的檔案與資料夾，權限最小最安全）
     > ⚠️ 若選 `3`，**請勿手動在 Google Drive 網頁先建立 `VaultwardenBackup` 資料夾**，交由容器初次檢查時自動建立，否則 `rclone` 會因看不到人工建立的資料夾而報錯。
   - `service_account_file>` 直接按 `Enter` 跳過
   - `Edit advanced config?` 輸入 `n`
   - `Use web browser to automatically authenticate rclone with remote?` 輸入 **`n`**（NAS 無桌面瀏覽器，**必選 `n`**！）
3. 此時 NAS 終端機會顯示一串指令，例如：
   ```text
   rclone authorize "drive" "eyJzY29wZSI6ImRyaXZlLmZpbGUifQ"
   ```
   - 在**您平時使用的電腦（Windows / macOS）** 安裝 [rclone](https://rclone.org/downloads/) 並執行該指令。
   - 瀏覽器會跳出 Google 登入與授權頁面，完成授權後，電腦終端機會印出一段包含 `token` 的 JSON 字串（位於 `Paste the following into your remote machine --->` 下方）。
   - 將該 JSON 字串完整複製貼回 NAS 的 `config_token>` 提示中並按 `Enter`。
4. 最後完成確認：
   - `Configure this as a Shared Drive (Team Drive)?` 輸入 `n`
   - `Keep this "GoogleDrive" remote?` 輸入 `y`
   - `e/n/d/r/c/s/q>` 輸入 `q` 離開

5. 驗證兩個備份目標皆可正常讀取：
   ```bash
   docker run --rm \
     -v /volume1/docker/vaultwarden/rclone:/config/rclone \
     ghcr.io/ttionya/vaultwarden-backup:latest \
     rclone config show
   ```
   確認同時看得到 `[BitwardenBackup]` 與 `[GoogleDrive]` 兩個區段即可。

### 步驟 4：在 Synology Container Manager 匯入「專案」

1. 開啟 DSM **Container Manager** → 左側選擇 **專案 (Project)** → 點擊 **新增 (Create)**。
2. 填寫專案名稱：`vaultwarden`。
3. **路徑 (Path)** 選擇 `/docker/vaultwarden`（即 `/volume1/docker/vaultwarden`）。
4. Container Manager 會自動偵測 `docker-compose.yml` → 選擇 **使用現有的 docker-compose.yml**。
5. 啟動專案後，確認：
   - `vaultwarden` 狀態為 **健康 (healthy)**。
   - `vaultwarden-backup` 狀態為 **執行中**，且記錄檔出現：
     - `RCLONE_REMOTE: BitwardenBackup:/backups`
     - `RCLONE_REMOTE: GoogleDrive:/VaultwardenBackup`
   - 首次啟動會自動在您的 Google Drive 建立 `/VaultwardenBackup` 資料夾。

---

## 🔒 必看：設定 HTTPS 反向代理（強制要求）

Bitwarden Web Vault 與瀏覽器擴充套件依賴瀏覽器 Web Crypto API，**非 `localhost` 必須透過 HTTPS 才能登入與註冊**。

1. 前往 DSM **控制台 → 登入入口 → 進階 → 反向代理伺服器 → 新增**：
   - **描述**：`Vaultwarden`
   - **來源**：協定 `HTTPS` / 主機名稱 `vw.yourname.synology.me`（或您的自訂網域）/ 連接埠 `443`
   - **目的地**：協定 `HTTP` / 主機名稱 `localhost` / 連接埠 `8200`
   - **自訂標題**頁籤 → **新增 → WebSocket**（會自動加入 `Upgrade` 與 `Connection` 標頭，確保多裝置即時同步）
2. 前往 DSM **控制台 → 安全性 → 憑證**，為該網域申請並套用 Let's Encrypt 憑證。
3. 回到 `.env` 更新對外網址並鎖定本機監聽，再於 Container Manager 專案頁面重新套用 / 重建：
   ```env
   DOMAIN=https://vw.yourname.synology.me
   VAULTWARDEN_BIND_IP=127.0.0.1
   ```

---

## 🛡️ 首次使用安全強化

1. **關閉公開註冊**：用瀏覽器開啟 `https://vw.yourname.synology.me` 完成個人帳號註冊並開啟 2FA 後，立即修改 `.env` 並重新套用專案：
   ```env
   SIGNUPS_ALLOWED=false
   ```
2. **啟用 `/admin` 管理後台（選用）**：
   預設 `ADMIN_TOKEN=` 留空代表完全停用 `/admin` 頁面。如需啟用，請執行以下指令產生 Argon2id 雜湊：
   ```bash
   docker exec -it vaultwarden /vaultwarden hash
   ```
   將輸出的 `$argon2id$...` 用**單引號**填入 `.env`：
   ```env
   ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'
   ```

---

## 💾 手動測試備份與災難還原

### 手動觸發一次完整備份（驗證 NAS 本機 + Google Drive 上傳）

```bash
docker exec vaultwarden-backup bash /app/backup.sh
```

執行成功後，應同時在以下兩處看到加密備份檔 `backup.YYYYMMDD_HHMMSS.7z`：
- NAS 本機：`/volume1/docker/vaultwarden/backups/`
- Google Drive：`我的雲端硬碟 / VaultwardenBackup /`

### 災難還原（從本機或 Google Drive 下載的 `.7z` 還原）

> [!WARNING]
> 還原操作會覆蓋 `./data` 內的現有資料庫與金鑰，執行前請務必先停止專案並保留舊 `./data` 備份。

```bash
cd /volume1/docker/vaultwarden

# 1. 停止兩個容器
docker compose down

# 2. 備份目前資料夾以防萬一
cp -a ./data ./data.before-restore

# 3. 若要從 Google Drive 還原，請先將 backup.YYYYMMDD_HHMMSS.7z 下載至 ./backups/
# 4. 執行互動式還原（會詢問確認 y/N 與壓縮檔 ZIP_PASSWORD 密碼）
docker run --rm -it \
  --mount type=bind,source="$(pwd)/data",target=/bitwarden/data \
  --mount type=bind,source="$(pwd)/backups",target=/bitwarden/restore \
  ghcr.io/ttionya/vaultwarden-backup:latest restore \
  --zip-file backup.YYYYMMDD_HHMMSS.7z

# 5. 重新啟動服務
docker compose up -d
```
