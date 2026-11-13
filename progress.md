# Review Mode — Implementation Progress

> Last updated: 2026-11-13

## Goal

Add Review Mode (git diff browsing, staging, and proofreading annotations) to the
ijevim TUI editor.

ユーザー向けの操作説明は [レビューモード](/review) を参照してください。このページは
実装状況と設計上の判断を記録します。

## Current status

レビューモードの主要機能は実装済みです。workspace のテスト、clippy
（`-D warnings`）、rustfmt はすべて通っています。

### Completed

#### Diff model and navigation

- `crates/ijevim-tui/src/review/state.rs`
  - `ReviewState`, `DiffFile`, `Hunk`, `DiffLine`, `Annotation`
  - `ReviewArgs`（`WorkingTree` / `Staged` / `Against` / `Range` / `File`）
- `crates/ijevim-tui/src/review/rows.rs`
  - diff を行単位に平坦化する `DiffRow` モデル
  - ハンクジャンプ、行番号から行への変換
- カーソルは `ReviewState::cursor_row` を単一の情報源とし、keymap と renderer が
  同じ行モデルを共有

#### Git integration

- `crates/ijevim-tui/src/review/diff.rs`
  - `git -C <root>` で実行し、プロセスの cwd に依存しない
  - `--src-prefix=a/ --dst-prefix=b/` を明示し、`diff.mnemonicprefix` の影響を排除
  - quoted path、mnemonic prefix、タイムスタンプ付きヘッダを解析
  - `\ No newline at end of file` を保持
  - ハンク内の `+++` / `---` で始まる行をコンテンツとして扱う
  - `hunk_to_patch()` で `git apply --cached` 用の patch を再構築
  - `stage_hunk` / `unstage_hunk` / `stage_file` / `unstage_file` / `discard_file`
  - staged 状態は `git diff --cached` から導出
  - `reload()` でファイル選択と行位置を保ったまま再取得

#### UI

- `crates/ijevim-tui/src/review/render.rs`
  - ファイル一覧、unified / side-by-side diff、プレビュー
  - 行番号ガター、カーソル、stage マーカー、アノテーション件数
  - アノテーション一覧、コメント入力、ヘルプ、検索 / フィルタプロンプト
  - 複数行コメントの範囲ハイライト、検索マッチの背景ハイライト
- `src/keymap/review.rs`
  - FileList / DiffView / Preview / CommentInput / Annotations / SearchInput /
    FilterInput のサブモード
  - ハンク / ファイル操作、複数行選択、検索、フィルタ

#### Annotations

- `crates/ijevim-tui/src/review/annotations.rs`
  - 保存先: `<git-dir>/ijevim/review/annotations.json`
  - worktree の `.git` ファイルにも対応
  - 旧 `.review/annotations.json` は読み取りフォールバック
  - JSON version 2（`end_line` 追加、v1 も読み込み可能）
  - `created_at` は ISO 8601 UTC
  - serde は使わず手書き JSON を維持

#### Tests

- `crates/ijevim-tui/src/review/diff.rs`
  - parser、prefix、quoted path、no-newline、added / deleted
  - 一時 Git リポジトリを使った stage / unstage / discard / staged 導出
- `crates/ijevim-tui/src/review/render.rs`
  - フィルタ、検索表示、複数行アノテーション、visual 選択
- `tests/editor_flow.rs`
  - 実 Git リポジトリでのレビュー起動、コメント、複数行コメント、
    ファイル操作、検索、フィルタ

## Key Decisions

- GitHub / GitLab の PR 連携はスコープ外。ローカルの Git リポジトリだけを扱う。
- デフォルトの diff は `HEAD` 基準。
- アノテーションは worktree を汚さないよう Git ディレクトリ配下に保存する。
- アノテーションはファイルパスと行番号で紐づける。内容追従は行わない。
- 検索はスマートケース。`n` / `N` は検索が有効な間だけ検索移動になる。
- ファイルフィルタはパスの大文字小文字を無視した部分一致。
- stage 状態をメモリで反転せず、index から再導出して表示と実状態のずれを防ぐ。

## Known limitations

- 未追跡ファイルは stage 済み / intent-to-add でない限り diff に現れない。
- バイナリファイルはハンクを持たないため一覧から除外される。
- アノテーションは行番号ベースのため、コード変更で位置がずれることがある。
- `--staged` 以外では staged 判定のために `git diff --cached` を追加実行する。
- 構文ハイライトは diff 表示には適用していない。
