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

## Working agreement with Claude

**All Claude Code prompts drafted in this project must enforce the following sections of `Buzz_Project_Development_Procedure.md` (kept at `C:\Users\buzzs\OneDrive\Desktop\Projects\Buzz_Project_Development_Procedure.md`):**

- **3.2a — Working-tree hygiene.** Every Claude Code session ends with a clean `git status`. No "I'll commit this later." Backend deploys (Supabase migrations, Edge Functions) and the corresponding git commits are paired operations.
- **3.2b — Backend deploys are git operations.** Any `npx supabase db query`, `npx supabase functions deploy`, or `npx vercel env add` must be followed by a git commit of the source in the same session.
- **4.1a — Path-filter Vercel deploys.** If this project deploys to Vercel from a subfolder of a multi-folder repo, the Vercel project's `vercel.json` must include `"ignoreCommand": "git diff --quiet HEAD^ HEAD -- ."` so commits outside the deployed folder don't trigger rebuilds.

When drafting prompts for Claude Code, include verifications for these rules where relevant — e.g. "verify clean working tree before starting," "commit any backend deploy source in the same session," etc.

---

## Recent changes / project log

*Append major changes here as the project evolves so future sessions have context. Format: date, one-line summary, commit hash if applicable.*

- [YYYY-MM-DD] Initial project setup
