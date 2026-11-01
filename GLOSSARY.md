# Pi Session Navigator

Pi Session Navigator is a Pi extension for finding and navigating local sessions. This glossary defines its vocabulary, including concepts for optional supervisor integration. A definition does not establish that a capability is implemented or supported.

## Language

### Sessions and identity

**Session**:
A persistent Pi conversation history, distinct from any running instance that advances it. A saved session does not establish that an executor is active.
_Avoid_: Process, executor when referring to the conversation history.

**Session ID**:
The identifier of a session, distinct from its file path and the identifiers of entries within its history.

**Session file**:
The file that stores a session's history. Its existence alone is not evidence of activity or safe resumption.

**Entry**:
An individual record within a session's history, identified by an entry ID. An entry is not a separate session.

**Executor**:
A running Pi instance that advances a session through model and tool execution. Observing a session does not make the observer an executor.

**Fork**:
A separate session derived from an existing session's history. Its origin records provenance, not a parent-child ownership relationship.

### Discovery and navigation

**Navigator**:
The Pi Session Navigator extension, responsible for session discovery and navigation rather than worker orchestration.

**Discovery**:
The identification of sessions within the locations Navigator examines. Discovery does not establish liveness, permission to control a session or safe attachment.

**Session inventory**:
The collection of sessions known to Navigator through discovery. It is distinct from the dashboard that presents it.

**Dashboard**:
Navigator's session-browsing interface, where users inspect the inventory and choose navigation actions.

**Search**:
The narrowing of the session inventory using a query or filters. The term does not imply that Navigator indexes conversation contents.

**Read-only inspection**:
Access to a session's saved history without changing it or starting an executor. Read-only access does not, by itself, limit how much content is read.
_Avoid_: Resume, attachment.

**Live observation**:
Read-only access to updates from an active session. Observation does not confer control or start another executor.

**Live attachment**:
A connection to an existing executor for supported interaction with its session. Attachment is distinct from starting an executor and does not itself grant authorization.
_Avoid_: Resume.

**Native resume**:
Continuation of a saved session through Pi's own session mechanism. Resume is distinct from connecting to an existing executor.
_Avoid_: Live attachment.

**Navigation history**:
The record of prior Navigator locations used to provide a back path. It is separate from a session's conversation history and fork provenance.

**Back**:
Navigation to a prior location in navigation history. The term does not imply that an executor is restarted or a prior session is resumed.

### Roles and relationships

**Parent**:
A session that owns one or more child sessions in an orchestration relationship. This is the role called a director elsewhere in the project documentation.
_Avoid_: Director as a separate role.

**Child**:
A session owned by a parent in an orchestration relationship. This is the role called a worker elsewhere in the project documentation.
_Avoid_: Worker as a separate role.

**Ownership**:
The orchestration relationship between a parent and its child. Ownership is distinct from fork provenance and does not, by itself, establish a viewer's control permissions.

### Status and control

**Supervisor**:
The optional service responsible for orchestration and managed execution. Navigator uses its supported capabilities without taking over those responsibilities.

**Managed session**:
A session whose execution is managed by the supervisor. Management is independent of whether the session is currently running, idle or waiting.

**Unmanaged session**:
A session whose execution is not managed by the supervisor. An unmanaged session may still have an active executor.
_Avoid_: Standalone as a session status.

**Standalone mode**:
Navigator operation without supervisor integration. It describes Navigator's operating mode, not a session's activity or management status.

**Running**:
An activity status indicating that an executor is actively advancing the session. Saved activity alone is not evidence of this status.

**Idle**:
An activity status indicating that an executor is available but is not currently advancing the session or waiting for required input or approval.

**Waiting**:
An activity status indicating that further execution requires input or approval.

**Disconnected**:
A connectivity status indicating that contact with a previously reachable supervisor or executor has been lost. It does not establish that execution has stopped.

**Unknown**:
A status indicating insufficient evidence to determine a session's condition. It is not a synonym for inactive.

**Resumable**:
A session confirmed eligible for native resume without competing with an active executor. A session file alone does not establish eligibility.

**Last activity**:
The most recent recorded activity associated with a session. It is not proof that an executor is still alive.

**Heartbeat**:
A recurring liveness signal from a monitored service or executor. It is distinct from recorded conversation activity.

**Viewing**:
Read-only access to session information through inspection or live observation. Viewing does not confer control.

**Control**:
The ability to request actions that affect a session's execution, subject to authorization. Control is distinct from viewing and parent-child ownership.

**Authorization**:
Permission to perform a particular control action. Discovering a session, viewing it or attaching to its executor does not itself establish that permission.

**Pending approval**:
A request awaiting an authorization decision before an action may proceed. Displaying the request does not give the viewer permission to approve it.
