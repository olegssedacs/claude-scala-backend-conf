---
name: wire-process
description: >
  Wire an already-existing event-sourced domain Process (a long-running business workflow /
  state machine) into the event-sourcing infrastructure and app bootstrap — the plumbing that
  connects a finished process (state, protocol, command/event handlers, syntax, optional
  orchestration, reactors) and its Circe codecs to a Postgres journal, a process guardian
  actor, reactor sinks, the process status repo, and the *Env wiring. Use the existing
  IbanAllocation process as the reference. This is NOT about authoring a new domain process,
  writing handlers, orchestration, reactors, or codecs — those are assumed to already exist
  (authoring a new process is the ev-processes skill); this skill only does the infra/app wiring. Trigger whenever the user wants to "wire", "hook
  up", "register", "plug in", "connect", or "bootstrap" an existing process into event sourcing,
  or mentions wiring a process's journal, guardian, ProcessDefinition, ProcessDefinitions,
  EvProcesses, PostgresJournalDefinitions, ReactorUpdatesSinks, Reactors, ProcessesEnv, or ReactorsEnv.
  Trigger on: wire process, hook up process, register process, plug in process, connect process
  to journal, add process guardian, ProcessDefinition, ProcessDefinitions, EvProcesses,
  PostgresJournalDefinitions, PostgresJournals, Journals, ReactorUpdatesSinks, Reactors wiring,
  ProcessesEnv, ReactorsEnv, wire process event sourcing, orchestration wiring.
---

# Wire an existing Process into event-sourcing

The domain process is already written. This skill connects it to the runtime: a Postgres
journal, a **process guardian** actor that routes commands, reactor sinks that fan events out to
the existing reactors, the process status repo, and the `*Env` bootstrap. Mirror the existing
**`IbanAllocation`** process — it is the simplest fully-wired process (a 3-event state machine,
no orchestration, only a notification reactor). Throughout, `Xxx` / `xxx` / `XxxId` stand for the
process being wired (e.g. `WalletTopUp` / `walletTopUp` / `WalletTopUpId`).

## Processes ≠ Entities — separate machinery

Processes are wired through their **own** parallel infrastructure, *not* the entity machinery.
Do **not** touch `EntityDefinitions` / `EvEntities`. The process analogs are:

| Entity wiring          | Process wiring (use these) |
|------------------------|----------------------------|
| `EntityDefinition(s)`  | `ProcessDefinition` / `ProcessDefinitions` |
| `EvEntities`           | `EvProcesses` |
| `entitiesAc`           | `processesAc` (a dedicated `ActorSystem` in `ActorSystemEnv`) |
| `EntitiesEnv`          | `ProcessesEnv` |
| `entityDefs.xxx` in `Reactors.make` | `processDefs.xxx` in `Reactors.make` |
| `EntityCommandHandler` | `ProcessCommandHandler[Task, S, C, E]` |
| —                      | `ProcessStatusRepository` (lifecycle status), `ProcessCallbacks` (async signalling) |
| —                      | optional `Orchestration[S]` (state-machine automation) |

`PostgresJournalDefinitions`, `Journals`/`PostgresJournals`, `ReactorUpdatesSinks`, and
`Reactors` are **shared** — they hold *both* entity and process fields; you add a process field
to each, exactly like an entity, but the `Reactors.buildGroup` call references `processDefs.xxx`
rather than `entityDefs.xxx`.

## Precondition — what must already exist

This skill assumes the **domain** and **serializer** artifacts are done. Verify they exist
before wiring; if any are missing, they're out of scope here (write them first — scaffolding
a new process is the **ev-processes** skill; use **circe-codecs** for codecs, **scala-specs**
for tests):

- ID type `XxxId` (`domain-common/.../ids/StateId.scala`)
- The protocol + state in `processes/xxx/` (finop processes live in `processes/finops2/xxx/`):
  `XxxCommand extends ProcessCommand`, `XxxEvent extends ProcessEvent`, the `Xxx` state
  `extends ProcessState` with `initial`, and a `syntax` object exposing `S`, `C`, `E`, and the
  `ProcessCommonCommand` alias — `CC` in non-finop processes (kyb/ibanallocation), `PC` in
  finops2 processes
- `XxxCommandHandler extends ProcessCommandHandler[Task, S, C, E]` and `XxxEventHandler extends ProcessEventHandler[S, E]`
- Optionally `XxxOrchestration extends Orchestration[S]` (only if the process auto-advances; `IbanAllocation` has none)
- The reactors the process needs (often just `generalNotificationReactor`; finops processes add tx/finOp/cp projections)
- `XxxSerializers` Circe codecs for state + events

