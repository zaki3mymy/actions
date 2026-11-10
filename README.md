# actions

Shared GitHub Actions reusable workflows.

## Workflows

### Claude Code

Claude Code を使った Issue ドリブン開発を自動化するワークフロー群。

| ワークフロー | 説明 |
|---|---|
| `claude-implement.yml` | Issue を実装し、PRを作成する |
| `claude-review.yml` | PRをレビューし、コメントを残す |
| `claude-fix.yml` | レビュー指摘を修正し、返信する |

### リリース

SemVer のタグと GitHub リリースを扱うワークフロー群。

| ワークフロー | 説明 |
|---|---|
| `label-pr-by-title.yml` | PR のタイトル（Conventional Commits）から、種類に応じたラベルを付ける。タイトルの形が違うときは失敗する |
| `next-tag.yml` | 前のタグからの PR のラベルで次の SemVer のバージョンを決め、タグを打つ |
| `create-release.yml` | タグから GitHub リリースを作る（リリースノートは自動生成） |

## 使い方

各リポジトリに薄いラッパーワークフローを置いて呼び出す。

`claude-review.yml` と `claude-fix.yml` は、リポジトリの関係者（`author_association` が `OWNER` / `MEMBER` / `COLLABORATOR`）と、連携する Claude のボットからのコメント・レビューのときだけ起動する。この制限は呼び出し先のワークフロー側で行うので、ラッパー側で指定する必要はない。

```yaml
# .github/workflows/claude-implement.yml
name: Implement Issue
run-name: "Implement: Issue #${{ inputs.issue_number }}"

on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: 'Issue番号'
        required: true
        type: number

jobs:
  implement:
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: zaki3mymy/actions/.github/workflows/claude-implement.yml@main
    with:
      issue_number: ${{ inputs.issue_number }}
    secrets: inherit
```

```yaml
# .github/workflows/claude-review.yml
name: Review PR
run-name: "Review: ${{ github.event.issue.title }} (#${{ github.event.issue.number }})"

on:
  issue_comment:
    types: [created]

jobs:
  review:
    if: |
      github.event.issue.pull_request != null &&
      contains(github.event.comment.body, '@claude review')
    permissions:
      contents: read
      issues: write
      pull-requests: write
    uses: zaki3mymy/actions/.github/workflows/claude-review.yml@main
    secrets: inherit
```

```yaml
# .github/workflows/claude-fix.yml
name: Fix PR
run-name: "Fix: ${{ github.event.pull_request.title }} (#${{ github.event.pull_request.number }})"

on:
  pull_request_review:
    types: [submitted]
  pull_request_review_comment:
    types: [created]

jobs:
  fix:
    if: |
      (github.event_name == 'pull_request_review' && github.event.review.state == 'changes_requested') ||
      (github.event_name == 'pull_request_review_comment' && github.event.comment.in_reply_to_id != '' && !contains(github.event.comment.user.login, 'claudeapp-zaki3mymy'))
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: zaki3mymy/actions/.github/workflows/claude-fix.yml@main
    secrets: inherit
```

### PR のタイトルからラベルを付ける（`label-pr-by-title.yml`）

PR のタイトルから、種類に応じたラベルを付ける。タイトルは `<type>(<scope>)!: <説明>` の形にする。`(<scope>)` と `!` は省略でき、説明は日本語でよい。形が違うときは、チェックが失敗する。

- 使える type: `feat` `fix` `docs` `style` `refactor` `perf` `test` `build` `ci` `chore` `revert` `release`
- develop から main へのリリース PR は `release: ...` にする。

タイトルに合わせて、次のラベルを付ける。タイトルを変えたら、合わなくなったラベルは外す（外すのはこの4つだけ）。

| タイトル | ラベル | 使いみち |
|---|---|---|
| `feat!:` など `!` 付き | `breaking` | リリースノートの「破壊的変更」、メジャーを上げる |
| `feat:` | `feat` | リリースノートの「新機能」、マイナーを上げる |
| `fix:` | `fix` | リリースノートの「バグ修正」 |
| `release:` | `skip-release-notes` | リリースノートに載せない |

Dependabot と fork の PR は、トークンが読み取り専用なのでラベルは付けない（タイトルのチェックだけを行う）。

```yaml
# .github/workflows/label-pr-by-title.yml
name: Label PR by Title
run-name: "Label PR by Title: ${{ github.event.pull_request.title }} (#${{ github.event.pull_request.number }})"

on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

jobs:
  label-pr-by-title:
    permissions:
      pull-requests: write
    uses: zaki3mymy/actions/.github/workflows/label-pr-by-title.yml@main
```

### 次のバージョンのタグ（`next-tag.yml`）

前のタグからのコミットに含まれる PR のラベルを見て、次の SemVer のバージョンを決め、注釈付きのタグを打つ。本番のデプロイが成功した後などに呼び、打ったタグを出力する。

| 前のタグからの PR | 次のバージョン |
|---|---|
| `breaking` のラベルがある | メジャーを上げる（v1.2.3 → v2.0.0） |
| `feat` のラベルがある | マイナーを上げる（v1.2.3 → v1.3.0） |
| それ以外 | パッチを上げる（v1.2.3 → v1.2.4） |

