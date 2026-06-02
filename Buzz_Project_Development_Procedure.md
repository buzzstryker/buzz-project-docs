# Buzz Project Development Procedure
*Standard operating procedure for all new projects*
*Account: buzzstrykertest@gmail.com*
*Stored at: `C:\Users\buzzs\OneDrive\Desktop\Projects\Buzz_Project_Development_Procedure.md`*

> **Note on naming:** the auth/RLS reference project formerly called `late-add-v2` was renamed to `windex` in May 2026. All references in this doc use the new name; if you're searching old commits, notes, or scratchpads for the old name, this is the same project.

---

## Phase 1   Project Setup & Planning

### 1.1 Explore & Define
- Use chat engines (Claude, Perplexity, ChatGPT) to explore the problem space
- Ask each: *"Who would be best at what part of this project?"*   save answer to `ai_team.md`
- Define the project with a short directory name (no spaces, lowercase preferred)

### 1.2 Create Core Documents
Create these in the local project directory:
- `Project_Context.md`   what the project is, why it exists, data model, key decisions
- `Project_README.md`   overview, stack, how to run it
- `ai_team.md`   which AI tools to use for which tasks

**Use the template:** Copy `Project_Context_TEMPLATE.md` from `C:\Users\buzzs\OneDrive\Desktop\Projects\Project_Context_TEMPLATE.md` as your starting point for `Project_Context.md`. The template includes a "Working agreement with Claude" section that enforces key procedure rules in every Claude Code session — do not skip this section.

### 1.3 Create Claude Project
- Create a Claude Project with the same name as the local directory
- Upload `Project_Context.md` and `Project_README.md` to the project
- All analysis, architecture decisions, and major work sessions happen inside this project
- Claude retains full context across sessions   reopen the project next season and everything is there

---

## Phase 2   Infrastructure

### 2.1 GitHub
- Create a GitHub repo with the same name as the local directory
- Initialize with `.gitignore` appropriate for the stack (Next.js, Node, Python, etc.)
- First commit: project docs and initial scaffold (tell Claude Code to do it — see Phase 3)
- **Immediately after first commit: run `/gsd-new-project` in Claude Code**
  - This initializes GSD's planning structure (.planning/ directory)
  - GSD interviews you about what you're building and creates ROADMAP.md, REQUIREMENTS.md, STATE.md
  - All subsequent development phases are tracked here
  - If you skip this and build first, run `/gsd-map-codebase` first to map what exists, then `/gsd-new-project`
  - Without this, GSD has no planning structure and `/gsd-progress` will show 0 phases

### 2.2 Supabase (if needed)
- Create a Supabase project with the same name
- Collect keys from Supabase dashboard: **Project Settings → API**

**RLS posture — choose deliberately, document the choice:**

- **Single-owner project (all data accessible to one operator via service-role):**
  Disable RLS on all `public.*` tables. Document this choice in `CLAUDE.md`.
  Reads happen server-side via `SUPABASE_SERVICE_ROLE_KEY` which bypasses RLS
  anyway; enabling RLS without policies is a silent footgun (default-deny
  breaks any future client-side authenticated read).

- **Multi-user project (different users see different data, or any
  server-side code uses the SSR/authenticated client):**
  Enable RLS AND write at least one SELECT policy per table. Never enable
  RLS without a corresponding policy — RLS-enabled-no-policy is default-deny
  and will silently block the auth gate. Reference: windex's
  `015_rls_overhaul.sql` is the canonical pattern (every RLS-enabled table
  gets an explicit `CREATE POLICY` for SELECT, INSERT, UPDATE, DELETE).

- **The trap:** "I'll enable RLS now and add policies later" never works.
  The instant RLS is on, default-deny activates. Any authenticated-client
  read returns zero rows. The proxy redirects to `/login`. Users are locked
  out. Either ship policies in the same migration as the
  `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`, or don't enable RLS yet.

- **Diagnostic for "users can't log in but everything else works":**
  Run `SELECT * FROM pg_policies WHERE schemaname = 'public';` — if the
  table the auth gate queries (typically `family_users` / `app_users` /
  `user_profiles` / similar) has `rowsecurity=true` but zero policies,
  that's your lockout.

**Lesson learned the hard way (Family Spend, May 2026):** RLS enabled on 11
tables with zero policies. Auth gate queried `family_users` via SSR client,
got zero rows, rejected every sign-in as "not on family list" even though
the email was in the table. Fix was a 2-line migration adding permissive
SELECT policies on `family_users` and `members` for the `authenticated`
role. Should have been part of whatever migration enabled RLS in the first
place.

**Supabase CLI setup (run once per project in the project directory):**
```bash
npx supabase init
npx supabase link --project-ref [your-project-ref]
```
Project ref is the alphanumeric string in your Supabase project URL.

**Applying SQL migrations:**
```bash
npx supabase db query --linked -f supabase/migrations/001_initial_schema.sql
```

**Connection troubleshooting — IPv6-only DNS (common on home networks):**
Direct `psycopg2` connections to `db.xxxxxx.supabase.co` will fail if your router resolves that host to IPv6 only. Symptoms: `getaddrinfow` error or `hostname resolving error` in the stack trace. Fix: always use `--linked` flag with the Supabase CLI, which routes through the Management API instead of a direct TCP connection. Do not waste time debugging the direct connection.

**Python DB dependency — install before running any ingestion scripts:**
```bash
pip install psycopg2-binary
```

**Windows `.env.local` naming gotcha:**
Windows may save the file as `.env.local.txt` if created in Explorer. Claude Code can rename it:
```bash
mv .env.local.txt .env.local
```
Always verify the exact filename before running scripts.

### 2.2b Supabase Auth — Use OTP Code Flow, Not Magic Links

**Always use OTP code flow (6–8 digits depending on `mailer_otp_length`) instead of clickable magic links.** This is the standard for all projects going forward.

**Why OTP code flow is better:**
- Works identically on web and mobile — user enters the code, no link to tap
- No deep-link required on mobile — eliminates the entire `honcut://` scheme / Expo Go limitation problem
- No EAS dev build required just to test auth on a phone
- No URL fragment / hash stripping issues in the auth callback
- No custom SMTP rate limit problems (the same email-send path, but no redirect URL complexity)
- Works in any email client without browser/app switching issues

