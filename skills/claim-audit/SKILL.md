---
name: claim-audit
description: Check whether the factual claims this instance reports actually hold up. Extracts checkable assertions from recent run logs and notifications, re-verifies each against its own authoritative source, grades the evidence E0-E4, and contradicts the ones that do not survive. Notifies only on a change of state.
scorable: false  # meta skill: audits other skills' output, has no single gradable artifact
metadata:
  title: Claim Audit
  category: dev
  mode: write
  var: ""
  tags:
    - meta
    - verification
  requires: []
---

> **${var}** — View selector.
> - **empty** → `VIEW=audit` over the last 3 days of run logs (default).
> - a **positive integer** (e.g. `7`) → `VIEW=audit` over that many days, capped at 30.
> - any other **text** → `VIEW=check`: verify that single claim, right now.

<!-- Claim Audit is the truth-of-what-was-said companion to skill-health, which covers whether the run succeeded. Same doctrine as aeon's own create-prove: "a green diff review is not proof." Same doctrine as ai2human Onus: the burden of proof is on the evidence, not on the model's confidence. -->

## Overview

This instance reports things to you unattended. `skill-health` audits whether those runs
**succeeded**. `create-prove` proves a changed skill **actually ran**. Neither answers the
question you act on every morning:

> Was the thing it told me **true**?

This skill reads what the instance actually said, pulls out the assertions that are
checkable, re-checks each one against its own authoritative source, and reports the ones
that do not survive. It is deliberately conservative and deliberately quiet.

Two views share a preamble and branch:

- **audit** (default) — extract claims from recent `memory/logs/`, verify, and notify only
  when the state of the world **changed** since the last run.
- **check** — verify one claim the operator hands you, immediately, and always report.

## What counts as a checkable claim

Only assertions with a **pointer** to something outside the instance:

| kind | pointer | authoritative source |
|---|---|---|
| `commit` / `pr` / `issue` | `owner/repo` + sha / number | `gh api` |
| `release` | repo + tag | `gh api repos/{r}/releases/tags/{t}` |
| `tx` | chain + 0x hash | public RPC `eth_getTransactionReceipt`, or the chain's explorer API |
| `address_fact` | chain + address + asserted property | public RPC / explorer API |
| `gitlawb` | owner DID + repo + optional sha | `https://node.gitlawb.com/api/v1/repos/...` (public, no key) |
| `link` | URL | fetch it; confirm the page exists **and** supports the assertion |
| `number` | a named metric + the source it came from | re-fetch that source and compare |

Anything without a pointer — opinions, forecasts, "sentiment is turning", "this looks
promising" — is **not** a claim this skill audits. Do not flag it, do not score it, do not
mention it. Staying in scope is what keeps the signal worth reading.

## Evidence grades (E0-E4)

Grade the evidence **behind the claim**, not how confident the writing sounded. A claim
written forcefully with nothing behind it is E0.

| grade | meaning |
|---|---|
| **E0** | no pointer at all, or the pointer resolves to nothing |
| **E1** | a pointer exists but the source was unreachable, or the source does not actually contain the assertion |
| **E2** | a single non-authoritative source (blog, repost, aggregator) supports it; not cross-checked |
| **E3** | the authoritative source was fetched and the assertion matches it |
| **E4** | a deterministic, tamper-evident record confirms it (on-chain receipt, signed gitlawb ref certificate, a commit read from the API by hash) |

**E3/E4 confirm; E0/E1 do not; E2 is neither.** Never round upward because the claim is
plausible or because it agrees with what you expected. When the fetch fails, the grade is
**E1 — unreachable is not the same as true.**

## Verdict per claim

- **`confirmed`** — E3 or E4, and the source actively supports the assertion.
- **`unsupported`** — E0, E1 or E2. The claim may still be true; nothing checked proves it.
- **`contradicted`** — the source was reached and says something different. This is the one
  that matters most, and the only one that always warrants a notification.

