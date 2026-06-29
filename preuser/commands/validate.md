---
description: Heuristic pre-check of this repo's .preuser/config.yml — structural only; the authoritative validation runs on preuser's side when your PR opens.
---

Read the `.preuser/config.yml` at the root of the current repo and sanity-check it. This config is for
PR checks and may target a sandbox, a static URL, or a post-deploy GitHub Deployment URL. This is a
**heuristic, structural pre-check — NOT the real validator** (preuser's full schema runs server-side
when a PR opens; this just catches obvious mistakes early). If the file doesn't exist, tell the user
to run `/preuser:setup` first.

Check and report any problems:

- Top-level keys are only `target`, `up`, `journeys`, `modalities`, and `sealed`. Flag legacy
  top-level `target_url`, `url`, or `kind`.
- If `target:` is absent, treat it as `target.kind: sandbox`. If present, `target.kind` is exactly one
  of `sandbox`, `url`, or `github_deployment`; flag anything else.
- For `target.kind: sandbox`: `up:` is present. **`up.url`** looks like an `http(s)://` URL and points
  at a sandbox/local host rather than a public deployed host. Prefer `localhost`, `127.0.0.1`, `[::1]`,
  or a clearly sandbox-internal service host. If `up.url` points at Vercel/Netlify/Render/Railway/Fly/
  Heroku, GitHub Pages, staging/prod domains, or any other internet hostname, flag it as likely meant
  for `target.kind: url` or `target.kind: github_deployment`.
- For sandbox targets: **`up.run`** is present **unless** the repo has a compose file
  (`docker-compose.yml` / `compose.yaml` / `compose.yml`) — for a compose repo, `run` should be
  **absent** (preuser runs `docker compose up` itself).
- For sandbox targets: **`up.ready_timeout_s`** (if set) is an integer between 1 and 900. If it's a
  compose/cold-build repo and the timeout is the default 120, suggest raising it (≥300) so the first
  run doesn't time out into an `errored` verdict.
- For sandbox targets: **`up.env`** (if set) is a map whose keys look like POSIX env names
  (`[A-Z_][A-Z0-9_]*`) and do not start with `PREUSER_`. Values are plaintext/visible by contract;
  flag anything that looks like a real token/API key/shared password. `up.env` is only for disposable
  throwaway app variables, often a per-run seeded login that is also typeable as
  `<preuser-secret:KEY>`.
- For `target.kind: url`: `target.url` starts with `http://` or `https://`, is not localhost,
  loopback, link-local, metadata, or another obvious internal-only host; contains no CI placeholder
  syntax (`$`, `${{ ... }}`, or similar); contains no embedded username/password; and `up:` is absent.
- For `target.kind: github_deployment`: `up:` is absent; `environment` (if set) is a non-empty string;
  `wait_timeout_s` (if set) is an integer from 0 to 3600; `poll_interval_s` (if set) is an integer
  from 1 to 60. Remind the user that CI/CD must publish a successful GitHub Deployment status with
  `environment_url` or `target_url` after deployment; CI env syntax such as `$PREVIEW_URL` does not
  get expanded inside `.preuser/config.yml`; and the final URL must be reachable from preuser's hosted
  worker over public HTTP(S).
- For external URL targets, `target.auth` is optional and may contain only:
  - `basic.username_secret` + `basic.password_secret`, each naming a top-level `sealed:` entry;
  - `cookies`, a list of `{name, value_secret}`, where `name` is a simple cookie name and
    `value_secret` names a top-level `sealed:` entry;
  - `headers`, a list of `{name, value_secret}`, where `name` is a simple non-hop-by-hop request
    header name and `value_secret` names a top-level `sealed:` entry.
  Flag missing sealed references. Explain that these values are injected as browser context auth and
  are not typed by the AI user unless separately referenced by a journey.
- For external URL targets, flag unsupported network/auth fields such as `browser_auth`, top-level
  or global `headers`, `service_token`, `http_credentials`, `up.sealed_env`, raw bearer tokens, or
  credentials embedded in the URL. Explain that PR config supports Basic/cookie/header target auth
  and in-app typed credentials through `sealed:`, but does not provide VPN access or private
  allowlists.
- **`journeys`** is a non-empty list; each journey has a non-empty `name` (≤80 chars), `goal`, and
  `success`; journey `name`s are unique.
- **No RAW real secrets** (tokens, API keys, production passwords) appear anywhere in the file —
  especially not in a journey's `goal`/`success` text. A non-disposable credential is only ever
  provided as a **sealed value** under the `sealed:` map (a name → `sealed:v1:…` envelope, produced by
  the seal tool in `/preuser:setup`); a plaintext under `sealed:`, or a credential outside `sealed:`
  / disposable `up.env`, is wrong.
- Each `success` reads like a **visible end-state on one final screen** (not "a DB row exists", not a
  multi-step "and then …" sequence). If a `success` looks un-gradeable, point it out and suggest a
  fix (see `/preuser:setup` for the rules).
- The file is valid YAML with only known target keys for the selected kind; and only known `up:` keys:
  `url`, `run`, `setup`, `seed`, `health`, `ready_timeout_s`, `env`. (`secrets:` is no longer a
  field — flag it and tell the user to move the value to `sealed:` via `/preuser:setup`.)

If the target is sandbox, also suggest a local smoke test when the bring-up fields look plausible:
run the same setup/seed/run or compose command locally and curl `up.url + health`. Be explicit that
this only catches local bring-up mistakes; the authoritative run still happens in preuser's hosted
sandbox.

When relevant, remind the user that hosted preuser sees committed and pushed PR code/config, not
uncommitted local worktree edits.

End with the honest caveat: **a clean heuristic check does not mean the run will pass** — it means
the config is structurally plausible. The real schema validation happens when your PR opens, and a
journey only "passes" when the AI user actually reaches the `success` state in your live app.
