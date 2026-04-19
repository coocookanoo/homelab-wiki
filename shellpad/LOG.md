# shellpad — Project Log

A notebook-style shell with rich output rendering. Commands run in cells. Output is classified and rendered differently depending on type: text, JSON, table, markdown, AI. Text layout powered by [Pretext](https://github.com/chenglou/pretext) via Canvas. AI integration via local Ollama (qwen3:32b).

**Location:** `/home/adminbill/projects/shellpad/`
**Stack:** Bun + Canvas + Pretext + Ollama

---

## 2026-04-03 — Initial build + troubleshooting

**Who:** Bill + Claude Code

### What we built

Full prototype from scratch in one session:

- `src/model.ts` — Cell/Session types, OutputType enum (`text | json | table | markdown | image | error | ai`)
- `src/classifier.ts` — classify(raw, command): `@` prefix → ai, JSON detection, table heuristic, markdown headers/bullets
- `src/pty.ts` — Shell process wrapper, sentinel-based command completion detection
- `src/ai.ts` — Streams from Ollama `/api/chat` with qwen3:32b, parses NDJSON chunks
- `src/renderer.ts` — Canvas drawing with Pretext text measurement. drawCell() dispatches by OutputType. Markdown uses `prepareWithSegments` + `layoutWithLines`.
- `src/shell.ts` — Browser client: WebSocket, requestAnimationFrame render debouncing, smart scroll (only auto-scroll if already at bottom), strips Qwen3 `<think>...</think>` blocks
- `src/server.ts` — Bun WebSocket server with typed `ws.data` per-connection state
- `public/index.html` — Dark UI, canvas top, input bar bottom

### Troubleshooting log

#### node-pty + Bun: onData callback never fires

**Symptom:** `@axel` queries (AI path) worked fine. Regular shell commands stayed "running" forever — no output, no done message.

**Diagnosis:**
1. Added logging: server showed `[shellpad] recv type=run` and `[shellpad] pty.run: ls` — so the server was receiving commands and calling `pty.run()`.
2. No `[pty] chunk` logs followed — meaning `proc.onData()` from node-pty was never called after the command was written.

**Root cause:** node-pty is a native Node.js addon that registers file descriptor watchers via libuv. Bun uses its own event loop (not libuv). The native callbacks never fire when node-pty runs under Bun.

**Fix:** Replaced node-pty entirely with `Bun.spawn()` (persistent bash with piped stdin/stdout):
- Bash in non-interactive mode (piped stdin): no prompt output, no command echo, state persists across commands
- `pump()` async readers on stdout + stderr drive the data flow through Bun's native event loop
- Sentinel pattern unchanged: `cmd; echo "__SHELLPAD_DONE_${ts}__:$?"`
- Removed first-line (command echo) stripping — non-interactive bash doesn't echo commands
- node-pty import removed from pty.ts (package.json left alone)

#### WebSocket upgrade conflict (earlier)
`/` route matched both HTML and WebSocket upgrade requests → fixed with dedicated `/ws` route.

#### Bun `ws.data` vs Map (earlier)
Original server used `Map<WebSocket, Client>`. Bun's ServerWebSocket may not maintain reference equality between `open` and `message` callbacks → `clients.get(ws)` returned undefined, `message` handler silently returned early. Fixed with `Bun.serve<ClientData>` typed generic — `ws.data` is always valid.

#### Sentinel detection across chunks (earlier)
Initially checked each PTY chunk individually for sentinel. Bash can split `echo "__DONE__:0"` across two data events. Fixed by accumulating in `ws.data.pendingOutput` and matching the full buffer.

#### @axel flashing + disappearing (earlier)
Canvas redraws on every streaming chunk while scroll position wasn't tracked. Fixed with requestAnimationFrame debouncing + smart scroll (only scroll to bottom if user was already there).

### Architecture notes

**Pretext two-phase API:**
- `prepare(text, font, opts)` — one-time measurement, returns prepared blob
- `layout(prepared, width, lineHeight)` — cheap, call on every resize
- For markdown: `prepareWithSegments()` + `layoutWithLines()` to get per-line text back

**Sentinel pattern (shell commands):**
```
proc.stdin.write(`${cmd}; echo "__SHELLPAD_DONE_${ts}__:$?"\n`)
```
Server watches accumulated output for `/__SHELLPAD_DONE_\d+__:(\d+)/`. When matched, slices output before sentinel, trims, sends `done` message with exit code.

**Output classification priority:** `@` prefix → ai | valid JSON → json | 3+ tab-separated cols → table | `#`/`-`/`**` markers → markdown | fallback → text

### Current status

- `@axel` queries: working (streaming, markdown rendered, thinking blocks stripped)
- Regular shell commands: working ✅ (verified 2026-04-03)
- `cd` and stateful commands: should work (persistent shell session)
- Colors: stripped (no TTY; programs detect pipe and suppress ANSI colors — fine for now)
- Aliases from .bashrc: not loaded (non-interactive, non-login bash)

### Next steps

- Verify `ls` works end-to-end after Bun.spawn fix
- Test `cd` state persistence
- Improve text cell rendering (currently line-splits on `\n` — should handle long lines with Pretext wrap)
- Add cwd tracking (post-command `pwd` echo or trap)
- Consider sourcing .bashrc on shell init for aliases
