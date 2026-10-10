---
name: deploy
description: Deploy the current repository to Mojave and iterate until it is live — push, deploy, read the build log on failure, fix, redeploy, and verify the URL. Use when the user asks to deploy, ship, host, publish or put an app online with Mojave, or to fix a failing Mojave deploy.
---

# Deploy to Mojave

Mojave builds from **GitHub**, not from local files. The deploy loop is: commit, push, deploy, check the result, fix, repeat.

## 1. Check the tools

Call the `whoami` tool from the `mojave` MCP server.

- **Works:** continue.
- **"not logged in":** ask the user to run `mojave login` in their terminal. It opens a browser. Then retry.
- **No `mojave` tools at all:** the CLI is missing or outdated. Run `command -v mojave` to find out which.
  - Missing: ask the user before installing it with `curl -fsSL https://mojave.sh/install.sh | sh`.
  - Present: run `mojave update`.
  - Either way, finish with `mojave login`, then `/reload-plugins` or restart Claude Code.

## 2. Make sure GitHub has the code

The repo needs a GitHub `origin` remote. Commit the work and push the branch you are deploying. Only what is on GitHub gets built.

- If the repo is not on GitHub yet, ask the user before creating a remote (for example `gh repo create`). It may be private.
- A private repo needs the Mojave GitHub App installed. If deploy reports it cannot access the repo, send the user to https://mojave.sh/new to connect GitHub.

## 3. Deploy

**First deploy:** call `deploy` with no arguments to use the current directory's GitHub remote. Pass `root` for a monorepo app in a subdirectory. Before deploying, check the shape of the app against the `platform` skill.

**Later deploys of the default branch:** pushing triggers a deploy automatically. Don't call `deploy` again, which would build the same commit twice. Instead:
1. Call `list_deployments` for the project.
2. Find the deployment whose `commitSha` matches the commit you pushed. It may take a few seconds to appear.
3. Call `get_deployment` on it with `wait_seconds: 600`.

Call `deploy` directly to deploy another branch or to redeploy without a new commit, for example after changing env vars.

Read the `warnings` in the result. Uncommitted or unpushed local work means the deploy built something other than what you edited.

## 4. On failure: read, fix, push, retry

A failed `deploy` or `get_deployment` result includes `buildLogTail`. The real error is usually near the end, but the first error above it is the cause.

Common causes:
- **A dependency or lockfile is out of sync.** Run the install locally (`npm install`, `uv lock`, …) and commit the lockfile.
- **A command works locally but not on Mojave.** The build runs `npm ci` / `npm run build` on a clean Linux machine. Missing devDependencies, case-sensitive imports and unset env vars all show up here first.
- **A service exits or is never reachable.** It must listen on `0.0.0.0` on the port in `$PORT`, not `localhost`.
- **The build ran out of memory.** Raise `buildMemMiB` in `mojave.json`.

Fix the cause in the repo, commit, push and deploy again. Don't loosen the build to hide the error, for example by deleting type checks.

## 5. Verify

Once `status` is `ready`, fetch the `url` (`curl -sS -o /dev/null -w '%{http_code}' <url>`) and an API route if there is one.

The first request to a scaled-to-zero function or service can take a few seconds. A 5xx after a successful build is a runtime problem: read `get_runtime_logs` for the project.

Report the live URL to the user.

## Env vars and storage

- `set_env_var` stores a value as a secret by default. Changes take effect on the **next** deploy, so redeploy after setting them.
- `create_database` (Postgres → `DATABASE_URL`, Valkey → `KV_URL`) and `create_blob_bucket` (`BLOB_URL`, `BLOB_TOKEN`) inject credentials into the project's env. The app reads them from the environment, so never hard-code them. Redeploy afterwards. These resources bill while they exist, so confirm with the user before creating one.
- Deleting projects, databases and buckets is deliberately not available to you. Point the user to the dashboard or `mojave databases rm` if they ask.
