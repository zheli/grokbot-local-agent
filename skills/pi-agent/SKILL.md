---
name: pi-agent
description: >-
  Use when the user asks to “have Pi do/review/look at something,” “use Pi Agent,”
  “get a second opinion or comparison from another model,” or for heavy work such as
  batch file processing, writing code, long research with web search, or long-form
  first drafts: to save the Bot’s own usage quota, hand these off by default to the
  Pi Agent on the Bot’s machine. Invoke via native RPC mode and save sessions by
  task name so follow-up edits stay continuous.
---
# Calling the Pi Agent on the Bot’s machine

Pi is a CLI coding agent installed on the Bot’s machine. Default model is DeepSeek Flash with thinking level high; it can read/write files, run commands, and search the web. Full install and config details are in `docs/setup-guide.md` in this repo. Scripts live in `~/tools/pi-call/` (the repo’s `bin/` directory).

## Usage policy
The Bot’s own model quota is limited. Pi runs on the user’s own model account (DeepSeek by default), billed separately from the Bot. So heavy work goes to Pi by default; the Bot only schedules and reviews.

Hand off to Pi:
- Batch file work: bulk edits, data cleanup, format conversion.
- Writing code: scripts, small tools, tests, debugging.
- Long research and web search.
- Long-form first drafts: docs, reports, articles.
- When the user explicitly asks for Pi, a second opinion, or a same-question comparison.

The Bot handles itself:
- Short chat and questions answerable in one or two steps.
- Connector actions (Gmail, X, GitHub, etc.) — Pi does not have those connections.
- Confirmation before send/publish/push/deploy, and reporting back to the user.

Approach:
- Keep instructions short: state the goal, where inputs are, which file to write output to, and the done criteria — a few lines is enough. Do not paste large source text into the prompt; let Pi read files itself.
- Persist results as files: specify an output path in the prompt and have Pi write the file; after review, deliver that file directly without rewriting it yourself. For changes, continue in the same session and have Pi edit.
- Land important conclusions: when a task produces important conclusions, decisions, or data, the Bot should proactively write them into memory or related docs — do not leave them only in Pi’s session.