---

# Shared preamble

1. Read `memory/MEMORY.md` for context on what this instance is for, and `STRATEGY.md` for
   the operator's priorities — a contradicted claim about a priority area outranks a
   contradicted claim about a side interest.
2. Compute `${today}` (UTC date, `YYYY-MM-DD`).
3. Parse `${var}` → selector:
   - empty → `VIEW=audit`, `WINDOW_DAYS=3`
   - a positive integer → `VIEW=audit`, `WINDOW_DAYS=min(value, 30)`
   - anything else → `VIEW=check`, `CLAIM=<the text>`
4. Read `memory/claim-audit/last-report.json` if it exists (state for change detection).
   A missing file is normal on the first run — treat the previous state as empty, do not
   error, do not notify about the absence.
5. Dispatch to the matching view.

---

# Audit view

## 1. Gather what the instance said

Read `memory/logs/` for the last `WINDOW_DAYS` days. Include the notification bodies if the
instance keeps them; if it does not, the `### <skill>` sections of the logs are the record.

For each entry, pull out assertions that match a row in the *checkable claim* table above.
Carry the skill that said it and the date. Skip anything already reported as contradicted in
the last report — do not re-litigate a claim the operator has already seen contradicted.

Cap the work: **verify at most 25 claims per run**, newest and highest-consequence first.
If more qualify, say how many were deferred in the log — never silently drop them.

## 2. Verify each claim

For each claim, in order:

1. **Reach the authoritative source** listed in the table.
   - GitHub: `gh api`, which authenticates internally. Do **not** use raw curl for it.
   - Public HTTP: plain `curl -sS --max-time 20 -w '\nhttp=%{http_code}\n'`. If a specific
     host is flaky, retry once with **WebFetch**.
   - gitlawb: `curl -sS` against `https://node.gitlawb.com/api/v1/repos/<owner>/<repo>`
     (and `/commits` or `/certs`). Public repositories need no key and no identity.
   - On-chain: a public RPC or the chain's own explorer API. No key: if the only route needs
     one, that is an **E1** for this run, not a reason to skip the claim.
2. **Print the evidence you actually got** before judging:
   `claim=<short>  source=<url>  http=<code>  result=<found|miss|differs>`.
3. **Compare**, do not summarise. The assertion must be *in* the source, not merely
   consistent with it. "TVL fell 40%" is not confirmed by a page that shows today's TVL and
   says nothing about a change.
4. Assign the grade and the verdict.

If a fetch fails, record the true reason — `http-403`, `timeout`, `empty`, `no-pointer`.
**Never write "sandbox", "blocked", or "env not available"** — those are the stale excuses
that trained the operator to ignore failures. An unreachable source is an honest E1.

## 3. Decide whether this is worth a notification

Notify if **any** of these is true:

- at least one claim is **`contradicted`**;
- the overall state **changed** since `last-report.json` (a claim moved to contradicted, or
  a previously contradicted claim was resubmitted and still fails);
- a claim that `MEMORY.md` or `STRATEGY.md` marks as a priority is `unsupported` (E0/E1).

Otherwise **end quietly**: write the report file, append the log line, call nothing. A run
that finds nothing wrong should be silent. Standing alarms get muted, and a muted alarm is
worse than none.

Never notify about: claims you could not reach the source for on a non-priority topic, a
quiet day, or your own coverage limits.

## 4. Write state and log

1. Write `memory/claim-audit/last-report.json`:
   ```json
   {
     "checked_at": "${today}",
     "window_days": 3,
     "claims_checked": 12,
     "claims_deferred": 0,
     "by_verdict": {"confirmed": 9, "unsupported": 2, "contradicted": 1},
     "by_grade": {"E0": 0, "E1": 1, "E2": 1, "E3": 8, "E4": 2},
     "contradicted": [
       {"skill": "token-movers", "claim": "…", "source": "…", "http": 200, "said": "…", "actual": "…"}
     ],
     "unsupported_priority": []
   }
   ```
