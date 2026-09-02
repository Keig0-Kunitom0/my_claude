---
name: wt-dispatch
description: "`/wt:*` コマンド専用の内部スキル。git worktree と herdr ワークスペースを作り、そこで新しい Claude セッションを起動してタスクを投入する共通手順を提供する。USE WHEN: `/wt:issue` / `/wt:review` / `/wt:run` の実行中のみ。単独では使用しない。"
---

# メンバーセッション起動の共通手順

**これはスラッシュコマンドではない。** `/wt:issue` / `/wt:review` / `/wt:run` から
「読んで従え」と指示される共通部品で、単独では起動しない。

呼び出し元から以下4項目を受け取る。**手順はこのファイルにしかない。**
呼び出し元がここの手順を書き写すことは禁止する（コピペ増殖を避けるため）。

| 変数 | 意味 |
|---|---|
| `NAME` | メンバー名。ワークスペースの label と agent 名を兼ねる |
| `BASE_REF` | worktree の起点。既定は `HEAD`（= カレント） |
| `CHECKOUT_MODE` | `detached` または `branch`（`branch` のときは `BRANCH_NAME` も受け取る） |
| `BOOT_PROMPT` | 新セッションに投入する文字列 |

## このスキルがやらないこと

- **ブランチを作らない。** `CHECKOUT_MODE=branch` で明示された場合を除く。
  作業ブランチの作成は投入先のコマンド（`/implement-issue` 等）の責務
- **`git fetch` しない。** 理由は手順0を参照
- **タスクを実行しない。** `herdr agent prompt` は人間の打鍵を代行して文字列を流し込むだけで、
  展開・実行は新セッション側で起きる。よってこのスキルは投入先のコマンドに依存しない

---

## 手順0: パスと派生元を導出する

**ハードコードしない。** 呼び出し元の位置から導く。

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"
PREFIX="$(git rev-parse --show-prefix)"      # サブディレクトリで作業していれば "sub/"、リポジトリ直下なら ""
WORKTREE="${REPO_ROOT}-wt/${NAME}"
PROJECT_DIR="${WORKTREE}/${PREFIX}"          # 新セッションの claude が動く場所

BASE_SHA="$(git rev-parse "$BASE_REF")"
BASE_DESC="$(git rev-parse --abbrev-ref "$BASE_REF")"   # ブランチ名。detached なら "HEAD"
```

**`--show-prefix` で導出する理由**: リポジトリによって git ルートとプロジェクトディレクトリの
関係が違う（git ルート直下ではなく、その配下のサブディレクトリで作業するリポジトリがある）。
呼び出し元自身の位置から導けばハードコードが要らない。

**worktree を `<repo_root>-wt/` に置く理由**: herdr 既定の `~/.herdr/worktrees/` は
`Read`/`Edit` の許可範囲と sandbox の書き込み許可範囲の**どちらからも外れる**ため、
そこに置くと新セッションのファイルを読み書きできない。

**`git fetch` しない理由**: 既定の起点はカレント HEAD であり、**ローカルの状態そのものが起点**なので
fetch する意味がない。加えてサンドボックス内の `git fetch` は HTTPS + osxkeychain への XPC が
遮断されて失敗するため、実行するには `herdr pane run` によるサンドボックス脱出が必要になり、
**承認プロンプトが増える**。最新化するかは人間が事前に `git pull` するかで決める。

`BASE_REF` にリモート追跡参照（`origin/develop` 等）が明示された場合、それは**ローカルに
キャッシュされた状態**を指す。古い可能性は手順6で SHA を報告して可視化する。

## 手順1: 同名 worktree の存在を確認する

同じ issue や PR を2回起動するのは普通に起きる。

```bash
[ -e "$WORKTREE" ] && echo "既存: $WORKTREE"
```

存在する場合は**勝手に消さず**、`AskUserQuestion` で人間に選ばせる。

1. 既存のワークスペースにフォーカスする（`herdr workspace focus <workspace_id>`）
2. 撤収してから作り直す（撤収手順は README を参照）
3. 中止する

**勝手に消してはならない。** 未コミットの作業が入っている可能性がある。

## 手順2: worktree とワークスペースを作る

```bash
WT_JSON=$(herdr worktree create \
  --cwd   "$REPO_ROOT" \
  --path  "$WORKTREE" \
  --base  "$BASE_SHA" \
  --label "$NAME" \
  --no-focus)

WS=$(printf   '%s' "$WT_JSON" | jq -r '.result.workspace.workspace_id')
PANE=$(printf '%s' "$WT_JSON" | jq -r '.result.root_pane.pane_id')
```

`CHECKOUT_MODE=branch` のときのみ `--branch "$BRANCH_NAME"` を足す。

**`workspace_id` は `.result.workspace.workspace_id` にある。**
`.result.workspace_id` ではない（返り値は `{ type, workspace, tab, root_pane, worktree }`）。

### detached にする

**herdr は detached での作成に対応していない。** `WorktreeCreateParams` には
`base` / `branch` / `cwd` / `focus` / `label` / `path` / `workspace_id` しかなく、
`detach` に相当するパラメータが存在しない（herdr 0.8.0 のスキーマで確認）。

`--branch` を省略しても detached にはならず、herdr が `worktree/green-valley-dabb` のような
**ブランチ名を自動生成する**（0.8.0 で実測）。SHA は正しいが、放置すると撤収後に無名ブランチが残る。

**返り値の `is_detached` を見てから分岐する。無条件に detach しない。**

```bash
IS_DETACHED=$(printf '%s' "$WT_JSON" | jq -r '.result.worktree.is_detached')
AUTO_BRANCH=$(printf '%s' "$WT_JSON" | jq -r '.result.worktree.branch')

