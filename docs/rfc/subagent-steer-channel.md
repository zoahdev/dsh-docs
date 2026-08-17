# RFC: `subagent/steer` — a first-class steer channel for subagent sessions

> Status: community draft · related upstream discussion
> https://github.com/deepseek-ai/deepseek-harness/discussions/1968 ·
> 2026-08-17

## Summary

Today a parent can queue subagent sessions but cannot steer one mid-run. A
verified community experiment shows the gap is UI-gating, not architectural:
the API layer does not reject subagent steering — the shipped InputBar gates
it with `steeringAvailable = subagent === null`, and a client plugin can
restore the parent→subagent message path without any scheduler change.

This RFC makes the steer channel first-class with three small pieces:

1. **`subagent/steer` session event** — a steering message (+ optional turn
   budget) injected into the target subagent session as an ordinary user turn.
2. **InputBar gate removal** — steer becomes an explicit, discoverable action
   instead of a hidden client-plugin trick.
3. **Alignment with #1517 / #1473 / #2006** — steer composes with managed jobs
   and subagent model selection rather than competing with them.

No scheduler change and no session-log format change are required.

## Problem

- Subagent sessions are queue-only from the Composer: the parent can enqueue a
  task, then has no supported way to redirect it when the goal changes.
- The shipped UI hides the capability (`steeringAvailable = subagent ===
  null`), so the only working implementations today are zero-hack client
  plugins that know the internal API shape.
- Steering must not introduce a new execution model: the subagent loop already
  consumes session events, so a steer message should land as a normal
  user-turn injection into that session.

## Proposal

### 1. Session event: `subagent/steer`

```json
{
  "type": "subagent/steer",
  "data": {
    "sessionId": "<subagent session id>",
    "steer": "message text",
    "budget": { "turns": 3 }
  }
}
```

Semantics:

- `sessionId` must reference a live subagent session; otherwise the event is
  rejected with a typed error (`ERR_NOT_SUBAGENT` / `ERR_SESSION_DISPOSED`).
- `steer` is injected as a user-turn into that session, so the existing loop
  consumes it without new plumbing.
- `budget` is optional. When present, it caps the subagent's remaining turns
  for the current run; when absent, the steer message is unbounded like a
  normal user turn.
- The event is idempotent per (sessionId, steer, budget): re-delivery must not
  double-inject.

### 2. InputBar gating

- Relax `steeringAvailable = subagent === null` to a runtime query of live
  subagent sessions for the current parent.
- Keep queue-as-default; steer is an explicit action (button / shortcut), not
  a mode change.
- Surface a subagent picker when more than one live subagent exists.

### 3. Alignment with the managed-jobs direction

- #1517 (managed jobs) can adopt `subagent/steer` as one of its verbs; the
  event shape is small enough to embed in the job protocol.
- #1473 / #2006 (subagent model selection / inheritance) stay orthogonal:
  steer targets whatever model the subagent currently runs.

## Compatibility

- No scheduler change; no session-log format change.
- Unknown event types are already ignored by older clients, so shipping the
  event before the UI lands is safe.
- The working client plugin proves the message path exists on the current
  API; this RFC only standardizes it.

## Verification plan

1. e2e: parent starts a subagent, sends `subagent/steer`, asserts the injected
   turn lands in the subagent session and the subagent's next reply reflects
   the new goal.
2. API-level: steer to a disposed session returns `ERR_SESSION_DISPOSED`;
   steer to a non-subagent session returns `ERR_NOT_SUBAGENT`.
3. UI: with two live subagents, the picker lets the parent choose the target;
   steering does not disturb the queued parent flow.

## Open questions

- Should `subagent/steer` be persisted in the session log? (It is a user turn,
  so persist by default; flag for review.)
- Turn-budget semantics when the subagent is inside a tool call: defer the
  budget until the tool returns, or count the tool call as one turn?
- Should the parent receive an ack event when the steer is delivered
  (`subagent/steer-ack`), to make the UI optimistic-update safe?
