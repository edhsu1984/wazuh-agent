# Command Queue & Store Reference

## MultiType Queue storage layout and backpressure

| SQLite table | `MessageType` enum | Configuration keys | Backpressure strategy |
| --- | --- | --- | --- |
| `STATELESS` | `MessageType::STATELESS` | `agent.queue_size` (bytes), `events.batch_interval` (ms) | Synchronous producers optionally block on `std::condition_variable` until `QUEUE_STATUS_REFRESH_TIMER` expires; coroutine producers await a `steady_timer` before retrying when full. |
| `STATEFUL` | `MessageType::STATEFUL` | `agent.queue_size` (bytes), `events.batch_interval` (ms) | Same as above. |
| `COMMAND` | `MessageType::COMMAND` | `agent.queue_size` (bytes), `events.batch_interval` (ms) | Same as above. |

* `m_mapMessageTypeName` maps every `MessageType` to the SQLite table declared in `STATELESS_TABLE_NAME`, `STATEFUL_TABLE_NAME`, and `COMMAND_TABLE_NAME`, ensuring lookups remain consistent across the implementation.【F:src/agent/multitype_queue/include/multitype_queue.hpp†L19-L72】
* The maximum stored messages per table are capped by `agent.queue_size`, read through `GetBytesConfigInRangeOrDefault`. `events.batch_interval` configures `m_batchInterval` for batching consumers.【F:src/agent/multitype_queue/src/multitype_queue.cpp†L17-L44】
* Pushers use a `std::condition_variable` with the refresh timeout `config::agent::QUEUE_STATUS_REFRESH_TIMER` when `shouldWait=true`; coroutine pushers loop on a `steady_timer` with the same timeout until space is available.【F:src/agent/multitype_queue/src/multitype_queue.cpp†L45-L132】
* Batch consumers wait for either `messageQuantity` bytes or `m_batchInterval` milliseconds by combining a size poll with a `steady_timer`, guaranteeing bounded latency even under low throughput.【F:src/agent/multitype_queue/src/multitype_queue.cpp†L133-L210】

## Message envelope and command entries

* A queue item uses the `Message` wrapper that carries the JSON payload plus `moduleName`, `moduleType`, and optional metadata for round-tripping through persistence.【F:src/agent/multitype_queue/include/message_entry/message.hpp†L7-L46】
* Commands re-use the `module_command::CommandEntry` data model. Besides the payload `Parameters` JSON, it records module routing, execution mode, timestamp, and execution result state (status + message).【F:src/agent/command_handler/include/command_entry/command_entry.hpp†L8-L83】

### JSON field conventions

| Field | Source | Description |
| --- | --- | --- |
| `data` | Queue persistence rows | Raw event or command payload stored in the queue. |
| `moduleName` | Queue persistence rows | Logical producer identifier used for per-module dequeue. |
| `moduleType` | Queue persistence rows | Classifies producers (e.g., collector vs. handler). |
| `metadata` | Queue persistence rows | Transport metadata such as retry hints. |
| `parameters` | `CommandEntry` | Command-specific JSON arguments; validated per command signature. |
| `result` | `CommandEntry::ExecutionResult.Message` | Human-readable execution feedback. |
| `status` | `CommandEntry::ExecutionResult.ErrorCode` | Serialized `module_command::Status` enum. |
| `execution_mode` | `CommandEntry::ExecutionMode` | `SYNC` or `ASYNC` dispatch mode. |
| `time` | `CommandEntry::Time` | Milliseconds-since-epoch recorded at store time. |

## Command store schema and lifecycle

### SQLite schema

The `command_store` table resides in `command_store.db` and is created on demand with the columns listed below. All fields are `NOT NULL`, and `id` is the primary key.【F:src/agent/command_handler/src/command_store.cpp†L16-L96】

| Column | Type | Meaning |
| --- | --- | --- |
| `id` | TEXT PK | Command UUID. |
| `module` | TEXT | Target module resolved after validation. |
| `command` | TEXT | Command verb. |
| `parameters` | TEXT (JSON) | Serialized arguments. |
| `execution_mode` | INTEGER | `0` = sync, `1` = async. |
| `result` | TEXT | Latest execution message. |
| `status` | INTEGER | Encoded `module_command::Status`. |
| `time` | REAL | Epoch milliseconds recorded at enqueue. |

