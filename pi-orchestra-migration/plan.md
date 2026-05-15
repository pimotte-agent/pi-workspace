# Pi Orchestra Migration Plan

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    pi TUI (pim/tmux)                 │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │  User Chat       │  │  /queue, /formalize      │  │
│  │  /formalize X    │  │  Commands, Status View   │  │
│  └────────┬─────────┘  └───────────┬──────────────┘  │
│           │                        │                  │
│           ▼                        ▼                  │
│  ┌─────────────────────────────────────────────────┐ │
│  │          ┌──────────┐  ┌─────────────────────┐   │ │
│  │          │ pi-queue │◄─┤ pi-listeners (polls)│   │ │
│  │          │  (daemon)│  └─────────────────────┘   │ │
│  │          └────┬─────┘                            │ │
│  │               │ writes entries                   │ │
│  │               ▼                                  │ │
│  │          ┌──────────┐                            │ │
│  │          │ Task     │ ──► pi agent --print       │ │
│  │          │ Runner   │                            │ │
│  │          └────┬─────┘                            │ │
│  │               │                                  │ │
│  │  ┌────────────┼────────────┐                    │ │
│  │  │            │            │                    │ │
│  │  ▼            ▼            ▼                    │ │
│  │ ┌────┐   ┌───────┐                                │ │
│  │ │pi- │   │pi-    │                                │ │
│  │ │queue│   │auto- │                                │ │
│  │ │tools│   │formal│                                │ │
│  │ └────┘   └───────┘                                │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  Shared storage (JSONL files):                       │
│  ~/.pi/agent/orchestra-queue/   ← queue entries      │
│  ~/.pi/agent/orchestra-listener-state/ ← listener state│
│  ~/.pi/agent/orchestra-logs/    ← daemon logs        │
└─────────────────────────────────────────────────────┘
                    │
                    ▼ (log file)
              ┌──────────────┐
              │ pim's session│
              │  tail -f     │
              └──────────────┘
