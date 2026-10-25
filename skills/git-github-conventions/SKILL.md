---
name: git-github-conventions
description: Git and GitHub conventions for small projects (Go and Node apps on Google Cloud): commit message format, what to stage and what never to commit, .gitignore baselines, branches, pull requests and secret hygiene. Use when committing, writing commit messages or PR descriptions, setting up a repo or .gitignore, pushing to GitHub, or when a change might include secrets or key files.
---

# Git and GitHub conventions

Derived from the commit history of two small single-developer production apps (one Go, one Node) that worked directly on `main`. It keeps what worked, fixes what didn't (overused commit types, long subjects, secrets in tracked files), and assumes no CI to begin with.

## When to commit

- Commit and push only when the user asks. Never as a side effect of finishing a task.
- If on the default branch and a branch is wanted for a PR, create the branch first.
- Never force-push `main`, rewrite published history, or skip hooks unless the user explicitly asks for that specific action.

## Commit messages

Use a Conventional Commits style, or keep an existing project's style if it differs:

```
<type>: <imperative summary>
```

- Types in use: `feat`, `fix`, `refactor`, `chore`, `style`. Add `docs` and `test` as needed. A lowercase type, a colon and a space.
- Choose the type honestly. It is easy to label everything `feat` (spacing tweaks, a runtime version bump). Use `fix` for corrections, `style` for CSS/layout-only changes, `chore` for config, dependencies and `.gitignore`, `refactor` when behaviour is unchanged, `test` for tests only.
- Summary is imperative and specific, about 72 characters or fewer, and says what changed and where, for example `feat: manage redirects from a Firestore collection with an admin reload button`. Put anything longer in the body instead.
- Add a body when the reason isn't obvious (a blank line, then why it changed, not a list of files). Mention a data or index change that must be deployed separately.
- One logical change per commit. A schema change, its migration script and the code that uses it can be one commit. Unrelated tweaks cannot.
- Commit messages must not contain secrets, tokens or customer data.
- Follow any attribution trailer the environment asks for (for example a `Co-Authored-By:` line) and add no others.

## What to stage

- Run `git status` and `git diff` before staging. Stage named files (`git add path/to/file`), not `git add -A` or `git add .`.
- Leave out files the user is still working on and anything unrelated to the task. Working trees often hold uncommitted work in progress next to a requested change.
- Review the staged diff (`git diff --staged`) for secrets, debug output and local paths before committing.

## Never commit

- Secrets and credentials: API keys, Stripe or webhook secrets, tokens, passwords, OAuth client secrets, session secrets.
- Key and credential files: `service-account*.json`, `*-key.json`, any `<project>.json` Firebase/GCP key, `.env`, `cookies.txt`, `*.pem`.
- Deploy config containing secret values (`app.yaml`, `app-live.yaml`, `cloudbuild.yaml`). Use Secret Manager (`--set-secrets` on Cloud Run, or a resource path in `app.yaml` on App Engine) and keep only non-secret settings in tracked files.
- Build output and dependencies: compiled binaries (`/<binary-name>`, `/bin/`, `/tmp/`), `node_modules/`, generated CSS if it is rebuilt on deploy, `build-errors.log`.
- Editor and OS files: `.DS_Store`, `.vscode/`, `.idea/`.

### If a secret was committed

1. Rotate it first, in the provider's dashboard. Assume it is compromised the moment it was pushed.
2. Remove it from the tracked file and replace it with an environment variable or a Secret Manager reference.
3. Removing it from the latest commit does not remove it from history. If the repo is public or shared, rewrite history with `git filter-repo` and force-push (only with the user's explicit approval). For a private single-user repo, rotation is the essential step.
4. Add the file pattern to `.gitignore` and use `git rm --cached <file>` so it stops being tracked.

## .gitignore baselines

Go:

```
/<binary-name>
/bin/
/tmp/
*.exe
.env
.DS_Store
build-errors.log
service-account-key.json
*-key.json
```

Node:

```
node_modules/
.env
.env.*
.DS_Store
*.log
service-account*.json
*-key.json
cookies.txt
.gcloudignore
.vscode/
.idea/
```

Add the patterns when the repo is created, before the first commit. `.gitignore` does not affect files that are already tracked.

## Branches and pull requests

- Small solo changes may go straight to `main`. Use a branch and a PR for anything risky, large, or that needs review or a CI run: schema migrations, auth or payment changes, dependency upgrades, platform changes (App Engine to Cloud Run).
- Branch names: `<type>/<short-description>` (`fix/stripe-webhook-idempotency`, `feat/cloud-run-deploy`).
- PR title follows the commit format. PR body: what changed, why, how it was tested (`go vet && go test -race`, `node scripts/security/run-all.js`), and anything that must be deployed or configured separately (indexes, secrets, env vars).
- Squash-merge so `main` keeps one commit per change.
- Do not paste secrets, customer data or full logs into PRs or issues. They are visible to everyone with repo access and are kept even if later deleted.

## GitHub repository setup

- Create repos as **private** by default. Make one public only deliberately, and only after checking its whole history for secrets.
- Enable secret scanning and push protection (Settings, Code security) on every repo. It blocks pushes that contain recognised API keys.
- Protect `main` once there is CI: require the checks to pass before merging.
- A minimal CI workflow is worth adding (`.github/workflows/ci.yml`): Go repos run `go vet ./... && go test -race ./...`, Node repos run `npm ci && node --test`.
- Pull requests from the command line need the `gh` CLI signed in (`gh auth login`). Without it, create PRs in the browser.

## Housekeeping

- Delete merged branches. Keep `main` deployable: it is what gets deployed.
- Tag releases of anything deployed to a live environment (`v1.4.0`) so a bad deploy can be rolled back to a known commit.
- `git log --oneline` should read as a changelog. If a message would not make sense there, rewrite it before pushing.
