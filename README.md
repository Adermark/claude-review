# claude-review

A shared Claude Code PR reviewer for all Adermark repos. It automatically reviews
every pull request with inline comments on bugs, security issues and convention
violations, and replies when you mention `@claude`. To turn it on in a repo, add
one small workflow file.

It's built on [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action).

## What it does

| | |
|---|---|
| **Automatic review** | It reviews every non-draft PR when it's opened, updated, reopened or marked ready. It posts inline comments on real problems and one short summary comment. |
| **`@claude` on demand** | Mention `@claude` in a PR comment, a review or an issue to ask a question, request another review or have it make a fix. |
| **Follows your rules** | It reads each repo's `CLAUDE.md` / `AGENTS.md` and reviews against those conventions. |
| **Low noise** | It skips formatting and naming nitpicks and anything a linter catches. It flags only problems it's confident about. |
| **One place to change** | The review logic lives only in this repo, and every repo picks up changes on its next run. |

## How it works

```
your-repo/.github/workflows/claude.yml        (small caller, about 20 lines)
        │  uses:
        ▼
Adermark/claude-review/.github/workflows/claude-review.yml   (the actual bot)
        │
        ├─ job "review"  → runs on pull_request events
        └─ job "respond" → runs when a comment contains @claude
```

GitHub runs only workflows that a repo contains itself, so each repo needs the
small caller file. The caller hands control to the reusable workflow here. That
workflow runs inside the calling repo, so it sees that repo's code and secrets.

## Setup

### Once

1. **Make this repo public.** It contains no secrets, and being public lets repos
   under any account call it. If you keep it private, open
   *Settings → Actions → General → Access* and choose
   *"Accessible from repositories owned by Adermark"*. Only Adermark repos can
   use it then.
2. **Install the Claude GitHub App:** <https://github.com/apps/claude>. Install it
   on every account or org whose repos should be reviewed and choose
   *All repositories*. The bot then comments as `claude[bot]`.
3. **Create a Claude token.** In a terminal, run:
   ```bash
   claude setup-token
   ```
   Copy the token it prints (it's valid for about a year). It uses your Claude
   Pro/Max subscription, so reviews count against your plan's usage limits.

### For each repo

1. Copy [`templates/claude.yml`](templates/claude.yml) to
   `.github/workflows/claude.yml` in the repo.
2. Add the token as a secret named `CLAUDE_CODE_OAUTH_TOKEN` under
   *Settings → Secrets and variables → Actions*. If the repo belongs to an org
   and your plan supports it, you can set this once as an org secret instead.
3. Open a PR. The review shows up within a minute or two.

## Configuration

Each caller can pass these optional inputs:

```yaml
jobs:
  claude:
    uses: Adermark/claude-review/.github/workflows/claude-review.yml@main
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      model: claude-opus-5-5
      extra_instructions: |
        This is a Python/Flask app; pay extra attention to SQL injection.
```

| Input | Default | Purpose |
|---|---|---|
| `model` | Claude Code default | The Claude model to use. |
| `extra_instructions` | *(empty)* | Repo-specific review guidance, added to the end of the prompt. |
| `trigger_phrase` | `@claude` | The word that summons the bot in comments. |

To change the review prompt for every repo, edit
[`.github/workflows/claude-review.yml`](.github/workflows/claude-review.yml).

## Using `@claude`

Some examples of comments it responds to:

- `@claude review again`
- `@claude why is this function async?`
- `@claude fix the issue you flagged on line 42`
- `@claude add a test for the empty-playlist case`

Only people with write access to the repo can trigger it.

## Notes and limits

- **Fork PRs are skipped**, because GitHub doesn't give secrets to workflows
  from forks.
- **A new push cancels a review that's still running** for that PR, so you only
  get feedback on the latest code.
- **Too many comments?** Remove `synchronize` from the caller's `pull_request`
  types. The bot then reviews only when a PR is opened, and you can write
  `@claude review again` when you want another pass.
- **Review-only mode:** in the caller, change `contents: write` to
  `contents: read` so `@claude` can never push commits.
- **Pin a version:** to stop changes here from reaching a repo right away, tag a
  release (for example `v1`) and use `@v1` instead of `@main` in that repo's caller.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The workflow fails with *"secret CLAUDE_CODE_OAUTH_TOKEN is required"* | The repo secret is missing or misspelled. |
| *"workflow was not found"* / access error | This repo is private and the calling repo isn't allowed to use it. |
| The workflow is green but no comments appear | The Claude GitHub App isn't installed on that repo. |
| Authentication error from Claude | The token expired or was revoked. Run `claude setup-token` again and update the secret. |
| Nothing happens on a PR | The PR is a draft or comes from a fork. |
