# 実装計画 v0.1 — Active Reader MVP

- 対象: [要件定義 v0.1](../spec/active_reader_requirements_v0.1.md) の §15 MVP / §16 受け入れ条件 1〜10
- 方針: 書かずに済むものは書かない。stdlib・ブラウザ標準機能を優先し、依存は 1 個だけにする。

---

## 1. 未決定事項への回答（§18）

| # | 決定 | 理由 |
|---|---|---|
| Q1 PDF | **入れない**（Markdown + txt のみ） | 推奨どおり。 |
| Q2 Quiz色 | **オレンジ** | 紫・青と見分けやすい。CSS変数1行で変更できる。 |
| Q3 外部検索 | **MVPでは無し** | §15 で MVP外。原文のみで評価。 |
| Q4 AIプロバイダー | **外部コマンド方式**: `ACTIVE_READER_AI_CMD`（既定 `claude -p`）に stdin でプロンプトを渡し、stdout を受け取る | Adapterクラスを作らずにベンダー非依存（NFR-06）を満たせる。`llm` や `ollama run` 等にも差し替え可。 |
| Q5 保存 | **JSONファイルを正本**（1文書1ファイル）。SQLiteは使わない | §13 で MVP は JSON 可。正本がそのまま JSON なのでエクスポート（FR-08）の実装がほぼ不要になる。 |
| Q6 復習 | **status フィールドだけ持つ**（スケジューリング無し） | §15 で MVP外。 |
| Q7 Skill実行形態 | **JSONファイルが API**。Web UI と CLI から同じ `ai.py` を呼ぶ | Claude Code などの Skill はエクスポートされた JSON を直接読み書きできる。MCP は後回し。 |

---

## 2. 構成

```text
skills/reread/tools/pyproject.toml  # 依存: markdown-it-py のみ / script: active-reader
skills/reread/tools/src/active_reader/   # uv プロジェクトはスキル内に同梱
  cli.py      # argparse: open / export / explain
  server.py   # http.server (ThreadingHTTPServer), 127.0.0.1 限定
  store.py    # JSON の読み書き（ロック + アトミック書き込み）
  ai.py       # プロンプト4種 + サブプロセス呼び出し + JSON パース
  static/
    index.html
    app.js    # 選択・アンカー・ハイライト・サイドパネル
    app.css   # 種別色は CSS 変数
skills/reread/tools/tests/test_core.py
```

### 採用しないもの（理由）

| 不採用 | 代わりに | 後で足す条件 |
|---|---|---|
| FastAPI / uvicorn | `http.server` | エンドポイントが 10 個を超えたら |
| SQLite | 1文書1JSON | 複数文書横断検索・履歴が必要になったら |
| フロントのビルド環境 / フレームワーク | 素の JS 1 ファイル | 画面が Reader + Review 以上に増えたら |
| `<mark>` で DOM を書き換えるハイライト | **CSS Custom Highlight API**（`CSS.highlights`） | — （DOM を壊さないので offset が安定する） |
| AI Adapter クラス階層 | 外部コマンド 1 本 | SDK 固有機能（ストリーミング等）が必要になったら |
| config.json / 設定コマンド | CSS 変数・環境変数・CLI 引数 | 設定項目が増えたら |

---

## 3. データ

保存先: `~/.active-reader/docs/<doc_id>.json`（`--data-dir` で変更可）。
`doc_id` = 文書の絶対パスの sha256 先頭12桁。`content_hash` は別に持ち、文書が編集されたかの判定に使う。

§9 のスキーマをベースに、MVP で使わない `review` / `sources` を省き、Quiz 用の `questions` を追加:

```json
{
  "schema_version": "1.0",
  "document": { "id": "doc_ab12cd34ef56", "title": "Example Book",
                "source_path": "/path/to/book.md", "source_type": "markdown",
                "content_hash": "sha256:..." },
  "annotations": [{
    "id": "ann_<uuid4>",
    "type": "unknown | explain | quiz",
    "selected_text": "Reciprocal Rank Fusion",
    "context_before": "...(32文字)", "context_after": "...(32文字)",
    "locator": { "heading": "3.2 Hybrid Search", "start_offset": 1024, "end_offset": 1046 },
    "note": "Unknown=何が分からないか / Explain=自己説明 / Quiz=出題観点",
    "status": "new | learning | understood | review_required | mastered",
    "created_at": "...", "updated_at": "...",
    "ai": {
      "explanation": null,
      "evaluation": null,
      "questions": [{ "question": "...", "answer": null, "grading": null }]
    }
  }]
}
```

