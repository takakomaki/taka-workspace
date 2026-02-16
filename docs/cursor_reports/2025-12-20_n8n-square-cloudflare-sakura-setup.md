# 2025-12-20: Square決済（決済リンク）→ n8n通知（VPS + Cloudflare Tunnel）構築手順メモ

> 目的：Squareの「決済リンク」で支払い完了が発生したら、n8nで受けてメールを毎回送る（宛先2つ）。  
> 前提：エラー対応の枝葉は省略（本書は再現手順に集中）。

---

## 0. 何を作ったか（最終形）

- **Square**：決済リンクで支払い受付。支払い完了イベントをWebhooksで送る。
- **n8n**：Webhook受信 →（署名検証/重複排除/情報取得）→ メール2通送信。
- **さくらVPS（Ubuntu 24.04）**：n8n本体をDockerで常時稼働（Postgres併用）。
- **Cloudflare**：
  - ドメイン：`rsa-hub.com` をCloudflare Registrarで購入
  - Zero Trust（Cloudflare One）で **Tunnel** を作成し、VPSの `localhost:5678` に転送
  - 公開URLを2つ用意：
    - 管理画面：`https://n8n.rsa-hub.com`
    - Webhook受け口：`https://hooks.rsa-hub.com`

---

## 1. Square側（概要）

### 1-1. Square標準通知（補足）

- Square標準の通知メールは **複数宛先へ直接送る**ことはできない。  
  → 複数宛先にしたい場合は **転送**または **自動化（n8n等）**を使う。

### 1-2. Square Webhooks（今回の本命）

- Square Developerでアプリを作成し、Webhooks購読を設定する。
- 受け口URL（後述）に対して、支払い完了（例：`payment.updated` で `COMPLETED`）を送る。

---

## 2. n8n（ローカルPCでの更新と起動を楽にする）

> これは「PC上でn8nを動かしていたときの運用」。最終的にはVPSへ移行したが、ローカル運用の整理として残す。

### 2-1. npx運用（バージョン固定で起動）

- 例（1.x最新系で固定）：

```bat
set DB_SQLITE_POOL_SIZE=2
set N8N_RUNNERS_ENABLED=true
npx -y n8n@1.123.6
```

### 2-2. `n8n` コマンド1発で起動できるようにする（Windows）

- 既存のグローバル n8n を外す（ある場合）：

```bat
npm uninstall -g n8n
```

- `n8n.cmd` を作り、`n8n` と打つと npx固定起動するようにした。
  - 置き場所：`C:\Users\<あなた>\AppData\Roaming\npm\n8n.cmd`（PATHに入っている）

`n8n.cmd` 例：

```bat
@echo off
set DB_SQLITE_POOL_SIZE=2
set N8N_RUNNERS_ENABLED=true
npx -y n8n@1.123.6
```

---

## 3. Cloudflare（ドメインとTunnelの前提）

### 3-1. ドメイン

- `rsa-hub.com` を Cloudflare Registrar で購入し、Cloudflareで管理（DNSもCloudflare）。

### 3-2. サブドメイン運用方針（Vercel等のLP量産前提）

- **サブドメイン方式**で増やす（推奨）：
  - `lp1.rsa-hub.com`, `seminar.rsa-hub.com`, `campaign-2026.rsa-hub.com` …など
- n8nは業務自動化のハブとして切り出す：
  - `n8n.rsa-hub.com`（管理画面）
  - `hooks.rsa-hub.com`（Webhook受け口）

---

## 4. さくらVPS（Ubuntu 24.04）でn8nを常時稼働

> VPSに置く理由：Webhookは「届いた瞬間」に受ける必要があるため、PCを落としても動く **24/7環境**が必要。

### 4-1. VPS作成

- OS：Ubuntu 24.04 amd64
- パケットフィルタ：最小は **TCP 22（SSH）** だけでOK（Tunnelで公開するため 80/443 の開放は不要）
- SSH鍵：`ssh-keygen -t ed25519` で作って公開鍵を登録

### 4-2. Docker + Postgres + n8n（VPS内で実行）

SSHでVPSへログイン後、以下を実行してコンテナを起動（最初はホーム直下にcomposeを置いて運用）：

```bash
sudo apt-get update -y
sudo apt-get upgrade -y

curl -fsSL https://get.docker.com | sudo sh

cat > .env <<'EOF'
N8N_EDITOR_BASE_URL=https://n8n.rsa-hub.com
WEBHOOK_URL=https://hooks.rsa-hub.com

N8N_ENCRYPTION_KEY=__REPLACE_WITH_RANDOM__
N8N_RUNNERS_ENABLED=true
N8N_SECURE_COOKIE=true
N8N_LOG_LEVEL=info

POSTGRES_DB=n8n
POSTGRES_USER=n8n
POSTGRES_PASSWORD=__REPLACE_WITH_RANDOM__
EOF
```

