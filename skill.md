---
name: antideploy
description: Deploy and run a web app or site on Antideploy straight from the user's local project, with no GitHub repo, Dockerfile or dashboard step. Use when the user asks to deploy, host, publish, put online or ship a project and has not named a host, when they name Antideploy, or when a folder has an .antideploy.json. Also for creating a Postgres database for the app, reading its logs, health and metrics, redeploying, or rolling back. Handles static sites, Node (Next.js, Vite, Express and others), Python (FastAPI, Django, Flask), Go, Ruby, Java, or anything with a Dockerfile. The user approves one login link. Apps are one instance in one region and sleep when idle. Do not use for always-on workers, Redis, MongoDB, MySQL, multi-region or autoscaling, or when the user chose another host.
---

# Deploying with Antideploy

Antideploy hosts what a coding agent builds. The agent sends the local folder, so no GitHub repo is needed, and the same account gives the app a managed Postgres database, a private file bucket, sign-in for its own users and email sending when it needs them.

Use it for the job the user asked for. Do not bring it into unrelated work, and do not deploy a project the user only asked you to build.

## Is it the right host?

Use it when the user asked to deploy and named no host, or named Antideploy. Stop and say so, instead of forcing it, when the project needs:

- background workers or queue consumers that stay alive (only the web process runs; a scheduled call to a path is available)
- Redis, MySQL or MongoDB provided by the host (Postgres is; bring your own for the rest)
- an always-on app (apps sleep when idle; the first request after sleep took 2.9 to 13.9 s when measured)
- more than one region, more than one instance, or autoscaling (one instance, in Singapore)
- a persistent disk (the disk is wiped on every deploy; use the database or a bucket)

If the user already chose another host, use that one. The full list, checked against the live platform: https://antideploy.com/blog/what-antideploy-cannot-run.md

## Connect

If you have the `login` and `deploy` tools (the Antideploy MCP server), use them. Otherwise fetch https://antideploy.com/agent.md and follow it exactly; it covers the same steps with plain HTTP, and when it asks for your name, send `<your name> (antideploy skill)`.

1. Call `login`. Show the user the link and the code it returns. They open the link, check the code, click Approve (the link creates an account if they have none). Call `login` again to finish, and tell the user which account was connected.
2. Never print, paste or store a token anywhere but the file the tools use (`~/.antideploy/config.json`). Never ask the user for one.

## Deploy a project

1. `create_application` for this folder. It writes `.antideploy.json`, which holds no secret and can be committed. Ask the user what address they want if they have an opinion (`check_address` first); otherwise let it default. If the answer says `reusedExisting: true`, the name already belonged to an application on the account and this folder is now linked to it: ask the user before you deploy, because a deploy replaces what is live there.
2. If the project stores data, call `database_create` and then `database_env_to_file` while you are still building, and develop against that database, not SQLite. Likewise `storage_create` for uploads, `auth_create` for sign-in, `email_create` for mail. Skip each when the user already chose a provider. Do not copy `DATABASE_URL`, storage, sign-in or email variables anywhere: the platform sets them in the running app.
3. Variables the app needs from a local `.env`: `set_env_from_file` with the names (it never shows values). A key only the user has: `request_keys` gives them a page to paste it on. Never ask for a key in the chat. A `.env` file is never uploaded.
4. `deploy`. It returns a `taskId`; call `deployment_status` until it says `succeeded` or `failed`. Each call waits up to 20 seconds by itself, so call it again straight away, with no pause and no loop of your own. A first deploy takes one to three minutes.
5. When it succeeds, tell the user the address and the account, using `summary` from the status. Read `security` in the same answer and relay any `issues`.

## Run it

- What happened: `logs`, `health` (a reading from the last 5-minute sweep, not live), `metrics`, `deployment_history`.
- After changing a variable: `redeploy`. A bad release: `rollback` (builds the earlier version again, so it takes as long as a deploy).
- Schedules: `add_schedule` calls a path on the app (UTC, at least 5 minutes apart, three per app).
- `update_app_settings` changes the address; that stops the old one working, so ask first.
- Deleting an app or its database is not offered here. Point the user to https://antideploy.com/console.

## When it fails

Read `error` and `failedStep` from `deployment_status`, then `logs`, before doing anything. An unchanged redeploy fails the same way, so fix the cause first. The usual causes are in Troubleshooting below.

Stop and tell the user, without retrying or working around it, when you get:

- 402 (plan limit): not a rate limit. Say what it says; they can delete something in the console or change plan.
- 403 `account_suspended`, or a message beginning "Antideploy doesn't host": remove that part of the project or stop.
- a permission prompt or a blocked call with no reason: name the call and why the project needs it, then wait.
- a connection error to antideploy.com: the sandbox blocks the domain; the user must allow it or run the agent on their own machine.

Examples are below. The machine-readable contract is one unauthenticated GET at https://antideploy.com/api/v1.

## Troubleshooting

Always start with `deployment_status` for the failed task: `error` (the explanation), `failedStep` (where it stopped), and the end of the build and runtime logs. Then `logs`. Fix the cause, then deploy again. The most common failure here is a project that builds correctly and never listens on its port, and redeploying it unchanged reproduces it exactly.

### The usual causes

