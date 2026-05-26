# Ex9 — Reflection

## Q1 — Planner handoff decision

### Your answer

Looking at session `sess_6f1c23f9e1b3` (Ex7), the handoff signal doesn't
come from where you'd expect. The planner's round 1 output (ticket
`tk_5d75c8fe`, raw_output.json) shows:

```json
{"id": "sg_1", "description": "find venue near haymarket for 12",
 "assigned_half": "loop", "success_criterion": "candidate identified"}
```

The planner never assigns `"assigned_half": "structured"` — not in round 1
(`tk_5d75c8fe`) and not in round 2 after the rejection (`tk_91d673cc`:
description "retry with larger venue after rejection", still
`"assigned_half": "loop"`). The planner only ever thinks in terms of loop
work.

The actual handoff decision is made by the executor. 
Ticket `tk_21b9d4da`
(round 1 executor, raw_output.json) shows it called `handoff_to_structured`
as a tool with:

```
reason: "loop half identified a candidate venue; passing to structured half
         for confirmation under policy rules"
```

The trace.jsonl records what happens right after:
`{"event_type": "session.state_changed", "payload": {"from": "loop",
"to": "structured", "round": 1}}`. That's the handoff signal — a tool call
made at runtime, not something the planner decided upfront.

After the structured half rejected the booking
(`rejection_reason: "sorry, we can't accept this booking. reason: party_too_large"`
in trace.jsonl), the bridge re-ran the planner. Round 2 plan (`tk_91d673cc`)
still assigns `"loop"` — the structured half never shows up in any planner
ticket at all.

So the split of responsibility is: the planner decides what to do and in
what order; the executor decides at runtime when it's done exploring and
needs to hand off to the rule-based structured half. The
`handoff_to_structured` tool call is that decision made explicit — it writes
an IPC file and tells the bridge to switch state.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_5d75c8fe/raw_output.json` — round 1 plan, `assigned_half: "loop"`
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_21b9d4da/raw_output.json` — round 1 executor, `handoff_to_structured` tool call
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_91d673cc/raw_output.json` — round 2 plan, again `assigned_half: "loop"`
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/trace.jsonl` — four `session.state_changed` events: loop→structured→loop→structured→complete

---

## Q2 — Dataflow integrity catch

### Your answer

While working on Ex5 I found a self-verifying validation bug that would
have let any fabricated number pass uncaught.

The original `fact_appears_in_log` in integrity.py scanned both `r.output`
AND `r.arguments` on every tool call record. In session `sess_7836a6ef8dac`,
the trace.jsonl shows `calculate_cost` returning
`{"total_gbp": 356, "deposit_required_gbp": 71}` — so "total £356, deposit £71".
The executor then called `generate_flyer` passing those same values in as
arguments: `event_details={"total_gbp": 356, "deposit_required_gbp": 71, ...}`.
So now £356 exists in `generate_flyer`'s **arguments** record too.
`generate_flyer`'s output is just `{"path": "workspace/flyer.html",
"bytes_written": 1116}` — it doesn't compute any financial figures itself.

The bug: when `verify_dataflow` checked whether `£356` appeared in the
tool log, it found it in `generate_flyer`'s arguments and returned `ok=True`
— it was confirming the value against the call that *received* it, not the
tool that was used to *compute* it.

The fabrication scenario is concrete: change the FakeLLMClient to pass
`total_gbp=560` to `generate_flyer` instead. The flyer shows `£560`. A
human reviewer sees a plausible number and moves on. With the old check,
`fact_appears_in_log('£560')` finds 560 in `generate_flyer`'s arguments →
`ok=True` → fabrication goes undetected. With the fix —
`return any(_scan(r.output) for r in records)` — 560 isn't in
`calculate_cost`'s output (`total_gbp: 356`) → `ok=False,
unverified_facts=['£560']` → caught.

The reason this matters is that the check is specifically designed to catch
plausible fabrications that visual review would miss. A number like £560
looks reasonable for a party booking — you'd only know it's wrong if you
cross-reference it against what the tools actually returned.

To reproduce: seed `_TOOL_CALL_LOG` with `calculate_cost` output
`{"total_gbp": 356}`, call `verify_dataflow("Total: £560")`, assert
`result.ok is False`.

### Citation

- `sessions/examples/ex5-edinburgh-research/sess_7836a6ef8dac/logs/trace.jsonl` — `calculate_cost` output `{"total_gbp": 356, "deposit_required_gbp": 71}` and `generate_flyer` arguments
- `starter/edinburgh_research/integrity.py:112` — the fixed line: `return any(_scan(r.output) for r in records)`

---

## Q3 — First production failure

### Your answer

The first failure I'd expect if we shipped next week is a **lost
confirmation**: the Rasa webhook processes the booking and marks it
confirmed, but the network drops the response before the bridge reads the
HTTP reply. The bridge sees a timeout, doesn't know whether Rasa actually
committed the booking, and has to choose: retry (risk double-booking the
pub) or fail closed (risk the customer getting no confirmation). Without
extra state, neither option is safe.

The **ticket state machine** is what makes this recoverable.

In session `sess_6f1c23f9e1b3`, the bridge created four tickets across two
rounds: planners `tk_5d75c8fe` and `tk_91d673cc`, executors `tk_21b9d4da`
and `tk_fa5c8120`. Each ticket has a `state.json` that records whether it
finished as `pending`, `success`, or `error`. When `tk_fa5c8120` completed
— the round 2 structured call that confirmed the booking — that state was
written to disk before the process returned.

This is what saves you in the crash scenario: if the bridge dies after Rasa
responds but before `tk_fa5c8120` is written as `success`, the ticket stays
`pending` or `error`. On restart, the bridge sees that and knows it needs
to retry. If the ticket already shows `success`, the bridge knows the
booking went through and skips the retry — no double-booking.

Without the ticket state machine, the only thing on disk is the IPC file at
`ipc/handoff_to_structured.json` — it's either there or it's not. That
tells you a handoff was *requested* but nothing about whether it completed.
The ticket gives you the three-way distinction (`pending` / `success` /
`error`) that makes a crash-safe retry actually possible. That's the
difference between a demo and something you'd trust with a real booking.

### Citation

- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_fa5c8120/state.json` — round 2 executor ticket, terminal state
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/tickets/tk_21b9d4da/state.json` — round 1 executor ticket, terminal state
- `sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/trace.jsonl` — `session.state_changed` from→structured, to→complete (round 2)
