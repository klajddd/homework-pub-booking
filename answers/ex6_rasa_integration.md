# Ex6 — Rasa structured half

## Your answer

The RasaStructuredHalf subclass overrides run() to POST a booking
intent to Rasa's REST webhook and interpret the response. Input
payload flows: loop half produces raw booking data → StructuredHalf
calls normalise_booking_payload (via validator.py) to produce a
Rasa-shaped message with canonical types → urllib POST to Rasa →
parse response for {action: committed} or {action: rejected} custom
slots.

For offline mode we spawn a stdlib http.server thread that mimics a
Rasa webhook. It always confirms, which is enough for unit tests.
Rejection is exercised in Ex7 where the loop half's arguments drive
the decision.

Three design choices worth noting: (1) we raise ValidationFailed in
normalise_booking_payload and catch it in run() rather than letting
it propagate; the StructuredHalf contract demands a HalfResult. (2)
Network errors return success=False with SA_EXT_SERVICE_UNAVAILABLE
— the caller decides whether to retry. (3) The stable sender_id is a
hash of (venue+date+time) so the Rasa tracker is consistent across
retries within one session.

## Citations

- starter/rasa_half/validator.py — normalise_booking_payload + helpers
- starter/rasa_half/structured_half.py — RasaStructuredHalf.run + mock server
- sessions/examples/ex7-handoff-bridge/sess_6f1c23f9e1b3/logs/trace.jsonl — rejection event `party_too_large` from structured half in round 1
