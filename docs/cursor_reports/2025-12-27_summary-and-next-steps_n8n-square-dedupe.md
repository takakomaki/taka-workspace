## 2025-12-27 作業まとめ（n8n × Square 通知の安定化：重複排除）＋ 次回TODO

> 目的：Squareの決済リンク（payment.updated）をn8nで受けてメール通知する。  
> 本日の焦点：**「3通届く」→「1通にする（重複排除）」** と、詰まり箇所の解消。

---

## 📐 ワークフローの設計思想と構成

### 設計思想（2025-12-20からの継承）

1. **正規化優先**：Webhook直後にCodeノードでJSONを正規化し、以降のノードで一貫したパスで参照できるようにする
2. **早期フィルタリング**：COMPLETED以外のステータスは早い段階で除外
3. **重複排除**：Squareは同一決済を複数回送信するため、Postgresでidempotencyを担保
4. **データ取得**：Payments APIとOrders APIで詳細情報を取得
5. **メール送信**：2つの宛先にそれぞれ1通ずつ送信（重複排除付き）

### 推奨ワークフロー構成（2025-12-20時点）

```
1) Webhook (POST)
2) 署名検証（Square）- 推奨だが省略可能
3) IF：支払い完了だけ（payment.status == COMPLETED）
4) 重複排除（payment.id をキーに「既に処理済みなら終了」）
5) Square APIで詳細取得
   - Payments API：支払い詳細
   - Orders API：注文/商品名（＝セミナー名に使う想定）
6) メール送信（宛先2つ）
   - 支払者名/メール、金額、セミナー名、支払い日時、決済リンク名
```

### 現在の実装（2025-12-27時点）

```
Webhook (POST)
  → Code in JavaScript（正規化+COMPLETED判定）
    - WebhookのJSONを正規化
    - COMPLETED以外はreturn[]で早期終了
  → Execute a SQL query（Postgresで重複排除）
    - payment_idをINSERT（初回のみ成功）
    - 2回目以降は0行返る → 後続が動かない
  → 支払い詳細（HTTP Request: GET /v2/payments/{payment_id}）
  → 注文詳細（HTTP Request: GET /v2/orders/{order_id}）← 現在405エラー
  → Edit Fields（メール用に整形）
  → Send email（宛先2つ）
```

### 設計上の注意点

- **正規化Code**：Webhook直後に配置することで、以降のノードで `$json.data.object.payment.*` のパスで一貫して参照可能
- **重複排除**：Codeノードのstatic data APIが使えないため、Postgresで実装
- **注文詳細取得**：`payment.order_id` が存在しない場合があるため、Search Orders API（POST）への切り替えが必要な場合がある

---

---

## 今日の到達点（できたこと）

### 1) Square → n8n 受信・通し実行（メール送信まで）

- Square DeveloperのWebhook通知先は、n8nのProduction URLで固定：
  - `https://hooks.rsa-hub.com/webhook/55b41abc-1e6f-43a4-9a38-98b773e6c63a`
- Square側の Webhook Logs は **status code 200**（配信成功）。
- n8n側も `Webhook → 支払い詳細 → 注文詳細 → Edit Fields → Send email` まで **緑チェックで完走**し、メール送信できる状態になった。

### 2) 「Ifで止まる」問題の根本解決

- `status` が複数箇所に存在（例：`card_details.status = CAPTURED` と `payment.status = COMPLETED`）し、If判定がブレる問題があった。
- 結果として、Ifよりも **Webhook直後のCodeでpayloadを正規化＆COMPLETED以外はreturn[]で止める**方針にして安定化。
- 配線を確定（最終的な並び）：
  - `Webhook → Code in JavaScript（正規化+COMPLETED判定）→ … → 支払い詳細 → 注文詳細 → Edit Fields → Send email`

### 3) HTTP Requestのドメイン誤り修正

- Payments/Orders APIのホストは **`connect.squareup.com`** が正しい。
  - ✅ `https://connect.squareup.com/v2/payments/...`
  - ✅ `https://connect.squareup.com/v2/orders/...`
  - ❌ `connect.square.com` / ❌ `connect.squareupup.com`

### 4) Send emailの「Referenced node doesn't exist」修正

- メール本文が参照しているノード名と、実際のノード名が一致していなかった。
- 対策：HTTP Requestノードを
  - `支払い詳細`
  - `注文詳細`
  にリネームし、本文の参照を通す。

---

## 今日の課題（未完・次回やること）

### A) 3通届くのを1通にする（重複排除）

#### なぜ3通届く？
- Squareは同一決済に対して `payment.updated` を複数回送る（更新・再送・複数イベント）ことがある。

