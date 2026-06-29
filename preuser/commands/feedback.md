---
description: Send consented setup/rescue feedback to preuser so the team can follow up.
---

You are helping the user send a concise preuser setup/rescue pain report to the preuser team. This is
not a diagnostics command and it does not edit files.

## Hard rules

1. Do not submit anything until the user explicitly agrees and gives a contact email.
2. Do not send raw secrets, tokens, passwords, production credentials, customer data, or private
   source snippets. Summarize sensitive details instead.
3. Tell the user exactly what will be sent to `https://preuser.ai/api/plugin-feedback` before the
   POST: contact email, repo, command, summary, details, PR URL, run URL when present, and the public
   plugin release marker.
4. Send the public plugin release marker `preuser-plugin:v0.5.0` in
   `X-Preuser-Feedback-Client`. This is not a secret and not authentication; it is a spam-friction
   marker tying the request to this plugin release. You can see it because it is written directly in
   this command's instructions. Future plugin releases should use their own semver marker
   (`preuser-plugin:vX.Y.Z`); the backend accepts semver-shaped plugin markers without a backend
   change.

## Gather the report

Inspect local context first when available:

- repo: `gh repo view --json nameWithOwner -q .nameWithOwner`, or infer from `git remote -v`;
- local `.preuser/config.yml`, but include only a short redacted snippet or field summary;
- recent command/problem context from the conversation;
- PR URL, run page URL, commit SHA, target kind, and the smallest error message if the user has one.

Ask only for missing essentials:

- contact email for follow-up;
- one-sentence summary;
- approval to send the final report.

Use a compact details format. Good details name what hurt, what was expected, what actually happened,
and the evidence link. Do not paste long logs; summarize the relevant line and include the run/PR URL
when available.

## Submit

After showing the user the report and getting approval, POST JSON:

```bash
python - <<'PY' | curl -fsS \
  -H 'Content-Type: application/json' \
  -H 'X-Preuser-Feedback-Client: preuser-plugin:v0.5.0' \
  --data-binary @- \
  https://preuser.ai/api/plugin-feedback
import json

payload = {
    "email": "USER_EMAIL",
    "repo": "OWNER/REPO",
    "command": "/preuser:setup",
    "summary": "Short setup pain summary",
    "details": "Redacted concise details, including PR/run URLs when useful.",
    "pr_url": "https://github.com/OWNER/REPO/pull/123",
    "run_url": "https://preuser.ai/runs/...",
    "consent": True,
}
print(json.dumps({k: v for k, v in payload.items() if v not in ("", None)}))
PY
```

Replace every placeholder before running. If the response has `{"ok": true, "feedback_id": "..."}`,
show the feedback id and say preuser has enough context to follow up. If the POST fails, keep the
same redacted report in the chat and tell the user it was not submitted.