**How to implement:**
- Email template: use `{{ .Token }}` (the OTP code), NOT `{{ .ConfirmationURL }}`
- Sign-in flow: user enters email → gets OTP code by email → enters code in app → signed in
- Supabase call: `supabase.auth.signInWithOtp({ email, options: { shouldCreateUser: false } })` to send the code
- Verification: `supabase.auth.verifyOtp({ email, token, type: 'email' })` to verify
- No `/auth/callback` route needed — no redirect URL, no deep link, no hash fragment handling

**OTP length — match the UI to whatever Supabase actually sends:**

Supabase's `mailer_otp_length` defaults to 6 but can be configured up to 8.
The login UI must accept whatever length the project's Supabase config
produces. Don't hardcode the length; use a tolerant validator like
`pattern="\d{6,8}"` and `maxLength={8}` so the UI is drift-resistant.

If you do change `mailer_otp_length` via Management API, document in
`CLAUDE.md` (per 3.2b).

**Lesson learned the hard way (Family Spend, May 2026):** UI shipped with
6-digit hardcoded validation; Supabase project was configured for 8 digits.
Result: every successful OTP send produced a code the UI refused to accept.
Fix was a 4-line UI patch. The drift-resistant pattern (accept 6–8) means
this can't recur even if Supabase config changes.

**How to onboard users at launch — do NOT use inviteUserByEmail:**

`inviteUserByEmail` sends one email per user and hits Supabase's 2/hour rate limit at user 3. Use `admin.createUser` instead:

```typescript
await supabase.auth.admin.createUser({
  email: row.email,
  email_confirm: true,       // sends ZERO emails, no rate limit
  user_metadata: {
    partner_id: row.partner_id,
    role: row.role,
    display_name: row.display_name,
    is_admin: row.is_admin,
  },
})
```

This creates all users instantly with no emails sent. The `handle_new_user` trigger still fires and populates the linked profile row from `user_metadata`.

**Launch day flow:**
1. Run the admin script — all users created in seconds, zero emails
2. Send a group text: "App's updated. Open [app], enter your email, get a sign-in code"
3. Each user signs in on their own time — OTP code flow, one email per signin, never a rate limit problem

**Reference implementation:** `windex` used this exact pattern to onboard 19 users including 6 created in bulk within 3 seconds.



**Lesson learned the hard way (Windex / Honcut Phase 9, late 2025 – mid 2026):** Magic-link flow was tried first across multiple projects and cost weeks of cumulative debugging. The specific failure modes, all of which disappear under OTP:

1. **Expo Router strips URL fragments before the auth code can read them.** Magic-link URLs carry the session tokens in a hash fragment (`#access_token=...`). The router mounts, normalizes the URL, and the tokens are gone before any `useEffect` runs. Race condition timing was browser-dependent (Chrome usually fine, Safari usually broken, Firefox 50/50).
2. **Single-use link consumed by the wrong device.** User clicks the link on their phone but wanted to sign in on their laptop. Link is now spent — no recovery path.
3. **iOS Mail link previews.** Some iOS configurations fetch the URL to generate a preview, consuming the single-use link before the user actually taps it.
4. **Deep-link config that works in `expo start --web` breaks in the Vercel production build.** The Expo-generated deep-link config assumes a native context; web deploys ignore most of it.
5. **Built-in email relay rate limits.** Magic-link flows often pair with `inviteUserByEmail` for onboarding, which hits Supabase's 2/hour cap at user 3 (see section 2.2b's `admin.createUser` guidance).

Switching to OTP code flow eliminates all five — the code is a string the user types, never round-trips through a URL, and isn't bound to a device or single use.

### 2.2b.i Auth implementation gotchas — three bugs that survive the OTP switch

Switching from magic links to OTP eliminates the primary class of auth bugs but does NOT eliminate three subtler ones. Any Supabase + Expo (or Supabase + Next.js) project will hit these unless they're handled deliberately on day one.

**Gotcha 1: Capture URL params at module load, not in a useEffect.**

Even with OTP as the primary flow, there are auth contexts that still round-trip through a URL: password recovery, invite acceptance, OAuth callback. Expo Router (and Next.js App Router under some configs) will strip query params and hash fragments before any React effect runs. The auth code in `useEffect` sees an already-cleaned URL and behaves as if nothing was in it.

Fix: read `window.location.href` at **module load time** in a bootstrap file imported at the very top of the app entry, before the router is touched. Stash the parsed values in a module-level variable. The auth provider reads from that variable synchronously on mount.

```ts
// auth-bootstrap.ts — first import in the app entry
const initialUrl = typeof window !== 'undefined' ? window.location.href : '';
export const capturedAuthParams = parseAuthParamsFromUrl(initialUrl);
```

Then in the auth provider:
```ts
useEffect(() => {
  if (capturedAuthParams.code) { /* handle it */ }
}, []);
```

Without this, password recovery and invite acceptance silently no-op on first load, and the user is left staring at a login screen wondering why their email link "did nothing."

**Gotcha 2: Add a grace period before 401s trigger signout.**

Supabase sessions return transient 401s during token refresh, network blips, or right after a session change. A naive "401 → call `signOut()` → redirect to /login" handler will produce phantom logouts: user works for 30 seconds, leaves the tab idle, comes back, finds themselves logged out for no visible reason.

Fix: when a 401 is observed, mark the session as "uncertain" but do NOT sign out yet. Retry the request after a short delay (e.g. 5s). Only if the 401 persists past a grace window (30s is a reasonable starting value) trigger the actual signout. Most 401s self-resolve within a few seconds — the grace period catches the false positives.

```ts
let firstUnauthorizedAt: number | null = null;
const GRACE_MS = 30_000;

function handleResponse(res: Response) {
  if (res.status === 401) {
    firstUnauthorizedAt ??= Date.now();
    if (Date.now() - firstUnauthorizedAt > GRACE_MS) {
      supabase.auth.signOut();
    }
  } else {
    firstUnauthorizedAt = null;
  }
}
```

**Gotcha 3: Edge Functions called without a JWT need `--no-verify-jwt` at deploy time.**

Supabase defaults all Edge Functions to JWT-verified. If a function is called from a context that doesn't pass a user JWT — a webhook, a cron job, an in-app event before the session is established — the function returns 401 before any of its own code runs. The 401 is indistinguishable from a user-session 401 in client-side logging, making this an extremely painful diagnostic.

Fix: deploy those functions with `--no-verify-jwt`:

```bash
npx supabase functions deploy events --no-verify-jwt
```