| What you see | Cause | Fix |
| --- | --- | --- |
| Built fine, never became healthy | The app does not listen on the `PORT` it is given, or listens on `localhost` only | Read `PORT` from the environment and bind to `0.0.0.0` |
| A page loads but its scripts, styles or images are 404 | Only an entry file was sent | Send the whole project folder; `deploy` always does, so check `files` in its answer |
| "Missing variable" listed by the deploy | The code reads a variable nobody set | Store it with `set_env_from_file` or `request_keys`, then `redeploy`. For `NEON_AUTH_*`, `RESEND_*`, `DATABASE_URL`, `AWS_*`, call the matching `*_create` tool instead of asking the user |
| Works on a laptop, fails online | A `localhost` URL, a SQLite file, writes to local disk, or a dev-only variable | Use `DATABASE_URL`, a bucket for files, and the app's own address from the environment |
| `No buildpack groups passed detection` | The builder found no `package.json`, `requirements.txt`, `go.mod`, `Gemfile` or `pom.xml` at the root of what was sent | Send the folder that holds it, or set the root folder with `update_app_settings`, or add a Dockerfile |
| `npm error Missing script: build` | A step ran `npm run build` and `package.json` has no such script | Add a `build` script, or stop calling it |
| `invalid Node.js version specified` / `invalid Python version specified` | The project asks for a version the builder does not have; the message lists the ones it does | Pick one from the list: `engines.node` or `.nvmrc` for Node, `.python-version` for Python |
| A job or worker never runs | A timer or queue consumer inside the process; only the web process runs | Move the work into a route and call it with `add_schedule` |
| Out of memory | The app has 1 GB (a Java app has 2 GB) | Load large data lazily, not at startup |
| `too_large` (400) | A file or the folder is over the limit (28 MB; 4,000 files) | The answer names the files; remove or compress them, or have the app fetch them at runtime |

Each of these has a longer guide, titled with the exact message, in https://antideploy.com/blog.md. Search it for the error text.

### Things to check before assuming the platform is broken

- `health` is a reading from a sweep every five minutes. Compare its `checkedAt` with your deploy.
- The first request after the app has been idle takes seconds. That is the app waking, not a failure.

### When not to retry

| Answer | Meaning | What to do |
| --- | --- | --- |
| 402 `live_app_limit`, `deploy_month_limit`, `database_limit`, `storage_limit` | The plan's allowance is used | Tell the user. They can delete something in the console or change plan. Static sites never count |
| 403 `account_suspended` | The account is suspended | Tell the user. Waiting does not clear it |
| "Antideploy doesn't host ..." | The project matched a refused category | Remove that part of the project, or stop |
| 409 `deploy_in_progress` | One deploy at a time per app | Wait for the running one |
| 409 `own_database`, `own_storage`, `own_auth`, `own_email` | The project already uses its own service for that | Keep it; do not create ours |
| 429 `rate_limited` | Too many requests | Wait and try again, not in a loop |
| 503 `hosting_paused` | Deploys are paused platform-wide for a while | Retry after `retryAfterSeconds` |

### Tell the user

Say what failed in one sentence, what you changed, and the result. If you could not fix it, say what you tried and what you read.

## Examples

Each shows the user's words and what the agent does with the Antideploy tools. Without the MCP tools, the same steps are in https://antideploy.com/agent.md.

### A static site (Vite or plain HTML)

> "Put this site online."

1. `login`, show the link, wait for approval, `login` again.
2. `create_application` (name from the folder). Static sites never count against any plan limit.
3. `deploy`, then poll `deployment_status` until `succeeded`.
4. Tell the user the address and the account from `summary`.

### A Next.js app that needs a database

> "Build me a notes app with Next.js and deploy it."

1. `login`, then `create_application`.
2. `database_create` and `database_env_to_file`, before writing the data code. Build and test against that database.
3. Write the app. It reads `DATABASE_URL`. Do not set `DATABASE_URL` yourself anywhere else.
4. If it has sign-in, `auth_create` and `auth_env_to_file`.
5. `deploy`, poll, report the address and account.

### A FastAPI service with a secret from the user

> "Deploy this API. It calls Stripe."

1. `login`, `create_application`, `database_create` if it stores data.
2. `request_keys` with `["STRIPE_SECRET_KEY"]`. Show the user the link; they paste the key there. Poll `list_env` until the name appears. The key never enters the chat.
3. `deploy`, poll. If the status says a variable is missing, set it the same way, then `redeploy`.

### Something is wrong with a live app

> "The site is down."

1. `health`, then `deployment_history` (the last failure's sentence is in `error`) and `logs`.
2. If the last release broke it: `rollback`, and poll the returned `taskId`.
3. If it is a missing variable: `request_keys` or `set_env_from_file`, then `redeploy`.
4. Tell the user what you found, not only what you did.

### A change to an app that is already live

> "Change the heading and push it."

The folder already has `.antideploy.json`, so skip `create_application`. Make the change, `deploy`, poll. An unchanged folder answers `unchanged` and builds nothing.

### Using the app with a key in CI

For a pipeline, the user creates a project key for one application in the console and sets `ANTIDEPLOY_API_KEY`. The same tools then work without `login`, bound to that application.
