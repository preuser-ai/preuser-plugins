# preuser-plugins

The [Claude Code](https://claude.com/claude-code) plugin marketplace for **[preuser](https://preuser.ai)**: a product whose "test" is an AI user that uses your running web app like a human, on every PR or against a live URL, and posts a pass/fail with a video receipt.

preuser is useful beyond a binary check: the receipt can show where the AI user hit friction, and the journeys you define become a repeatable baseline for noticing workflow drift as the product changes.

This marketplace ships one plugin, **`preuser`**, for repo-owned PR checks. It helps your coding agent draft and triage `.preuser/config.yml`; it does not run preuser locally or push anything without confirmation. One-off live URL journeys still live in the preuser Console (`/journeys` or `/run-now`).

## Install (Claude Code)

```
/plugin marketplace add preuser-ai/preuser-plugins
/plugin install preuser@preuser-plugins
/reload-plugins
/preuser:setup
```

> Requires a Claude Code version with plugin-marketplace support and access to the private preview marketplace. `gittb/preuser-plugins` currently redirects to `preuser-ai/preuser-plugins`; use the `preuser-ai` path in docs and support.

## Commands

- **`/preuser:setup`**: interviews you, chooses the right PR target, works out any login path preuser needs, drafts `.preuser/config.yml`, and offers next steps. Draft-then-confirm; it writes only `.preuser/config.yml` and never commits, pushes, or opens a PR without your say-so.
- **`/preuser:validate`**: a quick heuristic pre-check of an existing `.preuser/config.yml`. Structural only; authoritative validation runs on preuser's side when your PR opens.
- **`/preuser:seal NAME`**: encrypts a test-account login value for the top-level `sealed:` map so the repo commits only `sealed:v1:...` ciphertext.
- **`/preuser:rescue`**: triages a config or run that did not behave as expected, using the local config, PR Check/comment, and run page evidence.

## The Two PR Target Paths

### 1. preuser spins up the app

Use the default sandbox target when there is no deployed preview environment, or when you want a fresh throwaway app per PR. The config has an `up:` block with the URL preuser should reach inside the sandbox and, for non-compose apps, the command that keeps the app serving:

```yaml
target:
  kind: sandbox
up:
  url: http://localhost:3000
  run: npm run start
  setup: npm ci && npm run build
  health: /
  ready_timeout_s: 300
journeys:
  - name: sign-up-and-land
    goal: A new visitor signs up and reaches their home screen.
    success: The home screen shows the signed-in user's email.
```

For Docker Compose repos, omit `up.run`; preuser detects the compose file and runs the stack. Use `up.env` only for disposable app variables, such as a per-run seeded login. `up.env` is plaintext and visible in receipts.

The easy smoke path is the runner image, which exercises the same app bring-up contract without running the AI user or producing a hosted verdict. It executes your repo's setup/seed/run or compose code, so the setup agent should offer it and wait for explicit approval before running it. For unfamiliar code, run it in a disposable dev VM or CI job.

For direct-run apps:

```bash
docker run --rm -p 3000:3000 \
  -v "$PWD:/workspace" \
  -e PREUSER_UP_URL=http://localhost:3000 \
  -e PREUSER_UP_HEALTH=/ \
  -e PREUSER_UP_RUN='npm run start -- --host 0.0.0.0' \
  ghcr.io/preuser-ai/preuser-runner-smoke:latest
```

Add `PREUSER_UP_SETUP` and `PREUSER_UP_SEED` when your `up:` block uses them. For compose repos, omit `PREUSER_UP_RUN`; the smoke image detects the compose file and needs `--privileged` for its in-container Docker daemon.

If the public image pull fails, fall back to the local check: run the same setup/seed/run or compose command and `curl` `up.url + up.health` before opening the PR. The first full preuser receipt still comes from the hosted PR run after the config is on the default branch.

### 2. your CI/CD deploys the app

Use `target.kind: url` for one stable staging URL:

```yaml
target:
  kind: url
  url: https://staging.example.com
```

Use `target.kind: github_deployment` when each PR or feature branch gets a unique URL after deploy:

```yaml
target:
  kind: github_deployment
  environment: preview
  wait_timeout_s: 900
```

This is the post-deploy path: preuser starts from the PR event, then waits for a successful GitHub Deployment for the PR head SHA and drives its `environment_url`, falling back to `target_url`. In GitHub Actions, set the deploy job's `environment.url` from your deploy output. In other CI systems, create/update a GitHub deployment status with `environment_url` after deploy succeeds.

Do not put `$PREVIEW_URL`, `${{ steps.deploy.outputs.url }}`, or provider-specific env syntax in `.preuser/config.yml`; it is read as a literal string. Also avoid deadlock: the deploy job must not wait for the preuser Check while preuser is configured to wait for deployment.

External targets must be reachable from preuser's hosted worker over public HTTP(S). VPN-only hosts, localhost, private IPs, internal DNS, and metadata/link-local addresses are rejected or blocked. For an in-app login, use `sealed:` values. If your preview is protected before the app UI loads, PR config supports HTTP Basic auth, cookies, and same-origin request headers via `target.auth`, with values referenced from top-level `sealed:`:

```yaml
target:
  kind: github_deployment
  environment: preview
  auth:
    basic:
      username_secret: preview_user
      password_secret: preview_pass
    cookies:
      - name: preview_token
        value_secret: preview_cookie
    headers:
      - name: Authorization
        value_secret: preview_authorization
sealed:
  preview_user: "sealed:v1:..."
  preview_pass: "sealed:v1:..."
  preview_cookie: "sealed:v1:..."
  preview_authorization: "sealed:v1:..."
```

Header values are attached only to the resolved target origin. For `Authorization: Bearer ...`, seal the complete `Bearer ...` value. VPN access, private allowlists, top-level/global browser headers, raw service-token fields, and credentials embedded in URLs are not supported in PR config.

## What You'll End Up With

A small `.preuser/config.yml` at your repo root: a `target:` block when you want something other than the default sandbox, an `up:` block only for sandbox bring-up, and a list of `journeys` where each journey has a plain-English `goal` plus a visible `success` criterion. That's it: no test code. For dynamic preview URLs, CI/CD must publish the URL after deployment as a GitHub Deployment status; preuser waits for that durable signal.

preuser reads `.preuser/config.yml` from the repo's default branch. A PR that first adds or changes the config is an activation PR; after it lands, the next eligible PR uses that config.

The full config reference lives at **<https://preuser.ai/get-started>**. That page and the product schema are the source of truth; this README stays at the workflow level.

## Debugging and Logs

The first-party feedback loop is the PR Check/comment plus the preuser run page. The run page shows live status while running and, when finalized, the video, timeline, screenshots, verdict result, and private Debug logs when worker/sandbox logs were captured.

`/preuser:rescue` inspects your config, classifies likely failures, and asks for the PR URL or run page URL when needed. Local smoke logs are the Docker terminal output. For hosted bring-up failures, open the run page as an authorized viewer and check the Debug logs card if it appears; otherwise keep the run page URL/run id, PR URL, commit SHA, and the relevant `.preuser/config.yml` snippet for support.

## Heads Up (Preview)

- preuser is in **preview** with a fail-closed repo allowlist. Installing the GitHub App on a repo that is not approved yet will not run anything. Request access at <https://preuser.ai/get-started>.
- When preuser runs, it records the AI user's screen and model activity as evidence. Keep secrets and real personal data out of journey text and out of the app surfaces the agent will see. Use disposable test accounts.
- The Check preuser posts is **advisory** during preview; it does not block merges.
