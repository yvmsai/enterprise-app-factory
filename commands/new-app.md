---
description: Build a new enterprise app (Spring Boot + React + PostgreSQL) from a design document and run it locally in Docker
argument-hint: <path-to-design-doc> [app-name]
---

Build a new application from the design document at: $ARGUMENTS

Follow the `app-factory-orchestrator` agent's pipeline, phases 1 through 8:

1. Read the design document. If no app name was given, derive a short kebab-case name
   from the document's title.
2. Create the app directory `<app-name>/` next to the design doc's folder (or in the
   current directory) and copy the design doc to `<app-name>/docs/design.md`.
3. Run phases 1–2 and **stop at checkpoint 1** for plan approval.
4. After approval, run phases 3–8, one feature at a time, keeping `docs/plan.md` ticked.
5. **Stop at checkpoint 2** with the build report and the local URLs.