```

---

## Part 1: Reuse Existing Packages

Install these npm packages first:

| Package | Purpose | Why |
|---------|---------|-----|
| `pi-subagents` | Subagent orchestration | Multi-step workflows (autoformalization) |

```bash
cd ~/.pi/agent
pi install npm:pi-subagents
```

**No MCP needed.** GitHub tools are native pi tools (registered via `pi.registerTool`).

---

## Part 2: Modular Extensions

Four modular extensions that work together via shared file-based storage. Each is independently loadable. All extensions are developed in the GitHub repository [pimotte-agents/orchestra-pi](https://github.com/pimotte-agents/orchestra-pi).

### Extension 1: `pi-queue` — Core Queue Manager

**Path (repo):** `packages/pi-queue/src/pi-queue.ts`
**Path (install):** `~/.pi/agent/extensions/pi-queue.ts`

The foundation — everything reads from and writes to this queue.

**Storage:** JSONL files in `~/.pi/agent/orchestra-queue/`

**Command handler:** `pi.registerCommand("queue", { handler: async (args: string, ctx) => {...} })` — subcommands parsed manually from the `args` string parameter. Output via `ctx.ui.setWidget("queue", lines)` for lists and `ctx.ui.notify()` for single-line messages.

**QueueEntry:**
```typescript
interface QueueEntry {
  id: string;
  createdAt: string;
  status: 'pending' | 'running' | 'done' | 'failed' | 'cancelled';
  paused: boolean;
  priority: number;          // higher = more important (default: 10)
  source: 'manual' | 'github-issues' | 'github-comments'
        | 'github-pr-reviews' | 'autoformalize';
  repo?: string;             // owner/repo
  prompt: string;
  model?: string;        // if omitted, uses the active pi session model
  agent?: string;
  maxRetries?: number;
  retriesRemaining?: number;
  qualityGate?: {            // for autoformalization
    maxIterations: number;
    currentIteration: number;
    remainingTodoCount?: number;
    isFormalization?: boolean;
    targetLanguage?: string;
  };
  validationScript?: string;
  taskId?: string;           // for continuation
  parentId?: string;         // chain linking (autoformalization)
  authSource?: string;
  series?: string;
  mode?: 'fork' | 'pr';
  tools?: string[];          // additional tools to enable
  issueNumber?: number;      // linked GitHub issue/PR
}
```

**TUI Commands (single handler, subcommands parsed from args string):**
- `/queue list [status]` — list entries, filter by status
- `/queue show <id>` — show full details
- `/queue add <prompt>` — add task (`--priority N`, `--repo`, `--model`)
- `/queue pause <id>` — skip in scheduling
- `/queue resume <id>` — unpause
- `/queue cancel <id>` — cancel (cascades to dependents)
- `/queue retry <id>` — retry failed task
- `/queue stats` — summary counts by status/source

**Command API:** `pi.registerCommand("queue", { handler: async (args: string, ctx) => { ... } })`
- `args` is raw string after `/queue`, e.g. `"list pending"`
- Use `ctx.ui.notify()` for single-line output
- Use `ctx.ui.setWidget("queue", lines)` for multi-line lists

**Default list ordering (`/queue list`):
Sorted by execution order: running tasks first, then pending by priority (highest first), ties broken by oldest queued.

**Daemon (runs within Pi's process via `setInterval`):**
- Managed by `QueueManager.startDaemon()` / `stopDaemon()`
- Polls every 1 second using internal `setInterval`
- Picks highest-priority pending (non-paused) entry
- Executes via `executeEntry()` hook (overridable)
- On completion: marks done, emits event
- On failure: decrements retries, re-queue or mark `failed`
- Cascade cancellation: dependents cancel when parent fails/cancels
- Starts on `session_start` event, stops on `session_shutdown`

**Events (for other extensions):**
Writes to `~/.pi/agent/orchestra-queue/events.jsonl`:
```json
{"event":"task_done","id":"...","status":"done"}
{"event":"task_failed","id":"...","reason":"..."}
{"event":"task_paused","id":"..."}
{"event":"daemon_started","pid":"..."}
{"event":"daemon_stopped"}
```

**Unit Tests:**
- `packages/pi-queue/tests/queue.test.ts` — entry CRUD, JSON serialize/deserialize
- `packages/pi-queue/tests/scheduler.test.ts` — priority ordering, stale cleanup, cascade cancel
- `packages/pi-queue/tests/daemon.test.ts` — polling loop, daemon lifecycle

---

### Extension 2: `pi-github-tools` — Native GitHub Tools

**Path (repo):** `packages/pi-github-tools/src/pi-github-tools.ts`
**Path (install):** `~/.pi/agent/extensions/pi-github-tools.ts`

GitHub API calls exposed as **native pi tools** (not MCP). Registered via `pi.registerTool`.

**Native Tools (exposed to agent):**
- `health` — check GitHub API connectivity
- `get_pr_comments` — fetch review threads for a PR
- `create_pr` — create a PR on upstream repo (target repo specified in args)
- `comment` — post comments (target repo specified in args):
  - Regular comment (`body`)
  - PR review (`body` + `review: true`)
  - Reply to inline (`body` + `reply_to_comment_id`)
  - New inline (`body` + `path` + `line`)
- `refresh_token` — refresh GitHub App token (internal)

**Internal functions (not exposed to agent):**
- Used by `pi-listeners` for polling GitHub (issues, comments, reviews)
- Used by `pi-queue` daemon for PR operations
- These are private utility functions, no tool registration

**Auth:** Uses existing GitHub OAuth/token
**Config:** `~/.pi/agent/orchestra-github.json`:
```json
{
  "github_app": { "app_id": "...", "private_key_path": "...", "installation_id": "..." },
  "github": { "pat": "github_pat_..." },
  "authorized_users": ["pimotte"]
}
```

**Unit Tests:**
- `packages/pi-github-tools/tests/client.test.ts` — API request/response handling
- `packages/pi-github-tools/tests/tools.test.ts` — tool input validation, error handling
- `packages/pi-github-tools/tests/auth.test.ts` — token refresh, app auth flow

---

### Extension 3: `pi-listeners` — GitHub Listener Pollers

**Path (repo):** `packages/pi-listeners/src/pi-listeners.ts`
**Path (install):** `~/.pi/agent/extensions/pi-listeners.ts`

Polls GitHub for events and enqueues tasks into pi-queue. Uses internal functions from `pi-github-tools`.

**Sources (polling):**
- `github-issues` — `GET /repos/{owner}/{repo}/issues?state=open`
- `github-comments` — `GET /repos/{owner}/{repo}/issues/comments?since={lastChecked}`
- `github-pr-reviews` — `GET /repos/{owner}/{repo}/pulls/{number}/reviews`

**Per-listener configs** in `~/.pi/agent/orchestra-listeners/` (one config per listener, each can watch multiple repos):
```json
{
  "name": "my-repo-comments",
  "repos": [
    { "upstream": "owner1/repo1", "fork": "pimotte/repo1" },
    { "upstream": "owner2/repo2", "fork": "pimotte/repo2" }
  ],
  "trigger": "@orchestra",
  "authorizedUsers": ["pimotte"],
  "labels": [],
  "action": {
    "mode": "pr",
    "promptTemplate": "Comment on issue/PR #{{issue_number}} by {{author}}:\n\n{{body}}\n\nURL: {{url}}",
    "priority": 10,
    "repo": "{{upstream}}",
    "series": "issue-{{issue_number}}"
  },
  "intervalSeconds": 120
}
```

**State** in `~/.pi/agent/orchestra-listener-state/`:
```json
{
  "lastChecked": "2026-05-15T09:00:00Z",
  "processedIds": ["owner/repo:123", "owner/repo:456"],
  "enabled": true
}
```

**Behavior:**
- Loaded as independent extension on session_start
- Each listener polls at configured interval (internal `setInterval`)
- On event match: enqueues task into pi-queue via `events.jsonl` or direct API access
- Dedup via processedIds (prefixed with repo slug)
- github-comments: reacts with 🚀 (best-effort)
- TUI: `/listeners` (single handler, subcommands parsed from `args`), `/listeners list`, `/listeners enable <name>`, `/listeners disable <name>`

**Unit Tests:**
- `packages/pi-listeners/tests/config.test.ts` — config parsing, validation
- `packages/pi-listeners/tests/state.test.ts` — state file I/O, enable/disable
- `packages/pi-listeners/tests/poller.test.ts` — event matching, dedup logic
- `packages/pi-listeners/tests/templating.test.ts` — template variable rendering

---

### Extension 4: `pi-autoformalize` — Autoformalization Loop

**Path (repo):** `packages/pi-autoformalize/src/pi-autoformalize.ts`
**Path (install):** `~/.pi/agent/extensions/pi-autoformalize.ts`

Orchestrates: formalize → review (list TODOs) → implement TODOs → repeat.

**Trigger:** `/formalize <prompt>` or `/formalize <worksheet-file>`

**Workflow:**
```
1. Creates queue entry with qualityGate configured
2. Formalization subagent:
   → Takes natural language, reads repo instructions
   → Produces formalization artifact
