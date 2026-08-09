# Telling a User Cancel Apart from an Internal Deadline

## Problem

A request to a provider can be aborted for two reasons that look identical at
the error: a user pressing Esc, and an internal deadline firing (a goal-judge
timeout, a memory-recall budget, the workflow wall-clock cap). Both abort the
same request signal and both make the SDK reject abort-shaped
(`APIUserAbortError`, or a DOMException `AbortError`). The two logging paths —
the `api_error` telemetry event and the OpenAI debug log — need to suppress the
first (a cancel is not a failure) and keep the second (a timed-out request is a
failure). They cannot tell them apart from the error alone.

#8398 (the predecessor PR) fixed the immediate noise with the approximately
correct gate `signal.aborted && isAbortError(error)`. That suppresses a user
cancel, but it also suppresses a timed-out internal request — hiding a real
failure. This PR makes the distinction real.

## The decision: which side is the default?

Since the two cases are indistinguishable at the error, the aborting side has
to say which it is. That leaves a polarity choice, and it is the whole design:

- **Suppress by default, deadlines opt out** (tag every deadline as a
  `TimeoutError`). A deadline that forgets to tag itself is silently
  suppressed — a telemetry hole with nothing to catch it. The deadline-producer
  set is open and growing (an earlier attempt took nine review rounds to find
  seven producers, and still missed one).

- **Report by default, cancels opt in** (tag every user cancel). A cancel that
  forgets to tag itself surfaces as one spurious `api_error` — the recoverable
  #8398 noise, not a silent hole. The user-cancel producer set is small and
  closed.

This PR takes the second. The contract cannot be a compile-time type, so the
default belongs on the safe side, and the set that must be complete should be
the small closed one.

## Design

- `USER_CANCEL_ABORT_REASON = 'qwen:user-cancel'` (in `utils/errors.ts`) is the
  one tag a deliberate user cancel carries.
- `isUserCancel(error, signal)` is true only when the signal was aborted **with
  that reason** and the error is abort-shaped. Both logging gates call it.
- The user-cancel producers set the tag:
  - TUI turn cancel (`useGeminiStream.ts` `cancelOngoingRequest`) — previously a
    bare `abort()`, now `abort(USER_CANCEL_ABORT_REASON)`.
  - ACP/daemon session (`Session.ts`) — already used `'qwen:user-cancel'`; now
    imports the shared constant so the producer and the gate cannot drift.
- Everything else — deadlines, budgets, watchdogs — is untagged and therefore
  reported. No per-producer conversion is needed for them.

## Not covered / open

- **The daemon prompt deadline still launders through the cancel tag.** When a
  `qwen serve` deadline fires, the bridge forwards a generic cancel and the
  agent re-stamps the model-facing signal `'qwen:user-cancel'` at the Session
  admission boundary — so it reads as a user cancel and is suppressed. The
  deadline is still surfaced through its `prompt_deadline_exceeded` terminal and
  the errored LLM span; only the provider-health `api_error` is affected. Fixing
  it needs deadline attribution to cross the ACP wire (via `CancelNotification`
  `_meta`) so the re-stamp can use a timeout reason. Deferred; called out here
  so it is not mistaken for handled.
- **Completeness of the cancel-producer set.** This PR tags the two producers
  that reach a model request today (TUI, ACP). Secondary aborts triggered by a
  cancel (auxiliary/background completions, batch tool aborts) should be audited
  before relying on the invariant broadly — a missed one is recoverable noise,
  but it is still noise.
- **`isAbortError` recognition.** It still matches `APIUserAbortError` by class
  name. Moving that to an `instanceof` check at the provider boundary (keeping
  `utils/errors.ts` provider-agnostic) is a good companion change but is
  independent of this polarity decision.

## Test plan

- `isUserCancel`: true only for an abort-shaped error on a signal tagged
  `USER_CANCEL_ABORT_REASON`; false for a bare abort, a `TimeoutError` reason, a
  different string reason, no signal, and a non-abort error.
- The telemetry gate: a tagged cancel is suppressed; an internal deadline
  aborting the same request is reported; a real failure racing a cancel is
  reported.
- `shouldSuppressErrorLogging` (OpenAI + Qwen families) agrees with the gate.
