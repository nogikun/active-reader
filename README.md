# Active Reader

AI 支援型アクティブリーディングツール（MVP）。
Markdown / テキストを読みながら「分からない」「自分で説明」「問題にする」マーカーを付け、AI に説明・評価・出題させる。

- 仕様: [docs/spec/active_reader_requirements_v0.1.md](docs/spec/active_reader_requirements_v0.1.md)
- 実装計画: [docs/plan/implementation_plan_v0.1.md](docs/plan/implementation_plan_v0.1.md)

## Claude Code / ChatGPT・Codex スキルとして使う（勉強のお供）

[skills/reread/](skills/reread/SKILL.md) が 1 つのスキルになっている（CLI 本体の uv プロジェクトは `skills/reread/tools/` に同梱）。

```bash
mkdir -p ~/.claude/skills
ln -s "$(pwd)/skills/reread" ~/.claude/skills/reread
```

ChatGPT／Codex（ローカルのファイル・コマンドを実行できる環境）にも同じスキルを登録する。Claude Code の登録はそのまま使える。

```bash
mkdir -p ~/.agents/skills
ln -s "$(pwd)/skills/reread" ~/.agents/skills/reread
```

このリポジトリには `.agents/skills/reread` から同じスキルへのリンクもあり、ここで起動した Codex から利用できる。見つからない場合はアプリ／会話を再起動する。ChatGPT はスキルを選択、Codex は `$reread` で明示的に呼び出すこともできる（[公式のスキル案内](https://learn.chatgpt.com/docs/build-skills)）。通常の Web チャットなど、ローカル実行できない環境は今回の対象外。

「この資料を一緒に読みたい: book.md」「マーカーを付けたところを説明して」「問題を出して」「復習したい」などと話しかけると、AI が UI を起動し、読書記録を CLI で読み書きしながらチューターをする。

Claude Code は従来どおり、`wait` をバックグラウンドで動かして終了通知を受け取る。ChatGPT／Codex は `wait --timeout 50` の結果を実行中の会話で待ち、UI から届いた依頼に対応する。Codex のバックグラウンド hook は完了だけでは新しい会話のターンを開始しないため、この待ち受け方式を使う（[公式の hooks 仕様](https://learn.chatgpt.com/docs/hooks)）。会話を終了した後は、UI で送信してからチャットで「送った依頼に対応して」と伝えるか、参照をコピーして貼る。待ち受けを終えるときは「読書を終える」と伝える。依頼キューはサーバーの再起動で消えるので、その場合は再送する。

読書記録と依頼の JSON は AI 共通。Claude Code と ChatGPT／Codex で同じ UI を利用できるが、同じサーバーで待ち受ける AI は 1 つにする。Kiro／Copilot 固有の連携は今後の対応範囲。

## CLI を直接使う

```bash
AR="uv run -q --project skills/reread/tools active-reader"
$AR open book.md          # ブラウザで開く。起動中なら同じ UI の左サイドバーに追加
$AR export book.md -o book.json
$AR set book.md <ann_id> explanation < result.json   # 結果の書き戻し（スキル用）
$AR terms book.md         # 検出した専門用語と文脈（未解説のみ）
$AR set-terms book.md < glossary.json               # 用語の一言解説を書き込む（スキル用）
$AR wait                  # UI で用語がタップされるのを待ち、依頼を JSON で出力（スキルが常駐させる）
$AR wait --timeout 50     # 最大 50 秒待つ。時間切れは {}（ChatGPT／Codex 用）
$AR wait --timeout 0      # キューの次の依頼を即時取得。空なら {}
$AR list                  # 読書記録の一覧
$AR delete ~/books/rust   # ファイル or フォルダ配下（プロジェクト）の記録を削除
$AR clean                 # 元文書が無くなった記録を削除
```

Reader 画面: 左に文書一覧、中央に本文、その横に目次（見出しが 2 つ以上あり、画面幅 1200px 以上のとき。読んでいる節をハイライト。狭いときはヘッダ右の目次ボタンで小窓として開く）、右に詳細。本文を選択するとポップアップで「分からない（紫）/ 自分で説明（青）/ 問題にする（オレンジ）」を選び、そのままコメント（任意）。既存のマーカーの上を選び直して Esc で削除。Markdown の画像（文書からの相対パス）も表示する。専門用語は自動検出して最初の出現に青い破線（右の「用語」タブには読んでいる節で初めて出てくる用語だけを表示）。タップすると待ち受け中のスキルが依頼を受け取り、UI には一言、チャットには詳しい説明が出る。マーカーも一覧の送信アイコン（1 件）や種類ごとの一斉送信アイコンで AI に送れる。左右のパネルは ⌘B / ⌥⌘B で開閉、境界ドラッグで幅変更。アイコンは Tabler Icons（MIT）を同梱。
UI には AI 機能は無い。右パネルの「ID をコピー」をチャットに貼って、reread スキルに説明・評価・出題を頼む。結果は UI に表示される。ヘッダの「記録管理」から削除・掃除もできる。

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