- offset は**レンダリング後の本文 `textContent` 上の位置**。
- `ai` はサーバーだけが書く。クライアントはユーザー入力フィールドだけを送る（競合防止）。

---

## 4. API（server.py）

| Method | Path | 内容 |
|---|---|---|
| GET | `/` , `/static/*` | ホワイトリストの静的ファイルのみ |
| GET | `/api/doc` | `{document, html, annotations}`。HTML は markdown-it-py（`html: False`）でレンダリング、txt は `<pre>` |
| PUT | `/api/annotations/{id}` | upsert。`ai` 以外のフィールドをマージ |
| DELETE | `/api/annotations/{id}` | 削除 |
| POST | `/api/annotations/{id}/ai/{op}` | `op` = `explain` / `evaluate` / `quiz` / `grade`。結果を保存し、更新後の annotation を返す |
| GET | `/api/records` | 全記録の一覧（§8 参照） |
| POST | `/api/records/delete` | `{"doc_ids": [...]}` の記録を削除 |
| POST | `/api/records/clean` | 元文書が無い記録 + `.tmp` 残骸を削除 |

守ること（手を抜かない部分）:
- `127.0.0.1` にのみ bind、`Host` ヘッダが `127.0.0.1` / `localhost` 以外は 403（DNS rebinding 対策）。
- 書き込み系は `Content-Type: application/json` 必須（他サイトからの CSRF でAI送信＝外部送信を起こさせない。NFR-03）。
- 保存は `threading.Lock` + 一時ファイル → `os.replace`（書き込み途中のクラッシュでデータを失わない）。
  `# ponytail: グローバルロック。複数文書同時編集が必要ならファイル単位に`
- AI 呼び出し中はロックを持たない（AI 実行 → ロック取得 → load・更新・save）。

---

## 5. フロントエンド（app.js）

1. `/api/doc` を取得し `#reader` に HTML を挿入。
2. **選択 → 種別付与（FR-03〜05）**: 本文で選択中に `u` / `e` / `q` キー → Unknown / Explain / Quiz。
   選択範囲から offset・前後32文字・直前の見出し（h1〜h6）を算出して PUT。
   フローティングツールバーは省略（キー1つで足りる）。
3. **アンカー復元（NFR-04）**: `textContent` 上で
   ① offset 位置の文字列が一致 → そのまま
   ② 不一致 → `selected_text` の全出現を探し、前後文脈の一致度が高く元 offset に近いものを選ぶ
   ③ 見つからない → サイドパネルで「位置不明」と表示（データは消さない）。
   offset → `Range` は TreeWalker でテキストノードを走査して変換。
4. **ハイライト**: 種別ごとに `new Highlight(...ranges)` を `CSS.highlights` に登録、`::highlight(unknown)` 等で色付け。
   クリック判定は `document.caretPositionFromPoint`（Safari は `caretRangeFromPoint`）→ `range.isPointInRange`。
5. **サイドパネル（§8.2）**: 種別ごとに切り替え。
   - Unknown: 選択文 / メモ / 「AIに説明させる」/ 説明 / 「理解済みにする」(status=understood)
   - Explain: 選択文 / 自己説明 / 「評価する」/ 評価。**評価前は AI 出力を一切出さない**（FR-10 Recall）
   - Quiz: 選択文 / 出題観点 / 「問題を生成」/ 問題ごとに回答欄と「採点」
   - 下部に全アノテーション一覧。クリックで該当箇所へスクロール（FR-17）。
   - メモ入力は 500ms debounce で PUT（NFR-07）。
6. **スクロール位置保持（FR-02）**: `localStorage[doc_id]`（try/catch で囲む）。

---

## 6. AI（ai.py）

