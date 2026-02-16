## 2025-12-27 作業まとめ（n8n × Square 入金通知）＋ 次回やること

> 目的：Square決済リンクの支払い完了（入金完了）を n8n で受け、**通知メールを毎回1通（宛先は2名）**にする。  
> 今日のゴール：まず「通る」→ 次に「重複を止める」。

---

## 1. 今日できたこと（成果）

### 1-1. ワークフローの「通し」は成功（メール送信まで）

- Square → n8n Webhook が届いて実行される（Square側Webhook logsが **200**）。
- フローのノードは最終的に **Send email まで緑チェック**で通る状態にできた。
- SMTP送信（nifty SMTP）でメール送信自体は成功。

### 1-2. 詰まりの原因を解消したポイント

- **Webhookのpayload構造が揺れる問題**を避けるため、Webhook直後に「正規化 + COMPLETED以外は止める」Codeを入れた。
- HTTP RequestのSquare API URLのドメイン間違いで落ちたのを修正：
  - ❌ `connect.square.com` 等
  - ✅ `https://connect.squareup.com/...`
- Send emailの本文が「存在しないノード名」を参照して落ちたので、ノード名をリネームして解消：
  - `HTTP Request` → `支払い詳細`
  - `HTTP Request1` → `注文詳細`

---

## 2. 今日の現状（残っている課題）

### 2-1. メールが「3通ずつ」来る

- Squareの `payment.updated` は同一支払いでも複数回（更新/リトライ）届くため、**重複排除（idempotency）が必須**。
- もともとCodeで重複排除しようとしたが、n8nのCodeノードで
  - `this.getWorkflowStaticData` が使えない
  - `$workflow.getStaticData` も使えない
  という制約があり、Codeノード単体では保存型の重複排除が難しい。

### 2-2. Postgresノードの `Execute step` がぐるぐるして終わらない

- Credentialはスクショ上「Connection tested successfully」で、設定値も正しい（Host=`postgres`, Port=`5432`, DB=`n8n`, User=`n8n`, SSL=Disable, SSH Tunnel=OFF）。
- ただしノードの手動実行で結果が返らず、原因切り分けが必要。

---

## 3. 今日確定した「正しい設定値」メモ

### 3-1. Square Webhook通知先

- `https://hooks.rsa-hub.com/webhook/55b41abc-1e6f-43a4-9a38-98b773e6c63a`

### 3-2. Square API URL

- Payments:
  - `https://connect.squareup.com/v2/payments/{{$json.data.object.payment.id}}`
- Orders:
  - `https://connect.squareup.com/v2/orders/{{$json.payment.order_id}}`

### 3-3. 重複排除用テーブル（DB側は作成済み）

VPS側のpostgresに直接 `psql` で作成済み：

```sql
CREATE TABLE IF NOT EXISTS webhook_dedupe (
  payment_id TEXT PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

（n8nのPostgresノードでCREATE TABLEを叩けなくても、テーブルは既に存在する）

---

## 4. 次回（明日以降）の作業リスト（最短で「1通」にする）

### A) まず「Executionsが見えない」問題を先に潰す（超重要）

現象：Square logs 200なのに、Executions一覧に増えない/見えないことがある。  
原因候補：**成功実行を保存していない**、または表示フィルタ。

1. n8n画面左下の **`Which executions is this workflow saving`** を開く
2. **Save successful executions** を ON（成功も保存）
3. フィルタ（漏斗）があれば解除

> これがOFFだと、重複排除後に「即終了（成功）」になった実行が見えず、確認不能になる。

### B) Postgresで重複排除（本命）

**配置位置（おすすめ）**  
Webhook → Code（正規化＋COMPLETED判定）→ **Postgres（重複排除）** → 支払い詳細 → 注文詳細 → Edit Fields → Send email

Postgresノード（Execute a SQL query）のQueryをこれにする：

```sql
INSERT INTO webhook_dedupe (payment_id)
VALUES ('{{$json.data.object.payment.id}}')
ON CONFLICT (payment_id) DO NOTHING
RETURNING payment_id;
```

期待動作：
- 初回（同payment_idが未登録）：1行返る → 後続が動く → メール1通
- 同payment_idの再送：0行 → 後続が動かない → メール増えない

### C) テスト手順（最短）

1. 新しい決済を1回発生
2. Square Webhook logs が 200 のことを確認
3. n8n Executionsで、Postgresノードの出力が
   - 初回：`payment_id` が返る
   - 再送：0件
   になっているか確認
4. メールが **1通だけ（宛先2名）**届くか確認

---

## 5. 補足（ぐるぐるが続く場合の切り分け）

Postgresノードがぐるぐるで返らない場合は、次を確認：

- **Postgresノードが本当にワークフロー実行中に動いているか**（手動Execute stepではなく、Squareの実行で確認）
- **Hostが `postgres` になっているか**（VPSのIP/localhostにすると失敗）
- **ノードのTimeout設定**（Optionsにあれば一旦短くして原因を明確化）

---

## 6. 今日の状態での“暫定運用”について

重複排除が入るまでの間は、同一支払いで複数回メールが飛ぶ可能性があるため、運用上は注意：

- 件名に `payment.id` など識別子を入れておく（同一と分かる）
- 受信側でフィルタリング（同一IDは無視）を暫定で行う


