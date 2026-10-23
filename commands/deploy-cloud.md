---
description: Prepare the current app for free cloud hosting (Render + Neon) and guide the user through going live
---

Use the `free-cloud-deploy` skill for the application in the current directory:

1. Check prerequisites: tests pass, `ci.yml` exists, and the code is pushed to a GitHub
   repo. If it isn't pushed, ask the user before creating a repo or pushing.
2. Generate or update `render.yaml` and the production Spring profile, and commit them.
3. Give the user the exact numbered steps to do themselves: create the Neon database,
   create the Render Blueprint, and enter each secret. List every environment variable,
   where its value comes from, and which dashboard it goes in.
4. Once the user says it's deployed, verify the live API health URL and the frontend URL,
   then report them.
