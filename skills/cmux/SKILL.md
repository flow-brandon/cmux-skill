---
name: cmux
description: >
  Control the cmux macOS terminal (Ghostty-based, agent-aware) via its CLI and Unix-socket API —
  workspaces, panes, surfaces, sending input to and monitoring parallel agent sessions, sidebar
  status/progress, notifications, and embedded browser panes. Use when the user names cmux (a cmux
  workspace, pane, surface, or an agent running in cmux), when spawning or monitoring visible
  parallel agent sessions in cmux, or when a long-running task inside cmux should report progress
  to the sidebar. Do NOT trigger on generic mentions of "workspace", "pane", "tab", or "agent" when
  cmux is not named — tmux, VS Code, Ghostty, and Claude Code's in-process subagents are NOT cmux.
  macOS only (14.0+).
---

# cmux

cmux is a native macOS terminal for running AI coding agents in parallel. Everything in its UI is scriptable through the `cmux` CLI (backed by a Unix socket at `/tmp/cmux.sock`): create workspaces, split panes, launch agents, type into their terminals, read their screens, set sidebar status, and drive embedded browser panes.

## Concepts and handles

```
Window (macOS window)
  └─ Workspace   — sidebar entry; one project / branch / task context (UI calls it a "tab")
       └─ Pane   — split region inside a workspace
            └─ Surface — a tab inside a pane: terminal, browser, or agent session
```

Commands identify targets by **short refs**: `window:1`, `workspace:2`, `pane:3`, `surface:4`. UUIDs are accepted as input; add `--id-format uuids|both` if you need UUID output.

**Ref gotchas — get these right or fail silently:**