```python
def run(prompt: str) -> dict:          # ACTIVE_READER_AI_CMD に stdin で渡す
def explain(doc, ann) -> dict           # FR-09: summary / prerequisites / meaning_in_context / examples / related
def evaluate(doc, ann) -> dict          # FR-11: correct / missing / wrong / vague / improved, score + score_reason
def generate_quiz(doc, ann) -> list     # FR-13: 自由記述優先で 1〜3 問
def grade(doc, ann, q) -> dict          # FR-14: score / feedback / model_answer
```

- プロンプトは各関数内の f-string。入力は §10 のとおり（文書タイトル・見出し・選択文・前後文脈・メモ・回答）。
- 前後文脈は保存済みの32文字ではなく、AI 呼び出し時に原文から**前後 ~800文字**を取り直して渡す。
- 出力は「JSON のみ返せ」と指示し `json.loads`。コードフェンスは剥がす。失敗したら `{"raw": 出力全文}` を保存（UI はそのまま Markdown 表示）。
- AI は**ボタン押下 / CLI 明示実行時のみ**呼ぶ（NFR-03）。

---

## 7. CLI（cli.py）

```bash
uv run active-reader open book.md            # 127.0.0.1:8765 で起動しブラウザを開く。使用中ならランダムポート（NFR-01）
uv run active-reader export book.md [-o out.json]   # 正本 JSON をそのまま出力（FR-08）
uv run active-reader explain book.md         # 説明未生成の Unknown をまとめて AI 処理（Phase C）
uv run active-reader list                    # 記録一覧（§8）
uv run active-reader delete PATH... [-y]     # 文書 or プロジェクト単位で記録を削除（§8）
uv run active-reader clean [-y]              # 元文書が無い記録・残骸を削除（§8）
```

共通オプション: `--data-dir`。`serve` / `quiz` / `review` / 設定系は MVP では作らない。

---

## 8. 記録の管理（一覧・削除・クリーン）

`~/.active-reader/docs/` に置くのは**読書記録 JSON だけ**。元文書には一切触れない（読むだけ・消さない）。

### 8.1 操作の定義

| 操作 | 対象 | 判定方法 |
|---|---|---|
| **list** | 全記録 | 各 JSON の `document` + アノテーション数 + `updated_at` の最大値 + 元文書の有無（`missing`） |
| **delete PATH** | PATH がファイル → その文書の記録 / PATH がディレクトリ → **配下すべての文書の記録（＝プロジェクト単位）** | 保存済み `source_path` が `PATH` と一致、または `Path.is_relative_to(PATH)`。PATH は `resolve()` して比較する。ディスク上に PATH が既に無くても文字列で判定できる |
| **clean** | 元文書が存在しない記録 + 残った `*.json.tmp` | `not Path(source_path).exists()` |

「プロジェクト」専用の概念・設定は作らない。**ディレクトリ = プロジェクト**として扱えば、`delete ~/books/rust-book` でそのプロジェクトの記録がまとめて消える。

### 8.2 store.py に足す関数

```python
def list_records(data_dir) -> list[dict]          # {doc_id, title, source_path, annotations, updated_at, missing}
def match(records, path) -> list[str]              # delete PATH の対象 doc_id
def delete_records(data_dir, doc_ids) -> list[str] # 削除した doc_id を返す
def clean(data_dir) -> list[str]                   # missing の記録 + *.tmp を削除
```

CLI と API は同じ関数を呼ぶだけ。

### 8.3 CLI

- `delete` / `clean` は**まず対象（タイトル・パス・アノテーション数）を表示し `y/N` で確認**。`-y` で確認スキップ。対象 0 件なら何もしない。
- この確認表示がそのまま dry-run を兼ねるので `--dry-run` は作らない。

```text
$ active-reader delete ~/books/rust-book
以下 3 件の読書記録を削除します（元文書は削除しません）:
  ch01.md   12 annotations  ~/books/rust-book/ch01.md
  ch02.md    4 annotations  ~/books/rust-book/ch02.md
  ch03.md    0 annotations  ~/books/rust-book/ch03.md (missing)
よろしいですか? [y/N]
```

### 8.4 UI

ヘッダに「記録管理」ボタン → ネイティブ `<dialog>` を開く（別ページは作らない）。

