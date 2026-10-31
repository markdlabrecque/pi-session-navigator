# Pi session navigation and execution safety

Research for [issue 2](https://github.com/markdlabrecque/pi-session-navigator/issues/2), against [the PRD](../Pi-Session-Navigator-PRD.md) at repository commit `3d4c72d6b847f241122b3f2f8f280d0ab88139d4`. Investigated on 2026-10-31. This is a feasibility record, not a product decision or an implementation plan.

## Findings that matter

- Pi 1.4.0 supports user-global extensions, slash commands, cross-project saved-session enumeration, native session replacement, forks, and in-process or RPC event observation. These are real APIs, not proposed Navigator mechanisms. [E] [D] [N] [R]
- Native `switchSession()` is not live attachment. It opens persisted history, aborts the outgoing session's active work, disposes its runtime, and constructs another runtime. A switch cannot promise that the outgoing job keeps running. [N]
- No native API for attaching a second client to an arbitrary existing interactive Pi process was found in the documented SDK, extension declarations, or RPC implementation. RPC controls a subprocess through its existing stdin/stdout; `RpcClient.start()` spawns one. Managed live observation is feasible through a host or adapter, but multi-client fan-out, reconnect, authorization, and durable command delivery are not supplied by those Pi APIs. [R] [H]
- Session persistence does not enforce one writer or one executor per saved session. Existing files are appended without a session ownership lock, and some open paths can repair or rewrite them. Native resume alone does not establish safe exclusive execution. [W]
- `SessionManager.listAll()` is not an all-machine process registry or a metadata-only index. It scans one configured storage scope and reads transcript bodies to construct previews and searchable text. There is a privacy conflict to resolve before using it for the PRD's default minimal-metadata inventory. No actual user sessions were enumerated or read in this research. [D]

## Scope, versions, and evidence rules

Installed package root, abbreviated `P` below:

```text
/home/mark/.local/lib/node_modules/@earendil-works/pi-coding-agent
```

`P/package.json:1-4` declares `@earendil-works/pi-coding-agent` **1.4.0**. Its repository is `earendil-works/pi`, package directory `packages/coding-agent`. The installed SDK, CLI, and extension APIs discussed here are from that package, not the older `@mariozechner` namespace.

The official `v1.4.0` tag resolves to commit [`4a080252f06caf63a525d7873c5d94d3c23887e0`](https://github.com/earendil-works/pi/commit/4a080252f06caf63a525d7873c5d94d3c23887e0). The [tag API](https://api.github.com/repos/earendil-works/pi/git/ref/tags/v1.4.0) reported a commit object. Installed source maps embed TypeScript source. Exact text comparison against the pinned upstream files passed for all nine inspected modules: `session-manager`, `agent-session-runtime`, `extensions/types`, `agent-session`, `rpc-client`, `rpc-mode`, `resource-loader`, `config`, and `event-bus`. Installed copies of eight principal guides also matched the tag byte for byte: extensions, SDK, sessions, session format, RPC, RPC commands, configuration, and CLI. Thus the pinned TypeScript links below describe the installed implementation, not merely a similarly named release.

Upstream `main` was separately sampled at [`0baf611f04a35618727ee15bbf8bd7f49f16175d`](https://github.com/earendil-works/pi/commit/0baf611f04a35618727ee15bbf8bd7f49f16175d). A complete diff of its session manager and runtime against 1.4.0 showed path-comparison changes, rollback of in-memory entry state on failed persistence, and faster fork copying. It did not add writer ownership or change switch teardown into attachment. This is a two-module upstream comparison, not certification of all upstream APIs. [U]

Labels used below:

- **Verified** means established by the installed documentation and implementation. It does not mean an end-to-end Navigator acceptance test passed.
- **Unsupported** means the inspected native interfaces do not supply the requested guarantee. An adapter might supply it.
- **Unknown** means the contract, external integration, or operating condition still needs evidence. Absence of a discovered API is not proof that no third-party package implements one.

Only repository documentation, installed package documentation/source/examples, and public upstream source were read. No model runs, extension installation, real session inventory, transcript inspection, credential access, production access, or concurrency experiments occurred. The delegated task was research only, with no further delegation and one new repository file.

## Global loading and commands

**Verified.** A file in `<agent-dir>/extensions/` loads as a personal extension. The default agent directory is `~/.pi/agent`; `PI_CODING_AGENT_DIR` or an SDK `agentDir` can select another. `pi.registerCommand("agents", { handler })` is the supported way to expose `/agents`. Personal package installs are also global through user settings. Project trust gates project resources, not the personal extension directory. [E] [C]

"Whenever Pi starts" needs qualification. `--no-extensions` disables discovered/configured extensions, resource exclusions can disable them, different agent directories have different resources, and SDK hosts can supply custom loaders. Extensions load in TUI, RPC, JSON, and print modes, but a custom dashboard needs `ctx.mode === "tui"`. `ctx.hasUI` is also true in RPC, where supported dialogs are forwarded but `custom()` does not provide a terminal component. A session-start hook can initialize resources; documentation says not to create long-lived resources in the factory, since some invocations load extensions without starting a session. [E] [C] [R]

This establishes global command feasibility under a shared configuration. It does not choose startup auto-opening, command-conflict behavior, or a distribution method.

## Discovery, storage, IDs, and metadata

**Verified API.** `SessionManager.list(cwd, sessionDir?, onProgress?, signal?)` lists sessions for a directory. `SessionManager.listAll(...)` lists across project directories in the default sessions root, or accepts an explicit custom directory. Both return `SessionInfo[]` and support progress/cancellation. The extension context exposes a read-only manager for its current session; static enumeration comes from the exported `SessionManager`, not `ctx.sessionManager.listAll()`. [D] [H]

**Verified scope limitation.** Default `listAll()` scans immediate directory or symlink children of `getSessionsDir()`, then their `.jsonl` files. An explicit custom directory is scanned directly, without recursive project discovery. Neither route searches arbitrary disks, alternate agent homes, other users, unknown `--session-dir` locations, or in-memory sessions. CLI/environment/settings can redirect storage; two projects with different overrides need not see the same inventory. Some unreadable or invalid files are silently omitted. The diagram's "native session index" is a filesystem scan here, not an authoritative global registry. [D] [C]

**Verified persisted format.** Default storage is:

```text
~/.pi/agent/sessions/--<encoded-cwd>--/<timestamp>_<session-id>.jsonl
```

The header contains `type`, format `version`, session `id`, creation `timestamp`, `cwd`, and optional `parentSession`. Format v3 is current; v1/v2 are migrated when loaded. Session IDs default to UUIDs but callers may supply custom IDs through the SDK or `--session-id`. Forks create a new session ID. Entry IDs identify nodes within a session tree and are not session IDs. A `parentSession` is a fork/new-session provenance path, not proof of worker ownership or director control. [F] [D]

**ID caveat.** Opening an existing valid header retains its session ID. This supports stable identity across resume, but custom IDs and imported/copied files defeat any assumption of machine-global uniqueness. `listAll()` returns file records without an ID-deduplication step. The product still needs an explicit collision/provenance policy; this research does not select one. [D] [W]

`SessionInfo` includes path, ID, cwd, optional name, optional parent-session path, created/modified dates, message count, first message, and all user/assistant text. The listing implementation computes `modified` from the maximum parsed message activity timestamp, falling back to header time or filesystem mtime. This is saved conversation activity, not a heartbeat. A name comes from the latest `session_info` entry. Old sessions may have empty cwd. No native fields here identify a Git project/worktree relationship, owner/controller, assignment, PID, executor generation, approval, heartbeat, or archived/closed state. A saved cwd can identify a worktree directory, but authoritative Git relationships require separate evidence. [D]

**Privacy and read-only caveats.** Listing reads the whole file and builds `firstMessage`/`allMessagesText`, even if a caller later discards them. `findById()` performs bounded header-only lookup for an exact ID within one scope; it is not a global metadata-list API. Header-only scanning can establish ID/cwd/creation but cannot yield the latest name or message activity without inspecting later entries. `listAll()` follows symlink directories and does not enforce the PRD's traversal policy. [D]

Do not treat `SessionManager.open()` or even `loadEntriesFromFile()` as a guaranteed read-only viewer. The latter appends a missing final newline after validating a header. Opening can migrate old entries and rewrite the file, or initialize an empty file. Saved files also omit in-flight token deltas, so polling them is not equivalent to observing live provider output. [W] [F] [R]

## Native switch, resume, fork, and lifecycle

**Verified.** CLI `--continue` continues the latest session for the current project; `--resume` and TUI `/resume` open the picker. `--session` opens a path, exact ID, or partial ID, with documented cross-project fork handling. `/tree` moves within one entry tree; `/fork` creates a separate session from an earlier user message; `/clone` copies the active branch to a new session. These do not attach to another process. [F] [C]

Extension command contexts offer `waitForIdle`, `newSession`, `fork`, `navigateTree`, `switchSession`, and `reload`. Session replacement returns a cancellation result and can run `withSession` using a fresh context. The old context is invalid after replacement. `AgentSessionRuntime` provides the corresponding host-side operations, and RPC has `switch_session`, `new_session`, `fork`, and `clone`. [N] [H] [R]

The installed `AgentSessionRuntime.switchSession()` does the following:

1. Emits cancellable `session_before_switch` with reason `resume`.
2. Opens and validates the target session/cwd. Opening itself can write as noted above.
3. Calls `teardownCurrent`, which awaits `session.abort()`, emits `session_shutdown`, invalidates/disposes the outgoing runtime, and releases its resources.
4. Constructs a new runtime for the target cwd, emits `session_start` with reason `resume` and `previousSessionFile`, and rebinds host subscriptions. [N]

Fork uses cancellable `session_before_fork` and also tears down the outgoing runtime. Quit, reload, new, resume, and fork have explicit shutdown reasons. `session_shutdown` means runtime teardown, not necessarily OS process exit. Abrupt process death cannot promise orderly handlers. `agent_end` closes one low-level run; retries, compaction, or queued work can continue. `agent_settled` is the notification that automatic work has stopped. [E] [N] [H]

**Unsupported guarantee.** Native replacement is not a detach/background operation that keeps the outgoing executor running. Nor is there a built-in Navigator back stack in these APIs. `previousSessionFile` and a fresh session context are useful inputs for navigation history, but history persistence, deleted targets, cancellation, and keeping work alive need a separate design and tests. `waitForIdle()` can wait rather than abort, but it is not background execution. [N] [H]

## Liveness and live observation

A file proves saved history, not a live executor. The following signals have different scopes:

| Signal | Verified scope | What it cannot prove |
|---|---|---|
| `ctx.isIdle()`, pending-message check, SDK session state | Current in-process runtime | State of another arbitrary session/process |
| `agent_start`, `agent_settled`, tool events, compaction/retry events | Runtime with a bound listener or RPC stream | Heartbeat after disconnect, crash recovery, global membership |
| RPC `get_state` | The process connected to that RPC pipe; includes session ID/file, streaming, compaction, pending count | Global process inventory, lease, or permission |
| Saved timestamps/mtime | Persisted activity | Running versus idle, approval wait, death, or exclusive ownership |
| Shell-tool `PI_SESSION_ID` / `PI_SESSION_FILE` | Values injected when that shell command starts | A persistent live registry; child processes may outlive their session |
| OSC 7501 program status | TUI terminal protocol reports working, blocked, done, error, idle when enabled/supported | Authenticated cross-terminal session registry or executor lease |

Sources: [D] [H] [R] [L]. OSC reports exclude prompts/model output, require terminal support unless forced, and are not forwarded by tmux/screen according to the terminal guide. Environment process markers are inherited and are not automatically set by SDK embedding. PID presence alone would also need process identity and stale/PID-reuse handling, which these persisted fields do not provide. [L]

**Verified observation hooks.** SDK `session.subscribe()` delivers live messages, tools, queues, compaction, retry, and settlement events for an owned in-process session. RPC streams corresponding events over stdout. On the wire, message updates are deltas; `message_end` is authoritative. `get_entries({ since })` can recover persisted append-order entries using an entry ID cursor, including abandoned branches; it is not a durable stream of all lifecycle/approval/command events. After runtime replacement, subscriptions must be rebound. [R] [N]

**Unsupported native attach.** The inspected RPC implementation uses stdin/stdout, not a discoverable socket listener. `RpcClient.start()` launches `node <cli> --mode rpc`; passing a live session's file to a new Pi process would open its history in a second runtime, not attach to the first. No public attach-by-PID/session-ID, multi-client observer role, or transport takeover was found in the reviewed interfaces. Existing TUI processes do not become RPC servers merely because another terminal knows their session ID. [H] [R]

A supervisor-owned SDK host or RPC subprocess has supported inputs for a live adapter. Turning those into multiple viewers requires an external protocol and fan-out implementation. Slow stdout consumption can stall RPC; session events generally have no request ID. Reconnection, replay, buffering, viewer backpressure, and session/runtime identity changes therefore need an explicit adapter contract. These are feasibility implications, not a choice of transport or hosting model. [R]

## Execution safety, multiple terminals, and adapter boundaries

**Unsupported native writer exclusion.** In the installed session manager, first persistence uses exclusive file creation `wx`, but subsequent entries use `appendFileSync` with no session lock or executor check. `_rewriteFile()` opens with `w`. Each manager keeps its own in-memory entry tree and leaf. There is no external-writer merge/watch step in the inspected manager. Two independent resumes can therefore run separate model loops with stale branch state even if individual appends happen to remain valid JSONL. OS append behavior is not an execution lease. Migration/rewrite and newline repair add further hazards. [W]

A scan of unbundled installed JavaScript found `proper-lockfile` use for settings, trust, auth, package management, and MCP OAuth, not session persistence. The presence of that dependency does not establish session protection. This finding is about the inspected persistence and runtime path, not a claim that every Pi-related file is unlocked. [W]

**Verified queue, limited guarantee.** Pi has steering and follow-up queues for one runtime. They are in-memory string arrays; RPC supports queue modes and reports prompt disposition, pending state, and queue changes. A request ID correlates a response, not durable submission, deduplication, authorization, or exactly-once execution. Pi's queues are not the supervisor's durable command outbox. Two clients need one authoritative host and supervisor policy rather than two Pi processes opening the same file. [Q] [R]

**Available public hooks for an adapter:**

- `session_start`, `session_shutdown`, and cancellable before-switch/before-fork hooks to register/unregister a runtime and track replacement. They do not themselves create a registry or block all external processes. [E] [H]
- `agent_start`, `agent_settled`, message/tool/queue events, and current runtime state to publish progress. Approval wait needs adapter-specific reporting rather than inference from a quiet file. [E] [R]
- `tool_call` can block execution; the supplied permission-gate example demonstrates fail-closed non-UI behavior. This can connect an approval adapter, but does not define authenticated supervisor grants or authorize every RPC/control operation. [H] [C]
- `sendUserMessage`, SDK `prompt`/`steer`/`followUp`, abort, and RPC commands can control an owned runtime. They do not target a foreign runtime by ID. [H] [R]
- `pi.events` is a Node `EventEmitter` shared within an extension host. It is not interprocess IPC, a durable event log, or supervisor discovery. The default resource loader creates its own event bus unless a host supplies one. [B]

No reviewed Pi API supplies a supervisor capability handshake, owner/controller graph, durable command acknowledgments, execution leases/fencing, heartbeat registry, per-viewer authorization, or audit/control-transfer contract. Those remain external integration requirements. No compatible supervisor was supplied or examined. Standalone browsing remains feasible; standalone **safe native resume of arbitrary discovered files** is not yet proven because unmanaged liveness and writer exclusion are unresolved. [D] [W] [H]

## Capability matrix against the PRD

"Partial" means some native mechanisms are verified but the full requirement is not. These judgments do not mark acceptance criteria complete.

| Requirement | Finding | Implication and evidence |
|---|---|---|
| SN-01 supported metadata or read-only storage | Partial | Exported listing exists, but reads message bodies; header scan can avoid bodies with reduced metadata. Open/load are not reliably read-only. [D] [W] |
| SN-02 all sessions, stable-ID deduplication | Partial | Existing readable persisted sessions in known roots need no installation marker. Custom roots, ephemeral sessions, ID collisions/copies, and deduplication remain outside the listing guarantee. [D] [F] |
| SN-03 project/worktree/activity/ID/status/relationships/process | Partial | cwd, saved activity, name and ID exist. Git mapping, relationships, PID and live state need external evidence. [D] [L] |
| SN-04 truthful running/idle/waiting/disconnected/etc. | Partial | Own-runtime events/state exist; a saved file cannot classify liveness or archive status. Supervisor heartbeat semantics unknown. [L] [R] |
| SN-05 search/filter | Partial | Native listing contains name/cwd/activity and searchable transcript text. Minimal-metadata title filtering is possible with separate extraction; status/relationship filters need adapter data. No Navigator UI tested. [D] |
| SN-06 global dashboard and back history | Partial | Global command/custom TUI and native switching exist. No built-in back stack; replacement invalidates runtime and aborts outgoing work. [E] [N] |
| SN-07 managed attach and inactive native resume | Partial | Native resume verified. Managed attach requires an external host/transport contract; inactive/exclusive status not proven by a file. [N] [R] [W] |
| SN-08 unmanaged live read-only fallback | Partial | Read-only file inspection is feasible, but must avoid mutating open/load helpers. No unmanaged native attach found; reliable unmanaged-live classification is unknown. [W] [L] |
| SN-09 multiple observers, no competing loops | Unsupported natively | SDK/RPC expose observation of one executor, not multi-client attachment. Fan-out and observer isolation need supervisor evidence. [R] [H] |
| SN-10 authoritative managed executor | External/unknown | Pi does not enforce supervisor ownership or prevent another process resuming the file. [W] |
| SN-11 durable IDs/acks/serialized instructions | Unsupported natively | Local queues and RPC request IDs exist but are not a durable supervisor queue or deduplication guarantee. [Q] [R] |
| SN-12 viewing/control leases/stale owner/reconnect | External/unknown | No native viewer/control lease model found. Adapter protocol and failure handling remain unverified. [H] [R] |
| SN-13 never two independent concurrent writers | Unsupported natively | Exclusive creation is not ongoing writer exclusion. Existing session appends/rewrites lack an ownership guard. [W] |
| SN-14 standalone operation | Partial | Discovery and read-only UI need no Subagents service. Safe resume still requires evidence of inactivity/exclusion. [D] [E] [W] |
| SN-15 relationships/progress/approvals/heartbeat | External/unknown | Pi has own-runtime progress and tool gates, not the proposed supervisor graph, health or approval records. [H] [R] |
| SN-16 viewing grants no control | External/unknown | SDK/RPC APIs are control interfaces, not per-viewer authorization. Supervisor must establish and enforce rights independently of discoverability. [C] [R] |

## Acceptance criteria implications

All eleven PRD checkboxes remain untested as end-to-end product criteria.

| PRD acceptance criterion, in original order | Evidence and remaining condition |
|---|---|
| 1. Two unrelated directories, same global inventory | Supported for shared agent/storage roots using global enumeration, not automatic across different overrides. Need fixture coverage for root scope and extension loading. [C] [D] |
| 2. Pre-installation sessions discovered | No Navigator registration is required for readable valid JSONL in scanned roots. Need legacy/missing-cwd/custom-root fixtures. [D] [F] |
| 3. Unmanaged live not falsely attachable | No native attach found. Need UI and safety assertions that discovery/resume never implies attachment or exclusive ownership. [R] [W] |
| 4. Three terminals observe one managed live session | Not demonstrated. Needs one executor host, multi-client event delivery, observer-only roles, and disconnect/backpressure tests. [R] |
| 5. Two terminals' instructions execute serially without duplicate loops/corruption | Not supplied by native request IDs/local queues. Needs durable supervisor serialization, deduplication and writer-exclusion evidence. [Q] [W] |
| 6. Switching preserves back path and does not stop jobs | Native replacement aborts outgoing work; no native back history. Requires clarified navigation/hosting semantics and a focused lifecycle prototype, not an assumption about `switchSession`. [N] |
| 7. Standalone browsing, inspection and safe native resume | Browsing is feasible; inspection must avoid write-capable loaders. Safe unmanaged resume remains unresolved without an exclusion/liveness argument. [D] [W] |
| 8. Absent/disconnected/incompatible supervisor leaves browsing available | Pi supplies independent extension APIs, but no supervisor negotiation. Needs a versioned capability contract and failure fixtures. [E] [H] |
| 9. Installing both extensions creates no duplicate supervisors/executors | Package identity avoids some duplicate resource loading, not process/service singleton ownership. Needs integration tests at the supervisor boundary. [C] [W] |
| 10. Director/worker relationships display correctly | Native fork provenance is not ownership/control. Needs authoritative supervisor records and relationship fixtures. [D] [F] |
| 11. Restart preserves discovery and reconnects viewers where supported | Files preserve saved history in known roots. Runtime subscriptions are not persistent; durable replay/reconnect depends on the adapter. [D] [N] [R] |

## Remaining investigations, bounded by the evidence

1. Define the optional supervisor's capability/version contract before claiming managed attach, viewer replay, queue durability, approval authorization, or control transfer. Determine which Pi process/SDK host owns each executor and who owns its lifetime. No supervisor implementation or endpoint was examined.
2. Resolve the standalone safety gap. What trusted evidence can exclude an unmanaged writer, including a process started before Navigator installation or with extensions disabled? Can the guarantee hold without Pi-level locking adopted by every writer? A recent file timestamp, missing heartbeat, or stale PID is insufficient. This is a feasibility blocker for a universal SN-13 claim, not authorization to choose adoption or resume policy.
3. Specify the intended discovery scope and metadata budget. Unknown custom roots and ephemeral sessions cannot be recovered by default `listAll()`. Decide separately how to handle symlinks, duplicated IDs, copies/imports, unreadable files and legacy cwd gaps. Current source proves the gaps; no broad filesystem scan is warranted.
4. Use synthetic fixtures only for later validation: legacy formats and newline repair, partial trailing writes, denied permissions, symlink escapes, copied/custom IDs, custom root layouts, and resume collisions. No real transcripts are needed. Source evidence already establishes lack of writer exclusion, so destructive multiwriter experiments are unnecessary for this research.
5. A later terminal prototype can verify custom-screen exit/back behavior and lifecycle cancellation with synthetic sessions and a fake/non-networked runtime. It must distinguish a dashboard/view change from runtime replacement and show what happens to in-flight work. No UX or product behavior is selected here.
6. Once an adapter exists, test three observers, two simultaneous submitters, duplicate IDs, lost acknowledgments, slow viewers, host/supervisor restarts, stale ownership, and authorization failures. Pi's own stream backpressure and runtime rebinding rules must be included. Installed 1.4.0 and future releases need separate compatibility checks. [R] [U]

## Source register

Local paths are relative to `P`. Local line numbers refer to installed files, not upstream TypeScript line numbers. Every upstream link is commit-pinned. Where installed source-map content exactly matched upstream, the linked TypeScript is also a readable record of the installed source.

- [E] Global extension and lifecycle guide: local `docs/extensions.md:13-60,128-193,324-400`; [upstream guide](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/extensions.md#L13-L60). Exact declarations: local `dist/core/extensions/types.d.ts:213-340,575-642,1186-1270`; [context and command types](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/extensions/types.ts#L325-L455).
- [C] Configuration/CLI/package/security contracts: local `docs/configuration.md:1-47`, `docs/cli.md:77-108,181-216`, `docs/packages.md:9-29,119-129`, `docs/security.md:1-88`; [configuration](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/configuration.md), [CLI](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/cli.md), [packages](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/packages.md), [security](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/security.md).
- [D] Discovery and metadata: local `dist/core/session-manager.d.ts:6-13,162-178,383-416`, `dist/core/session-manager.js:492-622,1421-1521`; [header and metadata types](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L43-L55), [SessionInfo](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L229-L262), [listing record construction](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L799-L893), [exact lookup and enumeration](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L1874-L2013). Root: local `dist/config.js:491-494,527-529`; [config source](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/config.ts).
- [F] Storage and native user behavior: local `docs/session-format.md:1-33,64-80,199-209`, `docs/sessions.md:1-53`; [format](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/session-format.md#L1-L80), [sessions](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/sessions.md). Defaults/new IDs: local `dist/core/session-manager.js:691-716`; [ID and migration code](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L264-L351).
- [N] Runtime replacement: local `dist/core/agent-session-runtime.js:77-247,292-311`, `dist/core/agent-session.js:1011-1030`; [teardown and switch](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session-runtime.ts#L139-L223), [new and fork](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session-runtime.ts#L226-L349), [dispose](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session.ts#L1412-L1440).
- [H] Public host/extension contracts and examples: local `docs/sdk.md:26-99`, `dist/core/extensions/types.d.ts:213-340,575-642,1186-1270,1405-1406`, `examples/sdk/11-sessions.ts`, `examples/sdk/13-session-runtime.ts`, `examples/extensions/permission-gate.ts`; [SDK](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/sdk.md), [session runtime example](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/sdk/13-session-runtime.ts), [permission gate](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/examples/extensions/permission-gate.ts).
- [R] RPC and observation: local `docs/rpc.md:1-100`, `docs/rpc-commands.md:5-193,592-746`, `docs/json.md:13-134`, `docs/rpc-extension-ui.md:5-28`, `dist/modes/rpc/rpc-client.js:20-72`; [RPC guide](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/rpc.md), [commands](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/rpc-commands.md), [event stream](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/json.md), [spawn implementation](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-client.ts#L55-L134), [RPC event binding](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-mode.ts#L329-L386), [stdin handling](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/modes/rpc/rpc-mode.ts#L747-L819).
- [W] Persistence, repair and absence of session ownership guards: local `dist/core/session-manager.js:328-369,665-682,718-727,754-816`; [load/repair](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L627-L670), [open/migrate](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L1029-L1103), [rewrite and append](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/session-manager.ts#L1124-L1196). Lock-use contrast: local `dist/core/settings-manager.js:4`, `dist/core/trust-manager.js:4`, `dist/core/auth-storage.js:7`, `dist/package-manager-cli.js:5`, `dist/extensions/mcp/oauth.js:17`. These imports are source inspection only; no stored auth/settings content was read.
- [L] Liveness signal limits: local `docs/environment-variables.md:9-49`, `docs/terminal-setup.md:221-233`, `dist/core/agent-session.js:1068-1070,1224-1226`; [environment guide](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/environment-variables.md#L9-L49), [terminal program status](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/docs/terminal-setup.md#L221-L233). State declarations are also in [H] and RPC state in [R].
- [Q] Local queues: local `dist/core/agent-session.js:95-97,1795-1818,1943-1966`, `docs/rpc-commands.md:14-41,268-300`; [in-memory queue fields](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session.ts#L390-L405), [enqueue methods](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/agent-session.ts#L2296-L2330).
- [B] Event bus scope: local `dist/core/event-bus.js:1-23`, `dist/core/resource-loader.js:246-248`; [event bus](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/event-bus.ts#L1-L33), [loader](https://github.com/earendil-works/pi/blob/4a080252f06caf63a525d7873c5d94d3c23887e0/packages/coding-agent/src/core/resource-loader.ts#L374-L382).
- [U] Sampled upstream after release: [session manager](https://github.com/earendil-works/pi/blob/0baf611f04a35618727ee15bbf8bd7f49f16175d/packages/coding-agent/src/core/session-manager.ts#L1124-L1209) and [runtime](https://github.com/earendil-works/pi/blob/0baf611f04a35618727ee15bbf8bd7f49f16175d/packages/coding-agent/src/core/agent-session-runtime.ts#L167-L223). These links are not installed-version citations.

## Verification record

Document-only validation applies. No extension code, tests, settings, PRD changes, issue edits, or supervisor files belong to this candidate.

- Read the installed README and relevant guides completely, following session, SDK, RPC, configuration, settings, security, TUI, keybinding, environment, terminal-status, tmux, package, and message-format references. Read the session and runtime SDK examples and permission-gate example. Unrelated model-provider customization guides were outside scope.
- Read relevant installed declarations and implementation paths; compared the nine embedded TypeScript sources and eight guides against the official release commit. Compared two key upstream-main modules separately. No runtime/model credentials were required.
- Required cheap checks for this document are `git diff --check`, exact SN-01 through SN-16 coverage, all eleven acceptance criteria, reference-label resolution, installed-source line-range bounds, and a changed-file scope check. The implementor handoff reports their results. No code compile/test gate or multi-terminal acceptance experiment was run; those would not validate a research-only change. No independent reviewer approval is claimed.