- Always use **prefixed refs** (`surface:46`). A bare number means *index*, not id — `--surface 46` targets the surface at index 46, which usually doesn't exist and fails silently.
- `read-screen` / `capture-pane` have **no `--pane` flag** — they take `--workspace` or `--surface` only. With no target they read **your own** surface (you'll read your own prompt and draw wrong conclusions). To read a pane, resolve it first: `cmux list-pane-surfaces --pane pane:N`, then `cmux read-screen --surface surface:N`.
- There is **no `send-surface` command** — target with a flag: `cmux send --surface surface:N "text"`.
- **Never append `2>/dev/null`** to cmux commands. Errors go to stderr with exit code 1; suppressing them is the #1 cause of mystery "(no output)" failures.

## Detect cmux, degrade gracefully

Every cmux-spawned terminal has env vars injected: `CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID`, `CMUX_SOCKET_PATH`.

```bash
command -v cmux >/dev/null && cmux ping >/dev/null || { echo "not in cmux"; }
cmux identify --json        # who am I: window/workspace/pane/surface context
```

If cmux isn't available, skip all cmux enhancements silently — never block the main task on them. (CLI symlink, if a machine lacks it: `sudo ln -sf "/Applications/cmux.app/Contents/Resources/bin/cmux" /usr/local/bin/cmux`.)

## Golden rules (non-disruptive automation)

1. **Scope to the caller workspace.** Anchor every command to `"$CMUX_WORKSPACE_ID"` / `"$CMUX_SURFACE_ID"`, not to whatever is visually focused — the user may be looking at a different workspace than the one you're running in. Fall back to `cmux identify --json` only if the env vars are missing, and say so.
2. **Never steal focus.** `select-workspace`, `focus-pane`, `focus-panel`, and focus-changing `tab-action` verbs are user-affecting, like clicking their mouse. Don't call them speculatively; pass `--focus false` wherever a creation/move verb supports it.
3. **Build layout additively, in one shot.** Prefer `new-pane --type ... --direction ... --url ...` (creates the pane already populated) over create-then-move-then-focus chains. If a command rejects a valid ref, report it — don't "fix" it by changing focus.
4. **Reuse one helper pane.** For auxiliary output (previews, logs, one-off shells), keep a single helper pane to the right of the caller terminal; repeated "open X" requests add surfaces (tabs) inside it rather than new splits. Inspect first with `cmux list-panes` / `cmux list-pane-surfaces`.
5. **Don't touch other agents' surfaces** — no keystrokes, closes, or focus changes in a workspace the user didn't name. Leave worker surfaces open when done; close only on explicit cleanup.

## Topology fast start

```bash
cmux tree                                                    # full hierarchy at a glance
cmux list-workspaces --json
cmux new-workspace --name "feature-x" --cwd /path/to/repo    # also: --command "claude ..."
cmux new-pane  --workspace "$CMUX_WORKSPACE_ID" --type terminal --direction right --focus false
cmux new-pane  --workspace "$CMUX_WORKSPACE_ID" --type browser  --direction down --url http://localhost:3000
cmux new-surface --pane pane:2 --type terminal --focus false # tab inside an existing pane
cmux rename-workspace "auth-refactor"                        # positional title
cmux workspace-action --action set-color --color Blue        # color, description, pin, ...
cmux move-tab-to-new-workspace --surface surface:7           # promote a tab to its own workspace
cmux close-surface --surface surface:7
```

## Send input and read screens

```bash
cmux send --surface surface:7 "npm test"     # type into a specific terminal
cmux send-key --surface surface:7 enter      # enter|tab|escape|backspace|arrows|ctrl+c|...
cmux read-screen --surface surface:7                   # what's on its screen now
cmux read-screen --surface surface:7 --scrollback --lines 200
```

Two-step send (text, then `send-key enter`) is the reliable way to submit a command or a prompt to an interactive TUI like Claude Code.

## Make your work visible (sidebar conventions)

Any task longer than ~10 seconds should report to the sidebar so humans can glance instead of switching:

```bash
cmux set-status phase "building" --icon hammer --color "#ff9500"   # status pill
cmux set-progress 0.5 --label "Running tests..."                   # progress bar (0.0–1.0)
cmux log --level success "All 42 tests passed"                     # info|progress|success|warning|error
cmux notify --title "Review needed" --body "auth-refactor: tests green, diff ready"
cmux trigger-flash --workspace "$CMUX_WORKSPACE_ID"                # blue-ring attention cue
cmux sidebar-state --json                                          # read everything back
```

Convention: `set-status phase <verb>` while working, `set-progress` for anything with known steps, `notify` **only** at completion or when blocked on a human — notifications interrupt; status pills don't. Clear your pills/progress (`clear-status`, `clear-progress`) when done.

## Orchestrating worker agents

One session acts as controller; workers run visibly. This is for *visible, terminal-level* parallelism — for invisible parallelism inside one session, use the harness's normal subagents instead.

**Placement rule — panes first, workspaces second.** A workspace encapsulates one "thing" the user is working on. Workers that fan out from that thing belong in **panes of the caller workspace**, so the user sees all of them at once without clicking around. Reach for a **new workspace** only when the work is genuinely independent of what the user is looking at (its own branch/worktree/repo and lifecycle), or when the fan-out would cramp the pane grid (more than ~4 workers) — and then group related workers into *one* new workspace, not one workspace each.

```bash
# 1a. Default: fan workers out as panes in the CALLER workspace — all visible at once
cmux new-pane --workspace "$CMUX_WORKSPACE_ID" --type terminal --direction right --focus false
cmux new-pane --workspace "$CMUX_WORKSPACE_ID" --type terminal --direction down --focus false
cmux list-pane-surfaces --workspace "$CMUX_WORKSPACE_ID"   # capture each worker's surface ref
cmux send --surface surface:12 "cd ~/src/repo-worktree-a && claude --permission-mode acceptEdits"
cmux send-key --surface surface:12 enter

# 1b. Independent unit of work only: its own named workspace
cmux new-workspace --name "worker: auth-middleware" --cwd ~/src/repo-worktree-a \
  --command "claude --permission-mode acceptEdits"

# 2. Give it its task (find its surface ref via: cmux list-workspaces, cmux list-pane-surfaces)
cmux send --surface surface:12 "Implement the auth middleware per docs/specs/auth.md"
cmux send-key --surface surface:12 enter

# 3. Monitor without interrupting — poll every 15-60s depending on task size
cmux read-screen --surface surface:12 --lines 40

# 4. Follow up / course-correct the same way you started it
cmux send --surface surface:12 "now add integration tests for the auth routes"
cmux send-key --surface surface:12 enter
```

Controller invariants (from hard experience):
- Drive workers through **explicit `surface:N` refs** captured at spawn time; never "the focused one".
- Prefer **bounded, plan-first delegations** — have the worker present a plan or diff you can `read-screen` before letting it land changes.
- Verify worker claims yourself (`git -C <worktree> diff`, run the tests) before reporting success upward.
- Give parallel workers **separate git worktrees or repos** — cmux gives them separate terminals, not separate filesystems.
- When a worker finishes or stalls, say so and leave its workspace open for human inspection.

**Native alternatives — prefer these when they fit:**
- `cmux hooks setup` (once, per machine) wires agent lifecycle events into cmux natively: sidebar state, unread badges, and the live event feed (`cmux feed tui`, `cmux events` for scripts). Much better than screen-polling for "is it done / does it need me".
- `cmux claude-teams [claude-args...]` launches Claude Code with Agent Teams wired in — teammate agents render as native cmux splits automatically, no manual orchestration needed.
- `cmux diff --last-turn --workspace workspace:N` opens a diff viewer for what an agent just changed — review without interrupting it.

## Browser panes (preview and verification)

When a dev server starts (`localhost:PORT` in output) or you produce an HTML file worth seeing rendered, offer/open a preview split instead of telling the user to open a browser:

```bash
cmux browser open http://localhost:3000                # browser split in caller's workspace
```

For verification/QA, the `cmux browser` group is a full driver (target a surface; workflow: **open → wait → snapshot → act → re-snapshot**):

```bash
cmux browser surface:9 wait --load-state complete --timeout-ms 15000
cmux browser surface:9 snapshot --interactive --compact   # accessibility tree with element refs
cmux browser surface:9 fill "#email" --text "qa@example.com"
cmux browser surface:9 click "button[type='submit']" --snapshot-after
cmux browser surface:9 wait --text "Welcome"
cmux browser surface:9 console list                       # plus: errors list, screenshot --out, eval
```

## When a command doesn't match this file

cmux evolves quickly and this file will drift. The installed binary is authoritative:

```bash
cmux --help              # full command list
cmux capabilities        # socket methods available in this build
cmux docs api            # fetch current API docs
```

If a documented flag or command errors, check `cmux --help` for the current spelling before working around it — and update this skill.
