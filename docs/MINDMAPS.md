# NanoClaw Mind Maps

A daily reference for understanding how NanoClaw works. Read top-to-bottom — each map builds on the previous one.

---

## Map 1 — Big Picture: What Is NanoClaw?

```
                          ┌─────────────────────────────┐
                          │         NanoClaw             │
                          │   (single Node.js process)  │
                          └─────────────┬───────────────┘
                                        │
          ┌─────────────────────────────┼──────────────────────────────┐
          │                             │                              │
   ┌──────▼──────┐             ┌────────▼────────┐           ┌────────▼───────┐
   │  CHANNELS   │             │   CORE LOOPS    │           │   AGENT LAYER  │
   │             │             │                 │           │                │
   │ • Telegram  │             │ • Message Loop  │           │ • GroupQueue   │
   │ • WhatsApp  │             │   (every 2s)    │           │ • Container    │
   │ • Slack     │             │ • Scheduler     │           │   Runner       │
   │ • Discord   │             │   (every 60s)   │           │ • Agent Runner │
   │ • Gmail     │             │ • IPC Watcher   │           │   (inside      │
   │             │             │   (every 1s)    │           │    container)  │
   └──────┬──────┘             └────────┬────────┘           └────────┬───────┘
          │                             │                              │
          └─────────────────────────────▼──────────────────────────────┘
                                        │
                                ┌───────▼────────┐
                                │  STATE / DATA  │
                                │                │
                                │ • SQLite DB    │
                                │ • Filesystem   │
                                │ • IPC files    │
                                └────────────────┘
```

**Key insight:** Everything is one process. Channels are plugins. The agent runs in an isolated container. They communicate through SQLite (for persistence) and files (for IPC).

---

## Map 2 — Message Flow: User Sends a Message

```
User types "@Sumi what's the weather?"
          │
          ▼
  ┌───────────────┐
  │   Channel     │  (Telegram, WhatsApp, etc.)
  │               │  Receives the raw message
  └───────┬───────┘
          │ onMessage() callback
          ▼
  ┌───────────────┐
  │  storeMessage │  Saves to SQLite messages table
  │  (db.ts)      │  (all messages stored, even non-trigger ones)
  └───────────────┘

          ... 2 seconds pass ...

  ┌───────────────────────────────┐
  │  Message Loop (index.ts)      │
  │  getNewMessages() from SQLite │
  │  Checks every 2 seconds       │
  └───────────────┬───────────────┘
                  │
        ┌─────────▼──────────┐
        │ Is it a registered │ NO → skip
        │ group?             │
        └─────────┬──────────┘
                  │ YES
        ┌─────────▼──────────┐
        │ Does message have  │ NO → store in DB, wait for trigger
        │ "@Sumi" trigger?   │     (accumulates context)
        └─────────┬──────────┘
                  │ YES
        ┌─────────▼──────────┐
        │ Is container       │
        │ already active     │ YES → write message to IPC input file
        │ for this group?    │      (container picks it up in 500ms)
        └─────────┬──────────┘
                  │ NO
                  ▼
  ┌───────────────────────────────┐
  │  GroupQueue                   │
  │  enqueueMessageCheck()        │
  │  Respects MAX_CONCURRENT=5    │
  └───────────────┬───────────────┘
                  │
                  ▼
  ┌───────────────────────────────┐
  │  processGroupMessages()       │
  │  • getMessagesSince() — gets  │
  │    ALL messages since last    │
  │    agent response             │
  │  • formatMessages() → XML     │
  │    <message sender="..." />   │
  └───────────────┬───────────────┘
                  │
                  ▼
  ┌───────────────────────────────┐
  │  runContainerAgent()          │
  │  • docker run nanoclaw-agent  │
  │  • writes JSON to stdin       │
  │  • reads stdout for markers   │
  └───────────────┬───────────────┘
                  │
          (inside container)
                  ▼
  ┌───────────────────────────────┐
  │  agent-runner/src/index.ts    │
  │  • Claude Agent SDK query()   │
  │  • Allowed tools: Bash, Read, │
  │    Write, WebSearch, MCP, ... │
  │  • MCP servers: nanoclaw,     │
  │    notion (if configured)     │
  └───────────────┬───────────────┘
                  │ writes output between markers
                  ▼
  ---NANOCLAW_OUTPUT_START---
  {"status":"success","result":"It's 72°F and sunny!"}
  ---NANOCLAW_OUTPUT_END---
                  │
  ┌───────────────▼───────────────┐
  │  Host parses output marker    │
  │  Strips <internal> blocks     │
  │  channel.sendMessage()        │
  └───────────────┬───────────────┘
                  │
                  ▼
        User receives reply ✓
```

