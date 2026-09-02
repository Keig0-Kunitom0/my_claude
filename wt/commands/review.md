---
description: "PR のコードレビューを別 worktree の新しいセッションに委譲する"
argument-hint: "<PR番号 / PR URL / ブランチ名>"
allowed-tools: ["Bash", "Read", "Skill", "AskUserQuestion"]
---

# /wt:review

`/review-pr:review-pr` を、別 worktree に立てた新しいセッションで開始する。
起動したら以降は人間がそのセッションを直接見る。

**引数**: `$ARGUMENTS`

---

## 引数の解釈（最初に行う）

| 渡された形 | 解決 |
|---|---|
| PR 番号のみ（`1234`） | そのまま使う |
| PR URL（`https://github.com/{owner}/{repo}/pull/1234`） | **末尾の数値**を PR 番号とする |
| ブランチ名（`1234/add-retry-to-webhook`） | PR 検出を省き、そのまま head ブランチとして使う |

```
/wt:review 1234
/wt:review https://github.com/foo/bar/pull/1234
/wt:review 1234/add-retry-to-webhook
```

URL または番号の場合は `gh pr view <PR番号> --json headRefName,number,title` で
head ブランチ名を取る。PR が見つからない場合は `AskUserQuestion` で確認する
（**勝手にデフォルトブランチへ倒さない** — レビュー対象が変わってしまう）。

**URL をそのまま `NAME` に使ってはならない。** `NAME` はブランチ名から作る（下記）。

## 手順は共通スキルにある

`wt-dispatch` スキルを読んで、その手順0〜6に従うこと。
**ここに手順を書き写してはならない。**

このコマンドが持つのは下表の4項目だけ。

| 変数 | 値 |
|---|---|
| `NAME` | `review-<サニタイズ済みブランチ名>`（32文字以内に切り詰める） |
| `BASE_REF` | **`origin/<PR の head ブランチ>`** |
| `CHECKOUT_MODE` | **`detached`** |
| `BOOT_PROMPT` | 下記 |

## `BASE_REF` だけカレントを見ない

`/wt:issue` と `/wt:run` はカレント HEAD が既定だが、**このコマンドだけは PR の head ブランチ固定**。

**なぜ**: レビュー対象は PR によって一意に決まる。呼び出し元がどのブランチにいるかは関係なく、
カレントを見ると「レビューしたい PR」と「実際に checkout される中身」が食い違う余地を残すだけ。

## detached にする理由

レビュー対象のブランチに呼び出し元が既にいると、**同じブランチを2箇所に checkout できず
worktree 作成が必ず失敗する**。レビューは読むだけなので、ブランチ名が無くても実害がない。

## `NAME` のサニタイズ

ブランチ名は `/` を含むのでそのままでは使えない。`wt-dispatch` の名前制約を満たすこと。

1. `/` を `-` に置換する
2. 小文字英数・`-`・`_` 以外を `-` に置換する
3. **32文字以内に切り詰める。** `review-` が7文字を消費するので、ブランチ名から使えるのは
   **25文字まで**。切り詰めは単語境界で行う
4. 先頭が小文字英字であることを確認する

```
1234/refactor-notification-delivery-pipeline
  → review-1234-refactor        （20文字。OK）

review-1234-refactor-notification   （33文字。invalid_agent_name で失敗する）
```

ブランチ名の先頭に issue 番号が付く規約のリポジトリでは、切り詰めても先頭の番号で
一意性が保たれる。**worktree を作る前に長さを確認すること。**

## `BOOT_PROMPT`

`<REPO_ROOT>` は `git rev-parse --show-toplevel` を**評価した実際の絶対パスに展開してから**渡す。

```
/review-pr:review-pr <PR番号>

レビュー結果の格納先は <REPO_ROOT>/.claude/reviews/ とする。
読み書き・既存レビューの探索とも、このパスを使うこと。
カレントディレクトリ配下ではなく、本体リポジトリの絶対パスであることに注意。
```

**コマンド名は `/review-pr:review-pr` と完全修飾で書く。**
プラグインコマンドは `plugin:command` で名前空間が付くため、修飾なしで解決される保証がない。

`/review-pr` は2回目以降に前回レビュー時点からの増分だけを見る設計なので、
**探索先の固定は書き込み先の固定と同じくらい重要**（worktree 内を探すと毎回「初回」になる）。

## 実行

1. `wt-dispatch` スキルを読む
2. 上表の値を当てて手順0〜6を実行する
3. 手順6の内容を報告する（`workspace_id` を必ず含める）