3. Quality gate (parallel):
   a) Scripted: run .agent/validation.sh (non-zero exit → fail check)
   b) Review subagent (pi-subagents):
      → Reads formalization + existing worksheets from repo
      → Lists remaining TODOs (incomplete proofs, style issues, gaps)
      → Returns numbered list of actionable items (not a score)
4. Decision:
   no TODOs & scripted checks pass → DONE
   TODOs remain & retries left → re-queue with TODO list
   TODOs remain & no retries → FAIL
```

**Quality gate:**
- **Scripted checks:** Validation script (`.agent/validation.sh` or project-configured)
- **Review:** Uses pi-subagents — a reviewer subagent that lists TODOs:
  - Style consistency with source material
  - Coverage of original intent
  - Proof completeness (TODOs like "fill in this sorry", "prove this lemma")
  - Structural issues
- **Max iterations:** Default 5, configurable
- **TODO format:** Each TODO is a numbered item the implementer must address

**TUI Commands (single handler, subcommands parsed from `args`):**
- `/formalize <prompt>` — start loop
- `/formalize resume --add-retries N` — reset max iterations
- `/formalize status` — show current loop state + current TODO list
- `/formalize cancel` — abort
- `/formalize skip` — skip current TODOs, move to next iteration

**Integration with pi-queue:**
- Creates entries with `source: 'autoformalize'`, `qualityGate` set
- On iteration: re-queues with `parentId` linking back, prompt includes TODO list
- Review TODOs are appended to the prompt of the next iteration:
  `"TODO list from previous review:\n1. ...\n2. ...\n\nImplement these fixes."`
- On completion: writes events.jsonl entry with final state

**Unit Tests:**
- `packages/pi-autoformalize/tests/loop.test.ts` — iteration lifecycle, max iteration enforcement
- `packages/pi-autoformalize/tests/quality-gate.test.ts` — scripted check parsing, TODO list handling
- `packages/pi-autoformalize/tests/integration.test.ts` — queue interaction, parent/child linking

---

### Extension Interaction Map

```
┌──────────────┐  writes entries   ┌──────────────┐
│ pi-listeners │ ───────────────► │              │
│ (polls GH)   │                  │   pi-queue   │
└──────────────┘                  │   (core)     │
┌────────────────────┐  writes   │              │
│ pi-autoformalize   │ ─────────► │              │
│ (loop controller)  │           │              │
└────────────────────┘           │              │
                                 │              │
