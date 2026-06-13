---
description: Heuristic pre-check of this repo's .preuser/config.yml — structural only; the authoritative validation runs on preuser's side when your PR opens.
---

Read the `.preuser/config.yml` at the root of the current repo and sanity-check it. This is a
**heuristic, structural pre-check — NOT the real validator** (preuser's full schema runs server-side
when a PR opens; this just catches obvious mistakes early). If the file doesn't exist, tell the user
to run `/preuser:setup` first.

Check and report any problems:

- **`up.url`** is present and looks like an `http(s)://` URL.
- **`up.run`** is present **unless** the repo has a compose file (`docker-compose.yml` /
  `compose.yaml` / `compose.yml`) — for a compose repo, `run` should be **absent** (preuser runs
  `docker compose up` itself).
- **`up.ready_timeout_s`** (if set) is an integer between 1 and 900. If it's a compose/cold-build
  repo and the timeout is the default 120, suggest raising it (≥300) so the first run doesn't
  time out into an `errored` verdict.
- **`journeys`** is a non-empty list; each journey has a non-empty `name` (≤80 chars), `goal`, and
  `success`; journey `name`s are unique.
- **No secrets** (tokens, passwords, API keys) appear anywhere in the file — especially not in a
  journey's `goal`/`success` text.
- Each `success` reads like a **visible end-state on one final screen** (not "a DB row exists", not a
  multi-step "and then …" sequence). If a `success` looks un-gradeable, point it out and suggest a
  fix (see `/preuser:setup` for the rules).
- The file is valid YAML with only the known top-level keys: `up`, `journeys`, `modalities`,
  `secrets`.

End with the honest caveat: **a clean heuristic check does not mean the run will pass** — it means
the config is structurally plausible. The real schema validation happens when your PR opens, and a
journey only "passes" when the AI user actually reaches the `success` state in your live app.
