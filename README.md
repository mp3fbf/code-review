# Claude + Codex Consensus Code Review

Dual AI code review with consensus. Both Claude (Anthropic) and Codex (OpenAI) review your PRs, and only after they agree (or Claude arbitrates disagreements) does auto-fix run.

## How It Works

```
PR Push
    │
    ├──────────────────┬──────────────────┐
    ▼                  ▼                  │
┌─────────┐      ┌─────────────┐          │
│ Claude  │      │   Codex     │          │
│ Review  │      │   Review    │          │
└────┬────┘      └──────┬──────┘          │
     │                  │                 │
     ▼                  ▼                 │
┌────────────────────────────────┐        │
│     CONSENSUS WORKFLOW         │        │
│     (waits for both)           │        │
└────────────┬───────────────────┘        │
             │                            │
    ┌────────┴────────┐                   │
    ▼                 ▼                   │
 Both Agree?      Disagree?               │
    │                 │                   │
    ▼                 ▼                   │
Post unified    Claude arbitrates         │
consensus       and posts verdict         │
    │                 │                   │
    └────────┬────────┘                   │
             │                            │
             ▼                            │
┌─────────────────────────────┐           │
│      AUTO-FIX WORKFLOW      │           │
│   (applies consolidated     │           │
│    suggestions)             │           │
└─────────────────────────────┘           │
```

## Why Consensus?

| Problem | Solution |
|---------|----------|
| Claude says "APPROVED" but Codex finds P1 bugs | Wait for both before deciding |
| Claude starts auto-fix before Codex finishes | Consensus blocks until both ready |
| Different reviewers find different issues | Consolidated list of all issues |
| Reviewers disagree | Claude arbitrates and resolves conflicts |

## Observability

| Event | Comment Posted |
|-------|----------------|
| Both approve | ✅ **AI Review Consensus: MERGE RECOMMENDED** |
| Both find issues | ⚠️ **AI Review Consensus: CHANGES REQUESTED** |
| Disagreement | 🤝 **AI Review Consensus** (arbitrated) |
| Codex timeout (not started) | ⏱️ **CLAUDE ONLY (timeout)** after 5 min |
| Codex timeout (started 👀) | ⏱️ **CLAUDE ONLY (timeout)** after 30 min |
| Auto-fix running | 🔄 **Auto-fix iteration X/3** |
| All fixed | ✅ **Auto-fix: Review approved** |

## Setup

### Prerequisites

1. **Claude GitHub App** installed ([github.com/apps/claude](https://github.com/apps/claude))
2. **ChatGPT Codex Connector** installed ([github.com/apps/chatgpt-codex-connector](https://github.com/apps/chatgpt-codex-connector))
3. **Secret**: `CLAUDE_CODE_OAUTH_TOKEN` configured

### Files to Add

Copy these files to your repository's `.github/workflows/`:

#### 1. `consensus-review.yml` (monitors both reviewers)

See [.github/workflows/consensus-review.yml](.github/workflows/consensus-review.yml)

#### 2. `consensus-retry.yml` (handles timeouts)

See [.github/workflows/consensus-retry.yml](.github/workflows/consensus-retry.yml)

#### 3. `auto-apply-review.yml` (applies fixes)

See [.github/workflows/auto-apply-review.yml](.github/workflows/auto-apply-review.yml)

### Quick Setup (Copy One File)

For a simpler setup, copy `examples/use-consensus.yml` to your repo:

```bash
# In your repo
mkdir -p .github/workflows
curl -o .github/workflows/ai-consensus.yml \
  https://raw.githubusercontent.com/mp3fbf/code-review/main/examples/use-consensus.yml
```

## Timeout Strategy

The workflow uses intelligent timeouts based on Codex's 👀 reaction:

| Scenario | Timeout | Why |
|----------|---------|-----|
| Codex reacted 👀 | 30 min | Started processing, will finish |
| No Codex activity | 5 min | Didn't see PR, won't come |
| Claude missing | 15 min | Usually responds quickly |

When timeout expires, proceeds with available review only.

## Bot Detection

| Bot | Username | Event Type | Detection |
|-----|----------|------------|-----------|
| Claude | `claude[bot]` | `issue_comment` | `"Claude finished"` |
| Codex | `chatgpt-codex-connector[bot]` | `pull_request_review` | `"Codex Review"` |

### Verdict Detection

**Claude approves if:**
- Contains `APPROVED`, `Recommendation: MERGE`, or `Excellent Work`
- No security (🔒), bug (🐛), or warning (⚠️) markers

**Codex approves if:**
- No `![P1 Badge]` in review or inline comments

## Triggering Reviews

- **Claude**: Comment `@claude review` on PR
- **Codex**: Comment `@codex review` on PR (requires "review" keyword!)

## Example Flow

```
1. Developer opens PR
2. Claude reviews: "✅ Looks good!"
3. Codex reviews: "⚠️ P1: SQL injection in auth.js"
4. Consensus: "🤝 DISAGREEMENT - Claude arbitrating..."
5. Arbitration: "Codex is correct. SQL injection is real."
6. Auto-fix: "🔄 Applying fix..."
7. Push → triggers new reviews
8. Both approve: "✅ MERGE RECOMMENDED"
```

## Files in This Repo

```
.github/workflows/
├── consensus-review.yml    # Main consensus monitor
├── consensus-retry.yml     # Timeout handler (scheduled)
└── auto-apply-review.yml   # Applies fixes after consensus
examples/
└── use-consensus.yml       # Simplified single-file setup
```

## Security Notes

- All workflows use `actions/github-script@v7` for API calls
- No untrusted input (comment bodies, PR titles) is used in shell commands
- `contains()` is used only for filtering, not interpolation
- Review bodies are written to files, not passed via shell variables

## License

MIT