---

## Map 3 — Container System: What's Inside the Sandbox

```
  HOST MACHINE                          CONTAINER (nanoclaw-agent:latest)
  ─────────────────────────────────     ─────────────────────────────────────

  groups/my-group/          ──(rw)───▶  /workspace/group/
    CLAUDE.md                             CLAUDE.md        ← agent's memory
    logs/                                 logs/
    conversations/                        conversations/   ← archived chats

  data/ipc/my-group/        ──(rw)───▶  /workspace/ipc/
    messages/                             messages/        ← agent sends msgs here
    tasks/                                tasks/           ← agent schedules tasks
    input/                                input/           ← host sends msgs here
    current_tasks.json                    current_tasks.json
    available_groups.json                 available_groups.json

  data/sessions/my-group/   ──(rw)───▶
    .claude/                ──(rw)───▶  /home/node/.claude/
      settings.json                       (SDK sessions, skills, memory)
    agent-runner-src/       ──(rw)───▶  /app/src/
      index.ts                            (agent's own code, customizable)

  [main group only]
  project root/             ──(ro)───▶  /workspace/project/
    src/, dist/, ...                      (can read NanoClaw code, not write)
    .env  ──(shadowed)───▶  /dev/null    (secrets never visible)

  groups/global/            ──(ro)───▶  /workspace/global/
    CLAUDE.md                             (shared context, non-main only)

  ─────────────────────────────────     ─────────────────────────────────────
  Credential Proxy :3001
    ↑ container calls ANTHROPIC_BASE_URL=http://host-gateway:3001
    ↑ proxy injects real API key / OAuth token
    ↑ container never sees real secrets
```

**Key insight:** Each group gets its own isolated filesystem. The agent runner code (`/app/src`) is writable per-group, so each group's agent can be customized independently.

---

## Map 4 — IPC Bridge: How Host and Container Talk

```
  HOST                              CONTAINER
  ──────────────────────────────    ─────────────────────────────────

  ── Container → Host (agent wants to send a message) ──────────────

  Container writes:
  /workspace/ipc/messages/abc.json
  { "type": "message", "chatJid": "...", "text": "Hello!" }

  IPC Watcher (ipc.ts) polls every 1s
  → reads file
  → calls channel.sendMessage(chatJid, text)
  → deletes file ✓

  ── Container → Host (agent wants to schedule a task) ─────────────

  Container writes:
  /workspace/ipc/tasks/xyz.json
  { "type": "schedule_task", "targetJid": "...",
    "schedule_type": "cron", "schedule_value": "0 9 * * *",
    "prompt": "Send morning briefing" }

  IPC Watcher → createTask() in SQLite → Scheduler picks it up ✓

  ── Host → Container (new message while container is active) ───────

  GroupQueue.sendMessage() writes:
  data/ipc/{group}/input/1234567890.json
  { "type": "message", "text": "<messages>...</messages>" }

  Container polls /workspace/ipc/input/ every 500ms
  → reads file → deletes it
  → feeds text into active MessageStream → SDK processes it ✓

  ── Host → Container (tell agent to stop) ──────────────────────────

  GroupQueue.closeStdin() writes:
  data/ipc/{group}/input/_close   (empty file)

  Container detects _close sentinel
  → ends MessageStream → query loop exits → container exits ✓
```

---

## Map 5 — Data Model: What's Stored Where