If all of the above are present, proceed. The wiring is pure plumbing — add the process's field
to a handful of **exhaustive case classes** and let `sbt compile` point you at the next missing site.

## Why the wiring is shaped this way

Event sourcing + CQRS: `Command → CommandHandler → Event → EventHandler → State`, events appended
to a **PostgreSQL** journal (dedicated `journal` schema, via `libs/postgres-journal`), **Reactors**
replaying the stream to build read models and fire side effects. A process additionally has a
lifecycle **status** (Running/Paused/Completed) tracked in
Postgres, and may carry an **`Orchestration`** that decides, from the current state, whether the
guardian should auto-send a `ProcessCommonCommand.Continue` to advance the machine. The wiring
layer binds the abstract domain handlers/orchestration/reactors to concrete journals, codecs,
guardian actors, and sinks.

## Reference files — read the IbanAllocation analog before each step

- `infra/domain-adapters/.../eventsourcing/processes/{ProcessDefinition,ProcessDefinitions,EvProcesses}.scala`
- `infra/domain-adapters/.../eventsourcing/common/{PostgresJournalDefinition,PostgresJournalDefinitions,Journals,PostgresJournals}.scala`
- `infra/domain-adapters/.../eventsourcing/reactors/{ReactorUpdatesSinks,Reactors}.scala`
- `modules/app/.../envs/{ActorSystemEnv,ProcessesEnv,ReactorsEnv}.scala`
- Domain anchors: `processes/ibanallocation/{package.scala,IbanAllocation.scala,IbanAllocationCommandHandler.scala,IbanAllocationEventHandler.scala}`,
  `serializers/.../ibanallocation/IbanAllocationSerializers.scala`
- For an **orchestrated** process, also read `processes/finops2/sepadeposit/SepaDepositOrchestration.scala`

## Checklist — follow in order

### 1. ProcessDefinitions — bind the handlers (+ orchestration)

`eventsourcing/processes/ProcessDefinitions.scala`. Add
`type XxxDef = ProcessDefinition[XxxId, Xxx.S, Xxx.Command, Xxx.Event]`, the `xxx: XxxDef` field
to the case class, the `xxx = xxxDef(...)` entry in `make`, and a `private def xxxDef(...)`
returning `new XxxDef { ... }` that overrides `commandHandler = new XxxCommandHandler(...)`,
`eventHandler = new XxxEventHandler`, `actorNameOf = _.valueUrlSafe`,
`initialState = Xxx.S.initial`, and `orchestration = None` (or `Some(XxxOrchestration)` for an
automated machine). If the command handler needs new dependencies, add them to
`ProcessDefinitions.make`'s parameter list too (and thread them from `ProcessesEnv`, step 8).

### 2. EvProcesses — guardian + command routing

