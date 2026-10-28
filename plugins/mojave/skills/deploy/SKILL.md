---
name: deploy
description: Deploy the current GitHub repository to Mojave and iterate until it is live — push, deploy, read the build log on failure, fix, redeploy, and verify the URL. Use when the user asks to deploy, ship, host, publish or put an app online with Mojave, or to fix a failing Mojave deploy.
---

# Deploy to Mojave

Mojave builds from **GitHub**, not from local files. The deploy loop is: commit, push, deploy, check the result, fix, repeat.

## 1. Check the connection

Call the `whoami` tool from the `mojave` MCP server. It reports the user and the one Mojave org this connection acts in.

- **Works:** continue. If the org isn't the one the user means, they can reconnect and pick another (`/mcp` in Claude Code, or remove and re-add the connector).
- **Needs authentication, or no `mojave` tools:** ask the user to run `/mcp`, choose `mojave`, and sign in. A browser opens on mojave.sh, where they approve the connection and pick an org. Then retry.

## 2. Make sure GitHub has the code

Mojave builds what is on GitHub, and the Mojave server can't see this machine. Check before every deploy:

- `git remote get-url origin` names a GitHub repository. That `owner/name` is what you deploy.
- `git status --porcelain` is empty. Commit the work.
- `git rev-list --count @{upstream}..HEAD` is `0`. Push the branch.

If the repo isn't on GitHub yet, ask the user before creating a remote (for example `gh repo create`); it may be private. A private repo needs the Mojave GitHub App installed. If deploy reports it can't access the repo, send the user to https://mojave.sh/new to connect GitHub.

## 3. Deploy

**First deploy:** call `deploy` with `repo` set to the `owner/name` from step 2, and `branch` if it isn't the default branch. Pass `root` for a monorepo app in a subdirectory. Before deploying, check the shape of the app against the `platform` skill.

**Later deploys of the default branch:** pushing triggers a deploy automatically. Don't call `deploy` again, which would build the same commit twice. Instead:
1. Call `list_deployments` for the project.
2. Find the deployment whose `commitSha` matches the commit you pushed (`git rev-parse HEAD`). It may take a few seconds to appear.
3. Call `get_deployment` on it with `wait_seconds: 120`, and repeat while it is still building.

Call `deploy` directly to deploy another branch, or to redeploy without a new commit (for example after changing env vars). It waits up to two minutes. If the result is still building, keep calling `get_deployment` with `wait_seconds`.

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
- `resize_database` changes a database's or KV store's vCPUs/memory or grows its volume. It **restarts** the instance (a brief outage) and volumes never shrink, so confirm with the user first. `list_backups` shows its backups.

## Domains, cost and traffic

- `add_domain` attaches a hostname and returns the DNS records the user must create at their DNS provider. Once they have, `verify_domain` activates it; HTTPS is issued automatically. DNS can take minutes to propagate, so a failed verify right after adding is normal. `list_domains` shows status.
- `get_usage` reports the org's spend this month by line (bandwidth, compute, builds, databases, blob, workspaces). `get_analytics` reports a project's visitors and top pages, referrers and countries.

## Not available

- Deleting projects, databases and buckets is deliberately not available to you. Point the user to the Mojave dashboard if they ask.
