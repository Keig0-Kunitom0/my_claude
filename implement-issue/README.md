# implement-issue

スペック駆動開発で GitHub issue の実装を自律的に進める、[Claude Code](https://code.claude.com/docs) のプラグインです。

```
/plugin marketplace add Keig0-Kunitom0/my_claude
/plugin install implement-issue@my-claude
```

要件定義 → 設計 → タスク分解 → 実装までを、ユーザーと合意したドキュメント（`requirements.md` / `design.md` / `tasklist.md`）を**唯一の正**として進めます。各ドキュメントはレビュー・approve を挟んで作り込み、実装はその計画に沿って自走します。

> 思想的な背景は「スペック駆動開発」（仕様書を信頼源とし、後続の実装工程を LLM に委ねる開発方式）に基づいています。

## 特徴

- **スペック駆動**: 合意したドキュメントだけを正として実装する。書かれていないことは勝手に作らない。
- **承認ゲート**: requirements / design / tasklist の各段階でユーザーの approve を挟み、認識のズレを防ぐ。
- **コンテキスト効率**: 重い読み込み（影響範囲調査・実装）をサブエージェントに委譲し、実装はチェックボックス単位でフレッシュな文脈で走らせることで、長時間の自走でも精度が落ちにくい。
- **再開できる**: 進捗・承認状態・作業対象をドキュメント側に永続化するため、セッションをクリアしても同じ issue を渡すだけで[続きから再開](#セッションをまたいだ再開)できる。
- **一貫性優先**: 新しくクラス・メソッド・テスト・テーブル・カラムを追加するとき、既存の前例に設計方針・命名・書き方・粒度を揃える。新規分だけを局所最適化してプロダクト全体の可読性を下げない（[詳細](#設計実装方針プロダクトの一貫性)）。
- **移植性**: GitHub CLI (`gh`) ベースでリポジトリ非依存。テスト/Lint コマンドはプロジェクト設定から判定する。
- **安全側の挙動**: `git push` はしない（commit まで）。テストを通すための既存仕様の変更や、失敗の握り潰しはせず、ユーザーに確認する。

### 向いているタスク / 向いていないタスク

計画をファイルに残し、承認を3回挟むぶんの**オーバーヘッドを払う価値があるか**が判断軸です。

| | 例 | 理由 |
|---|---|---|
| ✅ **複雑で時間のかかるタスク** | 複数コンポーネントにまたがる機能追加、大きめのリファクタ | 計画を requirements / design / tasklist に分解でき、実装が計画から逸れにくい |
| ✅ **段階的に、慎重に進めたいタスク** | 仕様の解釈を都度すり合わせたいもの | 各段階で approve を挟むため、認識のズレが実装まで持ち越されない |
| ✅ **中断が挟まる長期タスク** | 数日にわたる作業、セッションを何度も切り替えるもの | 承認状態・進捗・判断がファイルに残り、[続きから再開](#セッションをまたいだ再開)できる |
| ✅ **プロダクトのコアを触るタスク** | 決済・認証など、失敗時の影響が大きいもの | 実装前に設計をファイルとして残せるため、人・チームのレビューを通せる |
| ❌ **数行で終わる簡単なタスク** | typo 修正、定数の値変更 | ドキュメント作成と approve がオーバーヘッドになる |
| ❌ **急ぎのタスク** | 障害対応、即時のホットフィックス | 承認を複数回挟むぶん、かえって遅くなる |
| ❌ **要件が定まっていないタスク** | issue はあるが、目的・解決したい課題が言語化されていない | 何を作るかが決まらないまま requirements の往復が続く。**先に issue 側で言語化することを推奨** |

> 数行で終わる簡単なタスクの場合、コマンドは3ドキュメントの作成と承認ゲートを省いた**軽量フロー**（影響範囲の共有 → 方針合意 → 実装 → commit）を提案します。**軽量フローに切り替えるかは必ず確認を挟む**ため、勝手にステアリングドキュメントが省略されることはありません。とはいえ、上の ❌ に当てはまる作業は素の Claude Code や plan モードのほうが速いことが多いです。

### plan モードより優れている点

Claude Code 標準の [plan モード](https://code.claude.com/docs) も「調査 → 計画提示 → 承認 → 実装」を行いますが、計画は**会話の中にしか存在せず**、承認も実装前の1回きりです。`/implement-issue` は計画をファイルとして外部化することで、次の点で優位です。

| | plan モード | `/implement-issue` |
|---|---|---|
| **計画の置き場所** | 会話コンテキスト。自動圧縮やセッション終了で失われる | `.claude/steering/` にファイルとして永続化。git で追跡できる |
| **承認の回数** | 実装前に1回。以降は計画の解釈が AI 任せになる | requirements / design / tasklist の**3段階**。粒度の粗い合意から順に詰められる |
| **レビューの形** | チャット上でその場で返すだけ | ファイルなので **PR に載せてチームで非同期レビュー**できる |
| **長時間の自走** | 1つの文脈で走り続けるため、後半ほど精度が落ちる | 調査・実装をサブエージェントに分離し、**タスクごとにフレッシュな文脈**で実行 |
| **中断と再開** | 会話を失うと計画ごと消える | 同じ issue を渡すだけで[続きから再開](#セッションをまたいだ再開)。判断は `decisions.md`、詰まりは `blockers.md` に残る |
| **ガードレール** | 汎用（実装前に止まるだけ） | `git push` 禁止・既存仕様の無断変更禁止・失敗の握り潰し禁止を**コマンドに明記** |

要するに、plan モードが**1回の会話を丁寧に進めるための仕組み**なのに対し、`/implement-issue` は**複数セッション・複数人にまたがる作業を破綻させないための仕組み**です。issue 1つを数十分〜数日かけて仕上げる、レビューを人に見せる、といった場面ほど差が出ます。

> ただし plan モードの承認は harness が**強制**するのに対し、このコマンドの承認ゲートは Markdown の指示ベースで、100% の実行が保証されるわけではありません。確実に効かせたい制約は [Hooks](#フックで禁止事項を強制する) と併用してください。

## 構成

このリポジトリは **Claude Code プラグイン兼マーケットプレイス**です。リポジトリルートがそのままプラグインルートになります。

```
.claude-plugin/
├── plugin.json                            # プラグインの識別情報（name / version）
└── marketplace.json                       # カタログ（このリポジトリ自身を source: "./" で登録）
commands/
└── implement-issue.md                     # 親コマンド（フロー / 承認ゲート / 委譲を統括）
agents/
├── codebase-investigator.md               # 影響範囲・既存パターンを調査し要約を返す（読み取り専用）
└── task-implementer.md                    # tasklist の1チェックボックスを実装→テスト→commit
skills/implement-issue-spec-docs/
├── SKILL.md                               # 各ドキュメントの書き方（ステップ2-4で自動ロード）
└── templates/                             # requirements / design / tasklist / decisions / blockers の雛形
```

| 種別 | 名前 | 役割 |
|---|---|---|
| コマンド | `/implement-issue:implement-issue` | 全体のオーケストレーション |
| スキル | `implement-issue-spec-docs` | ステアリングドキュメントの作法とテンプレート |
| サブエージェント | `codebase-investigator` | 設計前の影響範囲調査（親の文脈を汚さない） |
| サブエージェント | `task-implementer` | 実装フェーズの1タスク実行 |

## 前提条件

- **Claude Code**（CLI / IDE 拡張のいずれか）
- **git**
- **GitHub CLI (`gh`)** … issue の読み込みに使用。認証済みであること。
- 対象が **GitHub の issue を運用しているリポジトリ**であること。

## 導入方法

### ステップ1: 前提ツールを確認・準備する

それぞれインストール済みか確認します。

```sh
claude --version   # Claude Code
git --version
gh --version
```

- **Claude Code** が未インストールの場合は [公式ドキュメント](https://code.claude.com/docs) の手順に従ってインストールしてください（CLI / IDE 拡張のいずれでも可）。
- **`gh`** が未インストールの場合は [GitHub CLI のインストール手順](https://github.com/cli/cli#installation) に従ってください（macOS なら `brew install gh`）。

次に `gh` を認証します。**この認証ができていないと issue を読み込めません。**

```sh
gh auth login          # ブラウザの指示に従って認証
gh auth status         # "Logged in to github.com" と表示されればOK
```

### ステップ2: プラグインをインストールする

Claude Code のセッション内で2コマンド実行するだけです。ファイルのコピーは不要です。

```
/plugin marketplace add Keig0-Kunitom0/my_claude
/plugin install implement-issue@my-claude
```

インストール時にスコープを選べます。

| スコープ | 効果 | 向いている場面 |
|---|---|---|
| **User** | 自分のマシンの**全プロジェクト**で使える。各プロジェクトには何も置かない | 個人で複数リポジトリに使いたい（**推奨**） |
| **Project** | そのリポジトリの `.claude/settings.json` に記録され、チームで共有される | チーム全員に配りたい |
| **Local** | そのリポジトリで自分だけ | まず1つのリポジトリで試したい |

> 更新は `/plugin marketplace update my-claude` → `/reload-plugins` です。`plugin.json` の `version` が上がったときに配信されます。

### ステップ3: 認識されているか確認する

Claude Code を（再）起動し、新しいセッションで以下を確認します。

- プロンプトで `/implement-issue:implement-issue` がスラッシュコマンド候補に表示される
- `/plugin list` に `implement-issue` が表示される

> サブエージェント（`codebase-investigator` / `task-implementer`）は、コマンド実行中に必要なタイミングで自動的に起動されるため、手動での確認は不要です。

### ステップ4: 使う前の準備（重要）

移植性のため、コマンドは**テスト/Lint コマンドをプロジェクト側から判定**します。判定できるよう、`CLAUDE.md` に明記しておくことを強く推奨します（未記載の場合はテスト実行時にユーザーへ質問されます）。

`CLAUDE.md` の記載例（プロジェクトの言語・ツールに合わせて記載）:

```md
## テスト / Lint
# 例) Node.js
- テスト実行: `npm test -- <ファイル>`   # 関連ファイルのみ指定可能
- Lint:        `npm run lint`
# 例) Python
- テスト実行: `pytest <ファイル>`
- Lint:        `ruff check .`
# 例) Ruby
- テスト実行: `bundle exec rspec <ファイル>`
- Lint:        `bundle exec rubocop`
```

また、作業ドキュメントは**リポジトリルート**（`git rev-parse --show-toplevel`）の `.claude/steering/<issue番号>-<タイトル>/` に作成されます。チームで共有・振り返りに使えるため **commit を推奨**します（不要なら `.gitignore` に `/.claude/steering/` を追加してください）。

リポジトリルートに `.claude/` が無い場合は、勝手に作らず**どこに作成するかをユーザーに確認**します（`.claude/steering/` を新規作成 / `.steering/` を新規作成 / パスを指定）。

> **サブディレクトリから起動した場合も、保存先はリポジトリルートです。** 例えばリポジトリ内のサブディレクトリを作業ディレクトリとして起動し、そこに `.claude/` があっても、ステアリングは `<root>/.claude/steering/` に作られます。コマンドは作成前に `git rev-parse --show-toplevel` を基点とした絶対パスで作業ディレクトリを作り、作成先を検証します。

## 使い方

issue 番号（または URL、補足ドキュメントのパス）を引数に渡して実行します。

```
# issue 番号を渡す
/implement-issue:implement-issue 1234

# issue の URL でも可
/implement-issue:implement-issue https://github.com/{owner}/{repo}/issues/1234

# プロジェクト固有のドキュメントを前提知識として読ませる（Markdown なら何でも可）
/implement-issue:implement-issue 1234 --context docs/glossary.md

# 複数指定もできる
/implement-issue:implement-issue 1234 --context docs/glossary.md --context docs/spec/billing.md

# 派生元を明示する（未指定ならカレントの HEAD から派生）
/implement-issue:implement-issue 1234 --base origin/develop

# 作業ブランチ名を明示する（未指定なら既存の命名規約から推測）
/implement-issue:implement-issue 1234 --branch 1234/my-custom-name
```

`--context` に渡したファイルは、**親コマンドと両サブエージェントに前提知識として渡ります**。用語の定義や業務上の制約など、コードを読むだけでは分からない情報を補えます。詳しくは[カスタマイズ](#ドメイン知識を---context-で渡す)を参照してください。

### 派生元は「今いる場所」

作業ブランチは**カレントの HEAD から派生**します。デフォルトブランチ（main/master）を自動解決することはしません。

派生元はその時々で違う（main/master のこともあれば develop、既存の作業ブランチのこともある）ため、**派生したいブランチに切り替えてから実行する**という明示的な行為を指定として扱います。別の起点にしたい場合のみ `--base` を渡してください。

`<base>` を checkout したり `git pull` したりはしません。そのため、

- **未コミットの変更がある状態でも失敗しません**
- **git worktree の中でも動きます**（同じブランチを2箇所に checkout できない制約に当たらない）

最新化するかどうかは、実行前に自分で `git pull` するかで決めてください。派生元の SHA は作成時に報告され、`requirements.md` のメタ情報にも記録されます。

### ブランチ名は既存の規約から推測します

`--branch` が未指定の場合、リポジトリの既存ブランチから命名規約を推測します。

1. 直近30本のリモートブランチのうち、**数字で始まるもの**（= 作業ブランチ）だけを見る
2. 区切り文字（`/` `_` `-`）と語の連結方式（kebab / snake）を多数決で判定する
3. 5本以上が同じ形なら採用し、`<issue番号><区切り><タイトルのスラッグ>` を組み立てる
4. **判定できなければ聞きます**（規約を読み取れないリポジトリで推測を押し通さないため）

規約をハードコードしないのは、このプラグインが複数のリポジトリで使われるためです。

進行は次の流れです。**太字の箇所でユーザーの approve を求めます。**

```
ステップ0  再開判定（既存の作業があれば、その続きから再開）
ステップ1  issue 理解 → 作業ブランチ作成（新規時のみ）
ステップ2  requirements.md 作成 ──▶ 【approve】
ステップ3  影響範囲調査（codebase-investigator）→ design.md 作成 ──▶ 【approve】
ステップ4  tasklist.md 作成 ──▶ 【approve】
ステップ5  実装ループ：チェックボックスを1つずつ task-implementer に委譲
           （各タスクで 実装 → 関連テスト/Lint → commit）
           完了報告（レビュー・PR 作成はこのコマンドの範囲外）
```

approve は「approve」「OK」「LGTM」など承認の意思を示す返答で行います。修正してほしい点を書けば、その指摘を反映してドキュメントを練り直します。

実装が詰まった場合（既存仕様の変更が必要 / 3回以上試しても原因不明 / 5回失敗）は、自走を止めてユーザーに対応方針を確認します。中断内容は `.claude/steering/<...>/blockers.md` に記録され、再開時の文脈復帰に使われます。

### セッションをまたいだ再開

セッションをクリアしても、**同じ issue 番号（または URL）を渡して再実行するだけ**で続きから再開できます。

```
/implement-issue:implement-issue 1234        # 1回目：requirements の approve まで進んで中断
（セッションをクリア）
/implement-issue:implement-issue 1234        # 2回目：design の作成から再開
```

ステップ0 で次を自動的に行います。

1. `requirements.md` の**メタ情報**に記録された Issue URL から既存の作業を特定する（見つからなければ `<issue番号>-*` ディレクトリ名、さらに `<issue番号>/*` ブランチの順にフォールバック）。探索はリポジトリ全体を対象とするため、旧配置のリポジトリ直下 `.steering/` や、過去の取り違えでサブディレクトリ配下に作られたものも見つかります（見つけた場合は移動せず、実際のパスを報告します）
2. あれば**再開モード**に入り、メタ情報のブランチ名で作業ブランチに `git switch` する（他ブランチへの checkout や `git pull` はしない）。完了済みコミットの確認にはメタ情報の派生元 SHA を起点に使う
3. `requirements.md` / `design.md` / `tasklist.md` の**ステータス行**（`draft` / `approved（日付）`）を読み、未承認の最初のフェーズから再開する
4. `decisions.md` / `blockers.md` を読み、合意済みの方針と中断理由を復帰する
5. 未コミットの変更があれば内容を提示し、活かして続けるか破棄するかを確認する
6. 再開位置をユーザーに報告し、合意を得てから作業を続行する

そのため `requirements.md` は先頭に**メタ情報**を持ちます。作業対象（Issue・ブランチ・PR）をここに永続化することで、会話が消えても対象を取り違えずに再開できます。

```md
## メタ情報

| 項目 | 値 |
|---|---|
| Issue | https://github.com/<owner>/<repo>/issues/1234 |
| ブランチ | `1234/add-retry-to-webhook` |
| PR | 未作成 |
```

承認状態は各ドキュメントの先頭にあるステータス行が唯一の正です（進捗管理用の別ファイルは作りません）。approve 済みのドキュメントを後から書き換えた場合は `draft` に戻り、再度 approve を求めます。実装フェーズの進捗は `tasklist.md` のチェック状態が正で、チェックはタスクの commit 後に入ります。

## ステアリングドキュメント

実行すると、作業単位ごとに `<リポジトリルート>/.claude/steering/<issue番号>-<タイトル>/` が作られ、以下のドキュメントが生成・更新されます。これらは「AIエージェントが向かう先を示す北極星」であり、実装の唯一の正となります。

| ファイル | 役割 | 作成タイミング |
|---|---|---|
| `requirements.md` | 何を・なぜ作るか（メタ情報・背景・仕様・スコープ外） | ステップ2（approve 必須） |
| `design.md` | どう作るか（設計・実装アプローチ・影響範囲・動作確認方法） | ステップ3（approve 必須） |
| `tasklist.md` | 1コミット粒度の実装タスク（チェックリスト） | ステップ4（approve 必須） |
| `decisions.md` | 設計判断・技術選定の記録（AI の判断・ユーザーとの合意の両方） | 重要な判断をしたとき（条件付き） |
| `blockers.md` | 中断した事象の記録 | 実装が中断したとき（条件付き） |

`requirements.md` / `design.md` / `tasklist.md` は先頭に **ステータス行**（`**ステータス**: draft` / `approved（日付）`）を持ちます。approve のたびに更新され、[セッションをまたいだ再開](#セッションをまたいだ再開)の判定に使われます。さらに `requirements.md` だけは **メタ情報**（Issue の URL / ブランチ名 / PR の URL）を持ち、再開時の作業特定に使われます。

### decisions.md / blockers.md の役割

長時間の自走では、コンテキストウィンドウの**自動圧縮**によって「なぜその設計にしたか」「どこでなぜ詰まったか」といった詳細が失われていきます。この2ファイルは、消えては困る情報をファイルとして外部に逃がし、**再開や振り返りを可能にする**ためのものです。`requirements`/`design`/`tasklist` と違い必須ではなく、該当する事象が起きたときにだけ追記されます。

- **decisions.md** … `design.md` に明記されていない設計判断や、トレードオフのある選択をしたとき、「決定者・決定・理由・検討した代替案・影響・反映先」を追記します。**AI が自律的に下した判断だけでなく、承認ループでユーザーに確認して決まった方針も同じファイルに記録します**（会話にしか残らない合意を外部化するため）。実装後に「なぜこの実装になっているか」を追跡でき、ヘッドレス実行（無人実行）で何が起きたかの把握にも役立ちます。
- **blockers.md** … 実装が中断したとき（既存仕様の変更が必要 / 3回以上試して原因不明 / 5回失敗）、「事象・再現条件・試したこと・原因・ユーザーに確認したいこと」を追記します。中断 → 再開のとき、コマンドは**まずこのファイルを読んで**文脈を復帰させてから作業を再開します。

いずれもテンプレートは `implement-issue-spec-docs` スキルの `templates/` に同梱されています。

## 設計・実装方針（プロダクトの一貫性）

新しく **モジュール / クラス / メソッド / テスト / テーブル / カラム** を追加するとき、このコマンドは**そのコード単体の最適さより、プロダクト全体で読んだときの一貫性**を優先します。新規追加分だけを最適化すると、全体を俯瞰したときに設計思想や書き方がばらつき、かえって可読性が下がるためです。

- **前例・類似機能がある場合** … 設計方針（層・責務の切り方）・命名・書き方（呼び出し方・エラーハンドリング）・クラス分割とテストの粒度・テストの書き方（マッチャーの使い方）を既存に揃えます。踏襲元は `design.md` の「既存パターンとの整合」に `path:line` で明示されます。
- **前例が無い場合** … 近い層・近い役割のコードを参照し、プロダクトの慣習から外れない形にします。`design.md` には「前例なし」と、何に合わせたかが書かれます。

### 踏襲元が健全でないときは方針を確認します

前例をそのまま真似ると問題がある場合（Linter の閾値に抵触している / 除外リストで違反を回避している / メタプログラミングで定義箇所が静的に追えない、など）は、AI が勝手に判断せず**ユーザーに3択で確認**します。

| 選択肢 | 内容 | その後 |
|---|---|---|
| 1. リファクタ先行 | 先に踏襲元をリファクタしてから機能を追加する | スコープが広がるため requirements / design を更新し、再 approve を求めます |
| 2. 新規分のみ健全化 | 一貫性は多少損なうが、新規追加分は Linter に抵触しない設計にする | 判断と理由を `decisions.md` に記録します |
| 3. 別タスクに送る | 今回は既存に揃え、リファクタは今回のスコープ外とする | 判断と理由・リファクタすべき対象を `decisions.md` に記録します（**issue の起票は行わず、ユーザーに委ねます**） |

判断は原則として設計フェーズ（ステップ3）で行います。実装中に発覚した場合、`task-implementer` は自分で決めずに中断し、親コマンド経由でユーザーに確認します。

## カスタマイズ（任意）

### ドメイン知識を `--context` で渡す

**プロジェクト固有の情報が書かれた Markdown なら、何でも渡せます。** スキル（`SKILL.md`）である必要はありません。

```
# スキルにまとめてある場合
/implement-issue:implement-issue 1234 --context .claude/skills/billing-domain/SKILL.md

# プロジェクトのドキュメントをそのまま渡す
/implement-issue:implement-issue 1234 --context docs/glossary.md

# 複数指定もできます
/implement-issue:implement-issue 1234 --context docs/glossary.md --context docs/spec/billing.md
```

渡せるものの例:

| ファイル | 内容 |
|---|---|
| `docs/glossary.md` | 用語集・ドメインモデルの定義 |
| `docs/spec/*.md` | 機能仕様書 |
| `docs/adr/*.md` | 設計判断の記録（なぜその構造になっているのか） |
| `.claude/skills/*/SKILL.md` | 既にスキルとしてまとめてある場合 |

ディレクトリを渡すと配下の `*.md` が展開されます（5ファイルを超える場合は展開せず、個別指定を促します）。

`.claude/skills/` にあるスキルが**自動では読み込まれない**のは意図的です。何が置かれているか予測できないものを実装の前提に採用すると、挙動が環境ごとに変わってしまうためです。読ませたいものは明示的に指定する方式にしています。

**渡す価値が高いのは、コードを読むだけでは絶対に分からない情報です。**

- 用語の定義（「契約」と「申込」は何が違うのか）
- 状態遷移のルール（`trial → active → suspended` は一方向、など）
- 業務上の制約（「解約後もデータは論理削除で残す」など）
- 権限・テナントの境界
- その構造を選んだ理由（ADR に書かれているような判断の経緯）

これが無いと、要件も設計も一般論の範囲でしか書けません。渡してあれば、**仕様として正しい実装**に寄せられます。

> 前提知識は「守らせるルール」ではなく「正しさを判定する材料」として扱われます。**前提知識に反する実装はせず、反する必要が生じた場合はユーザーに確認**します。また**そこに書かれていないことを、書かれているかのように断定しない**よう指示されています。
>
> なお、指定したファイルは親と両サブエージェントに渡るためトークン消費に効きます。500行を超える場合は、全体を読まず必要な箇所だけを検索して参照します。
>
> レビューのたびに読ませたいものが複数のドキュメントに散っている場合は、1つの Markdown にまとめておくと毎回の指定が楽になります。

### フックで禁止事項を強制する

mdの指示は「読まれれば従う」もので 100% の実行は保証されません。`git push` 禁止や Lint の自動実行など**確実に効かせたい処理**は、[Hooks](https://code.claude.com/docs/en/hooks) を `.claude/settings.json` に設定すると仕組みとして担保できます。

以下は `git push` をブロックする `PreToolUse` フックの**例（環境に合わせて調整してください）**:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -e '.tool_input.command | test(\"git[[:space:]]+push\")' >/dev/null 2>&1 && { echo 'git push is not allowed in this workflow' >&2; exit 2; }; exit 0"
          }
        ]
      }
    ]
  }
}
```

同様に、`PostToolUse`（Edit/Write 後）にフォーマッタや Lint を自動実行する設定も有効です。

## 開発とリリース（メンテナ向け）

### ステップ1: リポジトリを clone する

```sh
git clone https://github.com/Keig0-Kunitom0/my_claude.git
cd my_claude
git switch -c feat/your-change
```

### ステップ2: ローカルのまま読み込んで動かす

**インストールせずに、作業ツリーの状態を直接読み込めます。** マーケットプレイスへの登録も push も不要です。

```sh
claude --plugin-dir /path/to/my_claude/implement-issue
```

`--plugin-dir` にはプラグインルート（`implement-issue/`）を渡します。**リポジトリルートではありません。** このリポジトリは複数のプラグインを収録しているため、ルートを渡しても読み込まれません。

セッション中に `.md` を編集したら `/reload-plugins` で再起動なしに反映できます。

```sh
# 認識状態の確認（別ターミナルから確認する場合も同じフラグが必要）
claude --plugin-dir /path/to/my_claude/implement-issue plugin list
claude --plugin-dir /path/to/my_claude/implement-issue plugin details implement-issue
```

`plugin details` はコンポーネントの一覧と**トークンコストの見積もり**を表示します。

> インストール実体は `~/.claude/plugins/cache/` 配下に展開されます。ここを直接編集しても更新で上書きされるため、改修は必ずこのリポジトリ側で行ってください。

### ステップ3: 修正する

修正後、**必ず `version` を上げてください。**

```jsonc
// implement-issue/.claude-plugin/plugin.json
{
  "name": "implement-issue",
  "version": "0.2.0",   // ← 上げる
  ...
}
```

> **version を上げないと、既にインストールしている利用者に更新が届きません。** プラグインは version 文字列でピン留めされ、上がったときにだけ配信されます。

`.claude-plugin/marketplace.json`（リポジトリルート）にも同じプラグインの `version` があります。**優先されるのは `plugin.json` 側**ですが、カタログの表示と食い違わないよう両方を揃えてください。

| ファイル | 役割 | 優先度 |
|---|---|---|
| `implement-issue/.claude-plugin/plugin.json` | プラグイン自身の宣言 | **こちらが優先** |
| `.claude-plugin/marketplace.json` | カタログ上の表示情報 | plugin.json があれば無視される |

バージョンの上げ方の目安:

| 変更内容 | 上げ方 |
|---|---|
| サブエージェント・スキルの追加削除、フローや承認ゲートの変更 | マイナー（`0.1.0` → `0.2.0`） |
| テンプレートの追記、文言修正 | パッチ（`0.1.0` → `0.1.1`） |
| README のみの修正 | 上げなくてよい（配信物に影響しないため） |

### ステップ4: 検証する

```sh
# マニフェストと、agents / commands / skills の構成を検証
claude plugin validate /path/to/my_claude

# CI では警告もエラー扱いにする
claude plugin validate /path/to/my_claude --strict
```

リポジトリルートを渡すと `marketplace.json` と収録プラグインがまとめて検証されます。

### ステップ5: PR を作成してマージする

```sh
git add -A
git commit -m "feat: ..."
git push -u origin feat/your-change
gh pr create
```

**マージ先はデフォルトブランチ（`main`）です。** `marketplace.json` の source は `ref` を指定していないため、デフォルトブランチが参照されます。したがって **`main` にマージした時点で公開**されます。

```jsonc
// .claude-plugin/marketplace.json — ref 未指定 = デフォルトブランチ
"source": { "source": "github", "repo": "Keig0-Kunitom0/my_claude" }
```

> 安定版と開発版を分けたい場合は、`marketplace.json` の source に `"ref": "stable"` を指定し、`stable` ブランチにマージしたときだけ公開する運用にできます。

### ステップ6: 配信を確認する

利用者側（自分の環境でも）で取得します。

```sh
# CLI から
claude plugin update implement-issue

# または Claude Code セッション内から
/plugin marketplace update my-claude
/reload-plugins
```

`claude plugin update` は**適用に再起動が必要**です。更新後、`/plugin list` に上げた version が表示されれば配信成功です。

### まとめ

```
git clone
  ↓
claude --plugin-dir /path/to/my_claude/implement-issue で動作確認
  ↓
修正 + plugin.json の version を上げる（marketplace.json も揃える）
  ↓
claude plugin validate --strict
  ↓
PR 作成 → main にマージ = 公開
  ↓
claude plugin update implement-issue（利用者側）
```

## ライセンス

（必要に応じて記載してください）
