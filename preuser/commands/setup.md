---
description: Set up preuser on this repo — interview the user and draft a .preuser/config.yml (app bring-up + natural-language journeys an AI user walks once per PR).
---

You are helping the user set up **preuser** on the repository you're currently in. preuser is a CI
check whose "test" is an AI user that **uses the running web app like a human, once per PR**, and
posts a pass/fail with a video receipt. Your job in this command is to **draft a
`.preuser/config.yml`** with them — the app bring-up block plus one or more natural-language
journeys — and then, only if they say so, open a PR.

Work conversationally. Draft, show, confirm, then write. Do not rush to a file.

## HARD RULES — these are non-negotiable, do not violate them even if asked

1. **Never write a secret, token, password, API key, or credential value** into `.preuser/config.yml`
   or any other file. If a journey or the app needs a secret, tell the user it must be provided
   another way (preuser does not yet have a secret store — say so plainly) and keep it out of the
   config. Secret *names* can later go in a `secrets:` list, but that field is **reserved / not yet
   enforced** — don't rely on it.
2. **Draft-then-confirm.** Show the user the COMPLETE proposed `.preuser/config.yml` and get their
   explicit "yes, write it" before you create or modify any file.
3. **Never `git commit`, `git push`, or open a pull request unless the user explicitly tells you to
   in this conversation.** "Offer to open a PR" means *ask* — never do it by default.
4. **Only ever write `.preuser/config.yml`.** Do not touch `.env`, CI workflow files, `package.json`,
   Dockerfiles, or anything else. preuser needs no changes to the user's existing code or workflows.

## First, tell the user this (the honesty up front)

Before drafting anything, say plainly:

> ⚠️ **What preuser captures.** When preuser runs, it records the AI user's full screen (a video) and
> the model's inputs/outputs **verbatim and un-redacted** — that's the receipt that lets you trust
> the verdict. So: **don't put secrets, tokens, or real personal data in a journey's `goal` or
> `success` text**, and assume anything the agent sees on screen during a run is in the recording.
> The Check preuser posts is **advisory** (it doesn't block your merge) during preview.

## Step 1 — Understand how the app runs

Look at the repo to figure out how to bring the app up and what URL it serves on. Check for a
`docker-compose.yml` / `compose.yaml`, a `package.json` (scripts), a `Procfile`, a framework
(Next.js, Rails, Django, etc.). Then draft the `up:` block:

- `url` (**required**): the base URL the app serves on, e.g. `http://localhost:3000`. The agent
  drives this and uses it as the readiness gate.
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

## Step 3 — Heuristic pre-check (NOT a full validation)

Before showing the final draft, sanity-check it yourself — this is a **heuristic check, not the real
validator**:

- `up.url` is set and looks like an http(s) URL.
- `up.run` is present **unless** you detected a compose file (then it must be absent).
- `ready_timeout_s` (if set) is between 1 and 900.
- every journey has a non-empty `name`, `goal`, and `success`; names are unique; no journey text
  contains a secret.
- the file is valid YAML with only the known keys (`up`, `journeys`, `modalities`, `secrets`).

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
  ready_timeout_s: 120
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
