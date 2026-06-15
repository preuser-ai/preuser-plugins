---
description: Seal a credential for preuser — encrypt a login value with preuser's public key so you can commit it (the ciphertext) in .preuser/config.yml. preuser decrypts it trusted-side; the value is never shown to the AI user.
---

You are helping the user **seal a credential** for **preuser** so a journey can log in to an app the
AI user can't sign up for (a seeded admin, a test account). Sealing is the ONLY approved way to give
preuser a credential: the value is encrypted on this machine with preuser's public key, you commit
only the ciphertext, and preuser decrypts it trusted-side, in memory, per run — it is never written
in plaintext and never shown to the AI model.

`$ARGUMENTS` may name the secret (e.g. `admin_pass`). If not, ask.

## HARD RULES — do not violate

1. **Never echo, log, or write the raw value.** Take it from the user, pipe it straight into the seal
   tool, and only ever show/commit the resulting `sealed:v1:…` ciphertext.
2. **Never put a raw credential anywhere in `.preuser/config.yml`** — config is read from the default
   branch and shown in the PR receipt. Only the sealed ciphertext goes in, under `sealed:`.

## Steps

1. **Name it.** Use `$ARGUMENTS` as the secret name, or ask: a short token like `admin_user` /
   `admin_pass` (letters, digits, `_`, `-`, `.`). The journey will reference it as
   `<preuser-secret:NAME>` — you don't put the value in the journey text.

2. **Confirm the repo it binds to.** A sealed value is bound to ONE repo. Use the **base /
   default-branch repo** (`owner/name`) — the repo whose `.preuser/config.yml` preuser reads. Get it
   from `git remote get-url origin` (and if this is a fork, ask for the upstream `owner/name`). Show
   the user the exact `owner/name` you'll bind to and confirm.

3. **Take the value secretly and seal it.** The seal tool ships with the `preuser` package — if
   `python -m preuser.credential.seal --help` isn't available, install it first (`pip install preuser`,
   or run via `uvx --from preuser python -m preuser.credential.seal …`). Then ask for the value and run
   the committed tool — never assemble crypto yourself — piping the value on stdin so it never lands in
   argv/history:
   ```bash
   printf %s 'THE_VALUE' | python -m preuser.credential.seal --repo OWNER/NAME --name NAME
   ```
   (Prefer reading the value into a shell var via a hidden prompt over inlining it.) The tool prints a
   `sealed:v1:<base64>` line. If it warns about the default recipient, that's fine for the hosted
   preuser; only override `--recipient` if the user was given a specific public key.

4. **Place it in config.** Add it under a top-level `sealed:` map in `.preuser/config.yml`:
   ```yaml
   sealed:
     NAME: "sealed:v1:…"
   ```
   and make sure the relevant journey's `goal` says the AI user logs in (e.g. "Log in as the admin
   and …"); preuser substitutes the value where the AI user types `<preuser-secret:NAME>`.

5. **Show the diff and confirm** before writing. Reassure: the value is encrypted to preuser's key, is
   safe to commit, and is never shown to the AI model. A blob that won't decrypt (e.g. sealed for the
   wrong repo) fails the run `errored`, never a false pass — re-run `/preuser:seal` for the right repo.
