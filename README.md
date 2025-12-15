# Claude Auto-Fix: Automated Code Review Loop

Automatically applies Claude's code review suggestions to your PRs.

## How It Works

```
PR Push → Claude Reviews → Auto-Fix Applies → Push → Claude Reviews Again → Loop until done
```

1. Claude Code Review posts suggestions on your PR
2. Auto-Fix workflow detects the review and applies fixes
3. Commits with `[auto-fix]` tag and pushes
4. New push triggers another review
5. Loop stops when: all issues fixed OR 3 iterations reached

## Observability

The workflow provides clear feedback on every outcome:

| Scenario | Comment Posted |
|----------|----------------|
| Review approved (no issues) | ✅ **Auto-fix: Review approved, no fixes needed** |
| Applying fixes | 🔄 **Auto-fix iteration X/3** - Applying review suggestions... |
| Analyzed but no changes | ✅ **Auto-fix: Analyzed, no changes needed** |
| Max iterations reached | ⚠️ **Auto-fix limit reached (3 iterations)** |

## Setup (2 files needed)

### 1. Claude Code Review Workflow

Create `.github/workflows/claude-code-review.yml`:

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - uses: anthropics/claude-code-action@beta
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          allowed_bots: "claude"  # REQUIRED for auto-fix loop
          direct_prompt: |
            Please review this pull request and provide feedback on:
            - Code quality and best practices
            - Potential bugs or issues
            - Performance considerations
            - Security concerns
```

### 2. Auto-Fix Workflow

Create `.github/workflows/claude-auto-fix.yml`:

```yaml
name: Claude Auto Fix

on:
  issue_comment:
    types: [created, edited]  # Must include 'edited'!

jobs:
  apply-review:
    # Matches both "Code Review" and "Pull Request Review" (output format varies)
    if: |
      github.event.issue.pull_request &&
      github.event.comment.user.login == 'claude[bot]' &&
      contains(github.event.comment.body, 'Claude finished') &&
      (contains(github.event.comment.body, 'Code Review') || contains(github.event.comment.body, 'Pull Request Review'))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write

    steps:
      - name: Get PR branch
        id: pr-info
        uses: actions/github-script@v7
        with:
          script: |
            const pr = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            core.setOutput('branch', pr.data.head.ref);

      - name: Check iteration count
        id: check-iterations
        uses: actions/github-script@v7
        with:
          script: |
            const commits = await github.rest.pulls.listCommits({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            const autoFixCount = commits.data.filter(c =>
              c.commit.message.includes('[auto-fix]')
            ).length;
            if (autoFixCount >= 3) {
              core.setOutput('should_continue', 'false');
            } else {
              core.setOutput('should_continue', 'true');
              core.setOutput('iteration', String(autoFixCount + 1));
            }

      - name: Post starting comment
        if: steps.check-iterations.outputs.should_continue == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            const iteration = '${{ steps.check-iterations.outputs.iteration }}';
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '🔄 **Auto-fix iteration ' + iteration + '/3** - Applying review suggestions...'
            });

      - uses: actions/checkout@v4
        if: steps.check-iterations.outputs.should_continue == 'true'
        with:
          ref: ${{ steps.pr-info.outputs.branch }}
          fetch-depth: 0

      - name: Configure git
        if: steps.check-iterations.outputs.should_continue == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Apply Review Suggestions
        if: steps.check-iterations.outputs.should_continue == 'true'
        uses: anthropics/claude-code-action@beta
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          allowed_bots: "claude"  # REQUIRED
          direct_prompt: |
            Apply the code review suggestions from the review comment on this PR.

            Focus on fixing:
            - Security issues (marked with warning symbols)
            - Potential bugs (marked with bug symbols)
            - Performance improvements that have code suggestions

            Skip:
            - Items already marked as good/done
            - Pure style suggestions
            - Test recommendations

            After applying fixes:
            1. Stage changes: git add -A
            2. Commit with message: fix: apply review suggestions [auto-fix]
            3. Include brief list of what was fixed in commit body

      - name: Push changes
        if: steps.check-iterations.outputs.should_continue == 'true'
        run: git push origin HEAD:${{ steps.pr-info.outputs.branch }} || echo "Nothing to push"

      - name: Comment if max iterations reached
        if: steps.check-iterations.outputs.should_continue == 'false'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '⚠️ **Auto-fix limit reached (3 iterations)**\n\nPlease review manually.'
            });
```

## Requirements

1. **Secret**: `CLAUDE_CODE_OAUTH_TOKEN` configured in your repo
2. **Both workflows need** `allowed_bots: "claude"` - this allows the loop to work

## Key Discoveries

| Problem | Solution |
|---------|----------|
| `claude-code-action` creates comment then edits it | Trigger on `issue_comment: [created, edited]` |
| Initial comment has no review content | Check for `"Claude finished"` AND (`"Code Review"` OR `"Pull Request Review"`) |
| Bot-triggered workflows blocked | Add `allowed_bots: "claude"` to BOTH workflows |
| No visibility into what happened | Added observability comments for all outcomes |

## Example Output

```
PR Push
  ↓
Claude: "### Pull Request Review ... ⚠️ Security issue found..."
  ↓
Auto-fix: "🔄 Auto-fix iteration 1/3 - Applying..."
  ↓
Claude: "### Applied Code Review Suggestions ✅ ... fixed security issue"
  ↓
Push triggers new review
  ↓
Claude: "### Pull Request Review ... ✅ All issues resolved"
  ↓
Loop stops (nothing more to fix)
```

## License

MIT