#### Codeノードでの重複排除が失敗した理由
- `this.getWorkflowStaticData(...)` や `$workflow.getStaticData(...)` が使えないCodeノード形式があり、重複排除コードがエラーになった。

#### 推奨の実装：Postgresで重複排除（idempotency）

1) DB側に重複排除テーブルを用意（VPS上のpostgresに作成済み）
   - テーブル：`webhook_dedupe(payment_id PRIMARY KEY, created_at)`
2) ワークフロー内に Postgresノード（Execute a SQL query）を挿入し、以下を実行：

```sql
INSERT INTO webhook_dedupe (payment_id)
VALUES ('{{$json.data.object.payment.id}}')
ON CONFLICT (payment_id) DO NOTHING
RETURNING payment_id;
```

- 1回目：1行返る → 後続（HTTP/Email）へ進む → **メール1通**
- 2回目以降：0行 → 後続が動かない → **増えない**

#### 注意
- n8n上で Postgresノードの `Execute step` が「ぐるぐる」して終わらない症状があった（DB到達性自体はVPSで確認済）。
- ただし、VPS側の確認では `n8nコンテナ → postgres:5432` は **疎通OK**。
- そのため、次回は **PostgresノードのCredential/Query/実行方法**を落ち着いて再確認し、まず `SELECT 1;` が返る状態を作る。

---

## 次回（明日以降）の具体的TODO（最短ルート）

### 1) Postgresノードの接続確認（決済不要）

- Postgres Credential：
  - Host: `postgres`
  - Port: `5432`
  - DB: `n8n`
  - User: `n8n`
  - Password: `/home/ubuntu/.env` の `POSTGRES_PASSWORD`
  - SSL: Disable
  - SSH Tunnel: OFF

- Queryを `SELECT 1;` にして実行し、すぐ返るか確認。

### 2) 重複排除INSERTの導入（本命）

- 上記の `INSERT ... ON CONFLICT ... RETURNING` を、メール送信の前に挿入。
- その後、新規決済で「1通だけ」になるか確認。

### 3) Executionsが見えない問題の確認（必要なら）

- Auto refreshだけではなく、**ワークフローごとのExecutions保存設定**を確認：
  - `Which executions is this workflow saving`
  - “成功を保存”がOFFだと一覧に出ないことがある。

---

## 参考：VPS側で確認できたこと（機密以外）

- `docker compose ps` で `n8n` と `postgres` はUp
- `n8nコンテナ → postgres:5432` は open
- n8nのDB接続環境変数は `DB_POSTGRESDB_HOST=postgres` 等で正しく設定されている

---

## ⚠️ 緊急対応：Editor画面とExecutions画面の構成不一致

### 問題
- **Editor画面**：`Webhook → Code in JavaScript → Execute a SQL query → 支払い詳細 → ...`（正しい構成）
- **Executions画面**：`Webhook → Code in JavaScript → Code in JavaScript1（エラー）→ 支払い詳細 → ...`（古い実行履歴）
- Executions画面に表示されているエラーは**過去の実行履歴**（古いバージョンで実行されたもの）の可能性が高い
- メールが届かない原因を特定する必要がある

### 確認済み
- ✅ Editor画面に「Code in JavaScript1」ノードは存在しない
- ✅ ワークフローは「Saved」状態
- ✅ 再Activate済み

### 次の確認ポイント

1. **最新の実行が新しいバージョンで実行されているか確認**
   - Executions画面で最新の実行を開く
   - ノード構成が `Webhook → Code in JavaScript → Execute a SQL query → ...` になっているか確認
   - もし「Code in JavaScript1」が表示されていれば、それは古い実行履歴

2. **SquareからWebhookが来ているか確認**
   - Square Developer → Webhooks → Logs で最新の通知を確認
   - Status code 200 になっているか
   - 最新の通知時刻を確認

3. **Postgresノードの動作確認**
   - 「Execute a SQL query」ノードを開き、クエリが正しいか確認：
   ```sql
   INSERT INTO webhook_dedupe (payment_id)
   VALUES ('{{$json.data.object.payment.id}}')
   ON CONFLICT (payment_id) DO NOTHING
   RETURNING payment_id;
   ```
   - Credential設定を確認（Host: `postgres`, Port: `5432`, Database: `n8n`）

4. **新しい実行を発生させる**
   - Squareで新しい決済を発生させる
   - または、Editor画面で「Execute workflow」ボタンで手動実行
   - Executions画面で最新の実行を確認し、新しいバージョンで実行されているか確認

