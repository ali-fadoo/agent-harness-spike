# agent-harness-spike

A spike to find out whether GitHub Actions can run an agent that makes a code change and opens a pull request without a person doing it by hand.

The goal is not to build the harness. It is to prove the delivery mechanism works and to find out where it breaks.

## Status

| Part | State |
|---|---|
| Self-hosted runner picks up jobs | Working |
| Checkout, branch, commit, push | Working |
| Opening a pull request from CI | Working |
| OIDC token exchange for git auth | Working |
| Claude Code step | Blocked, see below |

The pipeline is proven end to end except the model call.

## What is in here

### `.github/workflows/pipeline-check.yml`

Does the whole job with a plain shell script and no model. Creates a branch, writes `spike-test.txt`, commits, pushes, and opens a pull request using the GitHub CLI.

This exists so the plumbing can be tested for free. If this one goes red, the problem is the pipeline. If this one is green and the other is red, the problem is the model step.

### `.github/workflows/claude-spike.yml`

The real spike. Uses `anthropics/claude-code-action@v1` and hands it a prompt asking for the same outcome. The model decides how to get there instead of following a script.

Currently fails on the first API call. See "What is blocked".

Both workflows are manual only. Neither runs on push or pull request.

## Running it

You need three things in place first.

1. The runner has to be alive. Open a terminal and run `cd ~/actions-runner && ./run.sh`. Leave the window open. If you close it, jobs will queue and never start.
2. The GitHub CLI has to be installed on the runner machine. Check with `gh --version`. Install with `brew install gh`.
3. Settings, Actions, General, Workflow permissions, and tick "Allow GitHub Actions to create and approve pull requests". Without this the push works and the pull request step fails.

Then go to the Actions tab, pick the workflow, and hit "Run workflow".

A successful run leaves behind a branch named `spike/plumbing-<timestamp>`, an open pull request, and a file called `spike-test.txt` containing `abc`.

## Permissions

Both workflows need these at the job level.

```yaml
permissions:
  contents: write
  pull-requests: write
  id-token: write
```

`id-token: write` is what lets the action request an OIDC token and swap it for a short-lived GitHub token. Nothing long-lived is stored in the repo.

## What is blocked

`claude-spike.yml` fails with a 400 from the API.

```
"error": "billing_error",
"result": "Credit balance is too low"
```

The key itself is fine. A bad key returns 401, not a billing error. The account behind the key has no credits.

Two ways to unblock it.

- Get a key from a funded Anthropic account. This is the better option for anything team-owned, since the credential does not belong to one person and can be rotated without affecting anyone's own work.
- Use a Claude subscription instead. Run `claude setup-token` and set the result as `CLAUDE_CODE_OAUTH_TOKEN` in place of the API key. Note that if both are set, the API key wins, so it has to be a swap and not an addition.

Nothing was charged for any of the failed runs. Cost came back as zero every time.

## Known limitations

The runner is a personal MacBook. That means this is not reproducible on anyone else's machine, and it stops working when the laptop sleeps or the terminal closes. It also caused a real failure during setup, because Claude Code read the model preference out of the local config instead of using a default. The model is now pinned in the workflow to stop that happening again.

The repo sits on a personal GitHub account rather than the org.

The prompt is hardcoded in the workflow file. Fine for a spike, not a design.

## What this does not answer

This is plumbing, not a harness. It says nothing about what tasks an agent should be given, how work gets scoped, what it is allowed to touch, or where a human signs off. Those are the design questions. This just shows the delivery path exists and works.
