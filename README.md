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
| `pr-title.yml` | PR のタイトルが Conventional Commits の形かをチェックし、種類に応じてラベルを付ける |
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

### PR のタイトル（`pr-title.yml`）

PR のタイトルが `<type>(<scope>)!: <説明>` の形かをチェックする。`(<scope>)` と `!` は省略でき、説明は日本語でよい。形が違うときは、チェックが失敗する。

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
# .github/workflows/pr-title.yml
name: PR Title
run-name: "PR Title: ${{ github.event.pull_request.title }} (#${{ github.event.pull_request.number }})"

on:
  pull_request:
    types: [opened, edited, reopened, synchronize]

jobs:
  pr-title:
    permissions:
      pull-requests: write
    uses: zaki3mymy/actions/.github/workflows/pr-title.yml@main
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
