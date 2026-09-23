# Claim Audit Skills Catalog

One skill. It checks whether the things your unattended instance tells you are true.

```text
run logs → extract checkable claims → verify each at its source → grade E0-E4 → contradict
```

## Start Here

| Skill | Use it when | Produces |
|---|---|---|
| [`claim-audit`](skills/claim-audit/SKILL.md) | Always, on a daily schedule. It runs quietly and only speaks when something is wrong | Per-claim verdict, evidence grade, and the source that contradicts it |

## Verification

| Skill | Best for | Default evidence | Do not use for |
|---|---|---|---|
| [`claim-audit`](skills/claim-audit/SKILL.md) | Auditing what the instance reported; verifying one claim on demand | The claim's own authoritative source (GitHub API, public HTTP, on-chain RPC, gitlawb node) | Opinions, forecasts, or claims with no pointer to check |

## Configuration

`requires: []` — no secrets, no setup. Enable it in `aeon.yml` on a daily schedule and it
audits the last 3 days. Pass an integer to widen the window (capped at 30 days), or pass
claim text to check one thing right now.

## Exit signatures

`CLAIM_AUDIT_OK` · `CLAIM_AUDIT_NOTIFIED` · `CLAIM_AUDIT_NO_LOGS` · `CLAIM_AUDIT_UNSUPPORTED_VIEW`
