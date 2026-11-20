# Askwell — Team Rules

Askwell is an AI support chat widget for any website. A team of agents builds it, run from JSM HQ. Every agent follows this file. JSM HQ adds your role, the team, the workflow rules and your tools when your session starts.

## Docs

| File | Read it for |
|---|---|
| `docs/project.md` | What we build: features, data model, API routes, security rules, design |
| `docs/plan.md` | The phases, in build order (human-written) |

Your task (from `get_my_task`) lists the `project.md` sections it needs. Read those, not the whole file.

## How to talk to the product owner

- Lead with the answer or result. No preamble, no recap at the end.
- Short sentences, plain words. No paragraph over 3 sentences.
- A table for comparisons, a numbered list for steps. Plain text for a short answer.
- Need a decision? Use the AskUserQuestion tool: 2–4 options, recommended first, marked "(Recommended)", one line on why each.
- One decision at a time unless they're linked. Never more than 3.
- Say what you know vs. what you're guessing.

## Before you open a PR

- Run all checks (see Commands). All must pass.
- PR body: what changed, how you tested, and the task's done-when list with each item checked or explained.
- Vercel deploys a preview of every PR. Check your change on it, and put the preview URL in the PR. If the repo isn't connected to Vercel yet (early Phase 0), say so in the PR and skip this check.

## Checking your change on a Vercel preview

- Get the preview URL from GitHub, never from the branch name (Vercel shortens long names): the Vercel comment on your PR (`gh pr view <n> --comments`), or the PR's deployment status (`gh api repos/<owner>/<repo>/deployments?ref=<branch>`). Wait until the deployment succeeds.
- Previews are protected. Send the header `x-vercel-protection-bypass: $VERCEL_AUTOMATION_BYPASS_SECRET` on every request. For browser tests, also send `x-vercel-set-bypass-cookie: true` on the first request.
- If a preview URL is blocked by your sandbox, call `request_access` for `*.vercel.app` (all previews) rather than the single URL.

## When you must stop and ask

| Situation | Do |
|---|---|
| You need to change `packages/shared`, or change or drop an existing table or column | Stop. That's a contract change. The lead plans it; the product owner approves it. |
| The task is unclear or conflicts with `project.md` | Stop. Ask. Don't guess. |
| You need an account, a key, or a sign-in | Stop. Say exactly what you need and where it goes (e.g. your env file). Accounts and sign-ins are the product owner's job. |

## Reviews and your next task

- Fix every valid Codex (or lead) comment on the same branch, per the risk label rules below. CodeRabbit is advisory only: you may fix real bugs it finds in the same single fix round, ignore nits, and it never blocks a merge or triggers an extra round.
- Once your PR is open, the lead may give you your next task. Take it on a new branch from `main`.
- When review comments arrive on an earlier PR, switch back to that branch, fix, push, and return. Don't call `set_my_status` for the earlier task: it would update your current one. HQ tracks the earlier PR by itself.
- Never start a task that needs code from one of your open PRs. If nothing independent is ready, say so.
- Never merge your own PR, never push to `main`, never force-push a branch under review.

## Database rules

- The app reaches the database only through `supabase-js`: `createOwnerClient()` (the owner's Clerk token, RLS applies) and `createServiceClient()` (secret key, server only), both in `apps/web/lib/supabase`.
- Migrations are SQL files in `supabase/migrations`, named `<UTC timestamp>_<name>.sql`, newer than the latest one on `main`. Never edit a merged migration.
- Every new table or function ships with its RLS policies and grants in the same migration (see `project.md` §7 and §9). Add a new table or column only if your task says so.
- Test migrations, policies and functions on pglite (`packages/db/supabase-pglite.ts`). Never run a local Supabase stack or a Docker database.
- The Supabase dev project is shared by every developer and every preview. Its data is throwaway, but never apply an unmerged migration to it. After your PR merges, apply it to dev through the Supabase MCP, recording the file's version, then regenerate the types (the MCP's type generator) and confirm there's no diff.
- The Supabase MCP is scoped to the dev project. Never touch the production database. The product owner applies migrations to production before promoting the build.
- Migrations are additive: the live version must keep working while one runs. Dropping or renaming a table or column happens in a later PR, once no deployed code uses it.

