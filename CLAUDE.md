# preuser-plugins - AI assistant primer

Read before substantive work in this repo. This repository is the agent plugin marketplace for
preuser, not the preuser product/control-plane repo. Keep it small: user-facing workflow docs live in
`README.md`; command behavior lives in `preuser/commands/*.md`; Claude plugin metadata lives in
`.claude-plugin/marketplace.json` and `preuser/.claude-plugin/plugin.json`; Codex plugin metadata
lives in `.agents/plugins/marketplace.json`, `preuser/.codex-plugin/plugin.json`, and
`preuser/skills/preuser/SKILL.md`.

## What This Repo Ships

One marketplace plugin: `preuser`, exposed as Claude Code slash commands and as a Codex skill.

- `/preuser:setup` creates or updates `.preuser/config.yml` for repo PR checks.
- `/preuser:status [OWNER/REPO]` checks the first-party `/api/repo-status` endpoint for GitHub App
  installation/selection, config presence, and server-side run gates.
- `/preuser:validate` performs a heuristic local config pre-check.
- `/preuser:seal NAME` helps the user create a repo-bound `sealed:v1:...` credential value.
- `/preuser:rescue` triages config, deployment handoff, reachability, auth, and run evidence.
- `/preuser:feedback` sends a consented setup/rescue pain report to preuser with a contact email.
- Codex uses the `preuser` skill for the same six workflows; the skill points at the command files
  rather than duplicating their operational instructions.

The plugin helps a coding agent configure preuser. It does not run the hosted service, change the
GitHub App allowlist, edit CI workflows by default, or produce a local preuser verdict.

## Sources Of Truth

- Product/config truth: the preuser product repo and <https://preuser.ai/get-started>.
- Marketplace overview: `README.md`.
- Command instructions: `preuser/commands/setup.md`, `status.md`, `validate.md`, `seal.md`,
  `rescue.md`, and `feedback.md`.
- Codex manifest + skill: `preuser/.codex-plugin/plugin.json`,
  `preuser/skills/preuser/SKILL.md`.
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
- `/preuser:status` is a read-only first-party preflight. The CLI path sends the user's GitHub CLI
  token to `preuser.ai`; never do that silently. It must not imply that App installation alone proves
  a future journey will pass or that fork PR heads are allowlisted.
- `/preuser:feedback` and the Codex feedback workflow must never send a report silently. They must
  show the redacted payload, require a contact email and explicit consent, avoid raw secrets/customer
  data, and include the public plugin release marker as spam friction rather than auth. Claude Code
  uses `preuser-plugin:v0.5.0`; Codex uses `preuser-codex-plugin:v0.5.0`.

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

Before opening or merging a PR, run the static plugin validation when the Claude Code CLI and Codex
skill/plugin helper scripts are available:

```bash
claude plugin validate . --strict
python3 /home/coder/.codex/skills/.system/skill-creator/scripts/quick_validate.py preuser/skills/preuser
python3 /home/coder/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py preuser
```

CI also checks marketplace source directories, README/plugin metadata command mentions, and that
`/preuser:seal` still references `python -m preuser.credential.seal`.
