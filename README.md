# Buzz Project Docs

Standing cross-project documentation. Lives in GitHub so it syncs across
machines without OneDrive.

## Contents

- **`Buzz_Project_Development_Procedure.md`** — Standard operating procedure
  for all new projects. Auth, RLS, Vercel config, working-tree hygiene,
  deployment patterns, lessons learned. Update whenever a new lesson is
  worth capturing for future projects.
- **`Project_Context_TEMPLATE.md`** — Starting template for a new project's
  `Project_Context.md`. Copy into a new project's local directory and rename.

## Workflow

**At the start of any session that might edit these docs:**
```bash
cd <path-to-buzz-project-docs>
git pull
```

**After any edit:**
```bash
git add <filename>
git commit -m "docs: <description>"
git push
```

Manual sync, not automatic — but reliable and version-controlled. Treat
commit history as the changelog: every commit message should explain WHY
the lesson was added, not just what was added.

## Setup on a new machine

```bash
cd <parent-of-Projects-folder>
git clone https://github.com/buzzstryker/buzz-project-docs.git
```

Clone alongside (NOT inside) your Projects folder so the relative path
`..\buzz-project-docs\` works from any project subfolder.
