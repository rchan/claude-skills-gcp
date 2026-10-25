# Claude Skills for Go, Node.js and Google Cloud

Three [Claude Code](https://claude.com/claude-code) skills that teach Claude how to build and maintain small web apps on Google Cloud the way an experienced developer would: thin handlers, one place that touches the database, secrets out of the repo, a safe admin area, sensible caching, and clean commits.

| Skill | Use it when |
|---|---|
| `go-gcp-conventions` | Building Go web apps (standard library, `html/template`) on App Engine or Cloud Run with Firestore, Cloud Storage and Secret Manager, or PostgreSQL/MySQL on Cloud SQL |
| `node-gcp-conventions` | Building Node.js apps (Express, Nunjucks) on App Engine or Cloud Run with Firestore through `firebase-admin`, or PostgreSQL/MySQL on Cloud SQL, with Stripe webhooks and multi-tenant workspaces |
| `git-github-conventions` | Committing, writing commit messages and PRs, and keeping secrets and key files out of Git |

## What they cover

- Project layout and the dependency pattern (one `server`/service layer, no database calls in handlers)
- Firestore data modelling, field naming, queries, composite indexes and transactions
- An SQL variant for PostgreSQL and MySQL on Cloud SQL (connections, migrations, pooling, types)
- Admin authentication (signed sessions, CSRF, login lockout), rate limiting and CSP
- In-memory page caching with explicit invalidation
- Redirects stored in the database, uploads to Cloud Storage, cron jobs and webhooks
- Secrets with Secret Manager, App Engine and Cloud Run deployment
- Testing patterns, and a list of pitfalls to avoid
- Git: commit format, what to stage, what never to commit, `.gitignore` baselines, PR and GitHub hygiene

## Install

Copy the folders you want into your personal skills directory:

```bash
cp -r skills/go-gcp-conventions skills/node-gcp-conventions skills/git-github-conventions ~/.claude/skills/
```

Or copy them into a project's `.claude/skills/` folder so they travel with the repository. Start a new Claude Code session afterwards.

Claude loads a skill automatically when your task matches its description. You can also invoke one directly, for example `/go-gcp-conventions`.

## Status and honesty

- The Go and Node skills are distilled from two real production apps. They have been used to review and extend those apps, not yet on a fresh project.
- The **Cloud Run** and **SQL (PostgreSQL/MySQL)** sections were written from general practice. They have not been tested against a real project, and API details should be checked against current documentation. Each says so at the top.
- These are guidance for an AI assistant, not a framework or a security audit. Review the code Claude produces, especially authentication, payments and anything that handles secrets.
- Names such as `server`, `pageCache` or `requireAdmin` in the skills describe patterns to reproduce, not packages to import.

## Contributing and feedback

Issues and pull requests are welcome, especially reports of where Claude followed a skill badly or where a recommendation is wrong for your stack.

## Licence

MIT. See [LICENSE](LICENSE).