5. **各ノードの実行結果を確認**
   - 最新の実行を開き、各ノードが緑チェック（成功）か確認
   - 特に「Execute a SQL query」ノードが成功しているか
   - 「Send email」ノードまで到達しているか

---

## 📧 メールを1通ずつ2か所に送信するためのPostgreSQL実装

### 要件
- 同じ`payment_id`に対して、2つの異なる宛先にそれぞれ1通ずつメールを送信
- 各宛先ごとに重複排除を管理（同じ宛先には1回だけ送信）

### 方法1：送信先をカラムに含めた1つのテーブル（推奨）

#### 1) テーブル作成（VPS上で実行）

```sql
CREATE TABLE IF NOT EXISTS webhook_dedupe (
  payment_id TEXT NOT NULL,
  recipient TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (payment_id, recipient)
);
```

#### 2) Postgresノードを2つ作成（各宛先用）

**Postgresノード1（宛先1用）**：
```sql
INSERT INTO webhook_dedupe (payment_id, recipient)
VALUES ('{{$json.data.object.payment.id}}', 'recipient1@example.com')
ON CONFLICT (payment_id, recipient) DO NOTHING
RETURNING payment_id, recipient;
```

**Postgresノード2（宛先2用）**：
```sql
INSERT INTO webhook_dedupe (payment_id, recipient)
VALUES ('{{$json.data.object.payment.id}}', 'recipient2@example.com')
ON CONFLICT (payment_id, recipient) DO NOTHING
RETURNING payment_id, recipient;
```

#### 3) ワークフロー構成

```
Webhook
  → Code in JavaScript（正規化+COMPLETED判定）
  → Postgresノード1（宛先1用の重複排除）
    → IF（返り値があるかチェック）
      → 支払い詳細
        → 注文詳細
          → Edit Fields
            → Send email（宛先1のみ）
  → Postgresノード2（宛先2用の重複排除）
    → IF（返り値があるかチェック）
      → 支払い詳細
        → 注文詳細
          → Edit Fields
            -> Send email（宛先2のみ）
```

> **注意**：この構成だと、支払い詳細と注文詳細を2回取得することになる。効率化したい場合は、HTTP Requestノードの結果を後続で再利用する構成に変更する。

### 方法2：シンプルな構成（支払い詳細を1回だけ取得）

#### ワークフロー構成

```
Webhook
  → Code in JavaScript（正規化+COMPLETED判定）
  → 支払い詳細
    → 注文詳細
      → Edit Fields
        → Postgresノード1（宛先1用の重複排除）
          → IF（返り値があるかチェック）
            → Send email（宛先1のみ）
        → Postgresノード2（宛先2用の重複排除）
          → IF（返り値があるかチェック）
            → Send email（宛先2のみ）
```

#### Postgresノードのクエリ（方法1と同じ）

**Postgresノード1（宛先1用）**：
```sql
INSERT INTO webhook_dedupe (payment_id, recipient)
VALUES ('{{$node["Code in JavaScript"].json.data.object.payment.id}}', 'recipient1@example.com')
ON CONFLICT (payment_id, recipient) DO NOTHING
RETURNING payment_id, recipient;
```

**Postgresノード2（宛先2用）**：
```sql
INSERT INTO webhook_dedupe (payment_id, recipient)
VALUES ('{{$node["Code in JavaScript"].json.data.object.payment.id}}', 'recipient2@example.com')
ON CONFLICT (payment_id, recipient) DO NOTHING
RETURNING payment_id, recipient;
```

> **注意**：`payment_id`の参照パスは、前のノード名に合わせて調整してください。

### 動作説明

- **初回実行**（新しい`payment_id`）：
  - Postgresノード1：1行返る → 宛先1にメール送信
  - Postgresノード2：1行返る → 宛先2にメール送信
  - **結果**：2通のメールが送信される

- **2回目以降**（同じ`payment_id`）：
  - Postgresノード1：0行返る → 後続が動かない → メール送信なし
  - Postgresノード2：0行返る → 後続が動かない → メール送信なし
  - **結果**：メール送信なし（重複排除成功）

### IFノードの設定（オプション）

Postgresノードの後ろにIFノードを置いて、返り値がある場合のみ後続に進むようにする：

- **Condition**：`{{ $json.payment_id }}` が存在する
- または、**Code in JavaScript**で `return $input.all().filter(item => item.json.payment_id);` のようにフィルタリング

### 既存テーブルの移行（既に`webhook_dedupe`テーブルがある場合）

既存のテーブル構造を変更する場合：

