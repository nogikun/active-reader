# 実装計画 v0.1 — Active Reader MVP（現行実装）

この文書は 2026-10-11 時点の実装構成と動作をまとめる。
[要件定義 v0.1](../spec/active_reader_requirements_v0.1.md) は当初の構想・将来候補を含む。現在の使い方は [README](../../README.md)、AI の読み方・書き戻し方は [reread スキル](../../skills/reread/SKILL.md) を参照する。

## 1. 基本方針

- Markdown・テキストの閲覧、マーカー、メモ、用語集と読書記録をローカルで扱う。
- Python 標準ライブラリとブラウザ標準機能を中心にする。実行時の依存は Markdown 描画用の `markdown-it-py` と日本語解析用の `janome`。
- AI の説明・評価・出題・採点はチャット上のスキルが行い、CLI で結果を書き戻す。UI サーバーは AI プロバイダーを呼び出さない。
- 保存は文書ごとの JSON ファイル。元文書には書き込まない。

## 2. 構成

```text
skills/reread/
  SKILL.md                         # AI チューターの手順
  evals/                           # スキルの評価シナリオとサンプル資料
  tools/
    pyproject.toml                 # uv プロジェクト / active-reader コマンド
    src/active_reader/
      cli.py                       # 起動・記録操作・AI 依頼の取得
      server.py                    # ローカル HTTP API と依頼キュー
      store.py                     # JSON 保存、一覧、削除、用語集
      nlp.py                       # 文節境界と専門用語の検出
      static/
        index.html
        app.js                     # 文書一覧・本文・目次・マーカー・用語
        app.css
    tests/test_core.py
```

## 3. データと保存

既定の保存先は `~/.active-reader/docs/<doc_id>.json`。`--data-dir` で変更できる。

- `document`: タイトル、元文書の絶対パス、形式、本文ハッシュ。
- `annotations`: 選択文字列、前後32文字、見出しと本文上の位置、種類、メモ、状態、日時、AI の結果。
- `glossary`: 用語ごとの一言解説と更新日時。
- `ai`: 説明（`explanation`）、評価（`evaluation`）、問題・回答・採点（`questions`）。

`doc_id` は元文書の絶対パスの SHA-256 先頭12桁から作る。文書を移動すると別の ID になり、旧記録は「元文書が見つからない」扱いになる。
マーカーの offset は、Markdown を描画した本文の `textContent` 上の位置。本文が変わった場合は、選択文字列と前後文脈で位置を探し直す。見つからなければ位置不明と表示し、記録は残す。

保存・削除はスレッド間とプロセス間のロックで保護する。書き込みは一時ファイルから `os.replace` で置き換える。
UI からの更新はユーザー入力フィールドだけを受け付け、AI 結果は `set` / `set-terms` で書き戻す。

## 4. HTTP API

| Method | Path | 内容 |
| --- | --- | --- |
| GET | `/`、`/static/{name}` | 許可した HTML・JS・CSS のみ配信 |
| GET | `/api/docs` | 登録文書と読書記録の一覧 |
| POST | `/api/docs` | 絶対パスで指定した文書を登録 |
| GET | `/api/docs/{doc_id}` | 文書情報・本文 HTML・マーカー・文節境界・用語・用語集 |
| GET | `/api/docs/{doc_id}/file?p={relative_path}` | 文書から相対指定した画像 |
| GET | `/api/docs/{doc_id}/annotations` | マーカーの更新確認 |
| GET | `/api/docs/{doc_id}/glossary` | 用語集の更新確認 |
| PUT / DELETE | `/api/docs/{doc_id}/annotations/{ann_id}` | マーカーの更新・削除 |
| POST | `/api/docs/{doc_id}/ask` | 用語またはマーカーの AI 依頼をキューに追加 |
| POST | `/api/requests/next` | AI 依頼を1件取得。待機時間は最大120秒 |
| POST | `/api/records/delete` | 指定 ID またはフォルダ配下の読書記録を削除 |
| POST | `/api/records/clean` | 元文書が見つからない読書記録と古い一時ファイルを掃除 |

