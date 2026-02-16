## 2026-02-16 作業まとめ（lp-project / 加藤先生LP / Vercel）

> 目的：加藤先生セミナーLP（Vol.01）の制作を「コピペ地獄」にならない運用に移行しつつ、Vercelで確認できる状態まで進める  
> 対象：`C:\doc\lp-project`（Next.js App Router / Tailwind）＋ Vercel（`lp-project`）

---

## 今日やったこと（成果）

- **世界観ブリーフ（Single Source of Truth）を新規追加**
  - `lp-project/docs/brief-kato.md`
  - LP上の判断基準（世界観・訴求の柱・トーン・リスク管理・免責）を固定

- **Vol.01用の「型付き・原稿単一ソース」を新規追加**
  - `lp-project/app/content/lp-copy-vol-01.ts`
  - Hero〜CTA〜FAQ〜免責まで、LPのコピーを一箇所に集約（今後の差し替えは基本ここだけ）

- **Hero/CTA を “見た目を崩さず” データ参照に変更**
  - 対象
    - `lp-project/app/components/sections/hero-section.tsx`
    - `lp-project/app/components/sections/consultation-cta-section.tsx`
  - 方針
    - DOM構造・Tailwindクラス・背景/SVG/余白/タイポの指定は維持
    - 文言（見出し/サブ/CTAラベル）だけ `lpCopyVol01` 参照に置き換え

---

## Git（lp-project側）での反映

- **リポジトリ**：`takakomaki/lp-project`
- **ブランチ**：`main`
- **コミット**
  - `a8dd529`：`brief-kato.md` と `lp-copy-vol-01.ts` を追加
  - `02f9d83`：Hero/CTA のコピーを `lpCopyVol01` 参照に配線（ビジュアル維持）

---

## Vercel での確認（現状）

- **Vercel CLI**：50.17.1（ログインユーザー：`takakomaki-7193`）
- **公開URL（本番エイリアス）**
  - `https://seminar.rsa-hub.com`
- **Git連携のデプロイURL（main）**
  - `https://lp-project-git-main-takakomaki-7193s-projects.vercel.app`

### メモ（CLI運用）
- CLIで `vercel deploy --yes` を実行すると、エージェント検知により非対話モード扱いになり **scope指定が要求**されるケースがあった。
- ただしこのプロジェクトは **Git連携で自動デプロイ**されているため、基本は「push → Vercel反映」で運用可能。

---

## 追加/変更したファイル一覧（lp-project）

- 追加
  - `docs/brief-kato.md`
  - `app/content/lp-copy-vol-01.ts`
- 変更
  - `app/components/sections/hero-section.tsx`
  - `app/components/sections/consultation-cta-section.tsx`

---

## ねらい（なぜこうしたか）

- LPの文章をコンポーネントに直書きし続けると、修正が発生するたびに「複数箇所を同時編集」→「不整合」→「コピペ地獄」になりやすい。
- そこで
  - **世界観の固定（ブリーフ）**
  - **コピーの単一ソース（型付きTS）**
  - **表示側は参照（データ駆動）**
 という構造にして、変更コストとミスを最小化した。

---

## 次にやること（おすすめ順）

1. **他セクション（仮骨格）も `lpCopyVol01` 参照に寄せる**
   - `philosophy/problem/commonSolution/trueCause/heaven/hell/proof` を「見た目最小変更」でデータ駆動へ
2. **`lpCopyVol01` を元に、セクション本文を確定（Claude/Grok連携）**
   - Claude：本文の整合（ストーリー）
   - Grok：見出し/CTA短文の大量案（variants）
3. **デザイン指示の単一ソースを追加（Gemini担当）**
   - 例：`app/content/lp-design-notes-vol-01.ts`（見出し階層、余白、色、装飾のガイド）