## Steps
1. Choose a working directory: give Pi only the directory needed for the task (the repo or `/tmp/<task>`). Do not give `/` or the entire home directory.
2. Pick a session name: short English kebab-case, one name per task (e.g. `news-i18n`). Follow-up edits on the same task reuse the same name; start fresh with a new name if you want a clean slate.
3. Decide the output location: if there is no better target, use `/tmp/pi-out/<session-name>/`; for work that should be kept or synced, write it where it belongs (the matching repo, docs, or notes directory).
4. Write the prompt: short but complete — goal, constraints, output file paths, done criteria — and end by asking it to report which files it wrote plus a one-sentence conclusion. Pi cannot see the chat history; give any needed background in one or two sentences, and point long material to file paths.
5. Invoke. For anything that may take more than a minute or two, always use `pi-task`; it assigns a task id, records status and progress, and stores results as files:
   ```bash
   ~/tools/pi-call/pi-task run <workdir> "<prompt>" --session <session-name> --owner <your-agent-id> [--timeout seconds] [--no-write] [--no-web]
   ```
   `--owner` is required: it tags the task to the Bot that dispatched it so completion notifications reach the right agent, and so `pending`/`ack` prevent different Bots from stealing reports or acking each other’s tasks. Without it the job still runs (owner is stored as `null`, with a warning on stderr), but notifications cannot be routed back. (You can also set a default via the `PI_TASK_OWNER` env var, but passing it explicitly is more reliable.) Your agent id is in your profile path (shaped like `/home/box/agent-data/agents/<your-agent-id>/profile.json`).
   Use the Shell tool’s built-in background mode (non-blocking). **Do not use `nohup`, `&`, or `setsid`.** The first line of output is `TASK <task-id>`; on finish it prints status and the `result.md` path. Exit codes: 0 completed, 1 failed, 124 timed out, 130 cancelled.
   Other subcommands: `status <ID>` (status, owner, duration, recent events), `log <ID> [-n 30]`, `list [--all] [--owner <id>|none]`, `cancel <ID>`, `pending [--owner <id>|none]` (finished but not yet reported), `ack <ID> [--owner <id>|none]` (mark as reported; owner mismatch is rejected with exit code 1).
   Task data lives under `~/.pi/tasks/<task-id>/`: `status.json`, `events.jsonl`, `result.md`, `prompt-<n>.txt`. After a Bot-machine reboot, still-running tasks are marked `interrupted`; ask the user whether to rerun.
   **Completion notify:** When `PI_NOTIFY_URL` and `PI_NOTIFY_SECRET` are set in the environment (as Bot-machine env vars, not written to files), `pi-task` POSTs `{source, id, owner, state, exit_code}` to that webhook after each finished task — no task content. The routine that receives it wakes up, runs `pending --owner <own id>`, reviews, reports, and `ack --owner <own id>`. Only handle tasks whose owner is yourself; do not review or ack other Bots’ tasks — forward them to the owner Bot with SendToAgent (priority true); if the routine environment cannot send messages, hand back to the main Bot to forward. For tasks with no owner, only remind the user — do not ack. Tasks forwarded from another Bot whose owner is you: review, report, and `ack` yourself. If notify is not configured it is skipped automatically; send results are recorded as `notify` events in `events.jsonl`. Set `PI_NOTIFY_OFF=1` to temporarily disable.
   **Fallback reporting:** Completion notify can be missed. So on every wake (user message, routine, etc.) when you have dispatched Pi work, first run `pi-task pending --owner <own id>`, review only your own finished-but-unreported tasks, deliver them, then `ack <ID> --owner <own id>`; only ack your own — leave others for the Bot that dispatched them.
   For short Q&A that finishes in tens of seconds, you can call foreground:
   ```bash
   ~/tools/pi-call/pi-run <workdir> "<prompt>" --session <session-name> [--timeout seconds]
   ```
   For read-only or a different model, use the lower-level script:
   ```bash
   ~/tools/pi-call/pi-call.mjs <workdir> deepseek/deepseek-flash "<prompt>" --thinking high [--allow-write] [--no-web] [--session <session-name>] [--timeout seconds]
   ```
   `pi-run` and `pi-call.mjs`: stdout is only Pi’s final reply; default timeout 300 seconds; exit codes 0 success, 1 error, 124 timeout, 127 pi not found.
   Permissions: `pi-task run` and `pi-run` enable all tools by default (read, search, run commands, write files, web search); `pi-call.mjs` without `--allow-write` is read-only tools only.
6. Verify results: Pi’s reply is only its claim. Confirm with `git diff`, tests, or reading files before reporting to the user; do not treat unverified conclusions as fact.
7. Continue editing: for `pi-task` jobs use `pi-task reply <task-id> "<what to change>"` (same session, same directory, inherits the original owner; rejected while the task is still running); for `pi-run` jobs call again with the same `--session`. The two session stores are separate and cannot continue each other.

## Rules
- This uses the user’s own model quota (and the user’s own search quota for web search). Call as needed; do not make pointless repeats or large empty batch runs.
- Do not hand off to Pi: pushing code, deploying, changing DNS or production services, sending messages or email, or deleting important files. The Bot does those itself, and only after getting the user’s consent.
- Do not let Pi read or output `~/.pi/agent/auth.json`, secret files, or keys from environment variables.
- Do not ack others’ tasks: always pass `--owner <own id>` on `pending`/`ack`; on owner mismatch, hand off to the Bot that dispatched the work.
- In user-visible replies, say you “had Pi do X,” and state the result plus what you verified.

## Troubleshooting
- `pi: command not found`: first `export PATH=$HOME/.local/bin:$HOME/.node22/bin:$PATH`.
- Prompt says no model available: DeepSeek login expired; the user must run `pi` in a desktop terminal and `/login` again. The Bot cannot log in for the user.
- Timeout: split into smaller tasks, or increase `--timeout`.
- Errors or wrong results: check the exit code and `--verbose` logs first; if the task itself went wrong, point out the issue in the same session and have Pi redo it — retry at most once or twice. If it still fails, take over yourself and tell the user where Pi fell short; do not keep empty-running.