Then handle the trust boundary inside the function itself (shared-secret header for webhooks, IP allow-list for crons, whatever's appropriate). Document the choice in `CLAUDE.md` so future-you doesn't reflexively flip it back during a "let's tighten security" pass.

**Reference implementation:** Windex's `events` Edge Function uses `--no-verify-jwt` because it's called from the app at startup before the session is fully hydrated. Without this flag, the app would 401-loop on every cold start.

**The unifying principle:** prefer flows that don't depend on URL handoffs in an Expo + Vercel deployment. Web router behavior, mobile router behavior, and browser URL normalization interact in ways that are hard to reason about and harder to debug. Anything you can do in-app (code entry, manual paste, in-app navigation) is more robust than anything that crosses an `https://...?token=...` boundary. Magic links were a clever idea that depended on the URL being a clean handoff channel. In Expo it isn't. OTP works because it sidesteps the channel entirely.

### 2.2c Email relay — start with built-in, switch to Resend only if needed

Supabase's built-in email relay is rate-limited (~2 emails per hour per
recipient). For most projects this is fine — a 5-person family signing in
once a week each is nowhere near the cap. The limit only fires during
testing bursts (rapid sign-in retries during auth migrations / debugging) or
if a project does bulk user provisioning via `inviteUserByEmail`.

**Default: stick with the built-in relay.** No setup, no env vars, no
ongoing service. Document this choice in the project's `CLAUDE.md` under
"Known operational behaviors" so future-you doesn't waste time debugging
a rate-limit hit during testing.

**When to switch to Resend:**
- Organic (non-testing) usage starts hitting the cap
- You need a custom From: address
- You're adding password-reset volume that won't fit under 2/h

**How to switch (when needed):**
- Sign up at resend.com (free tier: 3,000/mo, 100/day)
- Verify a sending domain (or use `onboarding@resend.dev` for testing only)
- Configure in Supabase dashboard → Authentication → Providers → SMTP Settings
- Then PATCH `rate_limit_email_sent` up via Management API:

```bash
curl -X PATCH "https://api.supabase.com/v1/projects/<ref>/config/auth" \
  -H "Authorization: Bearer ${SUPABASE_PERSONAL_ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"rate_limit_email_sent": 30}'
```

- Document the change in `CLAUDE.md` (per 3.2b — manual dashboard changes
  get committed as documentation)

**Reference projects on built-in relay:** windex (19 users in production)
and honcut both run without custom SMTP.

### 2.2d Pre-launch auth checklist

Before any Supabase + Expo (or + Next.js) project goes live, verify:

- [ ] Email template uses `{{ .Token }}`, not `{{ .ConfirmationURL }}` — OTP code, not magic link.
- [ ] Login UI validates 6–8 digits with a tolerant pattern (`\d{6,8}`), not a hardcoded length.
- [ ] App entry imports a bootstrap module that captures `window.location.href` at module load time, BEFORE the router mounts. Auth provider reads from the captured value, not from live `window.location` inside a `useEffect`.
- [ ] 401 handler wraps signout in a grace period (≥30s) — does NOT signout on the first 401.
- [ ] Every Edge Function called without a user JWT (webhooks, crons, pre-session app calls) is deployed with `--no-verify-jwt` AND has its own trust-boundary check inside the function.
- [ ] Onboarding script uses `admin.createUser` with `email_confirm: true`, NOT `inviteUserByEmail` (rate-limit-free bulk provisioning).
- [ ] `/login` page (or its layout.tsx) sets `dynamic = 'force-dynamic'` and `revalidate = 0` (no-cache for auth pages).
- [ ] If multi-project Vercel: `vercel env ls production` from each project dir confirms `EXPO_PUBLIC_*` (or framework equivalent) vars are present per-project.
- [ ] Live cross-origin testing on a real phone before launch — same-origin dev never catches CORS / Authorization-header / VERCEL_URL gotchas (section 4.1).

If any of these aren't true, the auth flow will work in dev and break in some specific user scenario at launch. Every item on this list comes from a real production failure.


Create `.env.local` in the project root with the following keys and populate them now:

```
SUPABASE_DB_PASSWORD=your-password-here
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
```

**Where to find each value:**
- `SUPABASE_DB_PASSWORD` — set during Supabase project creation; copy it immediately
- `NEXT_PUBLIC_SUPABASE_URL` — Supabase dashboard → Project Settings → API → Project URL
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — Supabase dashboard → Project Settings → API → anon/public key
- `SUPABASE_SERVICE_ROLE_KEY` — Supabase dashboard → Project Settings → API → service_role key
- `ANTHROPIC_API_KEY` — console.anthropic.com → API Keys

**Back this file up to Google Drive immediately:**
`G:\My Drive\BTest Projects\[Project Name]\.env.local`

This file never goes in GitHub. Drive is the only copy. It is the project's lifeline on any new machine.

### 2.4 Environment & Secrets Setup
Have Claude Code do this in one pass:
```
Create .env.local with the following keys (I will provide values):
- SUPABASE_DB_PASSWORD
- NEXT_PUBLIC_SUPABASE_URL
- NEXT_PUBLIC_SUPABASE_ANON_KEY
- SUPABASE_SERVICE_ROLE_KEY
- ANTHROPIC_API_KEY
- [any other keys needed]

Also update .gitignore to cover:
.env.local
.env*.local
.env
```

---

## Phase 3   Development

### 3.1 Install GSD   Get Shit Done
**GSD is a specific open-source tool, not just a motto.** It is a meta-prompting and spec-driven development system built for Claude Code. Install it at the start of every project.

**The problem GSD solves   context rot:**
Without GSD, Claude Code degrades as the session grows. At 50% context it starts cutting corners and forgetting requirements. At 70%+ it hallucinates. GSD fixes this by spawning fresh Claude subagent instances for each task   each gets a clean 200K context window. Task 50 has the same quality as Task 1.

**Install globally (works across all projects):**
```bash
npx get-shit-done-cc --claude --global
```

**Install locally for a single project:**
```bash
npx get-shit-done-cc --claude --local
```

**Core GSD slash commands inside Claude Code:**
```
/gsd-new-project     Start a new project (interviews you, builds spec, creates plan)
/gsd-discuss-phase   Lock in product decisions before planning begins
/gsd-plan-phase      Generate atomic task plans from the spec
/gsd-execute-phase   Execute a plan in a fresh subagent context
/gsd-quick           Fast path for small tasks, skips heavy planning
/gsd-fast            Trivial inline tasks — no subagents, no planning files
/gsd-progress        Check project status and resume from where you left off
/gsd-resume-work     Resume work from previous session with full context restoration
/gsd-pause-work      Create context handoff when pausing mid-phase
/gsd-debug           Systematic debugging with persistent state across context resets
/gsd-help            Show full command reference
/gsd-update          Update GSD to latest version
```

