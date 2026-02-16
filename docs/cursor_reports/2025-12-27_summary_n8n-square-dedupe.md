## 2025-12-27 作業まとめ（Square→n8n通知の安定化：重複排除まで）

> 目的：Square決済リンクの支払い完了をn8nで受けて、通知メールを毎回 **1回だけ**（宛先2つ）送る。  
> 方針：エラー対応の枝葉は省略し、「明日すぐ再開できる手順」に寄せる。

---

## 1) 今日できたこと（到達点）

### A. Square → n8n の本番Webhook受信は成功

- Square Developer → Webhooks → Logs で以下を確認：
  - **Status code 200**
  - Event type: `payment.updated`
  - Notification URL: `https://hooks.rsa-hub.com/webhook/55b41abc-1e6f-43a4-9a38-98b773e6c63a`

→ Squareからn8nへの到達性はOK。

### B. n8nフローの通しは一度成功（メール送信まで到達）

- ノード名を揃え（例：`支払い詳細` / `注文詳細`）、本文テンプレの参照エラー（Referenced node doesn't exist）を解消。
- `Send email` まで緑チェックで通る実行が発生し、メール送信が成功。

### C. 「3通届く」原因が見えた

- Squareは同じ決済に対してリトライ/更新で複数回 `payment.updated` を送ることがある。
- 重複排除が必要。

---

## 2) 今日ハマった点（原因の整理）

### A. n8nのCodeノード（新形式）では static data API が使えない

- `this.getWorkflowStaticData` → **使えない**
- `$workflow.getStaticData` → **使えない**

→ Codeノードだけで重複排除（idempotency）を持たせるのが難しい。

### B. HTTP RequestのURL typoで詰まった

- 正：`https://connect.squareup.com/...`
- 誤：`https://connect.square.com/...`

→ ドメイン間違いで接続できないエラーになった。

### C. Postgresノードの手動 Execute step が「ぐるぐる」することがある

- Credential自体は **Connection tested successfully**。
- ただしノードの手動実行が終わらないことがあった（原因はUI/実行状況依存の可能性）。

→ テーブル作成はVPS側で直接psql実行し、準備を先に完了。

---

## 3) 重要な現状（VPS/DB）

- docker-compose は `/home/ubuntu/docker-compose.yml`
- コンテナ：`ubuntu-n8n-1` / `ubuntu-postgres-1` が稼働中
- n8nコンテナ → `postgres:5432` の到達性（DNS/Port）はOK
- n8nコンテナ環境変数のDB設定（VPS側で確認済）：
  - `DB_TYPE=postgresdb`
  - `DB_POSTGRESDB_HOST=postgres`
  - `DB_POSTGRESDB_PORT=5432`
  - `DB_POSTGRESDB_DATABASE=n8n`
  - `DB_POSTGRESDB_USER=n8n`
  - `DB_POSTGRESDB_PASSWORD` は `.env` に保存（※機密）

### 3-1) 重複排除テーブルは作成済み

VPS上で psql から以下を実行して作成済み：

```sql
CREATE TABLE IF NOT EXISTS webhook_dedupe (
  payment_id TEXT PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4) 明日やること（最短で「3通→1通」にする）

### ゴール

- 同じ `payment.id` のWebhookが複数回来ても、メールは **1回だけ**送る。

### 手順

1. **重複排除の実装を「Postgres」ノードに寄せる**
   - Codeノードではstatic dataが使えないため、DBでidempotencyを担保する。

2. ワークフローの並び（例）
   - `Webhook`
   - `Code in JavaScript`（正規化+COMPLETED以外はreturn[]）
   - `Postgres (Execute a SQL query)` ← **ここが重複排除**
   - `支払い詳細 (HTTP Request: /v2/payments/...)`
   - `注文詳細 (HTTP Request: /v2/orders/...)`
   - `Edit Fields`
   - `Send email`

3. Postgresノード（重複排除）に入れるクエリ

```sql
INSERT INTO webhook_dedupe (payment_id)
VALUES ('{{$json.data.object.payment.id}}')
ON CONFLICT (payment_id) DO NOTHING
RETURNING payment_id;
```

> 期待動作：  
> - 初回（新しいpayment_id）→ 1行返る → 後続が動く → メール1回  
> - 2回目以降（同じpayment_id）→ 0行 → 後続が動かない → メール増えない

4. Postgres Credential（n8n内）
   - Host: `postgres`
   - Port: `5432`
   - Database: `n8n`
   - User: `n8n`
   - Password: VPS `/home/ubuntu/.env` の `POSTGRES_PASSWORD`（※貼らない）
   - SSL: Disable
   - SSH Tunnel: OFF

5. テスト方法
   - Squareで決済を1回発生
   - Square Logsで同じpaymentが複数回送られても、受信側メールが1通だけになるか確認

---

## 5) 追加の注意（運用）

- Square Logsは200でも、n8n側でExecutionsが見えない場合がある。
  - `Which executions is this workflow saving` の設定（成功実行を保存する）やフィルタの影響があり得る。
  - ただし「メールが届く/届かない」で本番挙動は判定できる。