```
  SQLite (store/messages.db)
  ┌─────────────────────┬────────────────────────────────────────────────┐
  │ Table               │ Purpose                                        │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ messages            │ All chat messages from all registered groups   │
  │                     │ Fields: id, chat_jid, sender, content,         │
  │                     │         timestamp, is_from_me, is_bot_message  │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ chats               │ Known groups/chats (metadata only)             │
  │                     │ Fields: jid, name, last_message_time, channel  │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ registered_groups   │ Groups the agent actively monitors             │
  │                     │ Fields: jid, name, folder, trigger_pattern,    │
  │                     │         is_main, requires_trigger              │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ sessions            │ Claude Agent SDK session IDs (conversation     │
  │                     │ memory per group folder)                       │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ scheduled_tasks     │ Recurring/one-time tasks                       │
  │                     │ Fields: id, group_folder, prompt,              │
  │                     │         schedule_type, next_run, status        │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ task_run_logs       │ History of every task execution                │
  ├─────────────────────┼────────────────────────────────────────────────┤
  │ router_state        │ Cursors: last_timestamp, last_agent_timestamp  │
  │                     │ (used to avoid re-processing old messages)     │
  └─────────────────────┴────────────────────────────────────────────────┘

  Two Cursors That Matter:
  ┌──────────────────────────────────────────────────────────────────────┐
  │ last_timestamp        — "what's the newest message I've SEEN?"      │
  │                         Advanced immediately when messages arrive.   │
  │                                                                      │
  │ last_agent_timestamp  — "what's the newest message I've PROCESSED?" │
  │  (per group)            Advanced after agent successfully responds.  │
  │                         Rolled back on error (so nothing is missed). │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Map 6 — Startup Sequence

```
  npm run dev  (or systemctl start nanoclaw)
        │
        ▼
  main() in src/index.ts
        │
        ├─ 1. ensureContainerSystemRunning()
        │      Verify Docker/container runtime is up
        │      Clean up orphaned containers from last run
        │
        ├─ 2. initDatabase()
        │      Open SQLite at store/messages.db
        │      Create/migrate schema (safe, idempotent)
        │
        ├─ 3. loadState()
        │      Restore lastTimestamp from DB
        │      Restore sessions (session IDs per group)
        │      Restore registeredGroups from DB
        │
        ├─ 4. startCredentialProxy(port 3001)
        │      HTTP server that intercepts container API calls
        │      Injects real API key/OAuth token
        │
        ├─ 5. Connect Channels (for each registered channel)
        │      import './channels/index.js'  ← auto-registers all channels
        │      channel.connect()             ← authenticates, starts listening
        │      Channels: Telegram, WhatsApp, Slack, Discord, Gmail
        │
        ├─ 6. startSchedulerLoop()
        │      Polls scheduled_tasks every 60s
        │      Runs due tasks via containers
        │
        ├─ 7. startIpcWatcher()
        │      Polls data/ipc/ every 1s
        │      Processes outbound messages/tasks from containers
        │
        ├─ 8. recoverPendingMessages()
        │      Checks for messages that arrived during a crash
        │      Re-queues them so nothing is lost
        │
        └─ 9. startMessageLoop()
               Polls SQLite every 2s forever
               The main heartbeat of NanoClaw
