## 2026-02-15 会話ログ要点（Cursor運用 / 文字コード / マルチカーソル）

> 対象：Cursor（エディタ運用）、SJIS→UTF-8、マルチカーソル、cursor_reports運用ルール

---

## 決まったこと（方針）

- **会話メモの保存運用**
  - 今後、必要な内容は `C:\doc\cursor_reports` に Markdown で保存
  - 保存タイミングは **節目ごとに私から「保存しますか？」と確認**する

- **命名規約（cursor_reports）**
  - 形式：`YYYY-MM-DD_HHMM__<type>__<topic>.md`
  - 例：`2026-02-15_1635__chat__cursor-workflow-encoding-multicursor.md`
  - ファイル名は **ASCIIのみ**、日本語タイトルは本文へ

- **新規ファイル（Ctrl+N）の既定言語**
  - `Ctrl+N` で作る新規ファイルの言語モードを Markdown に寄せたい
  - 設定：`files.defaultLanguage = markdown` を追加済み

---

## 今日やった/解決したこと

### 1) Cursorのタブが増えすぎる問題

- **目的**：WZ Editor 的に「独立表示」やタブ管理をしたい
- **結論**：
  - `Ctrl+Shift+N` で新規ウィンドウ → その中で `Ctrl+N` がシンプル
  - 1ファイルを別ウィンドウに分離する方法（タブ右クリック等）も利用可能

### 2) `.txt` を Cursor で開きたい（Shift-JIS問題）

- **結論**：可能
  - 文字化けする場合は Cursor の **Reopen with Encoding** で `Japanese (Shift JIS)` を選び、読み直す
  - その後 **Save with Encoding** で UTF-8 に変換して保存できる

#### UTF-8 と UTF-8 with BOM の違い

- **UTF-8（推奨）**：BOMなし（GitHub/開発では基本これ）
- **UTF-8 with BOM**：先頭にBOM付き（古いWindowsアプリが判別しやすい場合があるが、ツール次第で不具合も）

### 3) マルチカーソルの使い方

- `Ctrl + D`：次の一致を追加して同時編集
- `Alt + Click`：任意の場所にカーソル追加
- `Ctrl + Alt + ↑/↓`：上下にカーソル増殖
- まとめ資料を `cursor_reports` に保存済み：
  - `2026-02-15_cursor_multi-cursor_guide.md`

---

## 参考（今回作った/更新したファイル）

- `C:\doc\cursor_reports\2026-02-15_cursor_multi-cursor_guide.md`
- `C:\doc\cursor_reports\2026-02-15_summary_lp-project_taka-workspace_vercel_cloudflare.md`（別トピックの作業まとめ）

---

## 次回の入り口（おすすめ）

- Markdown運用を続ける前提で：
  - `.md` は Cursor をメイン
  - `.txt`（SJIS）は必要に応じて UTF-8 化
  - 節目ごとに `cursor_reports` に要点保存