**Resuming after a break:**
```
/gsd-progress
```
This shows where you left off and offers next actions. Use this at the start of every session instead of re-explaining context.

**When to use GSD vs plain Claude Code:**
- Use GSD for any project spanning multiple sessions, files, or phases
- Use `/gsd:quick` for simple one-off tasks under 30 minutes
- GSD overhead is not worth it for trivial single-file changes

**Keep GSD updated**   it releases frequently:
```bash
npx get-shit-done-cc --claude --global   # re-run to update
```

**Used on Honcut Hunt Log Project:** GSD managed context across multi-session builds throughout the 2025-26 development cycle.

---

### 3.2 Claude Code as Primary Dev Tool
- **Claude Code + GSD**   scaffolding, components, API routes, scripts, debugging
- **Claude.ai Project**   architecture decisions, analysis, data modeling, documentation
- **Perplexity**   live web data, current API docs, anything requiring real-time web access

**Reduce confirmation prompts — add this to the top of every Claude Code prompt:**
```
You have full permission to read, write, create, and execute files and commands for this project without asking for confirmation on each step. Complete the full task autonomously and report when done.

If at any point you need a decision or permission that you cannot proceed without, stop and run this alert before waiting:
powershell -c "[console]::beep(500,1000)"

When the full task is complete, run:
powershell -c "[console]::beep(1000,300); [console]::beep(1000,300); [console]::beep(1000,300)"
```
- Single long low beep = needs your input mid-task, come check
- Three short high beeps = task complete

**Claude Code operational gotchas:**

**Auto-approving git operations:**
Claude Code prompts "This command changes directory before running git, which can execute untrusted hooks. Do you want to proceed?" on every git command during long autonomous runs. To skip these during a session, start Claude Code with:
```powershell
claude --dangerously-skip-permissions
```
Or tell Claude Code at the start of the session:
```
I trust all git operations in this repo. Auto-approve all git commands for this session.
```

**Resuming a mid-flight crash:**
If Claude Code is interrupted mid-plan (ctrl+c, network drop, context limit), do NOT run `/clear`. The executor knows what was committed vs uncommitted. Tell it:
```
Resume from where you stopped. Finish the current task, commit, then continue through the remaining plans.
```
Clearing the context forces it to rediscover state and risks double-applying completed work.

### 3.2a Working-tree hygiene — end every session clean

**The rule:** at the end of every Claude Code session, `git status` shows no modified files and no untracked files (except those covered by `.gitignore`). No exceptions.

**Clean = working tree clean AND `origin/<branch>` matches HEAD.** Committed-locally is not done. The session is not over until the commit has been pushed and the corresponding deploy (Vercel auto-deploy from `main`, or whatever the project uses) has been verified Ready. A commit that sits unpushed for several minutes meets the literal "working tree clean" rule but violates the spirit — the work isn't backed up to GitHub, isn't deployed via auto-deploy, and is invisible to anyone else (Family Spend May 2026: a documentation commit sat unpushed for several minutes in this state).

**Why this matters (lessons learned the hard way, Windex May 2026):**

A "working in progress" pile is invisible to everyone except the machine it lives on. Windex accumulated, across several weeks:

- Two SQL migrations applied to live Supabase but never committed — irreplaceable if the laptop died
- An entire frontend feature (W/L columns, HistoryChart, player detail route, two npm deps) visible to the developer via Expo Go but not deployed to web — other users could not see it
- Two real production bugs (double-header on a route, silent score-save failures) whose fixes existed locally but never shipped
- A repointed icon/favicon and brand assets sitting unshipped

The pattern in every case was the same: **the work got built, the build worked locally, the developer moved on without committing.** Each individual day this seemed fine. Cumulatively it produced a working tree with 12+ dirty files and three different in-progress features tangled together, requiring a multi-step "what is all this stuff" triage before anything new could safely ship.

**Checklist at the end of every Claude Code session:**

- [ ] `git status` is clean, OR
- [ ] every dirty file has been categorized: shipped, reverted, stashed to a named branch, or explicitly deferred with a reason
- [ ] if any backend (Supabase) change was deployed in the session, the source SQL / function code is committed to git in the same session
- [ ] if any package.json dep was added in the session, package.json AND package-lock.json are committed in the same session
- [ ] no "I'll commit this later" — later is when it gets lost or tangled with the next feature
- [ ] `git push` has been run; `origin/<branch>` matches local HEAD
- [ ] Vercel deploy (if applicable) has reported Ready

**The "deploy without commit" anti-pattern, specifically:**

When you deploy a Supabase migration with `npx supabase db query --linked -f migration.sql`, that file is now *running* on prod but not in git. Commit it before doing anything else. Same for `npx supabase functions deploy <name>` — commit the source immediately. The deploy command and the git commit are a pair; treat them as one atomic operation.

**The "Expo Go is not a deployment" anti-pattern:**

Seeing a feature work in Expo Go on your phone proves the feature works on **your** device, pointed at **your** dev bundle. It does not mean other users can see it. The only thing that puts a frontend feature in front of users on `app.lateaddgolf.com` (or any Vercel-hosted PWA) is a commit to the deployed branch. If you spent more than 30 minutes building a feature and you haven't pushed it yet, ask yourself why — and if the answer isn't "it's broken," push it.

**Branching for in-progress work that legitimately can't ship yet:**

If you're mid-build on a feature that genuinely can't ship today (it's broken, it depends on a backend change you haven't made yet, it needs design review), create a feature branch:

```bash
git checkout -b feature/<short-name>
git add <files>
git commit -m "wip: <description>"
git push -u origin feature/<short-name>
```

