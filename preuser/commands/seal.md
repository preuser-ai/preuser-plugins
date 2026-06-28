---
description: Seal a PR-run credential for preuser — encrypt a login value with preuser's public key so you can commit only ciphertext in .preuser/config.yml.
---

You are helping the user **seal a credential** for **preuser's PR config** so a journey can log in to
an app the AI user can't sign up for (a seeded admin, a test account). This command is repo-bound and
writes ciphertext under `.preuser/config.yml`'s `sealed:` map. Console URL journeys use the Console's
Login details flow instead. Sealing is the ONLY approved way to give preuser a non-disposable PR-run
credential: the value is encrypted on this machine with preuser's public
key, you commit only the ciphertext, and preuser decrypts it trusted-side, in memory, per run — it is
never written in plaintext config and the AI model sees only the placeholder name, not the value.

`$ARGUMENTS` may name the secret (e.g. `admin_pass`). If not, ask.

## HARD RULES — do not violate

1. **Never echo, log, or write the raw value.** Take it from the user, pipe it straight into the seal
   tool, and only ever show/commit the resulting `sealed:v1:…` ciphertext.
2. **Never put a raw credential anywhere in `.preuser/config.yml`** — config is read from the default
   branch and shown in the PR receipt. Only the sealed ciphertext goes in, under `sealed:`.

## Steps

1. **Name it.** Use `$ARGUMENTS` as the secret name, or ask: a short token like `admin_user` /
   `admin_pass` (letters, digits, `_`, `-`, `.`). The AI user may type it internally as
   `<preuser-secret:NAME>` when it needs to enter the value. Do not put the placeholder or the value in
   `goal`/`success`; describe the login in human terms.

2. **Confirm the repo it binds to.** A sealed value is bound to ONE repo. Use the **base /
   default-branch repo** (`owner/name`) — the repo whose `.preuser/config.yml` preuser reads. Get it
   from `git remote get-url origin` (and if this is a fork, ask for the upstream `owner/name`). Show
   the user the exact `owner/name` you'll bind to and confirm.

3. **Take the value through a hidden terminal prompt and seal it.** The seal tool ships with the
   `preuser` package — if `python -m preuser.credential.seal --help` isn't available, install it first
   (`pip install preuser`, or run via `uvx --from preuser python -m preuser.credential.seal …`). Never
   ask the user to paste the raw value into chat. Run the committed tool — never assemble crypto
   yourself — piping the value on stdin so it never lands in argv/history:
   ```bash
   read -rsp "Value for NAME: " PREUSER_SEAL_VALUE
   printf '\n' >&2
   printf %s "$PREUSER_SEAL_VALUE" | python -m preuser.credential.seal --repo OWNER/NAME --name NAME
   unset PREUSER_SEAL_VALUE
   ```
   If this environment cannot safely accept hidden terminal input, give the user that command with
   `OWNER/NAME` and `NAME` filled in and ask them to run it locally, then paste back only the
   `sealed:v1:<base64>` line. If the tool warns about the default recipient, that's fine for the
   hosted preuser; only override `--recipient` if the user was given a specific public key.

4. **Place it in config.** Add it under a top-level `sealed:` map in `.preuser/config.yml`:
   ```yaml
   sealed:
     NAME: "sealed:v1:…"
   ```
   and make sure the relevant journey's `goal` says the AI user logs in (e.g. "Log in as the admin
   and …"); preuser substitutes the value where the AI user types `<preuser-secret:NAME>`.

5. **Show the diff and confirm** before writing. Reassure: the value is encrypted to preuser's key, is
   safe to commit as ciphertext, and is substituted at the browser boundary rather than placed in the
   model prompt. Use a disposable test credential because anything the app visibly renders can be
   recorded as pixels. A blob that won't decrypt (e.g. sealed for the wrong repo) fails the run
   `errored`, never a false pass — re-run `/preuser:seal` for the right repo.