```

---

## Map 7 — GroupQueue: Concurrency Control

```
  Problem: Multiple groups can have active agents simultaneously.
  Solution: GroupQueue limits to MAX_CONCURRENT_CONTAINERS (default 5).

  Per-group state:
  ┌──────────────┬───────────────────────────────────────────────┐
  │ active       │ Container is currently running for this group │
  │ pendingMsgs  │ New message arrived, needs a container        │
  │ pendingTasks │ Scheduled task queued, needs a container      │
  │ idleWaiting  │ Agent done with work, waiting for next msg    │
  │ retryCount   │ How many times we've retried after error      │
  └──────────────┴───────────────────────────────────────────────┘

  When a new message arrives for group A:
  ┌─────────────────────────────────────────────────────────────┐
  │ Container active? → YES → write to IPC input file (instant) │
  │                  → NO  → activeCount < 5?                   │
  │                             YES → start container now       │
  │                             NO  → add to waitingGroups[]    │
  └─────────────────────────────────────────────────────────────┘

  When a container finishes (drainGroup):
  ┌─────────────────────────────────────────────────────────────┐
  │ 1. Pending tasks? → YES → run next task (priority over msgs)│
  │ 2. Pending msgs?  → YES → start new message container       │
  │ 3. Nothing?       → release slot → drainWaiting()           │
  │                    Give freed slot to next waiting group     │
  └─────────────────────────────────────────────────────────────┘

  Error / Retry:
  ┌─────────────────────────────────────────────────────────────┐
  │ Failure → exponential backoff retry                         │
  │ Retry 1: 5s,  Retry 2: 10s,  Retry 3: 20s                  │
  │ Retry 4: 40s, Retry 5: 80s → drop (max 5 retries)          │
  │ Next incoming message resets retry counter                  │
  └─────────────────────────────────────────────────────────────┘