This gets the work off your machine and into GitHub (where it's backed up and visible) without polluting `master` or the working tree. Switch back to `master` when you're done for the session: `git checkout master`. Your working tree is clean, the in-progress work is safe on its branch, and `git status` no longer hides anything.

**Why this matters (operational view):** GSD's state tracking (`STATE.md`, `HANDOFF.json`) records what was committed. Uncommitted work is invisible to the next session and gets lost or double-done.

**Claude Code enforcement prompt:**
```
Before ending this session, verify the working tree is clean:
- Run git status
- Commit any modified or untracked source files
- If any files should be gitignored, add them to .gitignore and commit that too
- Run git push and verify origin/<branch> matches HEAD
- If applicable, verify Vercel deploy reports Ready
- Report the final git status and last 5 commits
```

### 3.2b Backend deploys are git operations

Any time you run one of these commands, a corresponding `git commit` for the source must follow within the same session:

| Deploy command | Source files to commit |
|---|---|
| `npx supabase db query --linked -f <migration>.sql` | The migration file itself |
| `npx supabase functions deploy <name>` | The function's `index.ts` and any imported helpers |
| `npx vercel env add <KEY>` | Update `.env.local.example` with the key (no value) |
| `npx vercel deploy` (manual, outside auto-deploy) | Whatever was changed before the deploy |

If git and Supabase ever disagree about what's running, **Supabase is the truth and git is the bug.** Get them back in sync immediately, before doing anything else.

**Why this matters (operational view):** the live database can get ahead of the repo. Six months later you clone the repo, run migrations, and the schema doesn't match prod — because someone ran a fix directly in the dashboard without committing a migration. This happened in Honcut (`hunt_logs` RLS policy applied manually, not captured in a migration until after the fact).

**Claude Code enforcement prompt:**
```
After any supabase db push, functions deploy, or vercel env change:
1. Confirm the source file (migration SQL, function code, .env.example) is committed
2. Run git log --oneline -3 to confirm the commit landed
3. Run git push and verify origin/<branch> matches HEAD
4. Report the commit hash and final git status
```

---

### 3.3 UI / Visual Design

- Have Claude Code design the site with a clear visual brief
- Keep images in `/public` or `/assets`
- Specify the aesthetic upfront (example: *"hunting-themed dark green, Georgia serif font"*)

### 3.4 Data & Scripts
- All data files in `/data`
- All utility scripts in `/scripts`
- Never hardcode local paths   always use paths relative to project root

---

## Phase 4   Deployment

### 4.1 Push to GitHub → Vercel
```
Tell Claude Code:
"Push current state to GitHub main branch. Then set up 
Vercel deployment   the app should auto-deploy on push to main."
```
- Vercel detects Next.js automatically
- Auto-deploy on every push to `main` after initial setup

**Setting Vercel environment variables via Claude Code:**
Claude Code can set Vercel environment variables directly using the Vercel CLI — you do not need to do this manually in the dashboard. Tell Claude Code:
```
Set the following environment variables on the Vercel project using the Vercel CLI:
- NEXT_PUBLIC_SUPABASE_URL
- NEXT_PUBLIC_SUPABASE_ANON_KEY
- SUPABASE_SERVICE_ROLE_KEY
- ANTHROPIC_API_KEY
Values are in .env.local. Apply to production environment and redeploy.
```

**Note:** Claude.ai (this chat) may incorrectly say Claude Code cannot do this — it can. If Claude Code initially balks, instruct it to proceed anyway using the Vercel CLI (`vercel env add`). Override both the chat and any initial hesitation from Claude Code.

**Multi-project repos (PWA / mobile-export pattern) — env vars are PER PROJECT, not per repo:**

When a single repo deploys to MORE than one Vercel project — common pattern: a Next.js web app at `myapp.com` AND a PWA built from a sibling subdirectory like `apps/mobile/` at `app.myapp.com` — Vercel scopes env vars to the project, not to the repo. Setting `EXPO_PUBLIC_*` on the web project does NOT propagate them to the PWA project. The CLI must be linked to (or run inside) the directory of the project you're configuring.

```bash
# Scope matters — run from inside the PWA's project directory:
cd apps/mobile
npx vercel env ls production   # confirms which project you're talking to
npx vercel env add EXPO_PUBLIC_SUPABASE_URL production
# (paste/pipe value from .env.local — same value as the web project)
```

**Symptoms of forgetting this:** the PWA deploy is "READY" in Vercel, the URL serves a 200 OK index.html, but the page renders blank or stuck on a loading spinner. The bundler (Metro / Webpack / etc.) saw `process.env.EXPO_PUBLIC_*` as undefined at build time, dead-code-eliminated the real branch, and kept only an error-throw fallback. First call to the env-dependent code (e.g., `createClient`) throws and React unmounts.

**Diagnosis (no device required):** fetch the bundle file referenced from `index.html` and grep for the value that should have been inlined (e.g., `https://<ref>.supabase.co`). Empty result = env vars not set at build time.

**Fix:** add the missing vars on the right Vercel project, then `npx vercel deploy --prod --yes` — adding env vars does NOT auto-rebuild existing deploys.

**Lesson learned the hard way (Honcut Phase 9 PWA launch):** the PWA at `app.honcutcreekranch.com` deployed cleanly but rendered blank on iOS Safari for 30+ minutes after first deploy. Root cause was zero env vars on the `honcut-mobile` Vercel project, while the sibling `honcut` Next.js project had its `NEXT_PUBLIC_*` vars correctly set. Always verify each Vercel project independently with `vercel env ls production` after adding a second project.

**Three more PWA-UAT-only bugs (Honcut, surfaced 2026-05-02 during real phone use):**

The PWA + Next.js-API split has at least four ways to fail that LOOK FINE in same-origin testing (web app calling its own /api/*) and only break when the PWA at `app.{domain}` calls APIs on `{domain}`. Account for these BEFORE shipping the PWA — none are catchable without live cross-origin testing.

1. **Env vars per Vercel project** — covered above.

2. **CORS headers on /api/\*.** Same-origin requests (web → /api/) need no CORS. PWA cross-origin requests (app.{domain} → {domain}/api) need `Access-Control-Allow-Origin: https://app.{domain}` on every response. Without it: browser blocks every fetch silently, mobile fetch wrapper sees a network error, UI shows "Load failed" (POST) or returns null (GET). Add via `next.config.ts` `headers()` config; restrict to known origins, NEVER `*` on a JWT-authenticated API.

3. **JWT auth on routes the mobile client forgot to authenticate.** During Phase 9 cutover, every Route Handler adopts `verifyAndGetUid` for cookie OR Bearer JWT. Easy to miss: the mobile-side fetch wrapper for *some* endpoints (e.g., a weather/auto-fill helper module) keeps doing plain `fetch(url)` without the Authorization header. Server returns 401, fetch sees `!res.ok`, helper returns null, UI shows the no-data fallback. Audit every cross-origin fetch in the mobile codebase and ensure each one calls the shared `getAuthHeaders()` helper.

4. **Server-to-server fetches that use `VERCEL_URL`.** If a Route Handler internally fetches another Route Handler (e.g., `/api/recommend` composing `/api/forecast`), the natural pattern is `fetch(\`https://${process.env.VERCEL_URL}/api/forecast\`)`. **DO NOT do this.** `VERCEL_URL` returns the deployment-specific `*.vercel.app` hostname, which is gated by Vercel Authentication (SSO challenge). The internal fetch returns HTML 401, your `if (res.ok)` falls through, and you get the no-data fallback path. Use `process.env.NEXT_PUBLIC_SITE_URL` (the canonical custom domain) instead — it's publicly reachable and not SSO-gated. Falls back to `localhost:3000` in dev. Set `NEXT_PUBLIC_SITE_URL` on the Vercel project as part of the env-var initial setup.

**Pre-PWA-launch checklist:**
- [ ] `vercel env ls production` from inside the PWA's project dir → all `EXPO_PUBLIC_*` (or framework-equivalent) vars present
- [ ] `next.config.ts` emits `Access-Control-Allow-Origin: https://app.{domain}` on `/api/*`
- [ ] `grep -rn "fetch(" apps/mobile/lib/` audited for missing `Authorization` headers (compare each call site against the server route's auth gating)
- [ ] `grep -rn "VERCEL_URL" app lib` returns zero hits (or is replaced by `NEXT_PUBLIC_SITE_URL`)
- [ ] Real phone UAT: sign in, log a hunt, auto-fill weather, run the recommender. Catch the bugs before partners do.

**Pre-route-rename checklist (whenever you rename a tab or route file in Expo Router / Next.js App Router):**
- [ ] Renamed the file
- [ ] Updated tab-bar registration (`Tabs.Screen name="..."` in `_layout.tsx`)
- [ ] `grep -rn "/<old-route-name>" apps/mobile/ app/` (or just root `app/` for Next.js) — every hit needs to be updated to the new path. The grep is the LAST step before commit. Common offenders:
  - [ ] `router.replace('/<old>')` and `router.push('/<old>')` calls
  - [ ] `<Redirect href="/<old>" />` JSX
  - [ ] `<Link href="/<old>" />` JSX
  - [ ] Auth-gate / root index files with hardcoded post-signin redirect targets
- [ ] For PWAs only: add **compat redirect stubs** at the old URL paths (e.g., `app/<old-route>.tsx` exporting `<Redirect href="/<new-route>" />`) and keep them for at least a few weeks. iOS Safari standalone PWAs cache the last-visited URL; without stubs, installed PWAs hit `+not-found.tsx` on reopen. Delete stubs once enough time has passed for all installed PWAs to re-cache.

This bit twice in one session (Honcut Harvest→Hunt rename, 2026-05-02): first the root auth gate's redirect target, then the sign-in screen's two `router.replace` sites — both pointed at the now-404'd old route. Phone-side symptom each time was the same: force-close + reopen the PWA → "The screen doesn't exist. Go to home screen." Each new miss required another deploy cycle. Run the grep BEFORE pushing the rename commit.

**Auth pages should be no-cache.** Add to the project's `/login` page (or a sibling `layout.tsx` if the page itself is `"use client"` — Next 16 rejects these exports on client components):

```typescript
export const dynamic = 'force-dynamic';
export const revalidate = 0;
```

Or equivalent cache-control headers on the route. Goal: auth UX changes (validator updates, copy fixes, layout tweaks) are immediately visible to users without forcing a hard-refresh. `/review`, `/import`, `/dashboard` etc. benefit from normal Next.js caching — only auth-flow pages should be no-cache.

**Lesson learned the hard way (Family Spend, May 2026):** A login UI fix shipped to prod but users had a cached old bundle. The cached version rejected codes the new version accepted. Hard-refresh fixed it per user, but the wider lesson is auth pages should never be cached.

### 4.1a Path-filter Vercel deploys in multi-folder repos

If a single repo deploys only one Vercel project (e.g. `windex`'s `windex-expo` folder is the only thing Vercel watches), Vercel still rebuilds on every push to the deployed branch — including pushes that touch only `windex-api/`, `windex-admin/`, or root-level docs.

**Fix:** add an `ignoreCommand` to the project's `vercel.json`:

```json
{
  "buildCommand": "npx expo export --platform web",
  "outputDirectory": "dist",
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "ignoreCommand": "git diff --quiet HEAD^ HEAD -- ."
}
```

`ignoreCommand` runs from the project's root directory (the folder Vercel is watching). `git diff --quiet HEAD^ HEAD -- .` exits 0 (clean) if nothing in this directory changed since the last commit — Vercel skips the build. Exits non-zero (dirty) if anything changed — Vercel builds normally.

**Known limitation: this form only inspects the tip commit of a push.** Rebase-merges where the meaningful change isn't the tip will silently skip the build. Workarounds: prefer squash-merge for mixed-scope PRs, or run `vercel --prod --force` from the affected project folder after the merge. Attempts to fix this via `VERCEL_GIT_PREVIOUS_SHA` failed in production three times — see Windex BACKLOG for the investigation trail before re-attempting. Short version: Vercel uses shallow clones AND its build-time git token is more restricted than the initial-clone token (no arbitrary-SHA fetch, and `--unshallow` also failed for unclear reasons because `2>/dev/null` was hiding the error). Re-attempt needs out-of-prod test infrastructure (staging Vercel project) so iteration doesn't break real builds.

Set this up once per Vercel project. **Lesson learned the hard way (Windex, May 2026):** without this, a backend-only hygiene commit to `windex-api/` triggered a full Expo bundle rebuild and a redeploy of the web app — wasted build minutes and a small risk that an unrelated bundler change could break prod via a commit that didn't even touch the web app.

**Second lesson, also the hard way (Windex, May 2026):** the original form `git diff --quiet HEAD^ HEAD -- .` silently skipped the windex-admin deploy for PR #17 because the merge landed four commits in one push and the meaningful windex-admin commit was buried mid-push (HEAD was a root-level docs commit, HEAD^ was another root-level docs commit, neither touched `windex-admin/`). Squash-merges were immune (single tip commit); rebase-merges were the vulnerable path. Switched both `windex-admin/vercel.json` and `windex-expo/vercel.json` to the `VERCEL_GIT_PREVIOUS_SHA` form above. Recovery for the missed deploy: `vercel --prod --force` from CLI.

**Third lesson, learned 24 hours later (Windex, May 2026):** the first attempt at the `VERCEL_GIT_PREVIOUS_SHA` fix shipped without any fetch prefix and broke windex-admin builds immediately — `fatal: bad object <sha>` because the previous-deploy SHA wasn't in Vercel's shallow clone. windex-expo got lucky (its previous SHA happened to be inside the shallow window); windex-admin didn't. Production stayed up (failed builds don't swap the alias) but every subsequent push errored at the ignoreCommand step until a fetch prefix landed.

**Fourth lesson, same day (Windex, May 2026):** the next attempt prefixed the diff with `git fetch origin "$SHA" --depth=1 --quiet 2>/dev/null || true` — fetch the specific previous-deploy SHA. Worked locally against a depth-10 clone of the same public repo, but failed on Vercel with the same `fatal: bad object <sha>`. The fetch silently errored (`2>/dev/null` swallowed it). Working theory: **Vercel's git clone token does not permit arbitrary-SHA fetches** (no `allowReachableSHA1InWant` style permission); only standard ref fetches work.

**Fifth lesson, same day (Windex, May 2026):** the next attempt swapped the SHA-targeted fetch for `git fetch --unshallow --quiet 2>/dev/null || true`, on the theory that ref-based fetch (which the initial clone proved Vercel's token can do) would succeed where SHA-targeted fetch failed. It didn't. windex-admin errored with the same `fatal: bad object <sha>` and a total ignoreCommand runtime of ~25ms — way too fast for a real network unshallow. Either `--unshallow` errored instantly or no-op'd silently; the `2>/dev/null` was still in place hiding the actual reason. **Reverted both `vercel.json` files back to the original `git diff --quiet HEAD^ HEAD -- .` form** (the version documented at the top of this section). The multi-commit-skip bug remains as a known limitation per the note above. Re-attempt requires out-of-prod test infrastructure (staging Vercel project) so iteration doesn't keep breaking real builds, and the next diagnostic should remove `2>/dev/null` to read Vercel's actual stderr.

### 4.1b — Vercel Pro defaults that break new project setup

When importing a new GitHub repo to Vercel under a Pro team, three
defaults will make the first deploy fail or appear broken:

**1. Vercel Authentication = ON ("Standard Protection")**
- Effect: production URL returns plain 404 to any visitor not logged
  into Vercel and a member of the team. Even the dashboard "Visit"
  button leads to 404.
- Build logs show successful deploy with a real route table — this is
  not a build problem.
- Fix: Settings → Deployment Protection → toggle "Require Log In" OFF
  → **click Save**. The toggle is pending until Save is clicked; the
  UI does not warn you about this.
- For projects with their own app-level auth (Supabase OTP, etc.),
  Vercel Authentication is redundant — keep it off.

**2. Node.js version defaults to the newest LTS (currently 24.x)**
- Effect: builds succeed but can silently produce broken runtime
  artifacts on Next.js 15+ in some configurations. Hard to diagnose
  because logs claim success.
- Fix: Settings → Build and Deployment → Node.js Version → pin to
  20.x (or 22.x). Trigger a redeploy to apply.

**3. Empty-repo first import can corrupt the project's internal slug**
- Effect: per-deployment URLs come out with the wrong project name
  baked in (e.g. `irrigation-monitor-m8b4ogvi5-buzzstryker.vercel.app`
  instead of `irrigation-monitor-app-m8b4ogvi5-buzzstryker.vercel.app`).
  Production URL 404s permanently. Cannot be fixed via settings;
  the corruption is deep in Vercel's project record.
- Symptom to watch for: in Deployment Details, the third domain
  alias in the list (the per-deployment hash URL) has the wrong
  project name.
- Fix: delete the Vercel project and re-import AFTER code has been
  pushed to the GitHub repo. The empty-repo import path is what
  triggers the bug; importing a repo with code present produces a
  clean project record.

**Preferred sequence for new projects:**
1. Create GitHub repo (empty)
2. Clone locally, scaffold the project, push to main
3. ONLY THEN import to Vercel from a non-empty repo
4. Immediately after first deploy: turn off Vercel Authentication,
   pin Node.js to 20.x, trigger redeploy

This avoids all three issues above. The opposite sequence (create
Vercel project against empty repo, then push code) is the path that
caused 2+ hours of debugging in the irrigation-monitor-app session.

### 4.1c Auto-updating PWA — service worker + build-SHA stamp

**Standing procedure: every Buzz project that ships as an installed PWA includes, from project setup, (1) a build-SHA stamp visible in-app and (2) an auto-updating service worker** (network-first app shell, registered as `/sw.js?v=<BUILD_ID>`, with a documented kill switch). Not optional, not a later add-on — wire it in at setup.

**Why (the failure mode this prevents):** an installed PWA caches its JS bundle. Without a self-updating service worker, the OS WebView serves that stale bundle **indefinitely** — every deploy becomes a manual delete/re-add for every installed user, and client-side bugs become undebuggable because you can't tell new code from a cached bundle (Windex, 2026-06, burned an afternoon on a "layout bug" that was really a stale bundle masking the fix). The build-SHA stamp makes "which bundle is this device running?" a two-second glance instead of a guess.

**The non-negotiable: FRESH CODE FIRST, offline as a bonus.** A cache-first service worker on the HTML or bundle recreates the exact staleness you're fixing. Network-first on the shell + cache-by-hashed-URL for assets is the whole pattern.

**Implementation: see `PWA_Auto_Update_Pattern.md`** — it has the canonical `sw.js` + registration code, the invariant core vs. the three per-project adapters (static-asset glob, build-ID env var, "safe to reload" hook), the kill-switch deploy procedure, and the verification ritual (ship → one final manual re-add to plant the SW → push a trivial change → confirm the in-app SHA flips with no re-add).

**Bootstrapping caveat:** the first SW can't update an app that has no SW yet, so shipping this needs **one** final manual delete/re-add. Every deploy after that is automatic.

**Related caveat (same doc):** iOS auto-zooms on focusing an input with `font-size < 16px`, shrinking the layout viewport and shoving left-anchored UI off-screen — it masquerades as a layout bug. Fix: every focusable input is `font-size >= 16px`.

**Reference implementation:** Windex (`windex-expo`) — `public/sw.js`, `lib/pwaUpdate.ts`, web-guarded registration in `app/_layout.tsx`, `EXPO_PUBLIC_BUILD_ID` via `VERCEL_GIT_COMMIT_SHA` in `vercel.json`, SHA stamped in the drawer footer.

### 4.2 Connect GoDaddy Domain to Vercel
- Done manually in GoDaddy DNS settings
- Add Vercel's A record and CNAME to GoDaddy
- Vercel handles SSL automatically once DNS propagates

### 4.3 Mobile / App Store (if applicable)
- Local testing: Expo Go
- Distribution: Apple Developer Account → TestFlight → App Store
- Android: Google Play Console

---

## Phase 5   Closeout & Handoff

### 5.1 Pre-Handoff Audit (Claude Code)
```
Perform a pre-handoff audit:
1. Scan all source files for hardcoded secrets, API keys, or tokens
2. Find all console.log/console.error statements and list them
3. Check .gitignore covers .env, node_modules, and output files
4. List any TODO/FIXME comments left in the code
5. Check package.json for unused dependencies
6. Report duplicate or temp files that should be cleaned up
7. Find any hardcoded local paths in scripts and replace with 
   relative paths from project root
```

### 5.2 Drive Backup   Secrets & Large Files
**Account:** buzzstrykertest@gmail.com
**Drive path:** `G:\My Drive\BTest Projects\[Project Name]\`

| File | Why |
|------|-----|
| `.env.local` | Supabase keys + passwords   CRITICAL |
| `.env.local.example` | Populated backup reference |
| Any `*.html` reference files >1MB | Gitignored by default |
| Source Excel/CSV files not in git | Raw data inputs for scripts |
| Any untracked docs (`.docx`, etc.) | Testing notes, reference docs |

**Do NOT back up:**
- `node_modules/` → `npm install`
- `.next/` → `npm run build`
- `.vercel/` → `vercel link`
- Any folder regeneratable by a script

**Claude Code copy prompt:**
```
Copy the following files to "G:\My Drive\BTest Projects\[Project Name]":
[list files with exact paths]
Copy, do not move. Confirm each file succeeded.
Then update SETUP.md to reflect the exact Drive path.
```

### 5.3 Drive Sync Rule
Drive is the **source of truth for secrets.**
- If `.env.local` changes locally → copy updated file back to Drive manually
- Do not rely on folder sync (node_modules bloat problem)
- On a new machine: pull from Drive, never push to Drive automatically

### 5.4 Generate SETUP.md Before Closing
```
Generate a SETUP.md that documents:
1. Every file needed from Google Drive and where it goes locally
2. Every command needed to get the app running from a fresh clone
3. Exact Drive path: G:\My Drive\BTest Projects\[Project Name]\
4. "Keeping Drive in Sync" rule for .env.local updates
```

### 5.5 Final Commits
```
git add SETUP.md .gitignore [any fixed scripts]
git commit -m "docs: finalize SETUP.md, fix paths, update gitignore"
git push
```

---

## Phase 6   New Machine Setup

### The Two-System Rule
- **GitHub** → all code, committed data, `SETUP.md`
- **Google Drive** → secrets, large reference files, raw source data not in git

### New Machine Workflow
1. `git clone [repo]`
2. Copy `.env.local` from Drive into project root
3. Copy any large reference files needed for current task
4. `npm install`
5. `npx get-shit-done-cc --claude --global` (reinstall GSD)
6. Work

---

## Key Files Every Project Should Have

| File | Purpose |
|------|---------|
| `Project_Context.md` | What it is, data model, key decisions (use template) |
| `Project_README.md` | Stack, how to run, overview |
| `SETUP.md` | New machine setup, Drive file list |
| `ai_team.md` | Which AI tools for which tasks |
| `.env.example` | Empty template   committed to git |
| `.env.local` | Real secrets   never committed, lives in Drive |
| `.gitignore` | Covers env, node_modules, build output |

**Template location:** `C:\Users\buzzs\OneDrive\Desktop\Projects\Project_Context_TEMPLATE.md`

---

## Tool Roster

| Tool | Role |
|------|------|
| Claude.ai Project | Planning, architecture, analysis, documentation |
| Claude Code | Primary development execution |
| GSD (`npx get-shit-done-cc`) | Context engineering   prevents context rot across long builds |
| Perplexity | Live web data, real-time research |
| GitHub | Source code version control |
| Supabase | Database + auth backend |
| Vercel | Hosting + auto-deploy from GitHub |
| GoDaddy | Domain management |
| Google Drive (buzzstrykertest@gmail) | Secrets + large file storage |
| Expo Go | Mobile app local testing |
| Apple Developer Account | App Store + TestFlight |

---

## Honcut Hunt Log Project   Reference Implementation
*Completed March/April 2026   the project that established this procedure*

- **Stack:** Next.js, Supabase, React, TypeScript, Python scripts, Node.js (PptxGenJS)
- **Claude Project:** "Honcut duck hunting analysis"   11 seasons of data, Pacific Flyway migration forecasting, PowerPoint deck generation
- **GSD:** Used throughout to manage context across multi-session builds
- **Drive folder:** `G:\My Drive\BTest Projects\Honcut Hunt Log Project\`
- **Note:** PowerPoint generation (`honcut_v2.js`) lives in the Claude project, not the GitHub repo   two separate workflows
- **Wind data:** Fetched via Perplexity (IEM/KBAB)   Claude sandbox has no outbound network access

## Windex (formerly late-add-v2) — Auth & Multi-User Reference Implementation
*Phase 9 launched May 2026*

- **Stack:** Next.js + Expo PWA + Supabase + Vercel
- **Why it's the auth reference:** OTP code flow with `admin.createUser` bulk provisioning, full RLS overhaul (`015_rls_overhaul.sql`), 19 users onboarded with no rate-limit issues
- **Local path:** `C:\Users\buzzs\Desktop\Projects\windex\`
- **Reference for:** sections 2.2b (OTP), 2.2b.i (auth gotchas), 2.2c (built-in relay posture), 2.2d (pre-launch checklist), full RLS policy patterns
- **Auth gotchas embodied:** module-load URL param capture in `auth-bootstrap.ts`, 30-second 401 grace period in the API client, `events` Edge Function deployed with `--no-verify-jwt`

## Family Spend — Single-Owner-to-Multi-User Migration Reference
*Auth migration completed May 2026*

- **Stack:** Next.js + Supabase + Vercel
- **Notable:** documented the "RLS enabled without policies = lockout" failure mode, the OTP-length-drift failure mode, and the `/login` no-cache pattern. `CLAUDE.md` captures the operational lessons.
- **Local path:** `C:\Users\buzzs\OneDrive\Desktop\Family Spend`

---

*Last updated: June 1, 2026*
*To update: reopen this Claude project and ask Claude to revise*
