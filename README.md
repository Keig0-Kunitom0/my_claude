# implement-issue

スペック駆動開発で GitHub issue の実装を自律的に進める、[Claude Code](https://code.claude.com/docs) のスラッシュコマンド集です。

要件定義 → 設計 → タスク分解 → 実装までを、ユーザーと合意したドキュメント（`requirements.md` / `design.md` / `tasklist.md`）を**唯一の正**として進めます。各ドキュメントはレビュー・approve を挟んで作り込み、実装はその計画に沿って自走します。

> 思想的な背景は「スペック駆動開発」（仕様書を信頼源とし、後続の実装工程を LLM に委ねる開発方式）に基づいています。

## 特徴

- **スペック駆動**: 合意したドキュメントだけを正として実装する。書かれていないことは勝手に作らない。
- **承認ゲート**: requirements / design / tasklist の各段階でユーザーの approve を挟み、認識のズレを防ぐ。
- **コンテキスト効率**: 重い読み込み（影響範囲調査・実装）をサブエージェントに委譲し、実装はチェックボックス単位でフレッシュな文脈で走らせることで、長時間の自走でも精度が落ちにくい。
- **移植性**: GitHub CLI (`gh`) ベースでリポジトリ非依存。テスト/Lint コマンドはプロジェクト設定から判定する。
- **安全側の挙動**: `git push` はしない（commit まで）。テストを通すための既存仕様の変更や、失敗の握り潰しはせず、ユーザーに確認する。

## 構成

```
.claude/
├── commands/
│   └── implement-issue.md                 # 親コマンド（フロー / 承認ゲート / 委譲を統括）
├── agents/implement-issue/
│   ├── codebase-investigator.md           # 影響範囲・既存パターンを調査し要約を返す（読み取り専用）
│   └── task-implementer.md                # tasklist の1チェックボックスを実装→テスト→commit
└── skills/implement-issue-spec-docs/
    ├── SKILL.md                           # 各ドキュメントの書き方（ステップ2-4で自動ロード）
    └── templates/                         # requirements / design / tasklist / decisions / blockers の雛形
```

| 種別 | 名前 | 役割 |
|---|---|---|
| コマンド | `/implement-issue` | 全体のオーケストレーション |
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

### ステップ2: コマンド一式を配置する

#### A. 自分のプロジェクトに組み込む（推奨）

このリポジトリの `.claude/` 配下の3点を、利用したいプロジェクトの `.claude/` にコピーします。

```sh
# 例: このリポジトリを一時的に clone してコピーする場合
git clone https://github.com/<your-account>/my_claude.git /tmp/implement-issue

# コピー先プロジェクトのルートで実行
mkdir -p .claude/commands .claude/agents .claude/skills
cp     /tmp/implement-issue/.claude/commands/implement-issue.md      .claude/commands/
cp -r  /tmp/implement-issue/.claude/agents/implement-issue           .claude/agents/
cp -r  /tmp/implement-issue/.claude/skills/implement-issue-spec-docs .claude/skills/
```

配置後、`.claude/` の構成が「[構成](#構成)」の図と一致していることを確認してください。

#### B. このリポジトリ単体で試す

```sh
git clone https://github.com/<your-account>/my_claude.git
cd implement-issue
# このディレクトリで Claude Code を起動する
```

### ステップ3: 認識されているか確認する

Claude Code を（再）起動し、新しいセッションで以下を確認します。

- プロンプトで `/implement-issue` がスラッシュコマンド候補に表示される
- `implement-issue-spec-docs` がスキル一覧に表示される

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

また、作業ドキュメントはリポジトリ直下の `.steering/<issue番号>-<タイトル>/` に作成されます。チームで共有・振り返りに使えるため **commit を推奨**します（不要なら `.gitignore` に `/.steering/` を追加してください）。

## 使い方

issue 番号（または URL、補足ドキュメントのパス）を引数に渡して実行します。

```
/implement-issue 1234
```

進行は次の流れです。**太字の箇所でユーザーの approve を求めます。**

```
ステップ1  issue 理解 → 作業ブランチ作成
ステップ2  requirements.md 作成 ──▶ 【approve】
ステップ3  影響範囲調査（codebase-investigator）→ design.md 作成 ──▶ 【approve】
ステップ4  tasklist.md 作成 ──▶ 【approve】
ステップ5  実装ループ：チェックボックスを1つずつ task-implementer に委譲
           （各タスクで 実装 → 関連テスト/Lint → commit）
           完了報告（レビュー・PR 作成はこのコマンドの範囲外）
```

approve は「approve」「OK」「LGTM」など承認の意思を示す返答で行います。修正してほしい点を書けば、その指摘を反映してドキュメントを練り直します。

実装が詰まった場合（既存仕様の変更が必要 / 3回以上試しても原因不明 / 5回失敗）は、自走を止めてユーザーに対応方針を確認します。中断内容は `.steering/<...>/blockers.md` に記録され、再開時の文脈復帰に使われます。

## ステアリングドキュメント

実行すると、作業単位ごとに `.steering/<issue番号>-<タイトル>/` が作られ、以下のドキュメントが生成・更新されます。これらは「AIエージェントが向かう先を示す北極星」であり、実装の唯一の正となります。

| ファイル | 役割 | 作成タイミング |
|---|---|---|
| `requirements.md` | 何を・なぜ作るか（背景・仕様・スコープ外） | ステップ2（approve 必須） |
| `design.md` | どう作るか（設計・実装アプローチ・影響範囲・動作確認方法） | ステップ3（approve 必須） |
| `tasklist.md` | 1コミット粒度の実装タスク（チェックリスト） | ステップ4（approve 必須） |
| `decisions.md` | 設計判断・技術選定の記録 | 重要な判断をしたとき（条件付き） |
| `blockers.md` | 中断した事象の記録 | 実装が中断したとき（条件付き） |

### decisions.md / blockers.md の役割

長時間の自走では、コンテキストウィンドウの**自動圧縮**によって「なぜその設計にしたか」「どこでなぜ詰まったか」といった詳細が失われていきます。この2ファイルは、消えては困る情報をファイルとして外部に逃がし、**再開や振り返りを可能にする**ためのものです。`requirements`/`design`/`tasklist` と違い必須ではなく、該当する事象が起きたときにだけ追記されます。

- **decisions.md** … `design.md` に明記されていない設計判断や、トレードオフのある選択をしたとき、`task-implementer` が「決定・理由・検討した代替案・影響」を追記します。実装後に「なぜこの実装になっているか」を追跡でき、ヘッドレス実行（無人実行）で何が起きたかの把握にも役立ちます。
- **blockers.md** … 実装が中断したとき（既存仕様の変更が必要 / 3回以上試して原因不明 / 5回失敗）、「事象・再現条件・試したこと・原因・ユーザーに確認したいこと」を追記します。中断 → 再開のとき、コマンドは**まずこのファイルを読んで**文脈を復帰させてから作業を再開します。

いずれもテンプレートは `implement-issue-spec-docs` スキルの `templates/` に同梱されています。

## カスタマイズ（任意）

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

## ライセンス

（必要に応じて記載してください）