- `/api/records` の一覧をテーブル表示（タイトル / パス / 件数 / 最終更新 / `missing` は灰色）。
- 各行に「削除」、テーブル上部に **ディレクトリ入力欄 +「このフォルダ配下を削除」**（= CLI の `delete DIR`）と「見つからない文書を掃除」（= `clean`）。
- 実行前に `confirm()` で件数とパスを表示。
- 今開いている文書の記録を消した場合は `location.reload()` して空の状態に戻す。

### 8.5 安全策（ここは省略しない）

- API の `doc_ids` は `^doc_[0-9a-f]{12}$` で検証し、`data_dir/docs/` 直下のファイルしか消さない（パストラバーサル防止）。
- 削除・クリーンも保存と同じ `threading.Lock` 内で行う（AI 結果の書き込みと競合しない）。
- 書き込み系 API と同じく `Content-Type: application/json` 必須 + Host 検証（他サイトからの削除要求を通さない）。
- ゴミ箱は作らない。確認ダイアログで十分とし、残したい記録は先に `export` してもらう。
  `# ponytail: ハードデリート。誤削除の復元要望が出たら docs/trash/ への移動に変える`

### 8.6 注意

doc_id はパス由来なので、**文書を移動すると旧記録は `missing` 扱いになり `clean` の対象になる**。確認表示にパスを出すのはこのため。移動に追従させたくなったら、`content_hash` が一致する記録を新パスに付け替える `relink` を足す（MVP 外）。

---

## 9. 実装順序（マイルストーン = 受け入れ条件）

| # | 内容 | 満たす受け入れ条件 | 目安 |
|---|---|---|---|
| M1 | `pyproject.toml`、`cli open`、server の GET 系、Markdown 表示 | 1, 2 | 0.5日 |
| M2 | 選択→キーで種別付与、Custom Highlight、サイドパネルのメモ、PUT/DELETE、store | 3, 4 | 1日 |
| M3 | アンカー復元、スクロール保持、一覧からジャンプ | 5 | 0.5日 |
| M4 | `export` | 10 | 0.1日 |
| M5 | `ai.py` + `/ai/explain` + Unknown パネル、`cli explain` | 6 | 0.5日 |
| M6 | `/ai/evaluate` + Explain パネル（Recall 優先） | 7, 8 | 0.3日 |
| M7 | `/ai/quiz` `/ai/grade` + Quiz パネル | 9 | 0.5日 |
| M8 | 記録管理: store の list/match/delete/clean、CLI `list`/`delete`/`clean`、API、UI `<dialog>` | — | 0.5日 |
| M9 | `tests/test_core.py`、README の使い方、`.mise.toml` で python/uv 有効化 | — | 0.3日 |

規模感: Python ~360行、JS ~350行、CSS ~70行。

---

## 10. 検証

- `tests/test_core.py`（pytest 1ファイル）:
  - store の保存→読込の往復、`ai` フィールドが PUT で上書きされないこと
  - `ACTIVE_READER_AI_CMD` を `cat` 等の偽コマンドにして、JSON パース成功 / `raw` フォールバック
  - Host ヘッダ不正・Content-Type 不正で 403 / 415
  - `match`: ファイル指定で1件、ディレクトリ指定で配下のみ（`/a/b` 指定で `/a/bc/x.md` を巻き込まない）
  - `clean`: 元文書を消した記録だけが消え、存在する文書の記録と元文書自体は残る
  - 不正な `doc_id`（`../x` 等）は 400
- 手動（ブラウザ）: §16 の 1〜10 を順に通す。条件5は**文書の先頭に段落を追加してから再オープン**し、マーカーが付いて来ることも確認。
- ブラウザ要件: CSS Custom Highlight API 対応版（Chrome 105+ / Safari 17.2+ / Firefox 140+）。

---

## 11. MVP 後に回すもの

Review 画面（§8.3）・復習スケジューリング（FR-15）・読書セッション（FR-16）・全文検索（FR-18。当面は `jq` / ブラウザ内検索）・外部検索（FR-12）・ショートカット/色の設定 UI・フローティングツールバー・PDF/EPUB・SQLite 化・MCP 公開・ゴミ箱・移動した文書の `relink`。
