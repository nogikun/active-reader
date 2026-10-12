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

「この資料を一緒に読みたい: book.md」「マーカーを付けたところを説明して」「問題を出して」「復習したい」などと話しかけると、AI が UI を起動して「こちらを開いてください」とリンクを案内し（ブラウザは勝手に開かないので、アプリ内ブラウザやアプリの横など好きな場所で開ける）、読書記録を CLI で読み書きしながらチューターをする。

Claude Code は従来どおり、`wait` をバックグラウンドで動かして終了通知を受け取る。Claude 用の hooks・設定・CLI の動作は変更しない。

ChatGPT／Codex のローカル環境では、初回に以下を実行して自動待ち受け用の hooks を登録する。

```bash
uv run -q --project skills/reread/tools active-reader install-hooks
```

既存の `~/.codex/hooks.json`（`CODEX_HOME` 設定時はその配下）に追加し、既存 hooks は保持する。変更前のファイルは `hooks.json.reread-backup` に保存する。Claude の設定には触れない。`/hooks` で追加した定義を確認・信頼した後、新しい会話でこのリポジトリの `reread` スキルを使う。古い配布版プラグインを使っている場合は、スキル本体も更新する。

スキルが読書を開始すると `listen --port <UI のポート>` でその会話だけを有効化する。同期 `Stop` hook が UI の依頼を待ち、用語・マーカーが届けば会話を自動継続する（[公式の hooks 仕様](https://learn.chatgpt.com/docs/hooks)）。依頼が無い場合も 50 秒ごとに継続するため、読書中は会話が稼働し、モデル利用が発生する。UI の「読書をやめる」は未保存のコメントを保存して、その UI のサーバーと AI の待ち受けを終了する。`listen --stop`、チャットの停止ボタン、会話の終了、UI の切断でも hook の待ち受けを解除する。hooks はローカル実行が必要で、cloud-orchestrated ChatGPT Work の plugin hooks は未対応（[公式の対応範囲](https://learn.chatgpt.com/docs/plugins)）。

hooks が未起動なら従来の `wait --timeout 50` で会話内の待ち受けを続ける。待ち受けを終えるときは「読書を終える」と伝える。停止後に UI で送った依頼は、チャットで「送った依頼に対応して」と伝えるか、参照をコピーして貼る。依頼キューはサーバーの再起動で消えるので、その場合は再送する。

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
$AR install-hooks        # ChatGPT／Codex 用 hooks を既存の設定に追加
$AR listen --port 8765   # この会話だけ自動待ち受け（通常はスキルが実行）
$AR listen --stop        # 自動待ち受けを終了
$AR list                  # 読書記録の一覧
$AR delete ~/books/rust   # ファイル or フォルダ配下（プロジェクト）の記録を削除
$AR clean                 # 元文書が無くなった記録を削除
```

Reader 画面: 左に文書一覧、中央に本文、その横に目次（見出しが 2 つ以上あり、画面幅 1200px 以上のとき。読んでいる節をハイライト。狭いときはヘッダ右の目次ボタンで小窓として開く）、右に詳細。本文を選択するとポップアップで「分からない（紫）/ 自分で説明（青）/ 問題にする（オレンジ）」を選び、そのままコメント（任意）。既存のマーカーの上を選び直して Esc で削除。Markdown の画像（文書からの相対パス）も表示する。専門用語は自動検出して最初の出現に青い破線（右の「用語」タブには読んでいる節で初めて出てくる用語だけを表示）。タップすると待ち受け中のスキルが依頼を受け取り、UI には一言、チャットには詳しい説明が出る。マーカーも一覧の送信アイコン（1 件）や種類ごとの一斉送信アイコンで AI に送れる。左右のパネルは ⌘B / ⌥⌘B で開閉、境界ドラッグで幅変更。アイコンは Tabler Icons（MIT）を同梱。
UI には AI 機能は無い。右パネルの「ID をコピー」をチャットに貼って、reread スキルに説明・評価・出題を頼む。結果は UI に表示される。ヘッダの「記録管理」から削除・掃除もできる。

- 読書記録は `~/.active-reader/docs/<doc_id>.json`（`--data-dir` で変更）。元文書には書き込まない。
- doc_id は文書の絶対パス由来。文書を移動すると旧記録は「見つからない」扱いになる。
- ブラウザ: Chrome 109+ / Safari 17.2+ / Firefox 140+（MathML と CSS Custom Highlight API）

Markdown の数式は `$E = mc^2$`（行内）と `$$\frac{1}{2}$$`（独立した数式、複数行も可）で書く。
TeX を MathML に変換してブラウザで描画するため、実行時に外部 CDN への接続は不要。
分数、総和、添字、行列、`aligned` に対応する。コード内の `$` やエスケープした `\$` は数式に変換しない。
数式の `$` と式の間には空白を入れない。文章と同じ行にある `$$...$$` は行内に表示する。
独立した行にある `$$...$$` は別行に表示し、前後に空行がなくても表示できる。
表の数式・コード内にある `|` も、列区切りとして分割せずに表示する。
『プログラマーのための圏論』の [Markdown 版](https://github.com/ktgw0316/milewski-ctfp-markdown) で使われる
`\cat`、`\Set`、`\Rel`、`\Cat`、`\Hask`、`\id`、`\idarrow[A]`、`\Lim`、`\Ran`、`\Lan`、`\Fop`、`\Nat` の略記にも対応する。
変換エラーがある式は元の TeX を表示する。任意の LaTeX パッケージの読み込みには対応しない。
数式を含む文書で保存済みのマーカーは前後の文脈で位置を復元するが、TeX 自体を選択していたマーカーは再選択が必要になる場合がある。

## 開発

```bash
task test   # = uv run --directory skills/reread/tools pytest -q tests ../evals
task share-evals  # 評価ログを匿名化し、共有用の dist/eval-logs.zip を作成
```

## セットアップ手順（Nix）
### 1. Nix をインストールする
```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
```
> **Determinate Systems 版を推奨。** `nix-command` と `flakes` が最初から有効なため、追加設定不要。

## ライセンス

本リポジトリの独自コード・ドキュメント・reread スキルは [MIT License](LICENSE) で公開する。
著作権者は `nogi (nogikun)`。利用・改変・商用利用・再配布ができ、コピーや重要な部分の再配布には著作権表示とライセンス全文を残す。
Python 配布パッケージにも同じ [LICENSE](skills/reread/tools/LICENSE) を含める。
ユーザーが読み込む書籍・記事・論文などの資料には、その資料の権利・ライセンスが適用される。

第三者のコード・データは、それぞれのライセンスに従う。実行時の依存と同梱素材は次のとおり（バージョンは `uv.lock` と同梱アイコンの表記に基づく）。

| コンポーネント | バージョン | ライセンス・表記 |
| --- | --- | --- |
| Janome | 0.5.0 | [Apache-2.0](https://github.com/mocobeta/janome/blob/0.5.0/LICENSE.txt)。同梱の MeCab-IPADIC 辞書には [NOTICE.txt](https://github.com/mocobeta/janome/blob/0.5.0/NOTICE.txt) の条件も適用 |
| markdown-it-py | 4.2.0 | [MIT](https://github.com/executablebooks/markdown-it-py/blob/v4.2.0/LICENSE)。移植元の [著作権表示](https://github.com/executablebooks/markdown-it-py/blob/v4.2.0/LICENSE.markdown-it) も保持 |
| mdurl（間接依存） | 0.1.2 | [MIT](https://github.com/executablebooks/mdurl/blob/0.1.2/LICENSE) |
| mdit-py-plugins | 0.6.1 | [MIT](https://github.com/executablebooks/mdit-py-plugins/blob/v0.6.1/LICENSE) |
| latex2mathml | 3.81.1 | [MIT](https://github.com/roniemartinez/latex2mathml/blob/3.81.1/LICENSE) |
| Tabler Icons（SVG を同梱） | 3.49.0 | [MIT](https://github.com/tabler/tabler-icons/blob/v3.49.0/LICENSE)。著作権表示と全文を [app.js](skills/reread/tools/src/active_reader/static/app.js) に保持 |

Janome を依存として利用しても、本リポジトリを Apache-2.0 に変更する必要はない（[Apache-2.0 第4条](https://www.apache.org/licenses/LICENSE-2.0)）。
依存パッケージや辞書もまとめて再配布する場合は、各配布物の著作権表示・ライセンス・NOTICE を保持する。
開発用依存には MIT、BSD、PSF-2.0、Apache-2.0 / BSD の選択式ライセンスが含まれ、それらも本リポジトリの MIT とは別に適用される。
