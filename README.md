# Claim Audit — an Aeon community skill pack

**Did the run succeed, or was the thing it told you true?**

`skill-health` answers the first question. Nothing in Aeon answers the second.

This pack adds one meta skill that reads what your instance actually reported, pulls out
the assertions that have a checkable pointer (a commit, a transaction, a URL, a release, a
number with a source), re-checks each against that source, and tells you which ones do not
hold up.

```text
skill-health   →  did the job run?
create-prove   →  did the changed skill actually execute?
claim-audit    →  was what it said true?
```

## Install

```bash
bin/install-skill-pack richard7463/aeon-claim-audit
```

Or copy `skills/claim-audit/` into `skills/` and enable it in `aeon.yml`.

**No secrets.** `requires: []` — this skill needs no API keys. GitHub reads go through
`gh api`, public HTTP through `curl`, and gitlawb records through its public node API.
If a verification layer needs a key before it can tell you anything, nobody installs it.

## What it does

**Default (daily, quiet).** Reads `memory/logs/` for the last 3 days, extracts up to 25
checkable claims, verifies each, and **notifies only if something is contradicted or the
state changed** since the last run. A run that finds nothing wrong says nothing. Standing
alarms get muted, and a muted alarm is worse than none.

**On demand.** `claim-audit <any claim text>` verifies one claim immediately and always
answers, with the grade and what would settle it if it is not confirmed.

## Evidence grades

The grade describes the evidence **behind the claim**, never how confidently the claim was
written. A forceful sentence with nothing behind it is E0.

| grade | meaning |
|---|---|
| E0 | no pointer, or the pointer resolves to nothing |
| E1 | source unreachable, or it does not contain the assertion |
| E2 | one non-authoritative source supports it; not cross-checked |
| E3 | the authoritative source was fetched and matches |
| E4 | a deterministic, tamper-evident record confirms it (on-chain receipt, signed ref certificate) |

Verdicts: `confirmed` (E3/E4), `unsupported` (E0–E2), `contradicted` (the source says
something different). **Unreachable is not the same as true** — a failed fetch is an
honest E1, never a pass.

## Deliberately narrow

It only audits claims with a pointer to something outside the instance. Opinions,
forecasts, and "sentiment is turning" are out of scope and are not mentioned. A short
honest list beats a long speculative one, and scope discipline is what keeps the signal
worth reading.

## Related

- [ai2human Onus](https://github.com/richard7463/a2h-onus) — the same grading discipline as
  a library: `verify_claim` returns a verdict plus a replayable receipt. Install it with
  `pip install -e ".[mcp]"` if you want hash-chained receipts instead of log lines. The
  skill works without it.

## License

MIT.
