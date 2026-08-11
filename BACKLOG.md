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

## Clean up test data by identifying attributes, never by id prefix

**Observed:** Windex, 2026-08-11.

A test sweep deleted rows matching `id LIKE 'ZZWXGATE%'` — the prefix used when
creating fixtures directly. It missed one, because that row had been created by
the **production code path under test** (an Edge Function), which mints its own
nanoid and knows nothing about test prefixes. The survivor looked like a real
player.

It got worse quietly: the row's `user_id` pointed at a test auth user, and the
FK is `ON DELETE SET NULL`. Deleting the auth user did not orphan the row
visibly — it **converted it into a pending player**, indistinguishable from the
54 genuine ones. The census read 85/55 instead of 84/54.

**The rule:** sweep on attributes that identify the data as test data regardless
of who created it — `display_name`, `email` (plus-addressing is ideal), a marker
column. Never on the id-generation scheme, because the moment a fixture is
created *by the system under test* rather than by the test, the prefix is gone.

**Corollary:** end every cleanup with a census assertion against known-good
totals, not just a "zero rows matching my filter" check. The filter is the thing
that was wrong; only an independent count catches that. Here `players = 84 AND
pending = 54` failed while every residue check passed, which is the only reason
it surfaced.

**Cost:** one nearly-missed 55th pending player, caught by the census assertion
rather than by the cleanup itself.

---

## Windex: a squatted player row is indistinguishable from a legitimate invite

**Observed:** Windex, 2026-08-11, during the Stage 3 squat rehearsal for
migration 056. **Zero occurrences in production.** Needs its own spec before
anyone builds it.

Migration 056 stops a stranger's unconfirmed signup from claiming a pending
player row, and the admin UI now surfaces that row rather than hiding it — a
strict improvement on the old behaviour, where the row was silently linked, the
invite button vanished, and `send-invite` 409'd with no explanation.

But the surfaced row reads **"Invited — awaiting first sign-in" with a Resend
Invite button** — pixel-identical to rows we genuinely invited. Three rows side
by side, two invited by an admin and one claimed by a stranger, looked the same.
The row that needs attention hides among the ones that don't.

**Rejected: neutral wording** (e.g. "Account pending"). It stops the UI from
falsely claiming we invited someone, but every row still looks alike — the
statement just gets vaguer. It fixes the false claim, not the actual problem,
which is that the row needing attention is not identifiable.

**The fix: distinguish self-signup from an invite we sent.** The discriminator
already exists and is the same fingerprint used to triage the original incident:

| origin | `invited_at` | `raw_user_meta_data` |
|---|---|---|
| `inviteUserByEmail` (send-invite) | set | Edge-Function `display_name` |
| `createUser` + `signInWithOtp` (invite-player) | NULL | Edge-Function `display_name` |
| **stranger self-signup** | **NULL** | **no `display_name`** — GoTrue defaults only |

So "we invited them" is `invited_at IS NOT NULL OR raw_user_meta_data ? 'display_name'`,
and anything else with an auth account is unclaimed-by-us.

**Open questions for the spec** — the reason this is not a one-line change:
pill wording; whether it needs a distinct colour rather than reusing the invited
amber; and whether the admin should be able to act on it directly (dismiss,
force-unlink, block the address) or only observe it. Exposing the flag also
needs a migration, since `get_players_auth_status()` does not currently return
it.

---

## Windex: recovering a squatted address confirms the squatter's account

**Observed:** Windex, 2026-08-11, during the Stage 3 squat rehearsal for
migration 056. **Recommendation: disable the password grant project-wide.**

### The finding

A stranger can `POST /auth/v1/signup` with someone else's email and the public
anon key, supplying **a password of their choosing**. Migration 056 stops that
unconfirmed account from claiming the victim's pending player row, which was the
reported harm — that part holds.

Recovery works by mailing a sign-in code to the address. It reaches the real
owner's inbox, they redeem it, and the row links to them. But redeeming sets
`email_confirmed_at` on **the squatter's account** — the same row, now confirmed
— which makes the squatter's password live. They could then use the public
password grant to hold a genuine session as that player.

The credential is inert only while the account stays unconfirmed
(`mailer_allow_unverified_email_sign_ins = false`). Recovery is exactly the act
that arms it.

**Language correction, recorded deliberately:** this branch was repeatedly
described as "self-healing" during development. That was wrong and the word has
been removed from the code. It **recovers the address**; it does not clean up
the account. The row ends up owned by the right person *and* by the squatter.

### Recommended fix: disable the password grant project-wide

Windex has **no user-facing password flow anywhere**. `login.tsx` only ever
calls `signInWithOtp` / `verifyOtp`; `signInWithPassword` is exposed on the auth
context but never invoked by any screen. The password endpoint exists purely as
attack surface.

This is a **config change, not code** — the same shape as the
`mailer_autoconfirm` fix, and it needs the same discipline:

1. Audit what actually uses the password grant before flipping it — including
   the admin app's auth path, which must be confirmed unaffected.
2. Read the setting back **by value**, and remember that a 200 plus a matching
   read-back proves storage, not propagation. Verify with a real sign-in
   afterwards, not just a GET.

### Rejected alternatives, and why

- **Delete and recreate the unconfirmed account** in the invite branch. Destroys
  an auth record on suspicion. An unconfirmed account is not necessarily hostile
  — it is just as likely a legitimate user who abandoned onboarding halfway.
- **Clear `encrypted_password` when linking on confirmation.** Mutates someone
  else's auth record from a path that runs during ordinary sign-in. Too much
  blast radius for a trigger on the auth critical path, and it silently breaks
  any future legitimate password use.

Both treat the symptom on a hot path. Disabling the grant removes the capability
outright, in one place, reversibly.

---

## Not yet placed

There is currently **no "silent save" section** in
`Buzz_Project_Development_Procedure.md`. The assert-the-outcome item above is
the natural seed for one — likely a new subsection under Phase 3, adjacent to
§3.2b (backend deploys are git operations), since both are about not trusting a
command's exit status as proof of effect.

The propagation item may also belong in §2.2c (email relay) or §2.2d (pre-launch
auth checklist) once it is confirmed on a second project.
