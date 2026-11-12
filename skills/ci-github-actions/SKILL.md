---
name: ci-github-actions
description: GitHub Actions continuous-integration standards for Spring Boot + React monorepos - backend verify with Testcontainers, frontend lint/test/build/audit, Docker image build check, caching and branch protection. Use when adding or changing the CI pipeline or when CI fails.
---

# CI with GitHub Actions

Copy [templates/ci.yml](templates/ci.yml) to `.github/workflows/ci.yml`.

## What the pipeline guarantees

Every push to `main` and every pull request runs these jobs:

| Job | Steps | Fails when |
|---|---|---|
| `backend` | Java 21 → `./mvnw -B verify` | Any test fails, coverage < 80% on services |
| `frontend` | Node 22 → `npm ci` → lint → test → build → audit | Lint error, failing test, type error, high/critical vulnerability |
| `docker` | Builds both images | A Dockerfile is broken |

Testcontainers works out of the box on `ubuntu-latest` because Docker is preinstalled.

## Rules

- Use `npm ci` (the lockfile) and the Maven wrapper, never `npm install` or a global `mvn`,
  so CI builds exactly what was committed.
- Cache dependencies with `setup-java` `cache: maven` and `setup-node` `cache: npm`.
- No secrets are needed for CI. If a job ever needs one, read it from GitHub Secrets via
  `${{ secrets.NAME }}` and never echo it.
- `permissions: contents: read` at the top level (least privilege).
- Recommend that the user enables branch protection on `main`: require the three jobs to
  pass and require a PR. The agent doesn't change repository settings itself.

## Pushing the workflow file

GitHub rejects a push that adds or changes `.github/workflows/*` when the token lacks the
`workflow` scope: "refusing to allow an OAuth App to create or update workflow ... without
`workflow` scope". `gh auth login` doesn't request that scope by default. Have the user
grant it once:

```bash
gh auth refresh --hostname github.com --scopes workflow < /dev/null
```

Redirecting stdin skips the interactive prompts. The user just opens
https://github.com/login/device, enters the code shown, and clicks **Authorize**. Then
retry the push.

## When CI fails

1. Get the logs: `gh run view --log-failed`.
2. Reproduce locally with the same command as the failing step.
3. Fix the code, not the gate. Never lower the coverage threshold or skip tests to go green.