```sql
-- 既存テーブルをバックアップ（オプション）
CREATE TABLE webhook_dedupe_backup AS SELECT * FROM webhook_dedupe;

-- 既存テーブルを削除
DROP TABLE webhook_dedupe;

-- 新しい構造で再作成
CREATE TABLE webhook_dedupe (
  payment_id TEXT NOT NULL,
  recipient TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (payment_id, recipient)
);
```

または、既存テーブルにカラムを追加する場合：

```sql
-- recipientカラムを追加（既存レコードはデフォルト値で埋める）
ALTER TABLE webhook_dedupe 
ADD COLUMN recipient TEXT NOT NULL DEFAULT 'default@example.com';

-- 既存のPRIMARY KEYを削除して新しい複合主キーを作成
ALTER TABLE webhook_dedupe 
DROP CONSTRAINT webhook_dedupe_pkey;

ALTER TABLE webhook_dedupe 
ADD PRIMARY KEY (payment_id, recipient);
```

> **注意**：既存データがある場合は、移行前にバックアップを取ってください。

---

## ⚠️ エラー対応：「注文詳細」ノードの405 Method Not Allowed

### エラーメッセージ
```
Method not allowed - please check you are using the right HTTP method
405 - Method Not Allowed
```

### 問題点

1. **`payment.order_id`が存在しない可能性**
   - `payment.id` は存在するが、`payment.order_id` が存在しない場合がある
   - Square APIでは、paymentオブジェクトに `order_id` が含まれていない場合がある
   - その場合、`GET /v2/orders/{order_id}` は使えない

2. **AuthenticationがNoneになっている**
   - Square APIは認証が必須
   - 現在：`Authentication: None`
   - 正しい設定：Headersに `Authorization: Bearer <Production Access token>` を追加

3. **参照パスの確認**
   - URL: `https://connect.squareup.com/v2/orders/{{$json.payment.order_id}}`
   - 前のノード（支払い詳細）の結果を正しく参照できているか確認が必要

### 解決手順

#### 1) Authentication設定を追加

「注文詳細」ノードの設定：

- **Send Headers**: **ON** にする
- **Header Parameters** に以下を追加：

| Name | Value |
|------|-------|
| `Authorization` | `Bearer <Production Access token>` |
| `Square-Version` | `2025-10-16` |
| `Content-Type` | `application/json` |

> **注意**：`<Production Access token>` は実際のトークンに置き換えてください（`Bearer` の後に半角スペースを入れる）

#### 2) `order_id`の存在確認と対処

**INPUTパネルで確認**：
- `$json.payment.order_id` が存在するか確認
- 存在しない場合（`null` または `undefined`）：**Search Orders API**を使用する必要がある

**`order_id`が存在する場合**：
- URL: `https://connect.squareup.com/v2/orders/{{$json.payment.order_id}}`
- または: `https://connect.squareup.com/v2/orders/{{$node["支払い詳細"].json.payment.order_id}}`

**`order_id`が存在しない場合**：
- 以下の「Search Orders API（POSTメソッド）」を使用

#### 3) URLの確認

正しいURL形式：
```
https://connect.squareup.com/v2/orders/{{$json.payment.order_id}}
```

または、前のノード名を明示的に指定：
```
https://connect.squareup.com/v2/orders/{{$node["支払い詳細"].json.payment.order_id}}
```

#### 4) Methodの確認

- **Method**: `GET` のまま（変更不要）

### 確認ポイント

1. ✅ **Send Headers** が **ON** になっているか
2. ✅ **Authorization** ヘッダーに `Bearer <トークン>` が正しく入っているか
3. ✅ **Square-Version** ヘッダーが `2025-10-16` になっているか
4. ✅ **URL** の参照パスが正しいか（INPUTパネルで確認）
5. ✅ **Method** が `GET` になっているか

### テスト方法

1. 「注文詳細」ノードを開く
2. 右上の **Execute step** をクリック
3. OUTPUTパネルで結果を確認
4. エラーが解消され、注文データが返ってくることを確認

### 参考：支払い詳細ノードの設定（比較用）

「支払い詳細」ノードが正常に動作している場合、同じ設定を「注文詳細」ノードにも適用：

- **Method**: `GET`
- **URL**: `https://connect.squareup.com/v2/payments/{{$json.data.object.payment.id}}`
- **Send Headers**: `ON`
- **Headers**:
  - `Authorization: Bearer <Production Access token>`
  - `Square-Version: 2025-10-16`
  - `Content-Type: application/json`

---

## 🔍 405エラーが続く場合の詳細トラブルシューティング

### ステップ1：INPUTパネルでデータを確認

「注文詳細」ノードのINPUTパネルで以下を確認：

1. **`payment.order_id`が存在するか**
   - 存在しない場合：`null` または `undefined` が表示される
   - 存在する場合：実際のorder_idが表示される

