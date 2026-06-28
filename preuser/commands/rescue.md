---
description: Rescue a preuser setup or run — triage .preuser/config.yml, deployment handoff, reachability, auth, and the PR/run evidence.
---

You are helping the user diagnose a preuser PR check or config. Be concrete and evidence-driven.
Do not edit files unless the user explicitly asks for a fix after the diagnosis.

## Start with the minimal context

Ask for any missing item you need, but first inspect what is local:

- `.preuser/config.yml`, if present.
- The PR URL or branch/commit SHA, if the issue happened on a PR.
- The preuser PR Check/comment or run page URL, if they have one.
- Whether this is a sandbox target, a static URL target, or a GitHub Deployment target.

State the capture warning when asking for run evidence: the run page/video/timeline may show app
screens and model activity, so they should share only runs from test/staging data.

## Local config triage

Run the same heuristic checks as `/preuser:validate`:

- only top-level `target`, `up`, `journeys`, `modalities`, `sealed`;
- `target.kind` is absent/`sandbox`, `url`, or `github_deployment`;
- sandbox targets have `up:` and external targets omit `up:`;
- journeys are Shape C: one `name`, one `goal`, one visible final-screen `success`;
- no raw secrets in config; real credentials are only `sealed:v1:...` under `sealed:`;
- no legacy or unsupported fields: `secrets`, top-level `url`, `target_url`, `kind`,
  `browser_auth`, top-level/global `headers`, `service_token`, `http_credentials`, raw bearer tokens,
  `up.sealed_env`;
- if an external target uses `target.auth`, it is only `basic`, `cookies`, and/or `headers`, and each
  `*_secret` / `value_secret` points at a top-level `sealed:` entry.

If the config has obvious problems, give the smallest patch suggestion. Do not rewrite the whole file
unless the user asks.

Mention the default-branch invariant when relevant: preuser reads `.preuser/config.yml` from the repo's
default branch, so an unmerged PR that changes only this config is an activation PR, not proof that the
changed config has already run.

## Classify the failure

Use the evidence to put the issue in exactly one primary bucket, then list secondary suspects.

1. **No preuser run appeared.**
   Check whether the GitHub App is installed, the repo is approved in preview, the config is on the
   default branch, and the PR targets the repo whose default branch preuser reads. Tell the user that
   discovery/Console setup never loosens the fail-closed repo allowlist.

2. **Config error.**
   Prefer the preuser PR comment/Check summary if present; that is the authoritative server-side
   schema result. Local validation is only a heuristic.

3. **Sandbox bring-up failed or timed out.**
   Inspect `up.url`, `up.health`, `up.run`, `setup`, `seed`, compose presence, and `ready_timeout_s`.
   Common fixes: app binds `0.0.0.0` not only `127.0.0.1`; compose repos omit `up.run`; slow cold
   compose builds use `ready_timeout_s: 300` or higher; `up.env` contains every disposable boot var
   the app needs. Offer the runner smoke image first, filling in the config's URL and health:
   `docker run --rm --privileged -v "$PWD:/workspace" -e PREUSER_UP_URL=<up.url> -e
   PREUSER_UP_HEALTH=<up.health-or-/> ghcr.io/preuser-ai/preuser-runner-smoke:latest`. If the
   public image pull fails, fall back to running the same setup/seed/run or compose command and
   curling `up.url + health`. Say clearly this is not a hosted preuser verdict.

4. **External target reachability/auth failed.**
   For `target.kind: url`, confirm `target.url` is literal public HTTP(S), not localhost/private/
   internal/VPN/metadata/link-local and not CI syntax. For `target.kind: github_deployment`, confirm
   the resolved deployment URL has the same public reachability. Credentials must not be embedded in
   the URL. If there is a pre-app gate, PR config supports HTTP Basic auth, cookies, and same-origin
   request headers through `target.auth` with values in top-level `sealed:`; VPN access and private
   allowlists are still unsupported.

5. **GitHub Deployment URL timed out.**
   Confirm the deploy job publishes a successful GitHub Deployment status for the PR head SHA, with
   `environment_url` or `target_url`, after the deploy succeeds. If `environment` is configured,
   check it exactly matches the deployment environment name. Warn about deadlock: deploy must not
   wait for the preuser Check while preuser waits for deploy. If the URL appears after preuser's
   timeout, rerun preuser or raise `wait_timeout_s`.

6. **In-app auth/login failed.**
   Distinguish in-app typed credentials from pre-app access gates. `sealed:` and `up.env` let the AI
   user type values into the app UI. `target.auth.basic`, `target.auth.cookies`, and
   `target.auth.headers` use sealed values to unlock supported pre-app gates before the page loads;
   VPN access and private allowlists are still unsupported. For a wrong sealed value, wrong repo
   binding, or renamed repo, re-run `/preuser:seal NAME` for the base/default-branch repo.

7. **The journey failed.**
   Read the run page video/timeline and compare the final frame to `success`. Fix journey text when
   `success` is not visible on one final screen, includes an "and then" sequence, or asserts backend
   state the grader cannot see.

8. **Infrastructure/error verdict.**
   Treat `errored` as preuser/harness/setup trouble, not an app-under-test failure. Do not tell the
   user to debug their app behavior until the bring-up/deployment/auth evidence supports that.

## First-party feedback loops

Use these in order:

1. Local heuristic: `/preuser:validate` or the checks above.
2. PR Check/comment: authoritative config errors and the run link.
3. Run page: live status while running; final video, timeline, screenshots, result, and private
   Debug logs when worker/sandbox logs were captured and the viewer is authorized.
4. For support/escalation: collect the run page URL or run id, PR URL, commit SHA, target kind, and
   the relevant config snippet.

The plugin itself does not fetch cluster logs. Use the run page's Debug logs card when it appears.
If it does not appear, collect the run page URL or run id, PR URL, commit SHA, target kind, and the
relevant config snippet for support.

## End state

End with:

- the primary failure bucket;
- the smallest config/deploy/auth change to try next;
- what evidence would confirm the fix on the next run.
