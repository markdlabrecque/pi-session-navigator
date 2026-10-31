# PRD: Pi Session Navigator

## Product goal
A standalone global Pi extension available whenever Pi starts, exposing all local Pi sessions, including unmanaged and pre-existing ones. Users can navigate among active, idle and historical sessions from multiple terminals.

## Agreed requirements
1. Navigator is separate from the Subagents Extension, with optional shared-supervisor integration.
2. `/agents` is available in Pi from any project directory/session.
3. Discover **all** Pi sessions on the machine, not only supervisor-managed sessions.
4. Show directors as controlled peers and workers as owned children without confusing the relationships.
5. Multiple terminals can navigate between sessions and observe the same active session.
6. Prevent conflicting simultaneous execution or writes to the same session.

## Extension boundary
Related PRD: [Pi Subagents Extension](../../subagents-reloaded/docs/Pi-Subagents-Extension-PRD.md). This link assumes sibling checkouts; replace it with a repository URL if published separately.

The extensions are designed primarily for use together, but each must remain useful without the other.

- Navigator owns global session discovery, search, viewing, switching and safe native resume.
- Subagents owns orchestration through its supervisor: spawning, assignments, worker lifecycle, messaging, grants, recovery and managed execution.
- Without Subagents or its supervisor, Navigator provides session browsing, read-only inspection and safe native resume. It does not start its own orchestration service. Unsupported live attachment remains read-only.
- With a compatible supervisor, Navigator displays relationships, status, assignments, events and pending approvals, and offers only supported, authorized actions.
- Navigator submits control actions through the supervisor API. Retiring a worker requests supervisor retirement, not a direct process kill; approving a tool request invokes supervisor authorization, not a local grant change.
- Subagents remains usable through tools and basic status output without Navigator. An optional "open in Navigator" action falls back to a session ID and basic status when Navigator is unavailable.
- Runtime capability discovery must tolerate an absent, disconnected or incompatible supervisor. Navigator keeps standalone browsing available and marks unsupported integrated actions unavailable.

Durable command queues, execution leases, authorization and single-executor enforcement belong to the supervisor. Navigator displays pending commands and control state but does not duplicate these mechanisms. Viewing a session never grants control. Managed live observation and queued instructions require compatible supervisor capabilities; they are not promises of standalone feature parity.

## Non-goals
Worker orchestration, tool delegation, cross-host navigation in v1, automatically adopting every historical session, guaranteed live attachment to unmanaged processes if Pi does not support it.

## Proposed UI
```text
/agents                 Global Pi Session Navigator
Personal assistant       Running     Standalone
  Website director       Running     Controlled peer
    Implementer A        Running     Owned worker
    Reviewer             Idle        Owned worker
  Research director      Running     Controlled peer
Legacy Drupal session    Resumable   Unmanaged
Old experiment           Historical  Unmanaged

[Enter] Open  [v] View  [/] Search  [Esc] Back
```
Global extension load is required; auto-opening dashboard at startup should be optional.

## Functional requirements
### Discovery
- SN-01 Use supported Pi session metadata APIs, or read-only scanning of documented storage if necessary.
- SN-02 Include pre-installation, unmanaged, managed, and historical sessions; deduplicate by stable ID.
- SN-03 Display project, worktree, last activity, session ID, status, owner/controller and active process when known.
- SN-04 Distinguish running, idle, waiting, disconnected, resumable, historical and unknown. A session file alone does not prove liveness.
- SN-05 Search/filter by project, status, last activity, relationship and title.
### Navigation
- SN-06 Open dashboard from any session and return to prior session with navigation history.
- SN-07 Attach to managed live sessions using supported transport; resume inactive sessions through Pi's native mechanism.
- SN-08 For unmanaged live sessions without safe attach support, allow read-only inspection; do not launch a competing executor.
- SN-09 Multiple terminals may observe one session without killing it or creating competing model loops.
### Execution safety
- SN-10 Use the supervisor's authoritative executor for each managed live session; never start a competing executor.
- SN-11 Submit instructions through the supervisor's durable command queue with IDs and acknowledgments; display pending state. Queue persistence and serialization belong to the supervisor.
- SN-12 Distinguish `viewing` from `controlling`; display supervisor-owned lease and stale-owner state, and request cancellation or reconnection through its API.
- SN-13 Never permit two independent Pi processes to append concurrently to one session.
### Integration
- SN-14 Navigator works independently when Subagents Extension is absent.
- SN-15 When supervisor exists, show workers, directors, progress, pending approvals and heartbeat health.
- SN-16 Viewing a session does not confer control rights. Authorization gates messaging, stopping and grants.

