# Active Reader

AI 支援型アクティブリーディングツール（MVP）。
Markdown / テキストを読みながら「分からない」「自分で説明」「問題にする」マーカーを付け、AI に説明・評価・出題させる。

- 仕様: [docs/spec/active_reader_requirements_v0.1.md](docs/spec/active_reader_requirements_v0.1.md)
- 実装計画: [docs/plan/implementation_plan_v0.1.md](docs/plan/implementation_plan_v0.1.md)

## Claude Code スキルとして使う（勉強のお供）

[skills/reread/](skills/reread/SKILL.md) が 1 つのスキルになっている（CLI 本体の uv プロジェクトは `skills/reread/tools/` に同梱）。

```bash
ln -s "$(pwd)/skills/reread" ~/.claude/skills/reread
```

あとは Claude Code で「この資料を一緒に読みたい: book.md」「マーカーを付けたところを説明して」「問題を出して」「復習したい」などと話しかけると、Claude が UI を起動し、読書記録を CLI で読み書きしながらチューターをする。

## CLI を直接使う

```bash
AR="uv run -q --project skills/reread/tools active-reader"
$AR open book.md          # ブラウザで開く。起動中なら同じ UI の左サイドバーに追加
$AR export book.md -o book.json
$AR set book.md <ann_id> explanation < result.json   # 結果の書き戻し（スキル用）
$AR terms book.md         # 検出した専門用語と文脈（未解説のみ）
$AR set-terms book.md < glossary.json               # 用語の一言解説を書き込む（スキル用）
$AR wait                  # UI で用語がタップされるのを待ち、依頼を JSON で出力（スキルが常駐させる）
$AR list                  # 読書記録の一覧
$AR delete ~/books/rust   # ファイル or フォルダ配下（プロジェクト）の記録を削除
$AR clean                 # 元文書が無くなった記録を削除
```

Reader 画面: 左に文書一覧、中央に本文、右に詳細。本文を選択するとポップアップで「分からない（紫）/ 自分で説明（青）/ 問題にする（オレンジ）」を選び、そのままコメント（任意）。既存のマーカーの上を選び直して Esc で削除。Markdown の画像（文書からの相対パス）も表示する。専門用語は自動検出して最初の出現に青い破線。タップすると待ち受け中のスキル（Claude）が起こされ、UI には一言、チャットには詳しい説明が出る。マーカーも一覧の送信アイコン（1 件）や種類ごとの一斉送信アイコンで Claude に送れる。左右のパネルは ⌘B / ⌥⌘B で開閉、境界ドラッグで幅変更。アイコンは Tabler Icons（MIT）を同梱。
UI には AI 機能は無い。右パネルの「ID をコピー」をチャットに貼って、reread スキル（Claude）に説明・評価・出題を頼む。結果は UI に表示される。ヘッダの「記録管理」から削除・掃除もできる。

- 読書記録は `~/.active-reader/docs/<doc_id>.json`（`--data-dir` で変更）。元文書には書き込まない。
- doc_id は文書の絶対パス由来。文書を移動すると旧記録は「見つからない」扱いになる。
- ブラウザ: Chrome 105+ / Safari 17.2+ / Firefox 140+（CSS Custom Highlight API）

## 開発

```bash
task test   # = uv run --directory skills/reread/tools pytest -q
```

## セットアップ手順（Nix）
### 1. Nix をインストールする
```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```
> **Determinate Systems 版を推奨。** `nix-command` と `flakes` が最初から有効なため、追加設定不要。
