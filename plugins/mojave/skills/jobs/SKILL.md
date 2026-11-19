---
name: jobs
description: Run work as Mojave jobs — one command or function across many microVMs, each task up to 3 hours, with every task's output and result collected. Use when the user wants to run something in parallel or for a long time on Mojave, such as agent runs, evals, scrapers, backfills or batch processing.
---

# Run jobs on Mojave

A job runs one command as N **tasks**. Each task gets its own microVM, runs for up to 3 hours, and ends when its command exits. Exit code 0 means success.

## 1. Check the connection

Call the `whoami` tool from the `mojave` MCP server. It reports the user and the one Mojave org this connection acts in.

- **Needs authentication, or no `mojave` tools:** ask the user to run `/mcp`, choose `mojave`, and sign in. A browser opens on mojave.sh, where they approve the connection and pick an org. Then retry.

## 2. Choose the code

- **No repository.** Leave out `project`. Tasks run on a stock Python or Node image. Mojave reads the runtime from the command (`python`, `uv`, `node`, `npx`), or you set `runtime`. Use this for scripts that need only the standard library.
- **The user's code.** Pass `project`: a project built from their GitHub repository (see the `deploy` skill). Tasks run its current deployment. `source` runs another branch, tag or commit of the repo, and `image` runs a public container image instead.

## 3. Run

Call `run_job` with:
- `command`, a shell line run with `sh -c` in the app directory. Or `run`, a function (`module:function` in Python, `file.mjs:export` in Node) whose return value becomes the task's result.
- `tasks: N`, or `inputs`: a list with one entry per task.

Each task sees:

| Variable | Value |
| --- | --- |
| `MOJAVE_TASK_INDEX` | The task's position, from `0` |
| `MOJAVE_TASK_INPUT` | The task's entry of `inputs`, or empty |
| `MOJAVE_TASK_INPUT_JSON` | That entry as exact JSON (only when `inputs` was given) |
| `MOJAVE_TASK_ATTEMPT` | `1`, or higher on a retry |
| `MOJAVE_TASK_RESULT` | A file. JSON written here becomes the task's result |

Size tasks with `vcpus`, `mem_mib` and `timeout_sec`. The default timeout is 15 minutes, and the maximum is 10800 (3 hours). `parallelism` caps how many tasks run at once.

Run one or two tasks first to check the command, then the full job.

## 4. Read the outcome

`run_job` waits up to two minutes (set `wait_seconds` up to 900). If the job is still running, call `get_job` with `wait_seconds` until it settles.

A settled job lists up to 10 tasks, failed ones first, each with the end of its output and its result. Then:
- `get_job_results` pages through every task's status and result. Pass `status: "failed"` to see only failures, and continue from `next`.
- `get_task_output` returns one task's captured output and its whole result.
- A result holding an integer too large for a float64 (an id, say) comes as its exact JSON text in `resultJson` instead of `result`.
- If `job.error` is set, no task ran because the code failed to build. It holds the end of the build log. Fix the repo and push, as in the `deploy` skill.

When a task fails, read its output and fix the command or the code. Don't rerun the same job unchanged.

## 5. Cost and safety

- Jobs bill per second for the CPU used, the memory held and the data sent. Usage comes out of the org's $5 monthly credit first. `get_usage` shows spend.
- If a job needs a card, `run_job` fails with a billing link. Give the link to the user, and run the job again only after they say they added the card.
- A task runs **at least once**. A platform failure can rerun it from the start, so make side effects safe to repeat, for example by keying writes on `MOJAVE_TASK_INDEX`.
- `cancel_job` stops a job that is running away or was started by mistake. Finished tasks keep their output.
- Get the user's go-ahead before starting a large job (hundreds of tasks, or many vCPU-hours).
