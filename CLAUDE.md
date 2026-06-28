# preuser-plugins - AI assistant primer

Read before substantive work in this repo. This repository is the Claude Code plugin marketplace for
preuser, not the preuser product/control-plane repo. Keep it small: user-facing workflow docs live in
`README.md`; command behavior lives in `preuser/commands/*.md`; plugin metadata lives in
`.claude-plugin/marketplace.json` and `preuser/.claude-plugin/plugin.json`.

## What This Repo Ships

One marketplace plugin: `preuser`.

- `/preuser:setup` creates or updates `.preuser/config.yml` for repo PR checks.
- `/preuser:validate` performs a heuristic local config pre-check.
- `/preuser:seal NAME` helps the user create a repo-bound `sealed:v1:...` credential value.
- `/preuser:rescue` triages config, deployment handoff, reachability, auth, and run evidence.

The plugin helps a coding agent configure preuser. It does not run the hosted service, change the
GitHub App allowlist, edit CI workflows by default, or produce a local preuser verdict.

## Sources Of Truth

- Product/config truth: the preuser product repo and <https://preuser.ai/get-started>.
- Marketplace overview: `README.md`.
- Command instructions: `preuser/commands/setup.md`, `validate.md`, `seal.md`, and `rescue.md`.
- Static validation: `.github/workflows/validate.yml`.

Do not restate product schema details in more places than needed. If a config field changes upstream,
update the relevant command prompt and only the README workflow text needed to keep installation and
support accurate.

## Non-Negotiables

- Never tell the agent to write raw real credentials into `.preuser/config.yml`.
- Non-disposable PR-run credentials go through top-level `sealed:` values only.
- `up.env` is plaintext by contract; describe it only as disposable sandbox app configuration.
- `/preuser:setup` must explain preuser and the setup plan before it starts drafting.
- `/preuser:setup` must canvas sandbox vs static URL vs post-deploy GitHub Deployment before repo
  probing or journey drafting.
- `/preuser:setup` must make app auth explicit: fresh signup, sandbox-seeded disposable login,
  sealed existing test account, or supported external `target.auth`.
- `/preuser:setup` should write `.preuser/config.yml` once the setup path is clear. Do not require
  repeated confirmations for the core config write; ask only for real decisions, secrets, local code
  execution, optional agent-note edits, commits, pushes, or PRs.
- `/preuser:setup` writes only `.preuser/config.yml` by default. With explicit user approval, it may
  add/update a short `CLAUDE.md`/`AGENTS.md` note that tells future agents to keep preuser journeys
  aligned as the product surface changes.
- `/preuser:setup` must not edit CI, env files, package files, Dockerfiles, commit, push, or open a
  PR unless the user separately asks.
- External target pre-app auth supports only `target.auth.basic`, `target.auth.cookies`, and
  `target.auth.headers`, with values referenced from `sealed:`.
- Do not invent unsupported config fields such as `secrets`, `browser_auth`, global headers,
  `service_token`, `http_credentials`, raw bearer tokens, or `up.sealed_env`.

## Target Model To Preserve

The setup/rescue docs must keep the two PR paths clear:

1. **Sandbox target**: preuser clones the PR commit and brings up the app from source. `up:` describes
   how preuser reaches the app in the sandbox. The smoke image
   `ghcr.io/preuser-ai/preuser-runner-smoke:latest` should be strongly recommended for sandbox
   configs so the agent can verify the bring-up env instead of guessing. It is not a hosted verdict.
   Direct-run smoke does not need `--privileged`; compose smoke does because it runs Docker inside the
   container.
2. **Deployed target**: preuser drives an already reachable URL. `target.kind: url` is for a stable
   staging URL. `target.kind: github_deployment` waits for a successful GitHub Deployment for the PR
   head SHA and uses its `environment_url` or `target_url`. CI env syntax in config is literal text,
   not interpolation.

For hosted runs, the first-party evidence path is the PR Check/comment and the run page. Debug logs,
when captured, are private run artifacts linked from the run page for authorized viewers; they are
not public log dumps.

## Editing Guidance

- Keep command prompts operational and specific. They are instructions to another agent, not marketing
  copy.
- When changing one command's config guidance, check the others for drift. `setup`, `validate`, and
  `rescue` intentionally overlap on supported fields and common failure modes.
- Keep README descriptive rather than exhaustive. It should explain the workflows and point to
  preuser's hosted reference for the full schema.
- Keep examples minimal and valid. Prefer one canonical YAML shape over many variants.
- Do not add generated artifacts or local smoke output to the repo.

## Validation

Before opening or merging a PR, run the static plugin validation when the Claude Code CLI is
available:

```bash
claude plugin validate . --strict
```

CI also checks marketplace source directories, README/plugin metadata command mentions, and that
`/preuser:seal` still references `python -m preuser.credential.seal`.
