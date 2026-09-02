---
description: "wt が作った worktree・ワークスペース・セッションを撤収する"
argument-hint: "[<issue/PR番号> | <label> | --workspace <id>] [--prune] [--force]"
allowed-tools: ["Bash", "Read", "AskUserQuestion"]
---

# /wt:clear

`/wt:issue` / `/wt:review` / `/wt:run` が作ったものを撤収する。

**引数**: `$ARGUMENTS`

---

## 1コマンドで4つ消える

`herdr worktree remove --workspace <id>` を1回叩けば、以下がまとめて消える（実測）。

| 対象 | |
|---|---|
| herdr ワークスペース | 消える |
| セッション（ペインのプロセス） | ワークスペースと一緒に消える |
| git worktree の登録 | 消える |
| ディレクトリ実体 | 消える |
| **作業ブランチ** | **残る** ← `--prune` で扱う |

**個別に `herdr workspace close` を呼ぶ必要はない。**

## 対象の解決

### wt が作ったものだけを絞り込む

worktree のパスが `<repo_root>-wt/` 配下かで判定する。

```bash
herdr workspace list | jq -r '.result.workspaces[]
  | select(.worktree.checkout_path // "" | contains("-wt/"))
  | "\(.workspace_id)\t\(.label)\t\(.worktree.checkout_path)"'
```

**label だけで照合しない。** herdr は label の一意性を保証しないため、パスによる絞り込みと
併用する。

### 引数の解釈

| 渡された形 | 解決 |
|---|---|
| **なし** | 上記の一覧から選ばせる（下記） |
| 番号（`1234`） | label が `issue-1234` / `review-1234-*` にマッチするものを探す |
| label（`issue-1234`） | label 完全一致 |
| `--workspace <id>` | 曖昧さゼロの直接指定。照合を省く |

複数マッチした場合は**勝手に選ばず**確認する。
1件も無い場合はその旨を伝えて終了する（エラーにしない — 既に撤収済みの可能性がある）。

### 一覧から選ばせる方法

**`AskUserQuestion` を `multiSelect: true` で使う。** 撤収は「溜まったものをまとめて片付ける」
用途が主なので、1つずつ聞くと候補の数だけ往復が発生する。

各選択肢の `label` にはワークスペースの label を、`description` には
**worktree パス・ブランチ名・`--prune` の判定結果**を入れる。何を消すのかが選ぶ前に分かる必要がある。

**候補が5件以上ある場合、`AskUserQuestion` は使えない。**
このツールは1問あたり選択肢が**2〜4個**に制限されている。撤収を忘れて溜まった状態こそ
引数なしで叩く場面なので、この上限には現実的に到達する。

候補が5件以上のときは、**ツールを使わず一覧をテキストで提示し、番号か label で指定し直してもらう**。

```
wt 管理下のワークスペースが 7 件あります。

  ws=w1Z  issue-1234                 1234/add-retry-to-webhook           未 push 3 件
  ws=w20  review-1234-webhook      (detached)                —
  ws=w21  issue-5678                 5678/fix-timeout-handling  push 済み
  ...

撤収するものを指定してください:
  /wt:clear 1234
  /wt:clear issue-5678 --prune
  /wt:clear --workspace w1Z
```

候補が1件のときも `AskUserQuestion` を使わない（選択肢が2個未満のため）。
内容を提示して実行の可否だけ確認する。

**このコマンドは特定の起動コマンドに紐づかない。** 対象の判定は「worktree のパスが
`<repo_root>-wt/` 配下か」だけなので、`/wt:issue` / `/wt:review` / `/wt:run` の
いずれが作ったものも同じように撤収できる。

| 作った側 | label の形 | 番号で引けるか |
|---|---|---|
| `/wt:issue` | `issue-<番号>` | 引ける |
| `/wt:review` | `review-<ブランチ名>` | 引ける（先頭が issue 番号のブランチ規約の場合）|
| `/wt:run` | `wt-<スラッグ>` または `--name` の値 | **引けない**。label 指定か一覧から選ぶ |

**同じ番号で issue と review の両方が存在しうる**（実装とレビューの並行）。
その場合は必ず両方を提示して選ばせる。

## このセッション自身が対象の場合

**実行するとこのセッションは終了する。** `herdr worktree remove` はサーバ側で実行され、
ペインが破棄されると呼び出し元のプロセスごと落ちるため、**削除は成功するが報告は返らない**。

カレントのワークスペースが対象かどうかを先に判定する。

```bash
CURRENT_WS=$(herdr pane current | jq -r '.result.pane.workspace_id')
```

一致する場合は、**削除を実行する前に**次を行う。

