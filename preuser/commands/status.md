---
description: Check whether preuser's GitHub App connector is installed and selected for this repo.
---

You are checking whether **preuser's GitHub App connector** is installed for a repository. This is a
read-only preflight. Do not edit files, install the App, change GitHub settings, open a PR, or run a
smoke test from this command.

## What this command can and cannot prove

This command can prove, using preuser's first-party status endpoint, whether the GitHub App with slug
`preuser-ai` is installed on the repo owner and selected for the repo.

It cannot prove the repo will run in hosted preuser. During preview, hosted PR runs are also gated by
preuser's fail-closed server allowlist and the repo's pause state. The status endpoint reports those
server-side gates when the caller is authorized, but it still cannot prove a future PR will pass the
journey or that a fork PR's head repo is allowlisted.

## Resolve the repo

If the user supplied an `OWNER/REPO` argument, use it. Otherwise resolve the current checkout:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

If `gh` is missing, the command is not available in this environment. If `gh` is not authenticated,
ask the user to run `gh auth login` with the GitHub account that can see/manage the repo, then retry.
For org repos, the signed-in GitHub user may need access to the org installation or the selected repo.

## Check the preuser App installation

Prefer the first-party status endpoint:

```bash
repo="OWNER/REPO"
curl -fsS \
  -H "Authorization: Bearer $(gh auth token)" \
  "https://preuser.ai/api/repo-status?repo=${repo}"
```

Before running that command, tell the user it sends their current GitHub CLI token to preuser.ai for a
one-time read-only repo-access check. Do not run it silently. If they do not want to send a CLI token,
ask them to sign in with GitHub at preuser.ai and open:

```text
https://preuser.ai/api/repo-status?repo=OWNER/REPO
```

If the response has `"state": "ok"` and `"connector": {"state": "selected", ...}`, report:

- **Connector:** installed and selected for this repo.
- **Config:** report `config.state` (`valid`, `missing`, `invalid`, `unknown`, or `not_checked`).
- **Run gate:** report `run_gate.can_run_own_branch_pr`, `own_branch_prs_allowlisted`,
  `pause.state`, and any `blockers`.
- **Next:** if `can_run_own_branch_pr` is false, name the smallest blocker to clear first. If it is
  true, the next proof is an eligible PR after `.preuser/config.yml` is on the default branch.

If the connector state is not `selected`, report:

- **Connector:** not installed, not selected for the repo, or unknown, using the returned state.
- **Likely fixes:** sign in with the GitHub account that owns/admins the repo, install/select the repo
  via https://github.com/apps/preuser-ai/installations/select_target, or ask an org admin to select it.
- **Caveat:** `state: forbidden` means the presented GitHub identity cannot view the repo; it does not
  prove the App is absent.

## Optional local context

If you are inside the repo, also mention whether `.preuser/config.yml` exists locally. Do not call a
local file "active" unless it is already on the default branch; preuser reads the config from the
repo's default branch when a PR opens.

End with a compact status summary, not a transcript of every `gh api` call.
