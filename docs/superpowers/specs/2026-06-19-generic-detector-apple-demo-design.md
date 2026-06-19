# 設計: アプリ汎用化 +「りんご測定器」公開デモ

日付: 2026-06-19
状態: 承認済み

## 目的

`judge` アプリ = 現状 `ぎょぴちゃん` 専用ハードコード。汎用化 → 被写体名を学習時に変更可能。公開サイト(GitHub Pages)の同梱デモを著作権クリアな「りんご測定器」に差し替え。あわせて公開サイトで地図パネル非表示バグを修正。

## 背景・現状

- 公開リポジトリ追跡ファイル = `index.html` + `model/*` のみ。`train.html`/`training_data/` は `.gitignore`(著作権で非公開)。
- `index.html:504` 地図パネル表示条件 = `train.html` 存在(HEADフェッチ) OR localStorage `gyopi-map-data`。公開サイトは両方なし → パネル永久非表示。フォールバック `fetch('training_data/index.json')` も404。
- 被写体名 `ぎょぴちゃん` がi18n辞書(kid/adult)・ラベル(`ぎょぴっぽさ`/`ぎょぴ度`)・判定文・クラス名・localStorageキー・同梱モデルに全面ハードコード。
- ビルド環境: tfjs-nodeなし。playwrightのみ。学習はブラウザ駆動(`train.html` が CDN tfjs+mobilenet 読込)。既存 `verify_*.mjs` が同方式。

## 決定事項(承認済み)

1. 被写体名を全ラベルに反映(`りんごっぽさ` 等もテンプレ化)。
2. 旧ぎょぴモデルは公開から外し、りんごモデルに差し替え。
3. りんご学習画像 = Wikimedia Commons の PD/CC0 を取得(ライセンスをコードで絞る)。
4. 地図ノードサムネ = PD/CC0画像 → 著作権クリア。

## 設計

### 1. 被写体名パラメータ化

- `train.html`: 名前入力欄追加。学習時に設定。localStorage `subject-name` 保存。
- `index.html`: `subject-name` 読込。i18n辞書を `{name}` 差し込み式に変更。
  - 対象: タイトル `{name}判定器`、判定文 `{name}だ！`/`{name}じゃない`、ラベル `{name}っぽさ`/`{name}度`、2クラス名 `{name}`/`{name}じゃない`、その他 `ぎょぴ` 含む全文字列。
  - kid/adult 両辞書。
- フォールバック: `subject-name` なし → デフォルト「りんご」。
- localStorageキー (`gyopi-map-data`/`gyopi-avg-features`/`gyopi-classifier` 等) は内部キー → 改名不要(機能上)。ただし汎用化の整合で改名検討可(必須でない)。

### 2. りんごモデル新規同梱

ビルドスクリプト `build_apple_model.mjs`:
1. Wikimedia Commons API でりんご写真 + 非りんご写真 各12〜16枚取得。ライセンス = PD/CC0 のみに絞る。一時フォルダ保存。
2. playwright で `train.html` 駆動 → 画像投入 → 学習。
3. 出力を書き出し:
   - `model/` (model.json / weights.bin / avg.json) ← りんご版に差し替え
   - `map-data.json` (サムネ + 埋め込み + yes/no) ← 新規同梱
   - 被写体名 = 「りんご」

### 3. 公開地図修正

- `makeMap` フォールバック順: localStorage → 同梱 `map-data.json` → (training_data)。
- 地図パネル表示条件に「同梱 `map-data.json` フェッチ成功」を追加 → 公開サイトで常時表示。

### 4. 影響ファイル

- `index.html`: 辞書テンプレ化、makeMapフォールバック、名前読込。
- `train.html`: 名前入力・保存、クラス名汎用化。
- 新規: `build_apple_model.mjs`、`map-data.json`、りんご版 `model/*`。
- `.gitignore`: `map-data.json` は公開(追跡)。`training_data/`/`train.html` は引き続き非公開。

## テスト・検証

- 名前変更 → judge画面 全ラベル反映を確認(playwright)。
- 名前未設定 → 「りんご」デフォルト確認。
- 公開相当(train.html/training_data なし)で地図パネル表示 + ノード描画確認。
- りんごモデルがりんご画像を正判定確認。

## 非対象(YAGNI)

- マスコット絵文字 🐟 の変更(名前のみ汎用化)。
- 内部localStorageキーの一括改名(機能不要)。
- 複数被写体の同時保持・切替UI拡張。
