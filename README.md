# claude-review

A Claude Code reviewer that works like a teammate. It's a normal GitHub user you
pick under **Reviewers** on a pull request. It reads the PR and submits a real
review: inline comments, a summary, and **Approve**, **Request changes** or
**Comment**. Mention it in a comment and it replies.

It's built on [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action)
and runs on a Claude subscription.

## How it works

```
You request a review from the bot user on a PR
        │  (pull_request: review_requested)
        ▼
your-repo/.github/workflows/claude.yml          (small caller, about 30 lines)
        │  uses:
        ▼
Adermark/claude-review/.github/workflows/claude-review.yml
        │
        ├─ job "review"  → Claude reviews the PR and submits one review as the bot user
        └─ job "respond" → Claude replies when a comment mentions @<bot user>
```

The bot is an ordinary GitHub account (a *machine user*). The workflow acts with
that account's personal access token, so every review and comment shows up under
the bot's name, the same as when a colleague reviews. GitHub only lets you pick
users and teams as reviewers, not apps, which is why the bot needs its own account.

## Setup

### 1. Create the bot account (once)

1. Sign out of GitHub, or open a private window, and sign up at
   <https://github.com/signup>.
   - **Email:** each GitHub account needs its own email address. A Gmail alias
     works, for example `you+reviewer@gmail.com`.
   - **Username:** for example `mimer-review`. This is the name you'll pick
     under Reviewers and mention with `@`.
2. Set up two-factor authentication on the account (GitHub requires it).
3. Optional: give it a display name such as *Claude Reviewer* and an avatar, so
   it's easy to recognise.

GitHub's terms allow each person one free machine account in addition to their own.

### 2. Give the bot access to your repos

For each repo, or once for all repos of an organization:

- **Repo:** *Settings → Collaborators and teams → Add people*, add the bot with
  the **Write** role.
- **Organization:** invite the bot as a member and add it to a team that has
  **Write** access to the repos.

Then sign in as the bot and accept the invitations
(<https://github.com/notifications> or the email). The bot doesn't show up as a
possible reviewer until it has accepted.

### 3. Create the bot's token (once)

While signed in as the bot:

1. Go to *Settings → Developer settings → Personal access tokens → Tokens (classic)*
   (<https://github.com/settings/tokens>) and choose **Generate new token (classic)**.
2. **Note:** `claude-review`. **Expiration:** 1 year (set a calendar reminder).
3. **Scopes:** tick `repo`.
4. Generate it and copy the token (`ghp_…`). You can't see it again later.

A classic token works across every account whose repos the bot was invited to.
A fine-grained token is limited to one owner.

### 4. Create the Claude token (once)

On your own computer, run:

```bash
claude setup-token
```

Copy the token it prints (`sk-ant-oat01-…`). It uses your Claude Pro/Max
subscription, so reviews count against your plan's usage limits.

### 5. Turn it on in a repo

1. Copy [`templates/claude.yml`](templates/claude.yml) to
   `.github/workflows/claude.yml` and replace `YOUR-BOT-USERNAME` with the bot's
   username.
2. Under *Settings → Secrets and variables → Actions*, add two repository secrets:

   | Name | Value |
   |---|---|
   | `CLAUDE_CODE_OAUTH_TOKEN` | the token from step 4 |
   | `BOT_GITHUB_TOKEN` | the bot's token from step 3 |

   If the repo belongs to an organization and your plan supports it, you can set
   these once as organization secrets instead.
3. Merge the workflow file into the default branch.

### 6. Use it

- **Get a review:** on a PR, open **Reviewers** and pick the bot. The review
  arrives after a few minutes.
- **Review again after fixes:** click the re-request icon (🔄) next to the bot's
  name under Reviewers.
- **Ask a question:** write `@<bot user>` in a comment, for example
  `@mimer-review why is this function async?`.
- **Have it fix something:** `@mimer-review fix the issue you flagged on line 42`.
  It pushes a commit to the PR branch.

## Configuration

Inputs the caller passes under `with:`:

| Input | Required | Purpose |
|---|---|---|
| `bot_user` | yes | The bot account's GitHub username. |
| `model` | no | The Claude model to use. Defaults to the Claude Code default. |
| `extra_instructions` | no | Repo-specific review guidance, added to the end of the prompt. |

To change how the bot reviews in every repo, edit
[`.github/workflows/claude-review.yml`](.github/workflows/claude-review.yml).
Every repo that points at `@main` picks up the change on its next run. To stop
changes from reaching a repo right away, tag a release (for example `v2`) and
use `@v2` in that repo's caller.

## Notes and limits

- **It only reviews when asked.** Opening or updating a PR doesn't trigger it.
- **Approvals count.** The bot has Write access, so its Approve counts toward
  branch protection rules that require reviews. If you don't want that, give it
  the Triage role instead; it can still comment, but its approvals won't count.
- **Fork PRs are skipped**, because GitHub doesn't give secrets to workflows
  from forks.
- **The bot can push.** When asked to fix something, it pushes commits as the bot
  account. Those pushes trigger your other workflows, like a person's would.
- **Keep the token safe.** `BOT_GITHUB_TOKEN` can do anything the bot account
  can do in the repos it was invited to. If it leaks, revoke it under the bot's
  *Settings → Developer settings* and create a new one.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The bot isn't in the Reviewers list | It hasn't been added as a collaborator, or it hasn't accepted the invitation. |
| No workflow run after requesting a review | The caller file isn't on the default branch yet, or `bot_user` doesn't match the username exactly. |
| *"secret … is required"* | One of the two secrets is missing or misspelled. |
| `Bad credentials` / 401 | The bot's token expired or was revoked. Create a new one and update `BOT_GITHUB_TOKEN`. |
| Authentication error from Claude | The Claude token expired. Run `claude setup-token` again and update `CLAUDE_CODE_OAUTH_TOKEN`. |
