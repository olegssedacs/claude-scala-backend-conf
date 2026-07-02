---
name: wire-entity
description: >
  Wire an already-existing event-sourced domain Entity into the event-sourcing infrastructure
  and app bootstrap — the plumbing that connects a finished domain entity (state, protocol,
  command/event handlers, view, reactors) and its Circe codecs to a Postgres journal, an actor
  guardian, reactor sinks, and the *Env wiring. Use the existing Brand entity as the reference.
  This is NOT about authoring a new domain entity, writing handlers, reactors, or codecs — those
  are assumed to already exist; the skill only does the infra/app wiring. Trigger whenever the
  user wants to "wire", "hook up", "register", "plug in", "connect", or "bootstrap" an existing
  entity into event sourcing, or mentions wiring an entity's journal, guardian, EntityDefinitions,
  PostgresJournalDefinitions, ReactorUpdatesSinks, Reactors, EvEntities, DomainReposEnv, or ReactorsEnv.
  Trigger on: wire entity, hook up entity, register entity, plug in entity, connect entity to
  journal, add entity guardian, EntityDefinitions, PostgresJournalDefinitions, PostgresJournals,
  Journals, ReactorUpdatesSinks, Reactors wiring, EvEntities, ReactorsEnv, wire event sourcing.
---

# Wire an existing Entity into event-sourcing

The domain entity is already written. This skill connects it to the runtime: a Postgres
journal, an actor guardian that routes commands, reactor sinks that fan events out to the
existing reactors, the read-model repo, and the `*Env` bootstrap. Mirror the existing
**`Brand`** entity — it is the simplest fully-wired entity (account, domain view, Postgres
projection, notifications). Throughout, `Xxx` / `xxx` / `XxxId` stand for the entity being
wired (e.g. `Merchant` / `merchant` / `MerchantId`).

## Precondition — what must already exist

This skill assumes the **domain** and **serializer** artifacts are done. Verify they exist
before wiring; if any are missing, they're out of scope here (write them first — use
**circe-codecs** for codecs, **scala-specs** for tests):

- ID type `XxxId` (`domain-common/.../ids/StateId.scala`)
- `XxxProtocol` (commands, events, replies) and the `Xxx` state with `initial`
- `entities/xxx/syntax.scala` exposing `S`, `E`, `Event`, `Command`, `C` (the glue every layer imports)
- `XxxCommandHandler` and `XxxEventHandler`
- `XxxView` + `XxxViewRepository` (trait) and the reactors (`XxxViewReactor`, projections)
- `XxxSerializers` Circe codecs for state + events

If all of the above are present, proceed. The wiring is pure plumbing — add the entity's
field to a handful of **exhaustive case classes** and let `sbt compile` point you at the next
missing site.

## Why the wiring is shaped this way

Event sourcing + CQRS: `Command → CommandHandler → Event → EventHandler → State`, events
appended to a **PostgreSQL** journal (dedicated `journal` schema, via `libs/postgres-journal`),
**Reactors** replaying the stream to build read models and fire side effects. The wiring layer
(in `infra/domain-adapters` and `modules/app/.../envs`) binds the abstract domain
handlers/reactors to concrete journals, codecs, actor guardians, and sinks. The single import every site uses is `import dev.fintech.domain.entities.xxx.syntax as Xxx`.

## Reference files — read the Brand analog before each step

- `eventsourcing/entities/{EntityDefinition,EntityDefinitions,EvEntities}.scala`
- `eventsourcing/common/{EvDefinition,PostgresJournalDefinition,PostgresJournalDefinitions,Journals,PostgresJournals}.scala`
- `eventsourcing/reactors/{ReactorUpdatesSinks,Reactors}.scala`
- `modules/infra/domain-adapters/.../brand/BrandViewRepositoryImpl.scala`
- `modules/app/.../envs/{ActorSystemEnv,EntitiesEnv,DomainReposEnv,ReactorsEnv}.scala`
- Domain anchors to grep for the exact aliases: `entities/brand/syntax.scala`,
  `reactors/BrandViewReactor.scala`, `serializers/.../brand/BrandSerializers.scala`

## Checklist — follow in order

### 1. Read-model repo impl (infra adapter)

`modules/infra/domain-adapters/.../xxx/XxxViewRepositoryImpl.scala`. If the existing
`XxxViewRepository` trait has no infra implementation yet, add one mirroring
`BrandViewRepositoryImpl`: `InMemViewRepository` + `XxxSnapshotViewRepository(session,
snapshotEveryNEvents)` wrapped in `CommonViewRepository`, plus a `make(...)` factory. If it
already exists, skip.

### 2. EntityDefinition — bind the handlers

`eventsourcing/entities/EntityDefinitions.scala`. Add
`type XxxDef = EntityDefinition[XxxId, Xxx.S, Xxx.Command, Xxx.Event]`, the `xxx: XxxDef` field
to the case class and `make`, and `private def xxxDef(timestampFn)` wiring
`new XxxCommandHandler(timestampFn)`, `new XxxEventHandler`, `actorNameOf = _.valueUrlSafe`,
`initialState = Xxx.S.initial`.

