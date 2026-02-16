# 2025-12-20 次回（明日）やることメモ：n8n × Square（C）→ n8n × GitHub（B）

> ゴール：  
> 1) Square決済リンクの「支払い完了」をn8nで受け、メールを毎回2通送る（Cを完成）  
> 2) その後、n8nとGitHubを連携して運用自動化（Bに着手）

---

## 0. いまの完成状態（前提）

- VPS（さくら）上で n8n + Postgres が Docker で稼働中（127.0.0.1:5678）
- Cloudflare Tunnel で公開済み
  - 管理画面：`https://n8n.rsa-hub.com`
  - Webhook受け口：`https://hooks.rsa-hub.com`
- Square Developerでアプリ作成済み
- Square Webhooks：`payment.updated` を n8n の Production URL（`hooks.rsa-hub.com/webhook/...`）へ送る設定済み

---

## 1. 明日の最優先：Square通知（C）を完成させる

### 1-1. n8n側：Webhookノード確認

- Webhookノードの **HTTP Method を POST** にする（SquareはPOST）
- Production URLが `https://hooks.rsa-hub.com/webhook/...` になっていることを再確認

### 1-2. セキュリティ（推奨）

- `n8n.rsa-hub.com` は管理画面なので、**Cloudflare Access** 等でログイン必須にする（あなたのメールだけ許可）
- `hooks.rsa-hub.com` は外部（Square）が叩くので公開のまま
  - 代わりに **Square署名検証**（必須推奨）と **重複排除** をn8nで実装

### 1-3. n8n：ワークフロー（推奨構成）

1) **Webhook**（POST）
2) **署名検証**（Square）
   - Square側の **Signature key** を使う（値は貼らない・安全に保管）
3) **IF**：支払い完了だけ
   - `payment.status == COMPLETED` のときだけ通す
4) **重複排除**
   - `payment.id` をキーに「既に処理済みなら終了」
5) **Square APIで詳細取得**
   - Payments APIで支払い詳細
   - Orders APIで注文/商品名（＝セミナー名に使う想定）
6) **メール送信（宛先2つ）**
   - 支払者名/メール、金額、セミナー名、支払い日時、決済リンク名（可能な範囲）を本文に入れる

> 注：決済リンク名は環境によって“そのまま取れない”場合があるので、まずは **Orderのline item名をセミナー名**として扱うのが安定。

### 1-4. すぐできるテスト

- Square側でテスト決済（可能なら少額）を行う
- n8nのExecutionsでイベント受信→メール2通送信まで確認
- 取りこぼし防止：決済受付期間中はVPS上のn8n/docker、cloudflaredが稼働していること

---

## 2. 次の段階：n8n × GitHub（B）の進め方（概要）

### 2-1. 目的例（どれをやるか決める）

- PR作成・レビュー依頼・ラベル付け
- Issue起票（問い合わせ/フォーム→Issue化）
- リリース/デプロイ通知（GitHub→Slack/メール）
- 変更ログ生成（PRタイトル一覧→リリースノート）

### 2-2. 事前準備

- GitHub側で **Personal Access Token（Fine-grained推奨）** か GitHub App を用意
  - 必要な権限：Issues / Pull Requests / Contents など（用途に応じて最小権限）
- n8nで **GitHubノード** or **HTTP Request**（GitHub API）で接続

### 2-3. 最初のおすすめワークフロー（簡単で効果大）

- 「フォーム/メール/Slack → GitHub Issue自動作成」
  - タイトル/本文テンプレを固定して、漏れなく追跡できるようにする

---

## 3. 作業再開チェックリスト（明日用）

- [ ] `https://n8n.rsa-hub.com` にログインできる
- [ ] VPSで `sudo docker compose ps` が両方Up（n8n/postgres）
- [ ] VPSで `sudo systemctl status cloudflared` が active (running)
- [ ] Square Webhookが `payment.updated` を購読している
- [ ] n8n Webhookが POST になっている


