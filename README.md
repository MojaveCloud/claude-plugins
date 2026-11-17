# Mojave for Claude

Build GitHub repos on [Mojave](https://mojave.sh) from Claude. Ask Claude to deploy your repo, and it pushes, builds it, reads the build log if something breaks, and fixes it. A new project is a job project: run its code across many machines with `mojave jobs run`.

## Install

In Claude Code:

```sh
claude plugin marketplace add MojaveCloud/claude-plugins
claude plugin install mojave@mojave
```

Then run `/mcp`, choose **mojave** and sign in. A browser opens on mojave.sh, where you approve the connection and pick the org Claude may work in. No CLI or API key needed.

Try it: *"deploy this repo to Mojave"*.

In claude.ai or Claude Desktop, add `https://mojave.sh/mcp` as a custom connector instead (Settings → Connectors).

## What Claude can do

- **Deploy** a GitHub repo and wait for the build.
- **Debug** a deployment from its build logs and runtime logs.
- **Roll back** to an earlier deployment, or forward again.
- **Manage env vars** for a project. Secret values stay write-only.
Claude can't delete projects. That stays with you in the dashboard.

## What's inside

| | |
| --- | --- |
| `plugins/mojave/.mcp.json` | Connects to Mojave's hosted MCP server, `https://mojave.sh/mcp`. |
| `skills/deploy` | The deploy loop: check git, push, deploy, read the log, fix, redeploy, verify. |
| `skills/platform` | How apps run on Mojave: static sites, `api/` functions, services, `mojave.json`, env vars. |

## What it runs and sends

- **Runs nothing locally.** The plugin is two skills (text) and one remote MCP server address. The deploy skill asks Claude to run read-only `git` commands (`remote`, `status`, `rev-list`, `rev-parse`) so it doesn't deploy unpushed work.
- **Sends** each tool call (project names, repository names, env var keys and values you ask Claude to set) to `https://mojave.sh/mcp` over HTTPS. Nothing goes to any other service.
- **Signs in with OAuth.** Claude Code stores the access token; Mojave never sees your Claude account. Each connection is limited to one Mojave org, and you can revoke it at any time under **Connected apps** in the dashboard.
- **Never returns** secret env var values to Claude.

Mojave's privacy policy: https://mojave.sh/docs/privacy

## Troubleshooting

- **`/mcp` shows mojave as needing authentication:** choose it and sign in.
- **Wrong org:** revoke the connection under Connected apps, then sign in again from `/mcp` and pick another org.

Docs: https://mojave.sh/docs · Contact: brandon@mojave.sh