### 3. EvEntities — guardian + command routing

`eventsourcing/entities/EvEntities.scala`. Add an `xxxGuardian: Actor` constructor param; add a
`guardianOf` case `(_: XxxId, _: Xxx.Command[_]) => Task(xxxGuardian)`; in `make`, build the
guardian (`name = "XxxGuardian"`, `definition = definitions.xxx`, `journal = journals.xxx`,
`sink = sinks.xxx`), `actorOf(xxxG, "xxxs")`, and thread it into the returned `EvEntities`.

### 4. PostgresJournalDefinition — journal tables + codecs

`eventsourcing/common/PostgresJournalDefinitions.scala`. Add the
`xxx: PostgresJournalDefinition[XxxId, Xxx.S, Xxx.Event]` field to the case class, the
`xxx = xxxDef(schema)` entry in `make`, and `xxx.schema` to the `schemas` list; add
`private def xxxDef(schemaName: String)` with `import XxxSerializers.given` returning
`PostgresJournalDefinition.of(idCodec = ..., stateCodec = PostgresJournalCodec.circe[Xxx.S],
eventCodec = PostgresJournalCodec.circe[Xxx.Event], schema = schemaName,
eventsTableName = "xxx_events", snapshotsTableName = "xxx_snapshots", idsTableName = "xxx_ids")`.
For the id codec: plain-UUID ids use
`PostgresJournalIdCodec.uuid[XxxId](_.value)(id => Right(XxxId(id)))`; UUIDv5-backed ids use
`PostgresJournalIdCodec.uuid[XxxId](_.value.value)(ofUUIDv5(XxxId.apply))`; non-UUID ids use
`PostgresJournalIdCodec.text` (see the `credentials` def). (Journal tables auto-create at
startup: `ActorSystemEnv.Postgres.make` runs the DDL from `schemas.flatMap(_.queries)` —
journal tables are NOT Flyway-managed.)

### 5. Journals — trait member + Postgres instance

`eventsourcing/common/Journals.scala`: add `def xxx: Journal[Task, XxxId, Xxx.S, Xxx.Event]` to
the `Journals` trait. `eventsourcing/common/PostgresJournals.scala`: add the matching
`override val xxx` field and `xxx = mkJournal(doobie, definitions.xxx)` in `make`.

### 6. ReactorUpdatesSinks — event fan-out sink

`eventsourcing/reactors/ReactorUpdatesSinks.scala`. Add the `xxx` field and
`xxx <- ReactorUpdatesSink.empty[Xxx.S, Xxx.Event]` in `empty`.

### 7. Reactors — group + recovery task

`eventsourcing/reactors/Reactors.scala`. Add `xxx: List[ShardedReactor]` to the case class, a
`mkXxx: List[Make[Xxx.S, Xxx.Event]]` param to `make`, a
`xxxGroup <- buildGroup("xxx", journals.xxx, entityDefs.xxx, mkXxx)` line, wire `xxx = xxxGroup._1`,
and add `xxxGroup._2` to `journalTasks`.

### 8. App bootstrap

- **ActorSystemEnv** / **EntitiesEnv** — no per-entity field; they call the `*.make` factories
  above (note `EvEntities.make`'s signature changed in step 3). Just confirm they compile.
- **DomainReposEnv** (`envs/DomainReposEnv.scala`) — add `xxxViewRepo: XxxViewRepository`, build
  `xxxViewRepo <- XxxViewRepositoryImpl.make(jdbc, viewProps.snapshotEveryNEvents)`, pass through.
- **ReactorsEnv** (`envs/ReactorsEnv.scala`) — instantiate the existing reactors
  (`val xxxViewReactor = new XxxViewReactor(domainRepos.xxxViewRepo)`, + projections) and register
  them via the new `mkXxx` param in `Reactors.make`, e.g.:
  ```scala
  mkXxx = List(
    entityView("domain.XxxViewReactor", xxxViewReactor, DomainViewSeqNrRepository[XxxId, XxxView](domainRepos.xxxViewRepo)),
    modelView("domain.XxxSomeProjectionReactor", xxxProjectionReactor),
    statelessFn("domain.XxxNotificationReactor", generalNotificationReactor[Xxx.S, Xxx.Event])
  )
  ```
  `entityView` → DomainView, `modelView` → DbProjection, `statefulFn`/`statelessFn` → Functions;
  recovery runs DomainView → DbProjection → StatefulFn. Finally add `_ <- sinks.xxx.add(reactors.xxx)`.

### 9. Postgres view table (Flyway)

`modules/app/src/main/resources/db/migration/app/V{next}__xxx_view.sql`. Add the snapshot/
projection table(s) the view repo and projection reactors read/write, if not already present.
Use the **flyway-migrations** skill. (Flyway is only for view/projection tables; the journal
tables in the `journal` schema auto-create at startup — never add them to Flyway.)

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