## Session states and actions
| State | Meaning | Safe action |
|---|---|---|
| Running managed | Live executor under supervisor | Observe; queue commands |
| Idle managed | Awaiting follow-up | Observe/resume |
| Waiting | Awaiting input or approval | Inspect/respond if authorized |
| Running unmanaged | Live process outside supervisor | Read-only unless native attach is supported |
| Resumable | Saved session without active executor | Native resume |
| Historical | Archived/closed session | Inspect/fork |
| Unknown/disconnected | Inconsistent or unreachable | Inspect/reconcile |

## Architecture
```text
Pi terminal A ─┐
Pi terminal B ─┼── Navigator extension ── Pi native session index (all sessions)
Pi terminal C ─┘             │
                             └── optional supervisor API
                                  ├── live status/relationships
                                  ├── attachment/event stream
                                  └── single-executor command queue
```
The live attach mechanism is the main feasibility risk. If Pi lacks an attach API, a managed session host/proxy may be needed. Do not claim full live attach until verified.

## Security and privacy
Respect OS user/file permissions. Avoid indexing transcript contents or secrets by default; store minimal metadata. Guard against symlink traversal, stale PIDs and PID reuse. Explicitly authorize control actions; do not equate discoverability with permission. Audit control transfers and command submission.

## Acceptance criteria
- [ ] Pi in two unrelated directories opens the same global session inventory.
- [ ] Sessions predating installation are discovered.
- [ ] Unmanaged live sessions are not falsely advertised as safely attachable.
- [ ] Three terminals observe one managed live session without interfering.
- [ ] Instructions from two terminals execute serially, with no duplicate agent loop or session corruption.
- [ ] Switching sessions preserves a back path and does not stop underlying jobs.
- [ ] Navigator browsing, read-only inspection and safe native resume work without Subagents Extension or its supervisor.
- [ ] An absent, disconnected or incompatible supervisor leaves standalone browsing available and integrated actions unavailable.
- [ ] Installing both extensions does not create duplicate supervisors or competing session executors.
- [ ] With supervisor, director and worker relationships display correctly.
- [ ] Restart preserves discovery and reconnects managed viewers where supported.

## Delivery phases
1. Pi API feasibility: extension global loading, session discovery, session switching, live attach.
2. Read-only global dashboard, search, session status.
3. Native resume and safe navigation/back stack.
4. Managed live observation and event stream.
5. Serialized command submission and control ownership.
6. Supervisor relationship integration, fault recovery and security hardening.

## Open questions
Exact Pi session storage/index APIs, safe live attach capabilities, whether Pi supports switching sessions inside one terminal, execution-lease strategy, command fairness, default startup dashboard behavior, unmanaged session adoption policy and cross-machine roadmap.

---

# Architecture and conventions
Draft v0.1, 2026-10-31. These are product requirements, not verified claims about Pi APIs. Validate extension hooks and live session control in a spike.

The supervisor is a non-LLM service, started by systemd on Omarchy, with a durable registry, message outbox, process monitoring and authorization. Suggested implementation: TypeScript/Node, SQLite WAL and Unix socket IPC. This is a proposal, not a locked technology choice.

A **worker ownership** relationship is distinct from a **director control** relationship. Directors are independent top-level Pi sessions, controlled by the personal assistant and owning flat worker groups. Workers cannot spawn workers.

## Shared integration contract
- Stable session, agent, assignment, operation and message IDs.
- Authenticated local IPC with per-operation authorization, never trust caller-supplied IDs alone.
- Registry records sessions, processes, owner/controller edges, status, heartbeat, and audit history.
- Event stream uses durable sequence numbers, acknowledgments and idempotency keys.
- Session Navigator reads registry data through the supervisor API, never its database or other private storage.
- The supervisor API is versioned independently of extension releases and advertises capabilities. Exact discovery, transport and Pi hooks remain subject to feasibility validation.
- Neither extension imports the other's implementation. Shared IDs and the public API connect them; installation together must not create duplicate supervisors or competing session executors.
- Supervisor enforces policy but never makes LLM decisions.
