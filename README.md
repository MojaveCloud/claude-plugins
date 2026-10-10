# Mojave for Claude

Deploy to [Mojave](https://mojave.sh) from Claude Code. Ask Claude to ship your repo, and it pushes, deploys, reads the build log if something breaks, fixes it, and hands you the live URL.

## Install

You need the Mojave CLI, logged in:

```sh
curl -fsSL https://mojave.sh/install.sh | sh
mojave login
```

Then add the plugin to Claude Code:

```sh
claude plugin marketplace add MojaveCloud/claude-plugins
claude plugin install mojave@mojave
```

Try it: *"deploy this repo to Mojave"*.

## What Claude can do

- **Deploy** a GitHub repo or a public container image, and wait for the build.
- **Debug** a deployment from its build logs and runtime logs.
- **Roll back** to an earlier deployment, or forward again.
- **Manage env vars** for a project. Secret values stay write-only.
- **Create storage:** Postgres, Valkey (Redis-compatible) or a blob bucket. Its credentials go straight into your project's environment and never into the chat.

Claude can't delete projects, databases or buckets. Those stay with you in the CLI and dashboard.

## What's inside

| | |
| --- | --- |
| `plugins/mojave/.mcp.json` | Runs `mojave mcp`, the MCP server built into the CLI. It uses your `mojave login`, so there are no extra keys. |
| `skills/deploy` | The deploy loop: push, deploy, read the log, fix, redeploy, verify. |
| `skills/platform` | How apps run on Mojave: static sites, `api/` functions, services, `mojave.json`, storage env vars. |

Other MCP clients can use the server directly:

```json
{ "mcpServers": { "mojave": { "command": "mojave", "args": ["mcp"] } } }
```

## Troubleshooting

Run `/mcp` in Claude Code to see the server's status.

- **"not installed":** install the CLI as shown above.
- **"predates `mojave mcp`":** run `mojave update`.
- **"not logged in":** run `mojave login`. You don't need to restart Claude Code after logging in.

Docs: https://mojave.sh/docs · Contact: brandon@mojave.sh
