---
description: "issue の実装を別 worktree の新しいセッションに委譲する"
argument-hint: "<issue番号 または issue URL> [--base <ref>]"
allowed-tools: ["Bash", "Read", "Skill", "AskUserQuestion"]
---

# /wt:issue

`/implement-issue:implement-issue` を、別 worktree に立てた新しいセッションで開始する。
起動したら以降は人間がそのセッションを直接見る。

**引数**: `$ARGUMENTS`

---

## 引数の解釈（最初に行う）

**先にオプションを切り出す。** `$ARGUMENTS` から `--base {ref}` を回収し、**残りを issue の指定として解釈する**。
オプションを混ぜたまま issue 番号を取り出そうとすると、値の側を issue 指定と誤認する。

**issue 番号を解決する。** 以降の `NAME` と `BOOT_PROMPT` は**番号**を使う。

| 渡された形 | 解決 |
|---|---|
| 番号のみ（`1234`） | そのまま使う |
| issue URL（`https://github.com/{owner}/{repo}/issues/1234`） | **末尾の数値**を issue 番号とする |

```
/wt:issue 1234
/wt:issue https://github.com/foo/bar/issues/1234
/wt:issue https://github.com/foo/bar/issues/1234 --base origin/develop
```

**URL をそのまま `NAME` に使ってはならない。** `NAME` は herdr の名前制約
（小文字英数と `-` `_` のみ・32文字以内）を満たす必要があり、URL は `/` や `:` を含むうえ
32文字を超える。

数値が取り出せない場合は `AskUserQuestion` で確認する（推測で進めない）。

## 手順は共通スキルにある

`wt-dispatch` スキルを読んで、その手順0〜6に従うこと。
**ここに手順を書き写してはならない**（コピペ増殖を避けるため）。

このコマンドが持つのは下表の4項目だけ。

| 変数 | 値 |
|---|---|
| `NAME` | `issue-<issue番号>` |
| `BASE_REF` | `--base <ref>` があればそれ。無ければ **`HEAD`**（カレント） |
| `CHECKOUT_MODE` | **`detached`** |
| `BOOT_PROMPT` | 下記 |

## 派生元はカレント HEAD

**デフォルトブランチを自動解決しない。** 派生元はその時々で違う（main/master のことも
あれば develop、既存の作業ブランチのこともある）。**派生したいブランチに切り替えてから
叩く**という明示的な行為を指定として扱う。

別の起点にしたい場合のみ `--base` を使う。

```
/wt:issue 1234                      → カレント HEAD から
/wt:issue 1234 --base origin/develop → origin/develop から
```

## ブランチは作らない

`CHECKOUT_MODE` は `detached`。**作業ブランチの作成は `/implement-issue` の責務**
（そのステップ1で `git checkout -b` する）。

ここでブランチを作ると責務が二重になり、ブランチ命名のためにこのコマンドが issue タイトルを
取りに行く必要が生じる。detached で箱だけ作れば、命名は `/implement-issue` に閉じる。

**`--base` を `BOOT_PROMPT` に渡さない。** worktree のカレントが既に正しい位置にあるため、
`/implement-issue` はそこから派生すればよい。渡すと二重指定になる。

## `NAME` は issue 番号だけで作る

`issue-<issue番号>` は `wt-dispatch` の名前制約（小文字始まり・32文字以内）を常に満たすため、
このコマンドでは切り詰めの考慮が要らない。

**issue タイトルを取りに行く必要はない。** ブランチ名を作らないため。

## `BOOT_PROMPT`

`<issue番号>` は「引数の解釈」で**解決した番号**を渡す（URL のままではなく）。
`<REPO_ROOT>` は `git rev-parse --show-toplevel` を**評価した実際の絶対パスに展開してから**渡す。

`--base` は渡さない（理由は上記）。

```
/implement-issue:implement-issue <issue番号>

ステアリングの格納先は <REPO_ROOT>/.claude/steering/ とする。
読み書き・再開判定の探索とも、このパスを使うこと。
カレントディレクトリ配下ではなく、本体リポジトリの絶対パスであることに注意。
```

**コマンド名は `/implement-issue:implement-issue` と完全修飾で書く。**
プラグインコマンドは `plugin:command` で名前空間が付くため、修飾なしで解決される保証がない。

**保存先の上書きが必要な理由**と**探索先も固定する理由**は `wt-dispatch` に書いてある。

## 実行

1. `wt-dispatch` スキルを読む
2. 上表の値を当てて手順0〜6を実行する
3. 手順6の内容を報告する（`workspace_id` を必ず含める）
