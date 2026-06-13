# preuser-plugins

The [Claude Code](https://claude.com/claude-code) plugin marketplace for **[preuser](https://preuser.ai)** — a CI check whose "test" is an AI user that *uses your running web app like a human, once per PR,* and posts a pass/fail with a video receipt.

This marketplace ships one plugin, **`preuser`**, whose job is to help your coding agent author your `.preuser/config.yml` — so you never hand-write the config.

## Install (Claude Code)

```
/plugin marketplace add gittb/preuser-plugins
/plugin install preuser@preuser-plugins
/reload-plugins
/preuser:setup
```

> Requires a Claude Code version with plugin-marketplace support. **Codex support is coming soon** — the Codex plugin path isn't published yet.

## Commands

- **`/preuser:setup`** — interviews you about how your app runs and what a user should be able to do, then *drafts* a `.preuser/config.yml` (the app bring-up block + one or more natural-language journeys) and offers to open a PR. Draft-then-confirm; it never writes secrets and never pushes without your say-so.
- **`/preuser:validate`** — a quick heuristic pre-check of an existing `.preuser/config.yml`. Structural only — the authoritative validation runs on preuser's side when your PR opens.

## What you'll end up with

A small `.preuser/config.yml` at your repo root: an `up:` block telling preuser how to bring your app up, and a list of `journeys` (each a plain-English `goal` + a `success` criterion an AI user walks toward). That's it — no test code, no workflow changes.

The full config reference lives at **<https://preuser.ai/get-started>** (the source of truth — this README intentionally doesn't restate the schema).

## Heads up (preview)

- preuser is in **preview** with a fail-closed repo allowlist — installing the GitHub App on a repo that isn't approved yet won't run anything. Request access at <https://preuser.ai/get-started>.
- When preuser runs, it records the AI user's screen (video) and the model's I/O **verbatim** — keep secrets and real personal data out of your journey text.
- The Check preuser posts is **advisory** during preview (it doesn't block merges).
