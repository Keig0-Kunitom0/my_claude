# my-claude

[Claude Code](https://code.claude.com/docs) のワークフロー系プラグインを配布するカタログです。

```
/plugin marketplace add Keig0-Kunitom0/my_claude
```

## 収録プラグイン

| プラグイン | 概要 |
|---|---|
| **[implement-issue](./implement-issue)** | スペック駆動開発で GitHub issue の実装を自律的に進める。requirements / design / tasklist を唯一の正とし、承認ゲートを挟みながら実装まで走らせる。 |
| **[review-pr](./review-pr)** | PR を11の専門サブエージェントに並列でレビューさせ、結果を集約して Markdown に永続化する。2回目以降は前回レビュー時点からの増分だけを見る。 |
| **[wt](./wt)** | git worktree と herdr ワークスペースを作り、そこで新しい Claude セッションを立ててタスクを投入する。実装やレビューを本体リポジトリと切り離して並行実行する。 |

### インストール

```
/plugin install implement-issue@my-claude
/plugin install review-pr@my-claude
/plugin install wt@my-claude
```

必要なものだけを入れて構いません。3つは互いに独立しています。

`wt` は `/wt:issue` で `implement-issue` を、`/wt:review` で `review-pr` を別セッションに投入しますが、**依存はしていません**（文字列を送るだけなので、入っていなければ投入先でコマンドが見つからず止まります）。単体では任意のプロンプトを投げる `/wt:run` として使えます。

## 共通する設計方針

いずれのプラグインも以下の考え方で作られています。

- **サブエージェントへの委譲**: 重い読み込みを専門サブエージェントに任せ、親のコンテキストを汚さない。長時間の作業でも精度が落ちにくい。
- **リポジトリ非依存**: GitHub CLI (`gh`) ベースで動作し、デフォルトブランチも動的に取得する。特定の言語・フレームワークを前提にしない。
- **中断と再開**: 進捗や結果をファイルに永続化するため、セッションをまたいでも続きから再開できる。
- **安全側の挙動**: `git push` はしない。失敗を握り潰さず、判断に迷う点はユーザーに確認する。

## 要件

- Claude Code
- [GitHub CLI (`gh`)](https://cli.github.com/) — 認証済みであること（`gh auth login`）

## ライセンス

MIT — 詳細は [LICENSE](./LICENSE) を参照してください。