> `N8N_ENCRYPTION_KEY` / `POSTGRES_PASSWORD` は強いランダム値で生成（例：`openssl rand -hex 32`）。

`docker-compose.yml`（例）：

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:1.123.6
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}

      N8N_EDITOR_BASE_URL: ${N8N_EDITOR_BASE_URL}
      WEBHOOK_URL: ${WEBHOOK_URL}
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      N8N_RUNNERS_ENABLED: ${N8N_RUNNERS_ENABLED}
      N8N_SECURE_COOKIE: ${N8N_SECURE_COOKIE}
      N8N_LOG_LEVEL: ${N8N_LOG_LEVEL}

      # security-forward defaults
      N8N_BLOCK_ENV_ACCESS_IN_NODE: "true"
      N8N_GIT_NODE_DISABLE_BARE_REPOS: "true"
    ports:
      - "127.0.0.1:5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  postgres_data:
  n8n_data:
```

起動：

```bash
sudo docker compose up -d
sudo docker compose ps
```

---

## 5. Cloudflare Tunnel（VPSのn8nをHTTPSで公開）

### 5-1. Tunnel作成（Cloudflare UI）

- Cloudflare **Zero Trust**（Cloudflare One） → **Networks** → **Tunnels**
- Tunnel作成（例：`rsa-hub-vps`）
- Connectorのインストール用に「トークンコマンド（cloudflared service install ...）」を取得

### 5-2. VPSにcloudflaredを入れてTunnelを常駐（VPS内）

APTで詰まったため、最終的には **GitHubの`.deb`直インストール**で導入：

```bash
cd /tmp
curl -L -o cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb || sudo apt-get -f install -y
cloudflared --version
```

Cloudflare画面に出ている「トークンの install コマンド」をVPSで実行（トークンは外部に漏らさない）：

```bash
sudo cloudflared service install <TOKEN...>
sudo systemctl status cloudflared --no-pager
```

### 5-3. 公開ルート（Public Hostname 相当）を2つ作る（Cloudflare UI）

Zero Trust の tunnel（`rsa-hub-vps`）内で「公開されたアプリケーション ルート」を追加：

- `n8n`：
  - Hostname：`n8n.rsa-hub.com`
  - Service：`http://localhost:5678`
- `hooks`：
  - Hostname：`hooks.rsa-hub.com`
  - Service：`http://localhost:5678`

> UI上の「ホスト名ルート（ベータ）」は **プライベート（WARP）系**になりやすいので、外部Webhook用途は「公開されたアプリケーション ルート」を使う。

### 5-4. 動作確認

- `https://n8n.rsa-hub.com` にアクセス → n8nの初期セットアップが出る → オーナー作成 → ログインできる
- n8nのWebhookノードの **Production URL** が `https://hooks.rsa-hub.com/webhook/...` になる

---

## 6. n8nワークフロー（Square決済完了→メール2通）

### 6-1. Webhook受信（n8n）

- Webhookノード
  - HTTP Method：**POST**（Square WebhookはPOST）
  - Production URL：`https://hooks.rsa-hub.com/webhook/<id>`

### 6-2. 推奨フロー（概要）

- Webhook受信
- Square署名検証（推奨）
- `payment.status == COMPLETED` のみ通す
- 重複排除（`payment.id` などで一度処理したものは弾く）
- Square APIで支払い詳細/注文情報を取得
- メール2通送信（宛先2つ、本文に：支払者名/メール、金額、セミナー名、支払い日時、決済リンク名）

---

## 7. セキュリティメモ（重要）

- `n8n.rsa-hub.com` は **Cloudflare Access** などでログイン必須にするのを推奨（管理画面をインターネットに裸で出さない）
- `hooks.rsa-hub.com` は外部サービス（Square等）が叩くため、基本は公開のまま  
  - 代わりに **署名検証**、または「推測困難なパス + トークン + レート制限」等で守る
- VPSの管理ユーザーのパスワード等、機密はチャットや公開場所に貼らない（漏れた場合は即変更）

---

## 8. いまの構成（控え）

- ドメイン：`rsa-hub.com`（Cloudflare）
- n8n管理：`https://n8n.rsa-hub.com`
- Webhook：`https://hooks.rsa-hub.com`
- VPS：さくらVPS Ubuntu 24.04（Docker + Postgres + n8n 1.123.6）