`eventsourcing/processes/EvProcesses.scala`. Add an `xxxGuardian: Actor` constructor param; add a
`guardianOf` case `(_: XxxId, _: Xxx.Command | _: ProcessCommonCommand) => Task(xxxGuardian)`
(the `| ProcessCommonCommand` arm is required so orchestration's `Continue` reaches the guardian);
in `make`, build `val xxxG = guardianOf(name = "XxxGuardian", definition = definitions.xxx,
journal = journals.xxx, sink = sinks.xxx)`, add `xxx <- actorSystem.actorOf(xxxG, "xxxs")` to the
for-comprehension, and pass `xxxGuardian = xxx` into the returned `EvProcesses`. (The local
`guardianOf` helper calls `GuardianActor.process[...]`, which already threads `statusRepo` and
`definition.orchestration` — nothing extra to wire there.)

### 3. PostgresJournalDefinition — journal tables + codecs

`eventsourcing/common/PostgresJournalDefinitions.scala`. Add the
`xxx: PostgresJournalDefinition[XxxId, Xxx.S, Xxx.Event]` field to the case class, the
`xxx = xxxDef(schema)` entry in `make`, and `xxx.schema` to the `schemas` list; add
`private def xxxDef(schemaName: String)` with `import XxxSerializers.given` returning
`PostgresJournalDefinition.of(idCodec = ..., stateCodec = PostgresJournalCodec.circe[Xxx.S],
eventCodec = PostgresJournalCodec.circe[Xxx.Event], schema = schemaName,
eventsTableName = "xxx_events", snapshotsTableName = "xxx_snapshots", idsTableName = "xxx_ids")`.
For the id codec: plain-UUID ids use
`PostgresJournalIdCodec.uuid[XxxId](_.value)(id => Right(XxxId(id)))`; UUIDv5-backed process ids
(most finops processes) use `PostgresJournalIdCodec.uuid[XxxId](_.value.value)(ofUUIDv5(XxxId.apply))`;
non-UUID ids use `PostgresJournalIdCodec.text` (see the `credentials` def). (Journal tables
auto-create at startup: `ActorSystemEnv.Postgres.make` runs the DDL from
`schemas.flatMap(_.queries)` — journal tables are NOT Flyway-managed.)

### 4. Journals — trait member + Postgres instance

`eventsourcing/common/Journals.scala`: add `def xxx: Journal[Task, XxxId, Xxx.S, Xxx.Event]` to
the `Journals` trait. `eventsourcing/common/PostgresJournals.scala`: add the matching
`override val xxx` field and `xxx = mkJournal(doobie, definitions.xxx)` in `make`.

### 5. ReactorUpdatesSinks — event fan-out sink

`eventsourcing/reactors/ReactorUpdatesSinks.scala`. Add the `xxx` field, the
`xxx <- ReactorUpdatesSink.empty[Xxx.S, Xxx.Event]` line in `empty`, and `xxx = xxx` in the yield.

### 6. Reactors — group + recovery task

`eventsourcing/reactors/Reactors.scala`. Add `xxx: List[ShardedReactor]` to the case class, a
`mkXxx: List[Make[Xxx.S, Xxx.Event]]` param to `make`, a
`xxxGroup <- buildGroup("xxx", journals.xxx, processDefs.xxx, mkXxx)` line (note **`processDefs`**,
not `entityDefs`), wire `xxx = xxxGroup._1`, and add `xxxGroup._2` to `journalTasks`.

### 7. App bootstrap — ReactorsEnv

`modules/app/.../envs/ReactorsEnv.scala`. In the `Reactors.make(...)` call, add the
`mkXxx = List(...)` argument registering the process's reactors, e.g. minimally:
```scala
mkXxx = List(
  statelessFn("domain.XxxNotificationReactor", generalNotificationReactor[Xxx.S, Xxx.Event])
)
```
(finops processes typically add `modelView(...)` projections: `txStatementReactor`,
`finOpViewReactor`, `cpTransactionsReactor`, etc.) `entityView` → DomainView, `modelView` →
DbProjection, `statefulFn`/`statelessFn` → Functions; recovery runs DomainView → DbProjection →
StatefulFn. Finally add `_ <- sinks.xxx.add(reactors.xxx)` to the sink-registration block.

### 8. App bootstrap — ProcessesEnv

`modules/app/.../envs/ProcessesEnv.scala`. There is **no per-process field** here — the process
is built inside `ProcessDefinitions.make` / `EvProcesses.make`, which `ProcessesEnv.make` already
calls. Only edit this file if step 1 added new dependencies to `ProcessDefinitions.make`: add the
matching `XxxEnv` dependency to `ProcessesEnv.make`'s params and pass it through the
`processDefinitions(...)` call. Otherwise just confirm it compiles. `ActorSystemEnv` (the
`processesAc` system) and the shared `ProcessStatusRepository` need no per-process changes.

### 9. Postgres view table (Flyway) — only if the process has projections

`modules/app/src/main/resources/db/migration/app/V{next}__xxx_view.sql`. Add the projection
table(s) the process's `modelView` reactors write, if not already present. Use the
**flyway-migrations** skill. A notification-only process (like `IbanAllocation`) needs **no**
migration — the shared `process_status` table already exists and the journal tables in the
`journal` schema auto-create at startup (never add them to Flyway).

## Verify & finish

- Compile with **sbt**, not IDE diagnostics: `sbt compile`. Format only the files you changed:
  `sbt "<module>/scalafmtOnly <path> ..."` (never `scalafmtAll` — see CLAUDE.md workflow rules).
- `sbt unit-test` if you touched anything testable.
- `git add` new files immediately. **Never** `git commit`. Do not add anything under `.claude/`.

## Conventions (from CLAUDE.md)

- Always use imports, never fully-qualified type paths. Scala 2 brace syntax. Pure FP — no
  `var` / `null` / throw. `final case class`; prefer `given`/`using`; derive `CanEqual`.
- Computational concepts: `trait` + companion `make` factory + anonymous impl, not a `class`.
- Keep comments minimal.