1. 撤収内容（worktree パス / ブランチと `--prune` の判定結果）を**先に報告する**
2. 「⚠ 実行するとこのセッションは終了します」と明示する
3. `AskUserQuestion` で続行の同意を得る
4. 同意が得られてから `herdr worktree remove` を実行する

**報告を後回しにしてはならない。** 削除後は報告する主体が存在しない。

## 手順

**ブランチ名は削除の前に取得する。** 削除するとディレクトリが消え、`git -C "$WORKTREE"` が
使えなくなる。

```bash
WORKTREE=<解決した checkout_path>
REPO_ROOT=$(git -C "$WORKTREE" rev-parse --path-format=absolute --git-common-dir | sed 's|/\.git$||')
BRANCH=$(git -C "$WORKTREE" branch --show-current)      # detached なら空文字
```

**対象は複数になりうる**（一覧から複数選択された場合）。以下を対象ごとに繰り返す。

1. 対象を解決する（上記）
2. **全対象について、ブランチ名と `--prune` の安全判定を先に済ませる**（下記）
3. このセッション自身が対象に含まれるなら、報告と同意を先に行う
4. 対象ごとに撤収する

   ```bash
   herdr worktree remove --workspace "$WS"            # 通常
   herdr worktree remove --workspace "$WS" --force    # --force 指定時のみ
   ```

   **1件失敗しても残りを続行する。** dirty で拒否された1件のために、他の撤収まで
   止める理由がない。失敗した対象と理由は控えておき、最後にまとめて報告する。

5. `--prune` かつ安全と判定されたブランチを削除する

   ```bash
   git -C "$REPO_ROOT" branch -d "$BRANCH"
   ```

6. `git -C "$REPO_ROOT" worktree prune` で登録の残骸を掃除する（**全対象の処理後に1回**）
7. 報告する

**このセッション自身が対象に含まれる場合は、それを最後に処理する。**
先に消すと残りの対象を撤収する主体が消える。

## dirty な worktree は拒否される

未コミット・未追跡のファイルがあると、herdr は `dirty_worktree_requires_force` で拒否する（実測）。

```json
{"error":{"code":"dirty_worktree_requires_force",
  "message":"... contains modified or untracked files, use --force to delete it"}}
```

**これは正しい挙動なので、自動で `--force` を付け直さない。**
拒否されたら、**何が残っているかを `git -C "$WORKTREE" status --short` で提示**したうえで、
`--force` を付けるかユーザーに判断させる。

## `--prune`: ブランチの削除

**既定ではブランチを削除しない。** `--prune` が指定されたときだけ、**安全と判定できた場合に限り**
削除する。

| 状態 | 判定 | 判定方法 |
|---|---|---|
| detached だった | 対象なし | `branch --show-current` が空 |
| 派生元からコミットが無い | **安全** | `git rev-list --count <派生元>..<BR>` が 0 |
| push 済み（upstream と一致） | **安全** | `git rev-list --count <BR>@{u}..<BR>` が 0 |
| デフォルトブランチにマージ済み | **安全** | `git branch --merged` に含まれる |
| **未 push のコミットあり** | **削除しない** | 上記いずれにも該当しない |

未 push のコミットがある場合は、`--prune` が指定されていても**削除せず**、
コミット一覧を添えて報告する。**`--prune` は「安全なら消す」であって「強制的に消す」ではない。**

```
⚠ ブランチ 5678/fix-timeout-handling に未 push のコミットが 3 件あるため残しました
    a1b2c3d fix: ...
  削除する場合: git branch -D 5678/fix-timeout-handling
```

`git branch -d`（小文字）を使う。`-D` は安全判定を握り潰すので使わない。
`-d` が拒否したら判定が誤っていたということなので、削除せず報告する。

## 報告

対象ごとに1行、最後に失敗があればまとめて出す。

| 項目 |
|---|
| 撤収した `label` と `workspace_id` |
| 削除した worktree パス |
| ブランチ: 削除したか、残したか（残した場合は理由と削除コマンド）|
| `--force` を使った場合はその事実 |
| **撤収できなかった対象と理由**（dirty 等）|

```
撤収しました:
  issue-1234  (w1Z)  → ブランチ 1234/add-retry-to-webhook は残しました（未 push 3 件）
  issue-5678  (w21)  → ブランチ 5678/fix-timeout-handling も削除（push 済み）

撤収できませんでした:
  review-1234-webhook (w20)  未コミットの変更があります
    M  app/models/user.rb
    ?? tmp/debug.log
    強制削除する場合: /wt:clear --workspace w20 --force
```

## 実行

1. 対象を解決する
2. ブランチ名と安全判定を先に済ませる
3. このセッション自身が対象なら、報告 → 警告 → 同意
4. 撤収し、報告する