- `skip-release-notes` の PR（develop から main へのリリース PR など）は数えない。中身の個々の PR は、マージコミットで main に入るので、前のタグからのコミットに含まれる（リリース PR をスカッシュマージすると、中身の PR が数えられなくなる）。
- 対象のコミットにすでに SemVer のタグがあれば、それを出力して新しいタグは打たない（再実行しても重複しない）。
- 数える PR がなければ、タグを打たない（出力は空）。
- 前のタグには、正式なタグ（`-rc.1` などが付かないもの）だけを使う。タグが1つもなければ `initial-version`（既定は `v0.0.1`）を打つ。
- `GITHUB_TOKEN` で push したタグでは、タグの push で動くほかのワークフローは起動しない。タグを使う処理は、呼び出し元で出力のタグを使って行う。

| 入力 | 既定 | 内容 |
|---|---|---|
| `ref` | 呼び出したワークフローのコミット | タグを打つコミット |
| `initial-version` | `v0.0.1` | タグがまだ1つもないときに打つバージョン |
| `push` | `true` | `false` のときは、次のバージョンを決めて出力するだけで、タグは打たない |
| `tag` | （空） | 指定すると、PR のラベルからバージョンを決めずに、このタグを打つ。前のタグより新しいバージョンであること |

| 出力 | 内容 |
|---|---|
| `tag` | 打ったタグ（`push: false` のときは打つ予定のタグ、対象のコミットにすでにタグがあればそのタグ）。打たないときは空 |
| `previous-tag` | 対象のコミットまでに打たれた、最新の正式なタグ（対象のコミットのタグを含む）。タグが1つもなければ空 |

```yaml
# CD（.github/workflows/cd.yml）の最後に足す例
  tag:
    needs: deploy
    if: github.ref_name == 'main'
    permissions:
      contents: write
    uses: zaki3mymy/actions/.github/workflows/next-tag.yml@main

  release:
    needs: tag
    if: needs.tag.outputs.tag != ''
    permissions:
      contents: write
    uses: zaki3mymy/actions/.github/workflows/create-release.yml@main
    with:
      tag: ${{ needs.tag.outputs.tag }}
```

デプロイの前にバージョンを知りたいとき（ドキュメントに埋め込むなど）は、2回に分けて呼ぶ。1回目は `push: false` でバージョンを決めるだけにし、2回目はその値を `tag` に渡してタグを打つ。2回目で計算し直さないので、デプロイしたものと打つタグが必ず一致する。

```yaml
  version:
    if: github.ref_name == 'main'
    permissions:
      contents: write
    uses: zaki3mymy/actions/.github/workflows/next-tag.yml@main
    with:
      push: false

  deploy:
    needs: version
    # 打たないとき（数える PR がないとき）は、前のタグを使う
    env:
      VERSION: ${{ needs.version.outputs.tag || needs.version.outputs.previous-tag }}
    # ...

  tag:
    needs: [version, deploy]
    if: needs.version.outputs.tag != ''
    permissions:
      contents: write
    uses: zaki3mymy/actions/.github/workflows/next-tag.yml@main
    with:
      tag: ${{ needs.version.outputs.tag }}
```

### リリース（`create-release.yml`）

タグから GitHub リリースを作る。リリースノートは GitHub の自動生成を使い、呼び出し元のリポジトリの `.github/release.yml` で、PR のラベルごとの見出しに振り分ける。

- タグは作成済みであること（`--verify-tag` を付けるので、タグがなければ失敗する）。
- タグは SemVer の形（`v1.2.3`、`v1.2.3-rc.1`）であること。`-` の付くタグはプレリリースにする。
- 同じタグのリリースがすでにあれば、何もせずに成功する。
- ノートの生成に使われるのは、**タグのコミットにある** `.github/release.yml`。タグのコミットにこのファイルがないと、ノートは見出しなしの一覧になる。
- `GITHUB_TOKEN` で push したタグでは、ほかのワークフローが起動しない。ワークフローの中でタグを打つときは、タグの push を待たずに、同じワークフローからこのワークフローを呼ぶ。

手でタグを push したときにリリースを作るラッパーの例：

```yaml
# .github/workflows/release.yml
name: Release
run-name: "Release: ${{ github.ref_name }}"

on:
  push:
    tags:
      - "v*.*.*"

jobs:
  release:
    permissions:
      contents: write
    uses: zaki3mymy/actions/.github/workflows/create-release.yml@main
    with:
      tag: ${{ github.ref_name }}
```

`.github/release.yml` の例：

```yaml
changelog:
  exclude:
    labels:
      - skip-release-notes
  categories:
    - title: 💥 破壊的変更
      labels:
        - breaking
    - title: 🚀 新機能
      labels:
        - feat
    - title: 🐛 バグ修正
      labels:
        - fix
    - title: 📦 依存関係の更新
      labels:
        - dependencies
    - title: 🔧 その他
      labels:
        - "*"
```

ラベル（`breaking`・`feat`・`fix`・`skip-release-notes`）は zaki3mymy/github の Terraform で作っている。

## 必要なシークレット

| シークレット | 説明 |
|---|---|
| `CLAUDE_APP_ID` | claudeapp GitHub App の App ID |
| `CLAUDE_APP_PRIVATE_KEY` | claudeapp GitHub App の秘密鍵 |
| `CLAUDE_REVIEW_APP_ID` | ClaudeReview GitHub App の App ID |
| `CLAUDE_REVIEW_APP_PRIVATE_KEY` | ClaudeReview GitHub App の秘密鍵 |
| `ANTHROPIC_API_KEY` | Anthropic OAuth トークン |