The helper converters keep the enum mappings stable: `StatusFromInt` maps `0..3` into `SUCCESS`, `FAILURE`, `IN_PROGRESS`, `TIMEOUT`, with all other values treated as `UNKNOWN`; `ExecutionModeFromInt` interprets `0` as `SYNC` and any other value as `ASYNC`.【F:src/agent/command_handler/src/command_store.cpp†L28-L63】

### Processing state machine

`CommandHandler::CommandsProcessingTask` coordinates queue ingestion and durable status tracking. The main transitions are:

1. **Idle → Fetch**: Await a command from the queue; if none, sleep for `WAIT_TIME` (1s) before retrying.【F:src/agent/command_handler/src/command_handler.cpp†L47-L88】
2. **Fetch → Validate**: For each dequeued entry, `CheckCommand` verifies the verb and parameters, populating module routing and execution mode from `VALID_COMMANDS_MAP`. Failures set status to `FAILURE` with message "Command is not valid" and immediately report the result before dropping the queue entry.【F:src/agent/command_handler/src/command_handler.cpp†L18-L104】
3. **Validate → Persist (IN_PROGRESS)**: Valid commands are inserted into `command_store` with `StoreCommand`; failures (e.g., DB errors) are reported as `FAILURE` with "Agent's database failure" before the queue message is discarded.【F:src/agent/command_handler/src/command_handler.cpp†L90-L119】
4. **Persist → Dispatch**:
   * **Sync** commands await `dispatchCommand`, update the row with the returned `ExecutionResult`, and log completion.【F:src/agent/command_handler/src/command_handler.cpp†L121-L131】
   * **Async** commands spawn a detached coroutine that performs the same update when the dispatcher completes.【F:src/agent/command_handler/src/command_handler.cpp†L132-L143】
5. **Startup cleanup**: Before processing, `CleanUpInProgressCommands` resolves lingering `IN_PROGRESS` entries—marking restarts as `SUCCESS` and other commands as `FAILURE`—to honor at-least-once semantics after agent restarts.【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】

### Synchronization with the queue

* `getCommandFromQueue` and `popCommandFromQueue` form the bridge between the transient queue and durable store: a command is first peeked for validation/persistence, then explicitly popped only after the store acknowledges the insert, preventing loss during validation failures.【F:src/agent/command_handler/src/command_handler.cpp†L67-L120】
* `reportCommandResult` receives every terminal or cleanup update so the caller (typically the queue manager) can emit responses or metrics. The queue itself stores messages by module, letting consumers filter by `moduleName` and `moduleType` during retrievals.【F:src/agent/multitype_queue/src/multitype_queue.cpp†L151-L210】

## API contracts for alternative persistence backends

When replacing SQLite with another storage engine, maintain the following guarantees:

1. **MultiType Queue storage API**: Any `IStorage` implementation must preserve atomic `Store`, `Retrieve*`, and `RemoveMultiple` semantics per table so that message counts drive backpressure correctly. Capacity checks (`GetElementCount`, `GetElementsStoredSize`) must remain accurate to enforce `shouldWait` blocking and coroutine timers, preserving at-least-once delivery from producers.【F:src/agent/multitype_queue/include/istorage.hpp†L9-L87】【F:src/agent/multitype_queue/src/multitype_queue.cpp†L45-L210】
2. **Command store durability**: `ICommandStore` backends must implement `StoreCommand`, `UpdateCommand`, and `GetCommandByStatus` with transactional integrity so that `IN_PROGRESS` records survive restarts and can be cleaned up before new processing begins. Returning stale or partial data breaks the restart recovery performed in `CleanUpInProgressCommands`.【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】【F:src/agent/command_handler/src/command_store.cpp†L98-L218】
3. **At-least-once dispatch**: Commands should only be removed from the queue after persistence succeeds, and dispatchers must update the stored status/results exactly once per execution attempt. Alternative stores must therefore support idempotent updates keyed by command `id` and ensure that restart cleanup can distinguish unfinished work (e.g., via `status = IN_PROGRESS`).【F:src/agent/command_handler/src/command_handler.cpp†L90-L171】
4. **Restart hygiene**: Any backend must expose a way to enumerate and update `IN_PROGRESS` commands at startup to either retry or mark them terminally failed, matching the current behavior where restarts clear dangling operations and report results upstream.【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】

By adhering to these contracts, alternate storage solutions can be introduced without regressing delivery guarantees or the command processing flow.
