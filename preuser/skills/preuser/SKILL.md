---
name: preuser
description: Set up, validate, and rescue preuser PR checks in a repository. Use when Codex is asked to configure preuser, write or update .preuser/config.yml, choose sandbox vs static URL vs GitHub Deployment targets, author natural-language journeys, check the preuser GitHub App connector, seal credentials, run a sandbox smoke test, triage a failed/errored preuser run, or send consented setup feedback to preuser.
---

# Preuser

## Overview

preuser is a CI check whose "test" is an AI user walking a real web-app workflow and posting a
pass/fail with a video receipt. This skill gives Codex the same setup and rescue behavior as the
Claude Code `/preuser:*` plugin commands.

## First Response

When the user asks for setup, briefly orient them before probing the repo:

> preuser is a UX guardrail for PRs: an AI user tries a real workflow in your app and leaves a
> video receipt. You're in good hands; I'll read the repo, gather the launch/auth intel, and get this
> wired up without making you babysit the setup.

Then ask the routing question before journey questions: whether PRs already deploy a preview/staging
app, or whether preuser should bring the app up from source in its sandbox. If there is a deployment,
ask whether it is one stable URL or a unique URL per PR/branch.

Keep the setup collaborative and concise. This skill means the user wants setup done: inspect the
repo, explain what you found, write `.preuser/config.yml` when the path is clear, and show the diff.
Ask only for real decisions or sensitive actions: target choice when ambiguous, secrets, local code
execution, optional repo-agent-note edits, commits, pushes, or PRs.

## Workflow Map

Read the referenced workflow before acting. These files are shared with the Claude Code plugin so
the two agent surfaces stay aligned.

- **Setup or update `.preuser/config.yml`:** read `../../commands/setup.md`.
- **Validate existing config shape:** read `../../commands/validate.md`.
- **Check GitHub App connector/status:** read `../../commands/status.md`.
- **Seal a login or preview-gate value:** read `../../commands/seal.md`.
- **Rescue a setup, PR check, or run:** read `../../commands/rescue.md`.
- **Send consented setup/rescue feedback:** read `../../commands/feedback.md`, then apply the Codex
  feedback override below.

Translate Claude command language naturally:

- A reference to `/preuser:setup` means "this setup workflow."
- A reference to `/preuser:validate`, `/preuser:status`, `/preuser:seal`, `/preuser:rescue`, or
  `/preuser:feedback` means the matching workflow above.
- `$ARGUMENTS` means the user-supplied wording in the current request.
- If a workflow tells the user to "run `/preuser:*`", say the equivalent Codex request instead, such
  as "ask Codex to use $preuser to validate the config."

## Non-Negotiables

- Never write raw real credentials into `.preuser/config.yml` or any other committed file.
- Use top-level `sealed:` values for non-disposable PR-run credentials.
- Treat `up.env` as plaintext disposable sandbox app configuration only.
- Do not invent unsupported config fields such as `secrets`, `browser_auth`, global headers,
  `service_token`, `http_credentials`, raw bearer tokens, or `up.sealed_env`.
- Do not commit, push, open a PR, edit CI, or run local smoke commands unless the user explicitly
  asks or approves that separate action.
- For sandbox targets, strongly recommend the runner smoke path after writing config so the launch
  environment is verified rather than guessed.
- For hosted run evidence, use the PR Check/comment and the run page. Debug logs, when captured, are
  private run artifacts available only to authorized viewers on the run page.

## Codex Feedback Override

When sending setup/rescue feedback from Codex, use this client marker instead of the Claude command's
marker:

```text
X-Preuser-Feedback-Client: preuser-codex-plugin:v0.5.0
```

The marker is public spam friction, not authentication. The backend accepts future semver-shaped
markers for both Claude and Codex plugin surfaces.

## End State

For setup, end by showing the written config or diff, explaining the next proof path, and offering
to open a PR only if the user wants. If a PR is opened and you have GitHub access, offer to monitor
the preuser Check/comment and link the user to the journey receipts.
