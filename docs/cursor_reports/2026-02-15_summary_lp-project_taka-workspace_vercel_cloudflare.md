## 2026-02-15 作業まとめ（lp-project / taka-workspace / Vercel / Cloudflare）

> 目的：GitHub + Vercel + Cloudflare の再セットアップを“忘れても再現できる形”に整理  
> 対象：LPシリーズ運用（サブドメイン固定＋パスで増やす）と、作業用ワークスペースの整備

---

## 今日やったこと（成果）

- **GitHub リポジトリ整理方針の決定**
  - 既存の複数リポジトリを無理に統合せず、今後の運用に合わせて **新規2本を主軸**にする方針
  - LP は `kekkankakumei` のセクション設計（ストーリー構造）が洗練されているため、これをベース採用

- **`lp-project` の新規作成（kekkankakumei app-web をベース）**
  - ローカル：`C:\doc\lp-project`
  - GitHub：`takakomaki/lp-project`
  - Next.js 16（App Router）として `npm install` / `npm run dev` / `npm run build` を確認

- **Vercel にデプロイ**
  - Vercel プロジェクト：`lp-project`
  - GitHub 連携による自動デプロイ

- **Cloudflare でサブドメイン固定（案A）**
  - Cloudflare 管理ドメイン：`rsa-hub.com`
  - サブドメイン：`seminar.rsa-hub.com` を Vercel（`lp-project`）へ接続
  - シリーズは **`seminar.rsa-hub.com/vol-xx`** のようにパスで増やす方針

- **`/vol-01`（固定ページ方式A）の実装**
  - `HeroSection` / `ConsultationCtaSection` を `seminar` を props 受け取りに変更
  - `app/vol-01/page.tsx` と `app/content/seminar-vol-01.ts` を追加
  - commit & push 済み（Vercel 自動反映）

- **`taka-workspace` の新規作成**
  - ローカル：`C:\doc\taka-workspace`
  - GitHub：`takakomaki/taka-workspace`
  - 初期構成：`docs/`, `workflows/`, `memory/`, `tools/`, `README.md`, `.gitignore`

- **`takakomaki/desktop-tutorial` をアーカイブ**
  - アーカイブは GitHub 上のリポジトリを read-only にするだけで、**ローカル PC のファイルは消えない**

- **Refined GitHub / RepoHide 周り**
  - ダッシュボードの「Top repositories」からアーカイブを完全に非表示にするのは難しい
  - RepoHide は `?tab=repositories` 等、特定ページ向け（ダッシュボードには効かないケースあり）
  - PAT（トークン）を作成し、1Password に保存

---

## 重要URL（すぐ戻れるリンク）

- **GitHub**
  - `lp-project`: `https://github.com/takakomaki/lp-project`
  - `taka-workspace`: `https://github.com/takakomaki/taka-workspace`

- **Vercel**
  - `lp-project`（プロジェクト）: `https://vercel.com/takakomaki-7193s-projects/lp-project`

- **公開URL**
  - サブドメイン（本番）: `https://seminar.rsa-hub.com/`
  - シリーズ Vol.01: `https://seminar.rsa-hub.com/vol-01`

---

## Cloudflare ⇄ Vercel 接続（再現手順）

### Vercel 側（`lp-project` → Settings → Domains）

- `seminar.rsa-hub.com` を **Production** に追加
- 表示される **DNS Records** の指示に従う

### Cloudflare 側（`rsa-hub.com` → DNS → Records）

- `seminar` の DNS レコードを追加（※Name の打ち間違い注意）
  - **Type**: `CNAME`
  - **Name**: `seminar`
  - **Target/Value**: `c4e19aee820e7032.vercel-dns-017.com`
  - **Proxy**: DNS のみ（灰色雲 / Proxied OFF）
  - **TTL**: 自動

### 反映確認

- Vercel Domains で `seminar.rsa-hub.com` が **Valid Configuration** になれば OK

---

## `/vol-01` の実装内容（要点）

- 変更
  - `app/components/sections/hero-section.tsx`
  - `app/components/sections/consultation-cta-section.tsx`
  - `app/page.tsx`（`seminar` を props で渡すように変更）

- 追加
  - `app/content/seminar-vol-01.ts`
  - `app/vol-01/page.tsx`

- ビルド確認
  - `npm run build` で `Route (app)` に `/vol-01` が出ることを確認

---

## 次回やること候補（優先順）

1. **Philosophy Section の構築**
   - 文章（理念/約束/反論の先回り）→ 情報設計 → 実装（content 外出し）
2. **Vol.02 以降の追加（固定ページ方式A）**
   - `app/content/seminar-vol-02.ts` を追加
   - `app/vol-02/page.tsx` を追加
3. **トップ（/）をシリーズのハブにする**
   - `/vol-01` などへの導線を追加（一覧・カードなど）

