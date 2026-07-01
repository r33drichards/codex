# Driving `app-server` from a scripting runtime (e.g. mcp-js / mcp-v8)

`codex app-server` speaks JSON-RPC 2.0 over stdio (newline-delimited JSON). Any
runtime that can spawn a child process and stream its stdin/stdout can drive it
— for example the [mcp-v8](https://github.com/r33drichards/mcp-js) `run_js`
JavaScript runtime. **No changes to `app-server` are required for this.**

A working end-to-end example (create a thread, start a `hello world` turn) lives
in the mcp-js repo at `examples/codex-app-server/`.

This note records two behaviors that matter when the client is a script rather
than a long-lived editor extension. Both were observed against `codex-cli`
`0.142.5`.

## 1. Keep stdin open until you have read the responses you need

The server processes requests asynchronously and flushes responses to stdout
from handler tasks. If the client writes its requests and immediately closes
stdin (EOF), the server can begin shutting down **before** those async responses
are flushed — so a naive `printf '…request…' | codex app-server` prints nothing.

A correct client keeps the stdin pipe open and reads stdout line-by-line,
closing stdin only after it has received the responses/notifications it needs.
This rules out a one-shot "write everything, capture all output" subprocess
model; the driver must be interactive.

## 2. A thread is persisted to disk when its first turn starts, not at `thread/start`

`thread/start` returns a `thread` object whose `path` points at the rollout file
that *will* back the session, and emits `thread/started` — but it does **not**
write that file yet. If the process exits after only `thread/start`, no rollout
is left on disk, and a later `thread/resume` by id cannot find it.

The rollout (its `session_meta` line and the turn's items) is written when the
first turn starts (`turn/start`). Because `turn/start` requires the
server-generated `threadId` from the `thread/start` response, creating a
*persisted* thread necessarily means: `thread/start` → read the id → `turn/start`
**in the same live process**.

This is why "create a `hello world` thread" persists the thread even when
inference then fails (e.g. `401 Unauthorized` at
`wss://api.openai.com/v1/responses` when no auth is configured): the user message
turn is what commits the rollout, and the inference error happens afterward.

## Minimal request sequence

```jsonc
// → initialize (request)
{"method":"initialize","id":0,"params":{"clientInfo":{"name":"my_client","title":"My Client","version":"0.1.0"},"capabilities":null}}
// → initialized (notification)
{"method":"initialized"}
// → thread/start (request) — response carries thread.id and thread.path
{"method":"thread/start","id":1,"params":{"cwd":"/work"}}
// → turn/start (request) — needs the threadId from the thread/start response
{"method":"turn/start","id":2,"params":{"threadId":"<id>","input":[{"type":"text","text":"hello world","text_elements":[]}]}}
```