┌──────────────┐  native tools │┌┤              │
│ pi-github-   │ ─────────────►││ Task Runner    │
│ tools        │ (registerTool)││ (runs pi --print)│
└──────────────┘               └──────────────────┘
```

**Key point:** pi-queue is the hub. Everything writes entries to it or reads events from it. GitHub tools are **native pi tools** (no MCP). Extensions are independent — enable/disable any combination.

---

## Part 3: Repository Structure

```
orchestra-pi/                          ← GitHub: github.com/pimotte-agents/orchestra-pi
├── packages/
│   ├── pi-queue/
│   │   ├── src/pi-queue.ts            # Main extension
│   │   ├── tests/queue.test.ts
│   │   ├── tests/scheduler.test.ts
│   │   ├── tests/daemon.test.ts
│   │   └── package.json
│   ├── pi-github-tools/
│   │   ├── src/pi-github-tools.ts
│   │   ├── tests/client.test.ts
│   │   ├── tests/tools.test.ts
│   │   ├── tests/auth.test.ts
│   │   └── package.json
│   ├── pi-listeners/
│   │   ├── src/pi-listeners.ts
│   │   ├── tests/config.test.ts
│   │   ├── tests/state.test.ts
│   │   ├── tests/poller.test.ts
│   │   ├── tests/templating.test.ts
│   │   └── package.json
│   └── pi-autoformalize/
│       ├── src/pi-autoformalize.ts
│       ├── tests/loop.test.ts
│       ├── tests/quality-gate.test.ts
│       ├── tests/integration.test.ts
│       └── package.json
├── examples/
│   ├── listeners/                     # Example listener configs
│   └── workflows/                     # Example workflow YAMLs
├── install/                           # Install helpers
│   └── install.sh                     # Copies extensions to ~/.pi/agent/extensions/
├── .agent/
│   └── validation.sh                  # Default validation (for testing)
├── package.json                       # Root workspace config
└── README.md
```

**Install from repo:**
```bash
git clone https://github.com/pimotte-agents/orchestra-pi.git
cd orchestra-pi
./install.sh
# Copies each extension to ~/.pi/agent/extensions/
```

**Test from repo:**
```bash
cd packages/pi-queue && pnpm test     # Run tests for a package
pnpm test                              # Run all tests (workspaces)
```

---

## Part 4: Implementation Order

### Phase 1: Core Queue (`pi-queue`)
1. Queue storage (JSONL files, entry structure)
2. Daemon (scheduling, priority, retries, cascade cancel) — managed by QueueManager via `setInterval`
3. TUI commands (`/queue` — single handler with subcommands parsed from `args: string`)
4. Event logging (events.jsonl for other extensions)
5. **Tests:** queue CRUD, scheduler priority ordering, stale cleanup, cascade cancel, daemon lifecycle

### Phase 2: GitHub Tools (`pi-github-tools`)
1. GitHub API client (internal functions for polling + PR operations)
2. Native tool registration (`create_pr`, `comment`, `get_pr_comments`, `health`)
3. Auth config (`~/.pi/agent/orchestra-github.json`), authorized_users = ["pimotte"]
4. **Tests:** API request/response, tool validation, auth flow, multi-repo routing

### Phase 3: Listeners (`pi-listeners`)
1. Listener configs + state management
2. Listener poller as independent extension (own `setInterval`, starts on `session_start`)
3. Uses internal functions from pi-github-tools
4. Enqueues into pi-queue via direct API access or `events.jsonl`
5. Authorized user checking (pimotte), trigger words, dedup
6. TUI: `/listeners` (single handler with subcommands parsed from `args`)
7. **Tests:** config parsing, state I/O, event matching, dedup, template rendering

### Phase 4: Autoformalization (`pi-autoformalize`)
1. Loop controller + quality gate
2. Integration with pi-subagents
3. Scripted validation execution (uses whatever is in the repo's validation script)
4. Human-in-the-loop commands (`/formalize resume`, `skip`, `status`)
5. Event emission for notifications
6. **Tests:** iteration lifecycle, scripted check parsing, TODO list handling, queue integration

### Phase 5: Polish
1. Log file setup (shared dir for `pim` observation)
2. TUI polish (status display, queue overview)
3. Config management and defaults
4. Documentation
5. Install script + example configs

---

## Part 5: Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Queue backend | JSONL files in `~/.pi/agent/orchestra-queue/` | Simple, inspectable, no DB |
| Daemon | setInterval within Pi process | Runs inside Pi's event loop |
| Extension style | Modular, independently loadable | Flexibility, testability |
| Extension communication | Shared files (events.jsonl) | Simple, no IPC complexity |
| GitHub integration | Native pi tools (`registerTool`) | No MCP dependency, simpler |
| GitHub API calls | Internal functions for listeners | Private, not exposed to agent |
| GitHub auth | Existing token, multi-repo | Supports any number of repos |
| Autoformalization | Subagent-based: formalize → review (TODOs) → implement → repeat | Leverages pi-subagents |
| Listener poller | Embed in daemon | Simple, no external webhook |
| Daemon output | Log file → `tail -f` from `pim` | Cross-user, observable |
| Notifications | Dropped | Not needed |
| Web search | Dropped | Can add later |
| Default model | Uses active pi session model | No hardcoded default |
| Validation scripts | Already in repos, resolved per-repo at runtime | |
| elan/Lake | Available, used only for validation | No Lean-specific logic in extensions |
| Testing | Unit tests per package, run with pnpm | Ensures correctness before install |
| Command API | Single handler per command, subcommands parsed from `args: string` | Pi API doesn't support `subcommands` |
| Distribution | GitHub repo → install script → `~/.pi/agent/extensions/` | Simple, version-controlled |

---

## Part 6: File Structure (Installed)

```
~/.pi/agent/
├── extensions/
│   ├── pi-queue.ts              # Queue daemon + commands
│   ├── pi-github-tools.ts       # GitHub API + native tools
│   ├── pi-listeners.ts          # Listener pollers
│   └── pi-autoformalize.ts      # Formalization loop
├── orchestra-queue/             # Queue storage
│   ├── daemon.pid
│   ├── <id>.json                # Queue entries
│   └── events.jsonl             # Events for extensions
├── orchestra-listeners/         # Listener configs
│   ├── my-repo-comments.json
│   └── ...
├── orchestra-listener-state/    # Listener state
│   ├── my-repo-comments.json
│   └── ...
├── orchestra-logs/              # Log files (visible from pim)
│   ├── daemon.log
│   └── listeners.log
├── orchestra-github.json        # GitHub config
└── AGENTS.md                    # Project instructions
```

---

## Ready to Build

All decisions confirmed:

- ✅ **Multi-repo support** — Listeners and tools handle arbitrary repo pairs
- ✅ **Username**: `pimotte`
- ✅ **Default model**: Active pi session model (no hardcoded default)
- ✅ **Validation scripts**: Already in repos, resolved per-repo at runtime
- ✅ **elan/Lake**: Available, used only for validation execution (no Lean-specific logic in extensions)
- ✅ **No MCP** — native pi tools
- ✅ **No notifications** — dropped
- ✅ **No web search** — dropped for now
- ✅ **No score-based quality** — TODO-based review loop
- ✅ **Modular extensions** — 4 independent modules
- ✅ **Repository**: github.com/pimotte-agents/orchestra-pi
- ✅ **Unit tests**: One test file per major module, run with pnpm

Plan is ready. Shall I start building Phase 1 (`pi-queue`)?
