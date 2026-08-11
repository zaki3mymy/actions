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

## 使い方

各リポジトリに薄いラッパーワークフローを置いて呼び出す。

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
      (github.event_name == 'pull_request_review_comment' && github.event.comment.in_reply_to_id != '')
    uses: zaki3mymy/actions/.github/workflows/claude-fix.yml@main
    secrets: inherit
```

## 必要なシークレット

| シークレット | 説明 |
|---|---|
| `CLAUDE_APP_ID` | claudeapp GitHub App の App ID |
| `CLAUDE_APP_PRIVATE_KEY` | claudeapp GitHub App の秘密鍵 |
| `CLAUDE_REVIEW_APP_ID` | ClaudeReview GitHub App の App ID |
| `CLAUDE_REVIEW_APP_PRIVATE_KEY` | ClaudeReview GitHub App の秘密鍵 |
| `ANTHROPIC_API_KEY` | Anthropic OAuth トークン |