if [ "$CHECKOUT_MODE" = "detached" ] && [ "$IS_DETACHED" != "true" ]; then
  git -C "$WORKTREE" checkout --detach "$BASE_SHA"
  [ "$AUTO_BRANCH" != "null" ] && git -C "$WORKTREE" branch -D "$AUTO_BRANCH"
fi
```

`is_detached` は `WorktreeInfo` の required フィールドなので必ず返る。
**事実を見て分岐すれば、herdr 側の挙動が変わっても壊れない。**

**detached にする理由**: 同じブランチを2箇所に checkout できないため、呼び出し元が対象ブランチに
いると worktree 作成が失敗する。加えて、作業ブランチの作成を投入先コマンドの責務として
残すため、この時点ではブランチを作らない。

## 手順3: CLAUDE.md をコピーする

```bash
[ -f "$REPO_ROOT/CLAUDE.md" ] && cp "$REPO_ROOT/CLAUDE.md" "$WORKTREE/CLAUDE.md"
```

**ルートに無ければコピーしない。サブディレクトリを探しに行かない。**
`CLAUDE.md` はリポジトリルートに置くのが正しい配置であり、イレギュラーを発見ロジックで
支えるより配置を正す方が安い。

**`.claude/` はコピーも symlink もしない。** `settings.local.json` を共有すると、
片方の設定変更がもう片方に波及する事故が起きる。

## 手順4: セッションを起動してタスクを投入する

```bash
herdr pane run    "$PANE" cd "$PROJECT_DIR"
herdr agent start "$NAME" --kind claude --pane "$PANE"
herdr agent prompt "$PANE" "$BOOT_PROMPT"
```

**初回のタスク投入は `agent prompt` で行う。** これは人間の打鍵を代行して文字列を流し込むだけで、
コマンドの展開・実行は新セッション側で起きる。

## 手順5: 着手を見届ける

```bash
herdr agent wait "$NAME" --until working --timeout 120000
```

**「起動した」と「働いている」は別イベント。** ここを省くと「起動したペインが消える」黙死を
検知できないまま完了報告してしまう。タイムアウトしたら成功と報告せず、人間に伝える。

## 手順6: 結果を報告する

| 項目 | 備考 |
|---|---|
| `NAME` | |
| **`workspace_id`** | **必須。** 撤収に要る（下記） |
| worktree パス | |
| **派生元** `<BASE_DESC> (<BASE_SHA の短縮>)` | fetch しないため、古いかどうかを人間が判断する材料 |
| 投入したタスク | |
| `agent wait` の結果 | `working` に到達したか |

**`workspace_id` を必ず出す。** `herdr worktree remove` は `workspace_id` を required で要求し、
`path` を受け付けない。ここで報告しないと後から撤収できなくなる。

---

## メンバー名（`NAME`）の制約

`herdr agent start <NAME>` の制約に従う。

| 制約 |
|---|
| 小文字英字で始まる |
| 小文字英数・`-`・`_` のみ |
| **1〜32文字** |

**33文字以上は `invalid_agent_name` で失敗する。**
名前をブランチ名から作る場合（`/wt:review`）は、**worktree を作る前に長さを確認して切り詰める**。
切り詰めは単語境界で行う。ブランチ名の先頭に issue 番号が付く規約のリポジトリでは、
切り詰めても先頭の番号で一意性が保たれる。

## `BOOT_PROMPT` に必ず含めるもの

**成果物の保存先を、本体リポジトリの絶対パスで指定する。**

`/implement-issue` も `/review-pr` も保存先を `git rev-parse --show-toplevel` で解決する。
新セッションは worktree 内で動くため、**これは worktree のパスを返す**。放置すると成果物が
worktree に書かれ、撤収時に消える。しかも `.claude/` は gitignore 対象で herdr の dirty 拒否に
引っかからないため、**無警告で消える**。

**「書き込み先」だけでなく「探索先」も固定する。** 再開判定は `<root>` 配下を探索するため、
書き込み先だけ本体に向けると2回目の起動時に既存の作業を見つけられない。

```
<成果物の種類>の格納先は <REPO_ROOT>/.claude/<ディレクトリ>/ とする。
読み書き・再開判定の探索とも、このパスを使うこと。
カレントディレクトリ配下ではなく、本体リポジトリの絶対パスであることに注意。
```

`<REPO_ROOT>` は呼び出し元で `git rev-parse --show-toplevel` を評価した**実際の絶対パスに
展開してから**渡す。新セッション側で評価させると worktree のパスになる。