```

---

## Map 8 — Agent Runner: Inside the Container

```
  container/agent-runner/src/index.ts
  (runs as node process inside Docker)

  1. Read stdin → parse ContainerInput JSON
     { prompt, sessionId, groupFolder, chatJid, isMain }

  2. Start MessageStream (async iterable)
     Push initial prompt into stream

  3. Start IPC poller (every 500ms):
     • Read /workspace/ipc/input/*.json → push into stream (multi-turn!)
     • Check for _close sentinel → end stream (container exits)

  4. Call Claude Agent SDK: query({ prompt: stream, ... })
     Options:
     ┌─────────────────────────────────────────────────────────┐
     │ cwd: /workspace/group   ← CLAUDE.md loaded from here   │
     │ resume: sessionId       ← continues conversation        │
     │ permissionMode: bypass  ← no approval prompts          │
     │ allowedTools: [Bash, Read, Write, Glob, Grep,           │
     │   WebSearch, WebFetch, Task, TodoWrite, Skill,          │
     │   mcp__nanoclaw__*, mcp__notionApi__*]                  │
     │                                                         │
     │ mcpServers:                                             │
     │   nanoclaw → ipc-mcp-stdio.js                          │
     │     (sends messages, schedules tasks, registers groups) │
     │   notionApi → npx @notionhq/notion-mcp-server          │
     │     (reads/writes Notion, if NOTION_TOKEN set)          │
     └─────────────────────────────────────────────────────────┘

  5. For each SDK result message:
     → Write to stdout between markers:
       ---NANOCLAW_OUTPUT_START---
       {"status":"success","result":"agent's reply text","newSessionId":"..."}
       ---NANOCLAW_OUTPUT_END---

  6. After query ends:
     → Wait for next IPC message (or _close)
     → If message → run new query (multi-turn session continues)
     → If _close → exit cleanly
```

---

## Map 9 — Channel System: How Channels Self-Register

```
  src/channels/index.ts  ← barrel import: import './telegram.js', etc.
        │
        │ Each channel file calls registerChannel() at module load time:
        │
        ▼
  registerChannel('telegram', factory)
        │
        ▼
  src/channels/registry.ts
  Map<string, ChannelFactory>
        │
        │ During startup, main() iterates registered channel names:
        ▼
  factory(channelOpts)
  channelOpts = {
    onMessage(chatJid, msg)     ← called for every inbound message
    onChatMetadata(jid, ...)    ← called for group name/metadata
    registeredGroups()          ← current registered groups snapshot
  }
        │
        ▼
  Returns Channel object:
  {
    connect()          ← authenticate, start listeners
    disconnect()       ← graceful shutdown
    sendMessage()      ← send text to a JID
    ownsJid(jid)       ← does this channel own this JID? (routing)
    setTyping()        ← show typing indicator (optional)
    syncGroups()       ← refresh group metadata (optional)
    isConnected()      ← health check
  }

  JID Format per Channel:
  ┌──────────────┬────────────────────────────────┐
  │ WhatsApp     │ 1234567890@g.us (groups)       │
  │ Telegram     │ tg:-100123456789               │
  │ Discord      │ dc:1234567890                  │
  │ Slack        │ slack:C1234567890              │
  └──────────────┴────────────────────────────────┘
```

---

## Map 10 — Security Model

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     SECURITY LAYERS                             │
  ├─────────────────────────────────────────────────────────────────┤
  │                                                                 │
  │  1. Credential Proxy                                            │
  │     • Container NEVER sees real API key                         │
  │     • All API calls go through host:3001                        │
  │     • Proxy injects real credentials before forwarding          │
  │     • .env file shadowed with /dev/null inside container        │
  │                                                                 │
  │  2. Filesystem Isolation                                         │
  │     • Each group gets its OWN folder (no cross-group reads)     │
  │     • Project source (src/) is read-only for main group         │
  │     • Non-main groups have NO access to project source          │
  │     • Agent cannot modify host application code                 │
  │                                                                 │
  │  3. IPC Authorization                                           │
  │     • Container identity = IPC directory it writes to           │
  │     • Non-main groups can ONLY send to their own JID            │
  │     • Only main group can: register new groups,                 │
  │       refresh group list, schedule tasks for OTHER groups       │
  │     • isMain cannot be set via IPC (only via host code)         │
  │                                                                 │
  │  4. Sender Allowlist (optional)                                 │
  │     • Per-group: who can send trigger messages                  │
  │     • Two modes: block (deny listed senders) or                 │
  │       drop (silently discard before even storing)               │
  │                                                                 │
  │  5. Mount Security                                              │
  │     • Additional mounts validated against allowlist at          │
  │       ~/.config/nanoclaw/mount-allowlist.json                   │
  │     • Allowlist is OUTSIDE project root (not mountable)         │
  │     • Containers cannot tamper with their own mount rules       │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Where to Look When...

| Question | File |
|----------|------|
| How does a message get from channel to agent? | `src/index.ts` → `processGroupMessages()` |
| How do I add a new channel? | `src/channels/telegram.ts` as template |
| How does the container get spawned? | `src/container-runner.ts` → `runContainerAgent()` |
| What tools does the agent have? | `container/agent-runner/src/index.ts` → `allowedTools` |
| How does agent send a message back? | `container/agent-runner/src/ipc-mcp-stdio.ts` |
| How are tasks scheduled? | `src/task-scheduler.ts` → `startSchedulerLoop()` |
| How are scheduled tasks created? | `src/ipc.ts` → `processTaskIpc()` case `'schedule_task'` |
| Where is the DB schema? | `src/db.ts` → `createSchema()` |
| How does session memory work? | SQLite `sessions` table + `data/sessions/{group}/.claude/` |
| How does concurrency work? | `src/group-queue.ts` → `GroupQueue` class |
| How are credentials handled? | `src/credential-proxy.ts` |
| What triggers the agent? | `src/config.ts` → `TRIGGER_PATTERN` (regex: `@Sumi`) |
| How do I change the bot name? | `.env` → `ASSISTANT_NAME=Sumi` |

---

## One-Liner Summaries

- **Channel**: Adapter between a messaging platform and NanoClaw's internal message format
- **Registered Group**: A chat that NanoClaw is monitoring (stored in SQLite)
- **GroupQueue**: Traffic controller — at most 5 containers at once, queues the rest
- **Container**: Isolated Docker sandbox where the Claude agent actually runs
- **Agent Runner**: The Node.js script inside the container that calls the Claude Agent SDK
- **Credential Proxy**: HTTP middleman that keeps API secrets out of containers
- **IPC**: File-based communication between host and container (no sockets, just files)
- **Session ID**: The Claude Agent SDK's conversation thread — persisted per group so memory survives restarts
- **Cursor (last_agent_timestamp)**: "How far have I processed?" — rolls back on error, advances on success
- **MCP nanoclaw**: Custom tool server giving the agent abilities: send messages, schedule tasks, register groups