サーバーは `127.0.0.1` にのみ接続を待ち受け、`Host` は `127.0.0.1` / `localhost` のみ許可する。
書き込み要求は `Content-Type: application/json` が必須。
Markdown の生 HTML は無効にし、文書側の画像を直接開く応答には sandbox を設定する。

## 5. Reader UI

- 左に文書一覧、中央に本文と目次、右にマーカー・用語の詳細。
- 本文を選択すると種別のメニューを表示。`u` / `e` / `q` キーでも「分からない」「自分で説明」「問題にする」を付与できる。メモは500ms待って保存する。
- ハイライトには CSS Custom Highlight API を使う。選択位置は文節境界に合わせ、マーカーの位置は offset と前後文脈から復元する。
- 専門用語の初出に破線を表示。用語またはマーカーの送信操作で AI 依頼を追加し、結果が書き戻されるまで更新を確認する。
- 目次は見出しが2つ以上あるときに表示。幅の狭い画面ではヘッダの目次ボタンで開く。
- 左右のパネルは開閉と幅の変更ができる。スクロール位置とレイアウトは `localStorage` に保存する。
- アイコンは Tabler Icons の SVG を同梱し、著作権表示と MIT ライセンス全文を保持する。

## 6. CLI と AI スキル

CLI は `uv run --project skills/reread/tools active-reader <command>` で実行する。

| コマンド | 用途 |
| --- | --- |
| `open FILE [--no-browser] [--port PORT]` | 文書を開く。起動中なら同じサーバーに追加 |
| `export FILE [-o FILE]` | 読書記録を JSON で出力 |
| `set FILE ANN_ID KEY` | stdin の説明・評価・問題・メモ・状態を書き戻す |
| `terms FILE [--all]` | 検出した専門用語と文脈を出力 |
| `set-terms FILE` | stdin の用語解説を書き戻す |
| `wait [--port PORT] [--timeout SECONDS]` | UI の AI 依頼を1件取得。時間切れは `{}` |
| `list` | 読書記録一覧 |
| `delete PATH... [-y]` | ファイルまたはフォルダ配下の読書記録を削除 |
| `clean [-y]` | 元文書が見つからない記録を掃除 |

全コマンドに `--data-dir` を指定できる。`open` の既定ポートは8765。新規起動時に使用中なら空きポートを選び、出力する URL で案内する。
スキルは `open --no-browser` で起動し、ユーザーが開けるリンクを案内する。
Claude Code はバックグラウンドで `wait`、ChatGPT／Codex は会話内で `wait --timeout 50` の結果を取得する。依頼を説明・評価・出題し、CLI で書き戻して再び待つ。
会話終了後に残った依頼は `wait --timeout 0` で取得できる。依頼キューはメモリ上にあり、サーバー再起動時に消える。

## 7. 記録管理と共有

CLI の `delete` / `clean` は対象を表示し、確認後に削除する。非対話実行では `-y` がなければ中止する。
UI の記録管理では `dry_run` で対象を確認してから削除する。削除対象は読書記録であり、元文書は保持する。
文書を移動した場合も旧記録は掃除対象になるため、必要な記録は先にエクスポートする。

個人の読書記録や評価の作業ファイルは Git 管理対象外。評価ログの共有時は [評価用の手順](../../skills/reread/evals/README.md) に従い、個人の絶対パスを匿名化する。

## 8. 検証

`task test` または `uv run --project skills/reread/tools pytest -q skills/reread/tools/tests skills/reread/evals` を実行する。
既存テストは記録更新・削除、Host と Content-Type の検証、文書の追加・画像表示、用語検出・用語集、依頼キューと待機時間を確認する。
スキルの説明・評価・出題の振る舞いは `evals/evals.json` のシナリオで確認する。
目次、選択メニュー、ハイライト、パネル操作は対応ブラウザで確認する。

## 9. 今後の対象

PDF / EPUB、専用の Review 画面、復習スケジューリング、読書時間の分析、全文検索、SQLite、MCP、文書移動時の記録の付け替えは現行実装に含めない。