## Security rules

`docs/project.md` §9 lists every rule. The ones you must never break:

- Read the plan on the server in every gated action. Never trust the client.
- Owner reads go through the owner client, so RLS scopes them. Never use the service client with a manual owner filter; use it only after the route has checked ownership.
- Render visitor messages and bot replies as text, never HTML.
- No secret in `NEXT_PUBLIC_*`, the widget bundle, or a commit. Never remove `.env` from `.gitignore`.

## Code conventions

| Area | Rule |
|---|---|
| Types | TypeScript `strict`. No `any`. Import API types from `packages/shared`; never redefine them. |
| API input | A zod schema per route, next to its type in `packages/shared` |
| API errors | `{ error: { code, message } }` with the right HTTP status |
| Tests | Vitest. At least one test per route and per pure function. |
| New dependency | Allowed. Name it and the reason in the PR body. |
| UI | Follow `project.md` §13 Design. Reuse components in `apps/web/components`. |
| App name | `APP_NAME` from `packages/shared`. Never hardcode it. |

## Commands

| Command | Does |
|---|---|
| `pnpm dev` | Run the app locally |
| `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` | Checks. All must pass before a PR. |
| `pnpm db:types` | Regenerate `packages/db/types.ts` from the dev project (needs `SUPABASE_ACCESS_TOKEN`; agents use the MCP's type generator instead) |

## Review by risk (one AI reviewer: Codex)

The `Risk label` workflow puts exactly one label on every PR from the files it changes. How much review a PR gets depends on that label. **Until `.github/workflows/risk-label.yml` is active on the default branch (or if a PR has no `risk:*` label yet), classify it by the paths it changes: docs/tests/copy-only = `risk:low`; migrations, auth/oauth, billing, RLS, `.github/workflows` or telecom = `risk:high`; everything else = `risk:medium`.**

- **`risk:low`** (docs, markdown, tests only, copy/translation files): merges as soon as required CI is green. The Codex review is **advisory**: `/workspace/automerge.sh` waits up to 10 minutes for Codex to post, logs what it said, resolves advisory threads and merges anyway. If Codex points at a real bug, fix it in a follow-up PR (and if merging manually via a script that blocks on unresolved threads, reply with the follow-up link and resolve the thread).
- **`risk:medium`** (other application code): the Codex review is **blocking**, with **one fix round at most**. Collect every finding, fix the real ones in one push, request one fresh Codex review, and stop. If that re-review still raises something, fix only genuine bugs (security, data loss, crashes, wrong behaviour) and add `needs:jorge` for anything you are unsure about. No further rounds.
- **`risk:high`** (migrations, auth, billing/payments, RLS/policies, `.github/workflows`, telecom/number code): the Codex review is **blocking** until it is clean. Batch each round's fixes into one push.

Rules for every PR:

- **Codex is the single AI reviewer.** Do not run agent cross-reviews, reviewer agents, "second opinion" sessions or self-review rounds.
- **CodeRabbit is advisory only.** You may read its comments and fix real bugs it finds in the same single fix round. It never blocks a merge and never triggers an extra round. Ignore its nits and style suggestions. It is not a required check. (`/workspace/automerge.sh` resolves CodeRabbit threads automatically before merging; if merging via a strict local script, reply "Advisory — ignored/deferred" and resolve them first.)
- **Branches do not need to be up to date with the base.** Strict up-to-date is off, so do not run update-branch just to refresh a green PR. A merge commit of the base branch does not need a new Codex review; the merge script accepts the review on the last non-merge commit.

## Before you open a PR

- **Run the local checks first:** lint, typecheck and the tests for what you changed. CI is the full test of truth, so do not run the whole suite locally.
- **Affected tests only.** Run the tests that cover the files you touched (for example the touched package's tests, or the test runner's "related"/filter mode), not every suite.
- **Keep PRs small:** aim for under 400 changed lines, not counting lockfiles and generated files. One concern per PR. Split bigger work into independent PRs that each pass CI on their own.
