# GitHub PR notifications for Slack

Reusable GitHub Actions workflow that posts **one Slack message per PR** and **keeps it updated** as the PR moves through its lifecycle. The same message is edited in place so the channel stays tidy and always shows the current status.

**How it works:**

| PR event | Slack message |
|----------|----------------|
| Opened / Reopened / Ready for review | Posts **Code Review Requested** (author, branch, link). |
| Reviewer requests changes | Updates the message to **Changes Requested** (adds reviewer). |
| PR merged | Updates the message to **Merged** (adds merged-by). |
| PR closed (not merged) | Updates the message to **Closed**. |
| PR converted to draft | Removes the Slack message and the hidden timestamp comment. |

So the channel sees a single, up-to-date line per PR instead of multiple separate notifications.

The workflow paginates through PR comments to find the hidden timestamp that links the Slack message to the PR, so it works for PRs with many comments (up to 10 pages of 100 comments).

## Using this workflow in your repository

1. **Add the `SLACK_BOT_TOKEN` secret** in your repo (Settings → Secrets and variables → Actions):
   - Create a Slack app or use an existing bot with the `chat:write` OAuth scope.
   - Invite the bot to the channel you want to post to.
   - Store the Bot User OAuth Token as `SLACK_BOT_TOKEN`.

2. **Create a workflow file** in your repo (e.g. `.github/workflows/slack-pr-notify.yml`) that calls this reusable workflow:

```yaml
name: Notify Slack on PR
on:
  pull_request:
    types: [opened, reopened, ready_for_review, converted_to_draft, closed]
  pull_request_review:
    types: [submitted]
jobs:
  notify:
    uses: OWNER/github-pr-slack/.github/workflows/slack-pr-notification-reusable.yml@main
    with:
      slack_channel_id: 'C09HB1Q3J69'
      # Optional: wait for CI before posting (opt-in; supply your check names)
      enable_wait_for_ci: true
      required_checks: 'tests,lint,prettier'
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

Replace `OWNER` with the GitHub org or user that hosts this repo (e.g. `nicasio` or `my-org`).

You can also use **repository variables** instead of literal values (e.g. `slack_channel_id: ${{ vars.SLACK_CHANNEL_ID }}`) so the same repo can drive the workflow from Settings → Variables:

```yaml
    with:
      slack_channel_id: ${{ vars.SLACK_CHANNEL_ID }}
      enable_wait_for_ci: ${{ vars.SLACK_ENABLE_WAIT_FOR_CI == 'true' }}
      required_checks: ${{ vars.SLACK_REQUIRED_CHECKS }}
```

### Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `slack_channel_id` | Yes | Slack channel ID (e.g. `C00HB0Q0J00`). |
| `required_checks` | No | Comma-separated CI check names to wait for. Only used when `enable_wait_for_ci` is true. If empty when wait is enabled, the workflow continues without waiting. |
| `enable_wait_for_ci` | No | Set to `true` to wait for CI checks to pass before posting. If false or omitted, the workflow posts to Slack immediately. |

**CI wait is opt-in.** Omit `enable_wait_for_ci` and `required_checks` to post to Slack immediately. To wait for CI first, set `enable_wait_for_ci: true` and `required_checks` to your repo’s check names (comma-separated).

### Secret

- **`SLACK_BOT_TOKEN`** (required): Slack Bot User OAuth Token with `chat:write`; the bot must be invited to the target channel.

### Configuring this repo (caller workflow)

The caller workflow (`.github/workflows/slack-notification.yml`) reads all configuration from repository variables. Set them under Settings → Secrets and variables → Actions → Variables (no defaults; all must be set as needed):

| Variable | Required | Description |
|----------|----------|-------------|
| `SLACK_CHANNEL_ID` | Yes | Slack channel ID (e.g. `C09HB1Q3J69`). |
| `SLACK_ENABLE_WAIT_FOR_CI` | No | Set to `true` to wait for CI before posting; omit or set to anything else to post immediately. |
| `SLACK_REQUIRED_CHECKS` | No | Comma-separated CI check names when `SLACK_ENABLE_WAIT_FOR_CI` is `true`. Omit or leave empty to continue without waiting for checks. |

If `SLACK_CHANNEL_ID` is not set, the `notify` job is skipped (no Slack API calls are made).
