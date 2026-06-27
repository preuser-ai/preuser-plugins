---
description: Set up preuser PR-sandbox checks on this repo — interview the user and draft a .preuser/config.yml (app bring-up + natural-language journeys an AI user walks once per PR).
---

You are helping the user set up **preuser PR-sandbox checks** on the repository you're currently in.
preuser can run on a PR sandbox or against a live/staging URL; this command is ONLY for the
PR-sandbox lane. Its job is to **draft a `.preuser/config.yml`** with them — the app bring-up block
plus one or more natural-language journeys — and then, only if they say so, open a PR. If the user
wants to point preuser at an already-running live/staging URL, stop drafting config and send them to
the preuser Console (`/journeys` or `/run-now`); do not add a `target_url` to `.preuser/config.yml`.

Work conversationally. Draft, show, confirm, then write. Do not rush to a file.

## HARD RULES — these are non-negotiable, do not violate them even if asked

1. **Never write a RAW real credential value** (a password, token, API key, or any plaintext secret)
   into `.preuser/config.yml` or any other file — that file is read from the repo's default branch and
   shown verbatim in the PR receipt. The **ONLY** way to give preuser a non-disposable credential is a
   **SEALED value in the `sealed:` map**, produced by running the committed seal tool (Step 2.5 below).
   The one exception is `up.env` for **disposable, throwaway app variables** that intentionally boot a
   per-run test account; `up.env` is plaintext and visible by contract, so never put a real secret
   there. There is no `secrets:` field anymore; if you see one in an old config, it's gone.
2. **Draft-then-confirm.** Show the user the COMPLETE proposed `.preuser/config.yml` and get their
   explicit "yes, write it" before you create or modify any file.
3. **Never `git commit`, `git push`, or open a pull request unless the user explicitly tells you to
   in this conversation.** "Offer to open a PR" means *ask* — never do it by default.
4. **Only ever write `.preuser/config.yml`.** Do not touch `.env`, CI workflow files, `package.json`,
   Dockerfiles, or anything else. preuser needs no changes to the user's existing code or workflows.
5. **No URL-lane keys in config.** Do not write top-level `target`, `target_url`, `url`, `kind`, or
   anything similar. `.preuser/config.yml` proves PR code by booting that PR in a sandbox; a fixed
   live URL belongs in the Console.

## First, tell the user this (the honesty up front)

Before drafting anything, say plainly:

> ⚠️ **What preuser captures.** When preuser runs, it records the AI user's full screen (a video) and
> the model's inputs/outputs — that's the receipt that lets you trust the verdict. So: **don't put
> tokens, real personal data, or anything you wouldn't want on screen into a journey's `goal` or
> `success` text**, and assume anything the agent sees on screen during a run is in the recording.
> The one exception is a **sealed credential**: a value provided via the `sealed:` map (Step 2.5) is
> substituted into the page only at the browser and is **redacted from the model I/O and the saved
> bundle** — so a test-account password handled that way does not appear in the captured model log.
> A value in `up.env` is different: it is plaintext app configuration for the disposable sandbox and
> is visible in config/receipts, so only use it for throwaway per-run values.
> The Check preuser posts is **advisory** (it doesn't block your merge) during preview.

## Step 1 — Understand how the app runs

Look at the repo to figure out how to bring the app up and what URL it serves on. Check for a
`docker-compose.yml` / `compose.yaml`, a `package.json` (scripts), a `Procfile`, a framework
(Next.js, Rails, Django, etc.). Then draft the `up:` block:

- `url` (**required**): the base URL the app serves on **inside the PR sandbox after preuser starts
  it**, e.g. `http://localhost:3000`. The agent drives this and uses it as the readiness gate. This is
  NOT a live/staging URL target. If you detect a public deployed host here (Vercel/Netlify/Render/
  Railway/Fly/Heroku, GitHub Pages, staging/prod domains, or any other internet hostname), stop and
  send the user to the Console unless they can explain why that host only exists inside the sandbox.
- `run`: the long-running serve command, e.g. `npm run start`. **Omit `run` entirely if the app is
  brought up by a compose file** (preuser runs `docker compose up` itself) — including `run` for a
  compose repo is wrong.
- `setup` (optional): one-time install/build, e.g. `npm ci && npm run build`. **Ask the user** —
  don't assume; build commands vary and a wrong one wastes the first run.
- `seed` (optional): migrate/seed fixtures (runs after `setup`, before `run`), e.g. a DB migrate.
  **Ask the user** whether the app needs seeding to be walkable — don't auto-fill it.
- `health` (optional, default `/`): a path appended to `url` that returns 2xx when the app is ready.
- `ready_timeout_s` (optional, default 120, max 900): how long to wait for `health` to go green.
  **For compose repos or anything with a cold multi-service build, suggest raising this to at least
  300** — a 120s default routinely times out on first build and produces an `errored` verdict that
  isn't the app's fault.
- `env` (optional): plaintext app variables for the throwaway PR sandbox. Use this when the app seeds
  a disposable login from env at boot (e.g. `DEMO_ADMIN_EMAIL`, `DEMO_ADMIN_PASSWORD`). Each key is
  also typeable by the agent as `<preuser-secret:KEY>`. **Warn clearly:** `up.env` is committed,
  visible, and shown in receipts — use only disposable per-run values, never API keys, production
  credentials, shared passwords, or customer data.

Be honest that you're **guessing** the build/seed/run commands from the repo — present them as a
draft and ask the user to confirm or correct each before you rely on it.

## Step 2 — Teach what makes a *gradeable* journey (this is the part that decides success)

A journey is one `goal` + one `success`, both plain English. The single thing that makes or breaks a
run is the **`success` criterion**, because a separate grader judges it against the **final frame** of
the walk. Teach the user these three rules, with examples:

- **It must be visible on one final screen ("shows, not stores").** The grader sees the last frame,
  not your database. ✅ gradeable: *"the new comment appears in the Comments list under the user's
  name."* ❌ not gradeable: *"a row is inserted into the comments table"* (the grader can't see your DB).
- **One end-state, not a sequence. "and then" = a second journey.** ✅ *"the dashboard shows the
  signed-in user's email."* ❌ *"sign up and then post a thread and then log out"* — split that into
  separate journeys (preuser runs each independently).
- **Describe the goal, not the clicks.** Say *what the user accomplishes*, not a click-by-click
  script — the AI user figures out the UI like a human. ✅ *"publish a new forum thread and see it in
  the Threads list."*

Then help them write **one** journey: a short `name` (≤80 chars, a stable handle), a `goal` (what the
user does), and a `success` (the visible end-state). Start with one good journey; more can be added
later as more list entries.

## Step 2.5 — A login the app can't self-signup for (only if a journey needs one)

Most journeys don't need this — skip it entirely if every journey can sign up or browse as a fresh
visitor. Do this step **only** when a journey requires logging in as an account the agent can't create
itself (a seeded admin, a pre-existing test user). If the app itself creates that disposable account
from env at boot, prefer `up.env` from Step 1 instead of sealing the same value twice. Use `sealed:`
for a repo-bound test credential that must not be visible in git/receipts. Then:

1. **Ask the user ONE question, in plain words** — no crypto vocabulary. For each credential the login
   needs, ask e.g.: *"What's the test-account password the AI user should log in with?"* (and the
   username/email if that's not already a fixed value you can put in the `goal`). The plaintext stays
   on their machine — say so. Pick a short snake_case name for each value, e.g. `admin_pass`,
   `admin_user`.
2. **Confirm the repo the value binds to.** It's sealed to the **base / default-branch repo**
   (`owner/name`) — the repo preuser reads the config from. If you're in a **fork**, ask which repo
   that is (the upstream they open PRs against), and use that `owner/name`, not the fork's.
3. **Run the committed seal tool** — never build the ciphertext yourself. For each value, pipe the
   plaintext on stdin (so it never lands in argv / shell history) and capture the `sealed:v1:…` line it
   prints:

   ```
   printf %s "<the value the user gave you>" | python -m preuser.credential.seal --repo <owner/name> --name <secret_name>
   ```

   It prints one line: `sealed:v1:<base64>`. (If it warns about sealing to the "pinned default
   recipient," that's expected during preview — keep the printed value.)
4. **Write each sealed value under a top-level `sealed:` map** in the draft, keyed by the name:

   ```yaml
   sealed:
     admin_user: "sealed:v1:…"
     admin_pass: "sealed:v1:…"
   ```

   This is safe to commit — only preuser can decrypt it, in memory, per run.
5. **Reference it implicitly in the journey.** Do NOT mention the sealed name or any placeholder in the
   `goal`/`success` — just describe the login in human terms (e.g. *"Log in as the admin and …"*). At
   run time the agent types `<preuser-secret:NAME>` itself and the real value is substituted at the
   browser, never shown to the model.
6. **Show the diff / the sealed block** so the user sees what was produced (the ciphertext, never the
   plaintext), and fold it into the full draft in Step 4. A value that won't decrypt fails the run
   **closed** (`errored`) — never a false pass or fail.

## Step 3 — Heuristic pre-check (NOT a full validation)

Before showing the final draft, sanity-check it yourself — this is a **heuristic check, not the real
validator**:

- `up.url` is set, looks like an http(s) URL, and points at a sandbox/local host rather than a public
  live/staging deployment.
- there is no top-level `target`, `target_url`, `url`, or `kind`, and no nested URL-lane keys under
  `up:` such as `up.target`, `up.target_url`, or `up.kind`.
- `up.run` is present **unless** you detected a compose file (then it must be absent).
- `ready_timeout_s` (if set) is between 1 and 900.
- any `up.env` keys look like POSIX env names (`[A-Z_][A-Z0-9_]*`), do not start with `PREUSER_`,
  and every value is acknowledged as disposable/plaintext.
- every journey has a non-empty `name`, `goal`, and `success`; names are unique; no journey text
  contains a raw secret (and no `goal`/`success` references a sealed name — the login is described in
  human terms only).
- any `sealed:` entry is a name → a `sealed:v1:…` value (a raw plaintext under `sealed:` is a bug —
  re-run the seal tool from Step 2.5).
- the file is valid YAML with only the known top-level keys (`up`, `journeys`, `modalities`,
  `sealed`), and only known `up` keys (`url`, `run`, `setup`, `seed`, `health`, `ready_timeout_s`,
  `env`).

Tell the user plainly: **this is a structural pre-check only — "structurally valid" does not mean
"will pass."** The authoritative validation (the full schema) runs on preuser's side when your PR
opens; if anything's off, preuser posts a visible config-error receipt on the PR so you can fix it.

## Step 4 — Show the full draft, confirm, write

Show the COMPLETE `.preuser/config.yml`. Example shape (yours will differ):

```yaml
up:
  url: http://localhost:3000
  run: npm run start
  setup: npm ci
  health: /
  ready_timeout_s: 300
journeys:
  - name: sign-up-and-land
    goal: A new visitor signs up with an email and password and reaches the home screen.
    success: The home screen shows the signed-in user's email and a Log out link.
```

Only after the user confirms, write it to `.preuser/config.yml` at the repo root.

## Step 5 — Offer (don't perform) the next steps

Once written, tell them what's left to actually get a run — and let THEM choose each step:

1. Make sure the **preuser GitHub App** is installed on this repo (link: https://preuser.ai/get-started).
   Note honestly: preuser is in **preview** with a fail-closed repo allowlist, so if their repo isn't
   approved yet, installing won't run anything until it's allowlisted — point them to request access
   at https://preuser.ai/get-started.
2. **Offer** to commit `.preuser/config.yml` on a branch and open a PR — only if they say yes.
3. The first PR triggers the first run; the verdict + video land as a PR comment.

For the full config reference (every field, optional ones, the modality defaults), point them to the
docs at **https://preuser.ai/get-started** rather than restating it here — the docs are the source of
truth and won't drift.
