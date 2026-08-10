# Backlog

Cross-project lessons that are confirmed but not yet folded into
`Buzz_Project_Development_Procedure.md`. Promote an item into the procedure doc
once it has bitten on a second project, or once it clearly belongs in a specific
phase; delete it here when promoted.

Each item records what was observed, why it matters, and what it cost — the cost
line is what justifies the rule to a future reader who is tempted to skip it.

---

## Supabase auth config writes propagate to GoTrue on a delay

**Observed:** Windex, 2026-08-10.

A Management API `PATCH /v1/projects/<ref>/config/auth` returns 200 and an
immediate follow-up `GET` returns the new value — but the running GoTrue
container keeps serving the *previous* config until it reloads.

Proven by three sends against the same stored template:

| send | elapsed since the config write | rendered |
|---|---|---|
| #1 | before the write | old template (expected) |
| #2 | ~20–45 seconds after | **old template** |
| #3 | ~110 minutes after | **new template** |

Storage was correct and the custom-content flag (`mailer_templates_custom_contents.MAILER_TEMPLATES_INVITE_CONTENT`)
was `true` the entire time. Nothing about the stored state distinguished the
broken send from the working one.

**Why it matters:** a 200 plus a matching read-back proves **storage**, not
**propagation**. This is the same class of error as trusting CLI deploy output,
one layer further out — and it is more dangerous because the read-back *feels*
like verification.

**The rule:** for any auth config change that affects email rendering
(`mailer_templates_*`, `mailer_subjects_*`), the read-back is necessary but not
sufficient. Wait, then do a verification send to a fresh plus-address and
confirm what actually rendered in the inbox. Never conclude a template is broken
from a send made shortly after the write — re-send later first.

**Also:** org audit logs return 404 on this plan, so a config write cannot be
timestamped after the fact. Record the write time yourself when sequencing will
matter to the diagnosis.

**Cost:** two wasted invite sends and an incorrect "the template write failed"
hypothesis that consumed a full diagnostic cycle.

---

## GoTrue OTP token storage — `confirmation_token`, not `recovery_token`

**Observed:** Windex, 2026-08-10.

For `inviteUserByEmail` and signup confirmation, GoTrue stores the OTP in
`auth.users.confirmation_token` as **`sha224(email || otp)`** — 56 hex
characters, with no `pkce_` prefix for server-side (implicit-flow) calls.
`recovery_token` stays **empty** for those flows; it is used by magic-link and
recovery instead.

Three consequences:

1. **The plaintext code is never stored**, so it cannot be read out of the
   database. Given a code from the inbox, confirm it before spending it:
   `node -e "console.log(require('crypto').createHash('sha224').update(email+otp).digest('hex'))"`
   and compare against `confirmation_token`. This isolates "wrong code" from
   "redemption rejected the code" before you burn a single-use token.
2. **Any instrumentation that reads `recovery_token` only is blind to every
   invite and signup.** This is the hard evidence for the existing Windex item
   about `otp_pending` instrumentation — it is not a theory.
3. **An invite-issued token redeems through `verifyOtp({ email, token, type:
   'email' })`** — the same call the app already makes for ordinary sign-in.
   `type: 'invite'` is not required. Verified end to end, which is why the
   `inviteUserByEmail` choice in Windex's `send-invite` Edge Function did not
   need reversing.

---

## Assert the outcome by reading state — never infer it from the call returning

**Observed:** Windex, 2026-08-10. This failure mode recurred **three times in a
single day, at three different layers**, which is why it is worth a standing
rule rather than three separate notes.

| layer | how it lied |
|---|---|
| PostgREST | `Prefer: return=minimal` returns 200 on a PATCH that matched **zero** rows |
| Management API | accepts **unknown config keys** and returns 200, changing nothing |
| PowerShell / tooling | a result-shape error iterated an empty id and reported three "failures" against a blank target — the intended deletes never even ran |

Three different systems, one shape: **the call returned, so the work must have
happened.** In each case the call was honest about its own success and silent
about the outcome.

**The rule:** after any state-changing call, read the state back and assert on
it *by value*. Specifically —

- Reject a write whose read-back does not contain the value you sent.
- Reject a write where a key you sent is **absent** from the read-back; an
  unrecognised key is a failure, not a no-op.
- On a delete or bulk update, assert the **residue count is zero** (or the
  expected remainder), not that the call returned.
- When iterating, assert the loop actually had targets. A loop over an empty
  collection completes "successfully" and does nothing.

Corollary already in the procedure doc for deploys (fetch the artifact back);
this generalises it to every layer.

---

## Not yet placed

There is currently **no "silent save" section** in
`Buzz_Project_Development_Procedure.md`. The assert-the-outcome item above is
the natural seed for one — likely a new subsection under Phase 3, adjacent to
§3.2b (backend deploys are git operations), since both are about not trusting a
command's exit status as proof of effect.

The propagation item may also belong in §2.2c (email relay) or §2.2d (pre-launch
auth checklist) once it is confirmed on a second project.
