---
description: Heuristic pre-check of this repo's PR-sandbox .preuser/config.yml — structural only; the authoritative validation runs on preuser's side when your PR opens.
---

Read the `.preuser/config.yml` at the root of the current repo and sanity-check it. This file is ONLY
the **PR-sandbox** contract: preuser brings this repo up from source and drives `up.url` inside that
sandbox. A live/staging URL target belongs in the preuser Console (`/journeys` or `/run-now`), not in
this config. This is a **heuristic, structural pre-check — NOT the real validator** (preuser's full
schema runs server-side when a PR opens; this just catches obvious mistakes early). If the file
doesn't exist, tell the user to run `/preuser:setup` first.

Check and report any problems:

- There are no URL-lane top-level keys such as `target`, `target_url`, `url`, or `kind`; flag them
  and tell the user to create live URL journeys in the Console instead.
- **`up.url`** is present and looks like an `http(s)://` URL. It is the sandbox-internal URL after
  preuser starts the PR app, not a live/staging target.
- **`up.run`** is present **unless** the repo has a compose file (`docker-compose.yml` /
  `compose.yaml` / `compose.yml`) — for a compose repo, `run` should be **absent** (preuser runs
  `docker compose up` itself).
- **`up.ready_timeout_s`** (if set) is an integer between 1 and 900. If it's a compose/cold-build
  repo and the timeout is the default 120, suggest raising it (≥300) so the first run doesn't
  time out into an `errored` verdict.
- **`up.env`** (if set) is a map whose keys look like POSIX env names (`[A-Z_][A-Z0-9_]*`) and do not
  start with `PREUSER_`. Values are plaintext/visible by contract; flag anything that looks like a
  real token/API key/shared password. `up.env` is only for disposable throwaway app variables, often a
  per-run seeded login that is also typeable as `<preuser-secret:KEY>`.
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
- The file is valid YAML with only the known top-level keys: `up`, `journeys`, `modalities`,
  `sealed`. (`secrets:` is no longer a field — flag it and tell the user to move the value to `sealed:`
  via `/preuser:setup`.)

End with the honest caveat: **a clean heuristic check does not mean the run will pass** — it means
the config is structurally plausible. The real schema validation happens when your PR opens, and a
journey only "passes" when the AI user actually reaches the `success` state in your live app.
