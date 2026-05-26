# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

In session `sess_6f1c23f9e1b3` (Ex7), the signal the question asks about is
visible in an unexpected place. The planner's round 1 output (ticket
`tk_5d75c8fe`, raw_output.json) reads:

```json
{"id": "sg_1", "description": "find venue near haymarket for 12",
 "assigned_half": "loop", "success_criterion": "candidate identified"}
```

The planner never writes `"assigned_half": "structured"` — not in round 1
(`tk_5d75c8fe`) and not in round 2 after the rejection (`tk_91d673cc`:
description "retry with larger venue after rejection", again
`"assigned_half": "loop"`). The planner treats the entire task as loop work.

The actual handoff decision lives in the executor. Ticket `tk_21b9d4da`
(round 1 executor, raw_output.json) shows the executor called
`handoff_to_structured` as a tool with:

```
reason: "loop half identified a candidate venue; passing to structured half
         for confirmation under policy rules"
```

The trace.jsonl records the consequence immediately after:
`{"event_type": "session.state_changed", "payload": {"from": "loop",
"to": "structured", "round": 1}}`. That is the handoff signal — a tool call,
not a planner assignment.

After the structured half rejected (`rejection_reason: "sorry, we can't
accept this booking. reason: party_too_large"` in trace.jsonl), the bridge
rebuilt the task and re-ran the planner. Round 2 plan (`tk_91d673cc`) again
assigns `"loop"`. The structured half never appears in any planner ticket.

The architectural lesson: the planner handles strategic decomposition (what
to do, in what order); the executor decides at runtime when open-ended
research should yield to deterministic rule enforcement. The
`handoff_to_structured` tool call is that decision made explicit — it writes
an atomic IPC file and signals the bridge to transition state.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_5d75c8fe/raw_output.json` — round 1 plan, `assigned_half: "loop"`
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_21b9d4da/raw_output.json` — round 1 executor, `handoff_to_structured` tool call
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_91d673cc/raw_output.json` — round 2 plan, again `assigned_half: "loop"`
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/trace.jsonl` — four `session.state_changed` events: loop→structured→loop→structured→complete

---

## Q2 — Dataflow integrity catch

### Your answer

During Ex5 development I found a self-verifying validation bug that would
have allowed any fabricated number to pass uncaught.

The original `fact_appears_in_log` in integrity.py scanned both `r.output`
AND `r.arguments` on every tool call record. In session `sess_7836a6ef8dac`,
the trace.jsonl shows `calculate_cost` recording output
`{"total_gbp": 356, "deposit_required_gbp": 71}` (summary: "total £356,
deposit £71"). The executor then called `generate_flyer` with
`event_details={"total_gbp": 356, "deposit_required_gbp": 71, ...}` — those
same values now also exist in `generate_flyer`'s **arguments** record.
`generate_flyer`'s output is just `{"path": "workspace/flyer.html",
"bytes_written": 1116}` — it produces no financial facts of its own.

The bug: `verify_dataflow` checking `£356` in the flyer found it in
`generate_flyer`'s arguments and returned `ok=True` — confirming the value
against the very call that received it, not the upstream tool that computed it.

The fabrication scenario is concrete and reproducible: change the
FakeLLMClient to pass `total_gbp=560` to `generate_flyer`. The flyer shows
`£560`. A human reviewer sees a plausible number and moves on. With the old
integrity check, `fact_appears_in_log('£560')` finds 560 in
`generate_flyer`'s arguments → `ok=True` → fabrication undetected. With the
fixed check — `return any(_scan(r.output) for r in records)` — 560 is not in
`calculate_cost`'s output (`total_gbp: 356`) → `ok=False,
unverified_facts=['£560']` → caught.

The check catches plausible fabrications that visual review misses precisely
because it cross-references each value against ground-truth tool outputs, not
against "does this look reasonable."

To reproduce: seed `_TOOL_CALL_LOG` with `calculate_cost` output
`{"total_gbp": 356}`, call `verify_dataflow("Total: £560")`, assert
`result.ok is False`.

### Citation

- `sessions/examples/ex5-edinburgh-research/sess_7836a6ef8dac/logs/trace.jsonl` — `calculate_cost` output `{"total_gbp": 356, "deposit_required_gbp": 71}` and `generate_flyer` arguments
- `starter/edinburgh_research/integrity.py:112` — the fixed line: `return any(_scan(r.output) for r in records)`

---

## Q3 — First production failure

### Your answer

The first failure I'd expect shipping next week is a **lost confirmation**:
the structured half's Rasa webhook processes the booking and marks it
confirmed, but the network drops the response before the bridge reads the
HTTP reply. The bridge sees a timeout, doesn't know whether Rasa committed,
and must decide: retry (risk double-booking the pub) or fail closed (risk the
customer getting no confirmation). Without additional state, neither option is
safe.

The **ticket state machine** is the primitive that surfaces this correctly.

In session `sess_6f1c23f9e1b3` the bridge produced four tickets across two
rounds: planners `tk_5d75c8fe` and `tk_91d673cc`, executors `tk_21b9d4da`
and `tk_fa5c8120`. Each ticket has a `state.json` that records its terminal
state (`pending`, `success`, `error`). When `tk_fa5c8120` completed — the
round 2 structured call confirmed — that state was durably written before the
process returned.

The value for the crash scenario: if the bridge process dies after Rasa
responds but before `tk_fa5c8120` reaches `success`, the ticket stays in
`pending` or `error`. On restart, the bridge reads that state and knows the
structured call must be retried. If the ticket already shows `success`, the
bridge knows the booking went through and skips the retry — no double-booking.

Without the ticket state machine the only durable signal is the IPC file at
`ipc/handoff_to_structured.json` — binary present/absent. That tells you a
handoff was requested but not whether it completed. The ticket gives the
three-way distinction (`pending` / `success` / `error`) that makes crash-safe
retry possible, which is the exact gap between a demo and a system you'd
trust with a real booking.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_fa5c8120/state.json` — round 2 executor ticket, terminal state
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_21b9d4da/state.json` — round 1 executor ticket, terminal state
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/trace.jsonl` — `session.state_changed` from→structured, to→complete (round 2)