2. Append to `memory/logs/${today}.md` under **one** `### claim-audit` heading:
   ```
   ### claim-audit (${var})
   - Window: <N> days · logs read: <N> · claims checked: <N> (deferred <N>)
   - Verdicts: confirmed <N> · unsupported <N> · contradicted <N>
   - Grades: E0 <N> · E1 <N> · E2 <N> · E3 <N> · E4 <N>
   - Contradicted: <skill> — "<claim>" (source <url>, http <code>)
   - Outcome: notified | quiet | CLAIM_AUDIT_NO_LOGS
   ```
3. Send, only if step 3 said so:

   ```
   ./notify "claim-audit · <N> claims checked, <N> contradicted

   ✗ <skill> on <date> reported: <what it said>
     the source says: <what it actually shows>
     <url> (http <code>)

   <repeat per contradicted claim, newest first, max 5>

   Unsupported (checkable pointer, source did not confirm): <N>
   Everything else checked out."
   ```

   No emoji beyond the single `✗` marker, no hedging, no "could potentially". State what was
   claimed and what the source shows.

---

# Check view

`VIEW=check`. The operator handed you one claim. Verify it and **always** report — they are
waiting on the answer.

1. Identify the pointer inside `CLAIM`. If there is none, answer `unsupported / E0` and say
   what would make it checkable ("a transaction hash, a repo and PR number, a URL").
2. Verify against the authoritative source for that pointer, exactly as in the audit view.
3. Notify in the same shape, with one addition — the grade on its own line, and the concrete
   next step when it is not confirmed:

   ```
   ./notify "claim-audit · check

   claim: <CLAIM>
   grade: E<n> · verdict: <confirmed|unsupported|contradicted>
   evidence: <url> (http <code>)
   <what the source shows, one line>
   <if not confirmed: what would settle it>"
   ```

4. Append one `### claim-audit` log line. Do **not** write `last-report.json` from this view —
   a single ad-hoc check is not the fleet's state.

---

## Network and permissions

`requires: []` — this skill needs **no secrets**. That is deliberate: a verification layer
that needs a key before it can tell you anything will not get installed.

- **GitHub:** `gh api` (auth is handled for you). Raw curl against GitHub is not equivalent.
- **Public HTTP and the gitlawb node:** plain `curl`; WebFetch as the one-retry fallback.
- **Nothing here is irreversible.** The only side effects are a file under `memory/`, a log
  line, and `./notify`.

## Rules

- **Fail closed.** Unreachable, empty, or ambiguous resolves to **unsupported**, never to
  confirmed. Silence is not confirmation.
- **Never invent a source, a status code, or a number.** If you did not fetch it, you did not
  check it.
- **Do not grade style.** Confidence in the writing is not evidence. Grade only what a source
  outside this instance actually shows.
- **Do not widen scope to seem useful.** Only assertions with a pointer, and at most 25 per
  run. A short honest list beats a long speculative one.
- **A quiet run is a success**, not a gap to fill.

## Exit signatures

`CLAIM_AUDIT_OK` (ran, nothing to report) · `CLAIM_AUDIT_NOTIFIED` ·
`CLAIM_AUDIT_NO_LOGS` (no logs in the window) · `CLAIM_AUDIT_UNSUPPORTED_VIEW`

## Related

- `skill-health` — did the run succeed. This skill starts where that one stops.
- `create-prove` — SHA-bound proof that a changed skill actually executed.
- [ai2human Onus](https://github.com/richard7463/a2h-onus) — the same grading discipline as a
  library: `verify_claim` returns a verdict plus a replayable receipt. If the operator has it
  installed (`pip install -e ".[mcp]"`), it can be used for the final verdict so each result
  carries a hash-chained receipt instead of a log line.
