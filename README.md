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

## Slack app setup

Before configuring the GitHub side you need a Slack bot token and the channel ID.

### 1. Create a Slack app (skip if you already have one)

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App → From scratch**.
2. Give it a name (e.g. "PR Notifications") and pick your workspace.
3. Under **OAuth & Permissions → Scopes → Bot Token Scopes**, add **`chat:write`**.
4. Click **Install to Workspace** (or reinstall if you added scopes later) and authorize.
5. Copy the **Bot User OAuth Token** (`xoxb-...`) from the OAuth & Permissions page -- this is your `SLACK_BOT_TOKEN`.

### 2. Invite the bot to the channel

The bot **must** be a member of the channel it posts to, otherwise the Slack API returns `not_in_channel`.

- Open the target channel in Slack, then either:
  - Type `/invite @YourBotName`, or
  - Open **Channel settings → Integrations → Add apps** and add your bot.

### 3. Find the channel ID

Right-click the channel name (or open channel details) and choose **Copy link**. The ID is the last path segment (starts with `C`, e.g. `C08B0BH8R99`). You can also find it at the bottom of the **About** tab in channel details.

## Using this workflow in your repository

1. **Add the `SLACK_BOT_TOKEN` secret** in your repo (Settings → Secrets and variables → Actions → **Secrets** tab).
   Store the Bot User OAuth Token (`xoxb-...`) you copied above.

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
    permissions:
      contents: read
      pull-requests: write   # required to store Slack message timestamp in a PR comment
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

The caller workflow (`.github/workflows/slack-notification.yml`) reads all configuration from repository **variables** and **secrets**. Set them under Settings → Secrets and variables → Actions.

> **Important:** GitHub has two separate tabs -- **Secrets** and **Variables**. The workflow uses `vars.*` for non-sensitive config and `secrets.*` for the token. Putting a value on the wrong tab will cause the job to be silently skipped or fail.

**Secrets** tab (Settings → Secrets and variables → Actions → Secrets):

| Secret | Required | Description |
|--------|----------|-------------|
| `SLACK_BOT_TOKEN` | Yes | Bot User OAuth Token (`xoxb-...`) with `chat:write` scope. |

**Variables** tab (Settings → Secrets and variables → Actions → Variables):

| Variable | Required | Description |
|----------|----------|-------------|
| `SLACK_CHANNEL_ID` | Yes | Slack channel ID (e.g. `C08B0BH8R99`). |
| `SLACK_ENABLE_WAIT_FOR_CI` | No | Set to `true` to wait for CI before posting; omit or set to anything else to post immediately. |
| `SLACK_REQUIRED_CHECKS` | No | Comma-separated CI check names when `SLACK_ENABLE_WAIT_FOR_CI` is `true`. Omit or leave empty to continue without waiting for checks. |

If `SLACK_CHANNEL_ID` is not set (or placed in Secrets instead of Variables), the `notify` job is silently skipped.

### Troubleshooting

**"No Slack message timestamp found, skipping update"** (e.g. when closing a PR)

The workflow stores the Slack message timestamp in a **hidden PR comment** (`<!-- slack-review-ts:... -->`) so it can update the same message on later events (review, merge, close). If that comment was never created, the workflow can't find the message and skips the update.

- **Check the run** that first posted the Slack message (e.g. when the PR was opened). Look for the step **"Save Slack message timestamp as PR comment"**. If it **failed**, the run will show an error (e.g. missing permission). If it was **skipped**, the workflow thought it was updating an existing message (`is_new` was false) instead of posting a new one.
- **Permissions:** The workflow needs `pull-requests: write` so it can create the comment. The reusable workflow and the caller both declare this; if you use a custom caller, add `permissions: pull-requests: write` (or `issues: write`) to the job that calls the reusable workflow.
- **One-time fix:** For a PR that already has a Slack message but no comment, you can re-trigger the workflow (e.g. reopen and close again) only after the permission fix is in place so the next "open" run creates the comment.
