# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

チャトレ事業（関ケ原・船田店）の月次収支管理 PWA。単一 HTML ファイル (`index.html`) にすべての UI・ロジック・スタイルをインライン化。

## 開発・動作確認

ビルド不要。ブラウザで直接開くか静的サーバで配信する:

```bash
python -m http.server 8080
# → http://localhost:8080/index.html
```

テストフレームワークなし。修正後はブラウザで動作確認が唯一の検証手段。

## アーキテクチャ

### データフロー

```
state (JS object)
  └─ setMd(k, md) → scheduleSave() → Supabase REST POST (upsert)
  └─ getMd(k)     ← dbLoad()       ← Supabase REST GET
```

- `state` はグローバル変数。`state[月キー]` = `{point, daisukeP, scout, staff, kyuryo, expenses[]}` の形式。
- Supabase テーブル: `budget`（`id="main"`, `data=JSON`）の 1 行だけ。全月分を 1 JSON に格納。
- 保存はデバウンス 800ms (`scheduleSave`)。失敗時は 4 秒後に自動リトライ。

### 月キー

`MKEYS = ["04","05",...,"03"]`（4月始まり会計年度）。`tab` 変数が現在表示中の月キーを保持。

### 収支計算 (`calcAll`)

```js
uriage = point × 0.57 / 1.05 × 1.1      // 売上収入（自動）
girl   = (point - daisukeP) × 0.3        // 女の子給料（控除）
dSell  = daisukeP × 0.57/1.05 × 1.1 × 0.98  // 大輔売上（控除）
incT   = uriage - girl - dSell - scout - staff - kyuryo
profit = incT - expT  // 関プ経費は除く
```

### 区分 (kubun)

- 通常モード: `["会社","関ケ","船田"]`
- 関プモード (`?p=kanpu2026` または `#kanpu2026`): `["会社","関ケ","関プ","船田"]` が追加される。関プ経費は利益計算に含まない独立セクション。

### 領収書アップロード

Supabase Storage `receipts/` バケットに直接 PUT。ファイル名は `timestamp_random.ext`。GAS_URL が設定されていれば Google Drive にも並行バックアップ（失敗しても無視）。

## 重要な定数・変数

| 変数 | 説明 |
|---|---|
| `SB_URL` | Supabase プロジェクト URL |
| `SB_KEY` | anon キー（RLS 前提でハードコード）。service-role キーは絶対に書かない |
| `GAS_URL` | Google Drive バックアップ用 GAS エンドポイント。未設定なら `"YOUR_GAS_URL_HERE"` のまま |
| `KANPU_KEY` | 関プモードのパスワード文字列 (`"kanpu2026"`) |
| `expF` | 経費入力フォームの一時状態 |
| `editTab`, `editId`, `editRecUrl`, `editTatekae` | 編集モーダルの一時状態 |

## レンダリング方針

`renderMonth()` と `renderAnnual()` は毎回 `innerHTML` を全書き換えする。部分更新ではなく全再描画。`rc()` だけは KPI 数値の高速インプレース更新に使用。

## CSS クラス命名規則

2〜3 文字の短縮形（`.hdr`, `.ch`, `.kpi`, `.fw`, `.tr`, `.mu` など）。HTML サイズ削減のための意図的な設計なので、リファクタで長い名前に変えない。
