# Project Context — [Project Name]

*Template — copy this file into a new project's local directory and rename to `Project_Context.md`, then fill in each section.*
*Master template lives at: `C:\Users\buzzs\OneDrive\Desktop\Projects\Project_Context_TEMPLATE.md`*

---

## What this project is

[One paragraph: what does it do, who is it for, why does it exist.]

## Stack

- **Frontend:** [e.g. Next.js, Expo, React]
- **Backend:** [e.g. Supabase, custom Node API]
- **Hosting:** [e.g. Vercel, custom]
- **Auth:** [e.g. Supabase OTP, none]
- **Other:** [e.g. third-party APIs, integrations]

**Next.js scaffold version pinning**

Do NOT use `npx create-next-app@latest` for new projects.

`@latest` pulls bleeding-edge Next.js (currently 16.x) and
bleeding-edge React (19.x), which may not be compatible with our
other dependencies (e.g. next-pwa, which has not been updated for
Next 15+, let alone 16).

Use an explicit version:

    npx create-next-app@15 ...

The currently-blessed Next.js major for new projects is **15.x**.
Update this number deliberately when a new major has been validated
end-to-end with our stack (PWA, Supabase, Vercel runtime).

## Repo structure

- **GitHub:** [repo URL]
- **Local path:** [e.g. `C:\Users\buzzs\OneDrive\Desktop\<project-name>`]
- **Default branch:** [`main` or `master` — confirm which]
- **Top-level folders:** [list and one-line description of each, if multi-folder]

## Data model

[Describe the core entities and their relationships. Skip if no persistent data.]

## Key decisions

[Bullet list of architectural choices that future-you will forget. Examples: why Supabase over Firebase, why a single repo with subfolders, why OTP auth, etc.]

## Live URLs (if deployed)

- **Production:** [URL]
- **Supabase project ID:** [if applicable]
- **Vercel project name:** [if applicable]

---

## Initial setup checklist

- [ ] **Push initial code to GitHub BEFORE creating Vercel project.**
  Importing an empty repo into Vercel can corrupt the project's
  internal slug. See Procedure §4.1b.
- [ ] **Disable Vercel Authentication** on the new project:
  Settings → Deployment Protection → toggle "Require Log In" OFF →
  click Save. Vercel defaults this ON for Pro teams. See §4.1b.
- [ ] **Pin Node.js version to 20.x** in Settings → Build and
  Deployment. Vercel defaults to the newest LTS (currently 24.x),
  which has known Next.js 15+ runtime compatibility issues. See §4.1b.
- [ ] **PWA auto-update (if shipping as an installed PWA):** build-SHA
  stamp visible in-app + auto-updating service worker installed. See
  §4.1c and `PWA_Auto_Update_Pattern.md`. Without it, installed PWAs
  serve stale cached bundles forever and every deploy needs a manual
  re-add.

---

## Working agreement with Claude

Procedure doc + template live at https://github.com/buzzstryker/buzz-project-docs (cloned locally at `C:\Users\buzzs\repos\buzz-project-docs\`). Pull before any session that touches auth, deploys, Edge Functions, or other procedure-governed areas.

**All Claude Code prompts drafted in this project must enforce the following sections of `Buzz_Project_Development_Procedure.md`:**

- **3.2a — Working-tree hygiene.** Every Claude Code session ends with a clean `git status`. No "I'll commit this later." Backend deploys (Supabase migrations, Edge Functions) and the corresponding git commits are paired operations.
- **3.2b — Backend deploys are git operations.** Any `npx supabase db query`, `npx supabase functions deploy`, or `npx vercel env add` must be followed by a git commit of the source in the same session.
- **4.1a — Path-filter Vercel deploys.** If this project deploys to Vercel from a subfolder of a multi-folder repo, the Vercel project's `vercel.json` must include `"ignoreCommand": "git diff --quiet HEAD^ HEAD -- ."` so commits outside the deployed folder don't trigger rebuilds.

When drafting prompts for Claude Code, include verifications for these rules where relevant — e.g. "verify clean working tree before starting," "commit any backend deploy source in the same session," etc.

---

## Recent changes / project log

*Append major changes here as the project evolves so future sessions have context. Format: date, one-line summary, commit hash if applicable.*

- [YYYY-MM-DD] Initial project setup
