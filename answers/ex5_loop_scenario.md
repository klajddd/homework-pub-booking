# Ex5 — Edinburgh research loop scenario

## Your answer

The planner produced two subgoals (ticket `tk_b689bacb`, session
`sess_7836a6ef8dac`): sg_1 "research Edinburgh venues near Haymarket for a
party of 6" and sg_2 "produce an HTML flyer with the chosen venue, weather,
and cost". Both assigned to loop. Both ran in the same executor session.

Turn 1 called venue_search, get_weather, and calculate_cost in parallel —
all three are parallel_safe=True because they only read fixtures. The trace
shows all three as simultaneous executor.tool_called events. Turn 2 wrote the
flyer via generate_flyer (parallel_safe=False — writes workspace/flyer.html).
Turn 3 called complete_task.

calculate_cost returned total_gbp=356 and deposit_required_gbp=71 for
haymarket_tap, party=6, 3 hours at bar_snacks tier. The cost formula is
`total = max(subtotal, min_spend) + service + hire_fee`, treating min_spend as
a floor rather than an additive term.

The dataflow integrity check verified 4 facts from the flyer against tool
outputs: £356 and £71 from calculate_cost's output, 12 from get_weather's
temperature_c, and "cloudy" from get_weather's condition. The check scans only
`r.output` fields — not `r.arguments` — so facts can only be verified if a
tool actually produced them, not just received them as input.

## Citations

- sessions/examples/ex5-edinburgh-research/sess_7836a6ef8dac/logs/trace.jsonl — tool call sequence with parallel venue_search+get_weather+calculate_cost
- sessions/examples/ex5-edinburgh-research/sess_7836a6ef8dac/logs/tickets/tk_b689bacb/raw_output.json — planner plan with both subgoals
- sessions/examples/ex5-edinburgh-research/sess_7836a6ef8dac/workspace/flyer.html — the produced HTML flyer
