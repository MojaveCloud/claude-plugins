---
name: platform
description: How apps run on Mojave — static sites, api/ serverless functions, long-running services (Node, Python, containers), mojave.json, Node/Python versions, env vars and managed storage. Use when writing, adapting or configuring an app to deploy on Mojave, writing a mojave.json, or deciding how a project should be served.
---

# Building for Mojave

Every deployment is one of three **kinds**. Mojave detects the kind at build time from committed files.

| Kind | What it is | When to use it |
| --- | --- | --- |
| **static** | Build output served from the edge, with SPA fallback and `404.html` support | No backend |
| **functions** | Static files plus `api/*.{js,ts,mjs}`, each served at `/api/<name>` and scaling to zero | Small stateless endpoints, webhooks, LLM calls under 30 s |
| **service** | One long-running process that owns its port. Every path is proxied to it, including websockets, SSE and streaming | SSR (`next start`), Express/Fastify/Hono, FastAPI/Django, websockets, in-process state |

Prefer static or functions: they cost nothing when idle. Use a service when the app needs a process that stays alive.

## Functions

```js
// api/hello.ts → GET /api/hello
export default async function handler(req) {
  // req: { method, path, query (raw string, no "?"), headers (lower-cased), body (string) }
  return { status: 200, headers: { 'content-type': 'application/json' }, body: JSON.stringify({ hello: 'world' }) }
}
```

- `body` is always a string, so parse JSON yourself. A non-string response body is JSON-stringified for you.
- The handler is not an Express/Next.js `(req, res)` handler and not a Fetch `Request`/`Response` one, so don't write it that way.
- Invocations are capped at **30 seconds** (504 after that). A thrown error becomes a clean JSON 500, and the stack goes to the runtime logs.
- TypeScript works without setup (esbuild bundles each function).

## Services

These are detected without config:
- Next.js without `output: 'export'`.
- A Node `start` script running `node`/`tsx`/… in a repo with a server dependency or a `.listen(`.
- A Python repo with no `package.json`: a Procfile `web:` process, or a FastAPI/Starlette/Flask/Django app.

Anything else opts in with `mojave.json` at the repo root:

```json
{ "kind": "service", "start": "node server.js", "port": 3000 }
```

**The service must listen on `0.0.0.0:$PORT`.** Binding `localhost`/`127.0.0.1` is the most common reason a service never becomes reachable.

A service runs as a single instance. Scale it vertically with `vcpus` and `memMiB`. Idle services are stopped and restarted on the next request. Services with live connections are never stopped, and `"alwaysOn": true` keeps one running (billed continuously). The filesystem is not durable storage, so use a managed store.

**Python** (service only, CPython 3.10–3.14, default 3.14):

```json
{ "kind": "service", "runtime": "python", "start": "uvicorn app:app --host 0.0.0.0 --port $PORT", "port": 8000 }
```

Python installs from `uv.lock`, `poetry.lock`, `Pipfile.lock`, `pdm.lock`, `requirements.txt` or `pyproject.toml`. Python functions are not supported.

**Containers** (when the account has them): `"runtime": "container"` with a public `"image"` or a `"dockerfile"`. Here `start` replaces the image CMD in exec form, without a shell.

## mojave.json fields

`kind`, `start`, `port`, `alwaysOn`, `vcpus`, `memMiB`, `runtime` (`node`/`python`/`container`), `python`, `node` (`"22"`, `"24"` default, `"26"`), `root` (monorepo subdirectory), `build` (extra Python build command), `image`, `dockerfile`, `buildVcpus`, `buildMemMiB`, `buildScratchMiB`.

Types are strict: write `"port": 3000` and `"node": "22"`. A mistyped field fails the build, and an unknown key only warns. The Node version otherwise comes from `engines.node`, then `.nvmrc`/`.node-version`.

## Builds

The build runs `npm ci`/`npm install` and `npm run build` on clean Linux. pnpm and yarn are used when there is a lockfile or `packageManager`. Commit the lockfile.

The build has no access to local files, `.env` or unpushed commits. Static output is found from the framework (Vite `dist/`, Next export `out/`, Astro, Eleventy `_site/`, …).

## Env vars and storage

- Project env vars (`set_env_var`) are exposed to the app as process env. They are secret by default and apply from the next deploy.
- Managed **Postgres** injects `DATABASE_URL`. Managed **Valkey** (Redis-compatible; `durable` or `cache` mode) injects `KV_URL`, which works with any Redis client. **Blob** storage injects `BLOB_URL` and `BLOB_TOKEN`.
- Read these from the environment and never hard-code credentials. Databases are reachable only from the project's own deployments, not from the user's laptop (a Valkey store can opt into a public TLS endpoint with `mojave kv expose`).

Full docs: https://mojave.sh/docs
