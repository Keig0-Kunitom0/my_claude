# wt

git worktree と herdr ワークスペースを作り、**そこで新しい Claude セッションを立てて
タスクを投入する**コマンド集。起動したら以降は人間がそのセッションを直接見る。

## コマンド

| コマンド | 用途 |
|---|---|
| `/wt:issue <issue番号\|URL> [--base <ref>]` | `/implement-issue` を別 worktree で開始 |
| `/wt:review <PR番号\|URL\|ブランチ名>` | `/review-pr` を別 worktree で開始 |
| `/wt:run "<プロンプト>" [--base <ref>] [--branch <name>] [--name <name>]` | 任意のタスク |
| `/wt:clear [<番号\|label>] [--prune] [--force]` | **撤収**（worktree・ワークスペース・セッション） |

```
/wt:issue  1234
/wt:issue  1234 --base origin/develop
/wt:review 1234
/wt:run --branch 1234/hotfix "この修正をコミットして"
/wt:clear  1234 --prune
```

## 前提

- [herdr](https://herdr.dev) 0.8.0 以降がインストールされ、セッションが起動していること
- `gh` が使えること（`/wt:review` の PR 情報取得に使う）
- `/wt:issue` は `implement-issue` プラグイン、`/wt:review` は `review-pr` プラグインが
  インストールされていること（**依存はしない**。文字列を送るだけなので、
  入っていなければ新セッション側でコマンドが見つからず止まる）

## 設計

### タスクを「呼ぶ」のではなく「打鍵を代行する」

```
呼び出し元のセッション
  /wt:issue 1234
     ├─ worktree 作成        herdr worktree create
     ├─ セッション起動        herdr agent start
     └─ 文字列を流し込む      herdr agent prompt <pane> "/implement-issue:implement-issue 1234"
                                                          └── ただの文字列。ここでは実行しない
                    ▼
        新しいセッション（別 worktree / 別ワークスペース）
          └─ ここで初めて展開・実行される
```

`herdr agent prompt` は人間の打鍵を代行するだけなので、**このプラグインは
`implement-issue` / `review-pr` に依存しない**。だから `/wt:run` が同じ仕組みで成立する。

### 責務の分担

| 作るもの | 作る主体 |
|---|---|
| worktree（ディレクトリ） | **wt** |
| herdr ワークスペース | **wt** |
| **作業ブランチ** | **投入先のコマンド**（`/implement-issue` 等） |

wt は `--branch` を明示された場合を除きブランチを作らない。worktree は detached で作り、
命名は投入先に任せる。これにより wt が issue タイトルを取りに行く必要がなくなる。

### 派生元はカレント HEAD

デフォルトブランチを自動解決しない。派生元はその時々で違う（main/master のこともあれば
develop、既存の作業ブランチのこともある）ため、**派生したいブランチに切り替えてから叩く**
という明示的な行為を指定として扱う。別の起点にしたい場合のみ `--base` を使う。

`/wt:review` だけは `origin/<PR の head>` 固定。レビュー対象は PR で一意に決まるため。

### fetch しない

カレント起点であればローカルの状態そのものが起点なので fetch する意味がない。加えて
サンドボックス内の `git fetch` は keychain への XPC が遮断されて失敗するため、実行するには
サンドボックス脱出が必要になり**承認プロンプトが増える**。最新化するかは人間が事前に
`git pull` するかで決める。

派生元の SHA は起動時に報告されるので、古いかどうかはそこで判断できる。

## 撤収

```
/wt:clear                    → wt 管理下の一覧を出して選ぶ（複数選択可）
/wt:clear 1234              → issue-1234 / review-1234-* を探して撤収
/wt:clear issue-1234        → label 完全一致
/wt:clear --workspace w1Z    → 直接指定
```

`herdr worktree remove` は1回で**ワークスペース・セッション・worktree 登録・ディレクトリ**を
まとめて消す。個別に `herdr workspace close` を呼ぶ必要はない。

**起動コマンドを問わず使える。** 対象の判定は worktree のパスが `<repo_root>-wt/` 配下かだけなので、
`/wt:issue` / `/wt:review` / `/wt:run` のどれが作ったものも撤収できる。

| 作った側 | label | 番号で引けるか |
|---|---|---|
| `/wt:issue` | `issue-1234` | 引ける |
| `/wt:review` | `review-1234-add-retry` | 引ける |
| `/wt:run` | `wt-<スラッグ>` / `--name` の値 | 引けない（label 指定か一覧から選ぶ）|

同じ番号で issue と review が並行している場合は、両方を提示して選ばせる。

### 残るのは作業ブランチだけ

既定では**ブランチを削除しない**。`--prune` を付けると、**安全と判定できた場合に限り**削除する。

| 状態 | `--prune` の挙動 |
|---|---|
| detached だった | 対象なし |
| 派生元からコミットが無い | 削除 |
| push 済み | 削除 |
| デフォルトブランチにマージ済み | 削除 |
| **未 push のコミットあり** | **削除しない**（コミット一覧を添えて報告） |

`--prune` は「安全なら消す」であって「強制的に消す」ではない。

### dirty な worktree

未コミット・未追跡のファイルがあると `dirty_worktree_requires_force` で拒否される。
**これは正しい挙動**なので自動で `--force` を付け直さず、残っているものを提示したうえで
ユーザーが判断する。

```
/wt:clear 1234 --force
```

### このセッション自身を撤収する場合

対象のワークスペースの中から `/wt:clear` を実行すると、**そのセッションは終了する**
（`herdr worktree remove` はサーバ側で実行され、ペインが破棄されると呼び出し元も落ちる）。

そのため、撤収内容の報告と警告を**削除の前に**出し、同意を取ってから実行する。
削除後は報告する主体が存在しないため。

## 制約

### メンバー名は32文字以内

`herdr agent start` の制約（小文字英字始まり・小文字英数と `-` `_` のみ・1〜32文字）。
33文字以上は `invalid_agent_name` で失敗する。

`/wt:review` はブランチ名から名前を作るため切り詰めが起きる。`review-` が7文字を消費するので
ブランチ名から使えるのは25文字まで。

### 成果物は本体リポジトリに書かれる

新セッションは worktree 内で動くため、`git rev-parse --show-toplevel` は worktree のパスを返す。
放置すると成果物が worktree に書かれ、撤収時に消える（`.claude/` は gitignore 対象なので
dirty 拒否にも引っかからず**無警告で消える**）。

`/wt:issue` と `/wt:review` は boot prompt で保存先を本体リポジトリの絶対パスに固定している。
`/wt:run` は自動では添えないので、**必要なら呼び出し元がプロンプトに含めること**。

## ファイル構成

```
wt/
├── commands/
│   ├── issue.md      → /wt:issue    プリセット表のみ
│   ├── review.md     → /wt:review   プリセット表のみ
│   ├── run.md        → /wt:run      プリセット表のみ
│   └── clear.md      → /wt:clear    撤収（wt-dispatch は使わない）
└── skills/wt-dispatch/SKILL.md      起動の手順0〜6。手順はここにしかない
```

起動系3コマンドは `NAME` / `BASE_REF` / `CHECKOUT_MODE` / `BOOT_PROMPT` の4項目と固有の注意点だけを持ち、
**手順は書き写さない**。新しいプリセットを足すときも、追加するのはこの4項目だけで済む。

`/wt:clear` は起動の手順を共有しないため `wt-dispatch` を読まない。撤収は
`herdr worktree remove` 1回で済み、共有すべき手順が無いため。
