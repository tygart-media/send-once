# send-once

A one-file wrapper that makes email sends idempotent. Route every
scheduled, scripted, or agent-driven send through it and it guarantees
**at most once** — in code, not in prose.

## The story

My morning-brief agent sent the same email 7 times in 3 minutes.

Nothing was broken. The send succeeded on the first try, but the approval
reply that confirmed it got lost. The agent, doing exactly what agents do,
retried. And retried. Each attempt looked brand new from the inside: no
error, no evidence the earlier attempt had landed. Seven identical emails.

This is the **duplicate completion** failure class: the original succeeds,
the response is lost, the retry re-sends. Gmail has no native idempotency
key (unlike Stripe, Resend, and friends), so there is nothing on the
provider side to catch it. Telling an agent "don't send twice" in the
prompt is not enforceable — agents re-plan, they retry, they lose context
across restarts. The guarantee has to live at the tool boundary, outside
the agent's reasoning loop. That's what this is.

## How it works

Three gates, in order, before any byte leaves the machine:

1. **Ledger check.** A local sqlite ledger keyed by a deterministic
   operation id (`loop-morning-brief-2026-09-17`). If this op already
   recorded `sent` or `verified`, refuse. Exit 0 — nothing to do.
2. **Sent-folder check.** Search Sent for the same recipient + subject in
   the last 24 hours. If a match exists, refuse. Sent is the source of
   truth, so this gate holds even if the ledger is lost, the run moved
   machines, or the send happened outside this tool entirely.
3. **Send once, verify on ambiguity.** After sending, if the result is
   ambiguous (timeout, empty output, lost response), **never blind retry**.
   Re-check Sent: if the send landed, record it and report honestly. If
   it can't be confirmed, exit 3 and hand it to a human.

Notably, an *inconclusive pre-check* is also a refusal. When the tool
can't verify what already happened, the safe move is to stop, not to
guess.

## Usage

```
send-once --key <op-id> --to <addr> --subject <s> --body-file <f> \
          [--account <acct>] [--window-hours 24] [--dry-run]
```

Exit codes:

| Code | Meaning |
| ---- | ------- |
| 0 | sent, or already sent (recorded in ledger / found in Sent) |
| 2 | refused as duplicate — a gate fired |
| 3 | send failed and Sent could not confirm it — a human must verify |

The operation key is your idempotency key. Make it deterministic:
`<what>-<YYYY-MM-DD>`, not a UUID. Same intent, same key, and a retry
becomes a no-op instead of a duplicate.

## Mailer contract

`send-once` shells out to a CLI you configure with `SEND_ONCE_MAILER`
(space-separated) or `--mailer`. That CLI must speak this protocol:

```
<mailer> +triage --query 'in:sent to:<addr> newer_than:<N>h' \
    --max 20 --format json [--account <acct>]
    -> JSON: a list of {"id", "subject", ...}, or {"messages": [...]}

<mailer> +send --to <addr> --subject <s> --body <text> [--account <acct>]
    -> exit 0 on success; a JSON "id"/"message_id" in stdout is
       recorded in the ledger when present
```

Any Gmail CLI that can list Sent and send mail can be adapted with a thin
shim. Defaults: ledger at `$XDG_DATA_HOME/send-once/ledger.db`
(`~/.local/share/send-once/ledger.db`), overridable with
`SEND_ONCE_LEDGER`.

## The invitation

This solved my problem, not everyone's. Take it, make it better. If you
build something better, come back — we'll be customer #1, and we'll pay
you for it.

## License

MIT. See [LICENSE](LICENSE).