2. **データ構造の確認**
   - `$json.payment.order_id` が正しいパスか確認
   - 前のノード（支払い詳細）のOUTPUTパネルで実際の構造を確認

### ステップ2：Headersの再確認

「注文詳細」ノードの設定画面で：

1. **Send Headers** が **ON** になっているか
2. **Header Parameters** に以下が**正確に**入っているか：
   - `Authorization`: `Bearer <実際のトークン>`（`Bearer`の後に半角スペース必須）
   - `Square-Version`: `2025-10-16`（正確な文字列）
   - `Content-Type`: `application/json`

3. **トークンの確認**
   - 「支払い詳細」ノードと同じトークンを使用しているか
   - トークンに余分なスペースや改行が入っていないか

### ステップ3：URLの参照パスを修正

INPUTパネルで `payment.order_id` が存在しない場合、以下のいずれかを試す：

**パターン1**（前のノード名を明示的に指定）：
```
https://connect.squareup.com/v2/orders/{{$node["支払い詳細"].json.payment.order_id}}
```

**パターン2**（配列の場合）：
```
https://connect.squareup.com/v2/orders/{{$json.payments[0].order_id}}
```

**パターン3**（paymentオブジェクトが直接ある場合）：
```
https://connect.squareup.com/v2/orders/{{$json.payment.order_id}}
```

### ステップ4：order_idが存在しない場合の対処（重要）

`payment.order_id` が `null` または存在しない場合、**Search Orders API（POSTメソッド）**を使用：

#### 方法A：Search Orders API（POSTメソッド） - payment_idで検索

**Method**: `POST`（GETではなくPOST）

**URL**:
```
https://connect.squareup.com/v2/orders/search
```

**Send Body**: `ON`

**Body Content Type**: `JSON`

**Body**（payment_idで検索）:
```json
{
  "query": {
    "filter": {
      "fulfillment_filter": {
        "fulfillment_types": ["SHIPMENT", "PICKUP", "DELIVERY"]
      }
    }
  },
  "limit": 1
}
```

または、より確実な方法（payment_idを直接指定できない場合）：
```json
{
  "query": {
    "filter": {
      "state_filter": {
        "states": ["COMPLETED", "OPEN"]
      }
    }
  },
  "limit": 10
}
```

その後、Codeノードで `payment.id` に一致する注文をフィルタリング。

**Headers**（同じ）:
- `Authorization: Bearer <Production Access token>`
- `Square-Version: 2025-10-16`
- `Content-Type: application/json`

#### 方法B：注文詳細を取得しない（シンプルな解決策）

`order_id` が存在しない場合、**注文詳細を取得せずに、支払い情報だけでメールを送信**する方法：

1. 「注文詳細」ノードを削除またはスキップ
2. 「支払い詳細」ノードの結果だけでメール本文を作成
3. セミナー名は、支払い情報から取得できる他のフィールドを使用

**メリット**：
- エラーが発生しない
- 処理がシンプルになる
- 支払い情報だけで十分な場合に有効

### ステップ5：デバッグ用のCodeノードを追加

「支払い詳細」と「注文詳細」の間に**Codeノード**を追加して、データを確認：

```javascript
// デバッグ用：前のノードのデータを確認
const payment = $json.payment || $json.payments?.[0] || $json;
const orderId = payment?.order_id;

console.log('Payment data:', JSON.stringify(payment, null, 2));
console.log('Order ID:', orderId);

// order_idがない場合の処理
if (!orderId) {
  console.warn('order_id is missing. Payment ID:', payment?.id);
  // ここでSearch Orders APIを使うか、エラーを返すか判断
}

return [{ json: { payment, orderId } }];
```

### ステップ6：最小限のテスト

1. 「注文詳細」ノードを開く
2. **Method** を `GET` に設定
3. **URL** を一時的に固定値でテスト：
   ```
   https://connect.squareup.com/v2/orders/テスト用のorder_id
   ```
4. **Headers** を正しく設定
5. **Execute step** を実行
6. エラーが解消されれば、参照パスの問題
7. エラーが続く場合は、認証またはエンドポイントの問題

### よくある原因と解決策

| 原因 | 解決策 |
|------|--------|
| Headersが設定されていない | Send HeadersをONにして、3つのヘッダーを追加 |
| トークンが間違っている | 「支払い詳細」ノードと同じトークンを使用 |
| `order_id`が存在しない | Search Orders API（POST）を使用 |
| 参照パスが間違っている | INPUTパネルで実際のデータ構造を確認 |
| URLのパスが間違っている | `https://connect.squareup.com/v2/orders/` が正しい |


