# NanoClaw

Personal Claude assistant that connects to WhatsApp and runs agents in isolated Linux containers via the Claude Agent SDK. See [README.md](README.md) for philosophy and [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for design decisions.

## Architecture Overview

Single Node.js process (host) that:
1. Connects to WhatsApp via Baileys
2. Stores messages in SQLite
3. Polls for new messages on a 2-second loop
4. Spawns Docker containers running Claude Agent SDK for each conversation
5. Routes agent responses back to WhatsApp

Each group gets an isolated container with its own filesystem, session, and CLAUDE.md memory.

```
WhatsApp (Baileys) --> SQLite --> Polling loop --> Docker container (Claude Agent SDK) --> Response
                                                        |
                                                  IPC (filesystem) <--> Host process
```

## Project Structure

```
nanoclaw/
├── src/                        # Host process source (TypeScript, ESM)
│   ├── index.ts                # Main orchestrator: state, message loop, agent invocation
│   ├── config.ts               # Constants, paths, trigger pattern, env config
│   ├── db.ts                   # SQLite database (messages, groups, tasks, sessions, state)
│   ├── router.ts               # Message formatting (XML) and outbound routing
│   ├── ipc.ts                  # IPC watcher: processes messages/tasks from container agents
│   ├── container-runner.ts     # Spawns Docker containers with volume mounts, streaming output
│   ├── container-runtime.ts    # Runtime abstraction (docker binary, start/stop, orphan cleanup)
│   ├── group-queue.ts          # Per-group queue with global concurrency limit (max 5)
│   ├── task-scheduler.ts       # Scheduled task execution loop (60s poll)
│   ├── team-manager.ts         # Agent Swarm/Teams support (inbox-based coordination)
│   ├── mount-security.ts       # Validates additional mounts against external allowlist
│   ├── dashboard.ts            # HTTP dashboard on localhost:3456
│   ├── logger.ts               # Pino logger with pretty printing
│   ├── env.ts                  # .env file parser (does NOT load into process.env)
│   ├── types.ts                # Shared TypeScript interfaces
│   ├── whatsapp-auth.ts        # Standalone WhatsApp QR auth script
│   └── channels/
│       └── whatsapp.ts         # WhatsApp channel: connect, send/receive, typing, group sync
├── container/                  # Agent container
│   ├── Dockerfile              # Node 22 + Chromium + claude-code + agent-browser
│   ├── build.sh                # Build script (uses CONTAINER_RUNTIME env var)
│   └── agent-runner/           # Code that runs inside the container
│       ├── src/
│       │   ├── index.ts        # Agent runner: reads stdin JSON, calls Claude Agent SDK
│       │   └── ipc-mcp-stdio.ts # MCP server providing send_message, schedule_task, etc.
│       ├── package.json        # Dependencies: claude-agent-sdk, mcp-sdk, cron-parser, zod
│       └── tsconfig.json
├── groups/                     # Per-group data (each folder = one group)
│   ├── main/CLAUDE.md          # Main channel memory and admin instructions
│   └── global/CLAUDE.md        # Shared read-only memory for all non-main groups
├── store/                      # Persistent data (gitignored)
│   ├── messages.db             # SQLite database
│   └── auth/                   # WhatsApp auth state (Baileys multi-file)
├── data/                       # Runtime data (gitignored)
│   ├── ipc/{group}/            # Per-group IPC: messages/, tasks/, input/
│   └── sessions/{group}/.claude/ # Per-group Claude session data
├── docs/                       # Documentation
│   ├── REQUIREMENTS.md         # Architecture decisions and philosophy
│   ├── SECURITY.md             # Security model and trust boundaries
│   ├── SPEC.md                 # Technical specification
│   └── DEBUG_CHECKLIST.md      # Debugging guide
├── .claude/skills/             # Claude Code skills (slash commands)
│   ├── setup/                  # /setup - First-time installation
│   ├── customize/              # /customize - Add capabilities
│   ├── debug/                  # /debug - Troubleshooting
│   ├── add-telegram/           # /add-telegram
│   ├── add-discord/            # /add-discord
│   ├── add-gmail/              # /add-gmail
│   ├── add-voice-transcription/# /add-voice-transcription
│   ├── convert-to-apple-container/ # /convert-to-apple-container
│   └── x-integration/          # /x-integration
├── package.json                # Host dependencies
├── tsconfig.json               # TypeScript config (ES2022, NodeNext, strict)
└── vitest.config.ts            # Test config
```

## Key Concepts

### Message Flow
1. WhatsApp messages arrive via Baileys websocket (`channels/whatsapp.ts`)
2. All chat metadata is stored; full message content only for registered groups
3. `index.ts` polling loop detects new messages every 2 seconds
4. Non-main groups require trigger word (`@Andy` by default) to activate
5. Messages are formatted as XML: `<messages><message sender="..." time="...">content</message></messages>`
6. `GroupQueue` manages concurrency (max 5 simultaneous containers)
7. Container is spawned with group-specific mounts via `container-runner.ts`
8. Agent responses stream back via stdout markers (`---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`)
9. Responses are sent back to WhatsApp, stripping `<internal>` tags

### Container Isolation
- Each agent runs in a Docker container (`nanoclaw-agent:latest`)
- Main group gets the full project mounted at `/workspace/project` (read-write)
- Non-main groups only get their own folder at `/workspace/group`
- IPC is namespaced per-group at `/workspace/ipc/`
- Sessions are isolated per-group at `/home/node/.claude/`
- Additional mounts validated against external allowlist at `~/.config/nanoclaw/mount-allowlist.json`

### IPC Protocol
Containers communicate with the host via filesystem-based IPC:
- **Outbound messages**: Container writes JSON to `/workspace/ipc/messages/`
- **Task operations**: Container writes JSON to `/workspace/ipc/tasks/`
- **Follow-up messages**: Host writes to `/workspace/ipc/input/` (container polls every 500ms)
- **Close signal**: Host writes `_close` sentinel to `/workspace/ipc/input/`
- Host polls IPC directories every 1 second (`IPC_POLL_INTERVAL`)

### Secrets Handling
- Secrets (`ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`) read from `.env` via `env.ts`
- Passed to containers via stdin JSON, never written to disk or environment
- `env.ts` intentionally does NOT load into `process.env` to prevent leaking to child processes
- Agent runner's `PreToolUse` hook strips secret env vars from Bash subprocess commands

### Database Schema (SQLite at `store/messages.db`)
- `chats` - Chat metadata (jid, name, last_message_time, channel, is_group)
- `messages` - Message content (id, chat_jid, sender, content, timestamp, is_bot_message)
- `scheduled_tasks` - Recurring/one-time tasks with cron/interval/once types
- `task_run_logs` - Task execution history with duration and status
- `router_state` - Key-value state (last_timestamp, last_agent_timestamp)
- `sessions` - Claude session IDs per group folder
- `registered_groups` - Group config (jid, name, folder, trigger, container_config, requires_trigger)

### Scheduled Tasks
- Created via `schedule_task` MCP tool inside containers
- Types: `cron` (recurring), `interval` (every N ms), `once` (one-time)
- Context modes: `group` (inherits chat session) or `isolated` (fresh session)
- Scheduler loop in `task-scheduler.ts` checks for due tasks every 60 seconds
- Tasks run as full container agents with all tools available

### Agent Swarms (Teams)
- `team-manager.ts` manages teams of specialized agents
- Teams defined by config files in `data/sessions/main/.claude/teams/{team}/config.json`
- Each team member has an inbox (JSON file) polled every 5 seconds
- Members process messages by spawning container agents
- Responses forwarded to team lead and to WhatsApp

## Development

### Commands
```bash
npm run dev          # Run with hot reload (tsx)
npm run build        # Compile TypeScript (tsc)
npm run start        # Run compiled output (node dist/index.js)
npm run typecheck    # Type-check without emitting
npm run test         # Run tests (vitest)
npm run test:watch   # Watch mode tests
npm run format       # Format with prettier
npm run format:check # Check formatting
./container/build.sh # Build agent container image
```

### Service Management (macOS)
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

### Testing
- Framework: Vitest
- Test files: `src/**/*.test.ts` (co-located with source)
- Skills tests: `.claude/skills/**/tests/*.test.ts` (separate config: `vitest.skills.config.ts`)
- Database tests use in-memory SQLite via `_initTestDatabase()`
- Container/channel tests mock dependencies with `vi.mock()`
- Run: `npm test` or `vitest run src/db.test.ts` for a specific file

### TypeScript Conventions
- ESM modules (`"type": "module"` in package.json)
- All imports use `.js` extension (required for NodeNext module resolution)
- Target: ES2022, module: NodeNext, strict mode enabled
- Zod v4 for runtime validation (in container MCP server)

### Code Conventions
- Logging: Use `logger` from `./logger.js` (pino), not `console.log`
- Error handling: Log with structured context: `logger.error({ err, groupName }, 'message')`
- State: Application state in module-level variables in `index.ts`, persisted to SQLite
- Testing internals: Prefix exported test helpers with `_` (e.g., `_initTestDatabase`)
- Channels: Implement the `Channel` interface from `types.ts`
- Atomic file writes: Write to `.tmp` then `fs.renameSync()` to final path
- Singleton guards: Subsystems track a `running` boolean flag to prevent duplicate starts
- Run commands directly for the user; don't tell them to run commands

### Environment Variables
| Variable | Default | Description |
|----------|---------|-------------|
| `ASSISTANT_NAME` | `Andy` | Trigger word and response prefix |
| `ASSISTANT_HAS_OWN_NUMBER` | `false` | Bot has its own dedicated phone number |
| `CONTAINER_IMAGE` | `nanoclaw-agent:latest` | Docker image for agent containers |
| `CONTAINER_TIMEOUT` | `1800000` (30 min) | Hard timeout per container |
| `IDLE_TIMEOUT` | `1800000` (30 min) | Close container after inactivity |
| `MAX_CONCURRENT_CONTAINERS` | `5` | Global concurrency limit |
| `LOG_LEVEL` | `info` | Pino log level (debug, info, warn, error) |
| `TZ` | System timezone | Timezone for cron schedules |

### Container Build Cache
Docker buildkit caches aggressively. `--no-cache` alone does NOT invalidate COPY steps. To force a clean rebuild, prune the builder volume first, then re-run `./container/build.sh`.

The `container/agent-runner/src/` directory is bind-mounted into containers at runtime and recompiled on each container start, so source changes there take effect without rebuilding the Docker image. Only rebuild for system dependency or npm package changes.

## Skills

Skills are Claude Code slash commands that transform the codebase. They live in `.claude/skills/{name}/SKILL.md`.

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/add-telegram` | Add Telegram as a channel |
| `/add-discord` | Add Discord as a channel |
| `/add-gmail` | Add Gmail integration |
| `/add-voice-transcription` | Voice message transcription via Whisper |
| `/convert-to-apple-container` | Switch Docker to Apple Container (macOS) |
| `/x-integration` | X (Twitter) integration |

Skills should transform code, not add configuration. Philosophy: "features as code changes, not config flags."

## Security Model

See [docs/SECURITY.md](docs/SECURITY.md) for the full model. Key points:
- **Container isolation** is the primary security boundary (not application-level checks)
- Main group is trusted (admin); non-main groups are untrusted
- IPC authorization enforced by directory namespacing per group
- Mount allowlist stored outside project root at `~/.config/nanoclaw/mount-allowlist.json`
- Blocked patterns: `.ssh`, `.gnupg`, `.aws`, `.env`, `credentials`, private keys, etc.
- Non-main groups can be forced to read-only mounts via `nonMainReadOnly` allowlist option
- Secrets passed to containers via stdin JSON, stripped from Bash subprocess environments
