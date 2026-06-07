---
name: link-new-entity
description: >
  Add a new event-sourced Entity and wire it end-to-end, from the pure domain layer
  through the infra adapters to the app bootstrap, using the existing Brand entity as the
  reference template. Use this skill whenever the user wants to add, create, link, wire, or
  scaffold a new entity / aggregate in this fintech backend — including phrasings like
  "add a new entity", "create a Merchant entity", "wire an entity from domain to infra",
  "link a new aggregate", "scaffold an event-sourced entity", or when they mention entity
  command/event handlers, entity guardians, EntityDefinitions, journals, view reactors, or
  the *Env wiring. Trigger even if the user does not say "skill" or enumerate the layers.
  Trigger on: new entity, add entity, create entity, link entity, wire entity, aggregate,
  event-sourced entity, command handler, event handler, guardian, journal, view reactor,
  EntityDefinitions.
---

# Link a new Entity (domain → infra → app)

Adding an event-sourced Entity means threading one new type through a fixed set of layers.
Mirror the existing **`Brand`** entity — it is the simplest *complete* entity (it has an
account, a domain view, a Postgres projection, and notifications). Read Brand's files for any
step you're unsure about and copy its structure exactly, swapping the names.

Throughout, `Xxx` / `xxx` / `XxxId` stand for your new entity (e.g. `Merchant` / `merchant` /
`MerchantId`).

## Why the wiring is shaped this way

This is event sourcing + CQRS. An entity is a state machine:
`Command → CommandHandler → Event → EventHandler → new State`. Events are appended to a
**Cassandra** journal; **Reactors** replay the event stream to build read models (domain views
+ Postgres projections) and fire side effects. The **domain** layer is pure and defines
everything abstractly; **infra** supplies Circe codecs, Cassandra journals, JDBC repos, and the
actor guardians; the **app** layer wires it together in `modules/app/.../envs/*Env.scala`.
Dependency direction: `domain` ← `infra/domain-adapters` ← `infra/api` ← `app`.

The single glue point every layer imports is the entity's **`syntax`** object
(`import dev.fintech.domain.entities.xxx.syntax as Xxx`), which re-exports the type aliases
`S`, `E`, `Event`, `Command`, `C` plus matching `val` accessors. Create `syntax.scala` early —
everything downstream depends on it.

Most infra/app collaborators are **exhaustive case classes** (`EntityDefinitions`, `Journals`,
`JournalDefinitions`, `ReactorUpdatesSinks`, `Reactors`, `EvEntities`) that enumerate every
entity. Add your field to each and the compiler will point you at the next missing one — lean
on `sbt compile` to drive the work rather than trying to hold all sites in your head.

## Decisions to confirm before starting

Ask the user (or infer from their request) and let the answers shape which steps apply:

1. **Entity name** — `Xxx` / `xxx` / `XxxId`.
2. **Does it own an account?** If yes (like Brand), the protocol unions with
   `TransactionEvent` / `TransactionCommand`, the state mixes in `HasAccount[XxxAccount]`, and
   the event handler delegates transaction events to `ApplyTransactionEventFn`. If no, it's
   simpler — skip those.
3. **Which read models / reactors** it needs: a domain view (`entityView`), Postgres
   projections (`modelView`), notifications/side effects (`statelessFn` / `statefulFn`).
4. **Commands & events** — the actual business operations. The skill scaffolds the structure;
   the user defines the semantics.

## Reference files — read the matching one before each step

Entity (domain):
- `modules/domain-common/.../common/ids/StateId.scala` — `BrandId`
- `modules/domain/.../entities/brand/Brand.scala` — state + `initial`
- `modules/domain/.../entities/brand/BrandProtocol.scala` — commands / events / replies
- `modules/domain/.../entities/brand/syntax.scala` — the glue object
- `modules/domain/.../entities/brand/handlers/{BrandCommandHandler,BrandEventHandler}.scala`
- `modules/domain/.../entities/brand/view/{BrandView,BrandViewRepository}.scala`

Reactors (domain):
- `modules/domain/.../reactors/BrandViewReactor.scala` — writes the domain view
- `modules/domain/.../reactors/BrandAccountPathReactor.scala` — example Postgres projection
- `modules/domain/.../reactors/ViewReactor.scala` — base trait

Infra:
- `modules/infra/serializers/.../domain/brand/BrandSerializers.scala`
- `modules/infra/domain-adapters/.../brand/BrandViewRepositoryImpl.scala`
- `modules/infra/domain-adapters/.../eventsourcing/entities/{EntityDefinition,EntityDefinitions,EvEntities}.scala`
- `modules/infra/domain-adapters/.../eventsourcing/common/{EvDefinition,JournalDefinition,JournalDefinitions,Journals}.scala`
- `modules/infra/domain-adapters/.../eventsourcing/reactors/{ReactorUpdatesSinks,Reactors}.scala`

App:
- `modules/app/.../envs/{ActorSystemEnv,EntitiesEnv,DomainReposEnv,ReactorsEnv}.scala`

## Checklist — follow in order

### A. Domain (`modules/domain`, `modules/domain-common`)

1. **ID** — `ids/StateId.scala`. Reuse an existing `StateId` if it fits; else add
   `final case class XxxId(value: UUID)` extending `EntityId` (or `HasAccountsStateId` if it
   owns an account), with `ofUuidString`, `valueUrlSafe`, and `given CanEqual`.
2. **Protocol** — `entities/xxx/XxxProtocol.scala`. Type aliases `Event` and `Command[+A]`
   (union with `TransactionEvent`/`TransactionCommand` if account-owning);
   `sealed trait XxxCommand[+A] extends EntityCommand[A] { def id: XxxId }`;
   `sealed trait XxxEvent extends EntityEvent` (each event carries `at: Timestamp`); reply ADTs
   with `Failure`/`Success` and `given CanEqual`.
3. **State** — `entities/xxx/Xxx.scala`. `final case class Xxx(override val id: XxxId, ...)
   extends EntityState` (+ `with HasAccount[XxxAccount]` if account-owning); companion with
   `given CanEqual` and `def initial(id: XxxId)`. The "already exists" guard compares
   `state == Xxx.initial(state.id)`.
4. **syntax** — `entities/xxx/syntax.scala`. Copy Brand's verbatim, swapping names. Exposes
   `S`, `E`, `Event`, `Command`, `C` (+ `Tx*` if account-owning) and the `val` accessors.
5. **Handlers** — `entities/xxx/handlers/`.
   - `XxxCommandHandler(timestampF: TimestampFn) extends EntityCommandHandler[S, Command, Event]`
     — match each command, return `Effect.persistOne(event, andThen = Reply.Success)`,
     `Effect.ignore(...)`, or `Effect.ofEither(...)`. Validate with `Either`; no exceptions.
   - `XxxEventHandler extends EntityEventHandler[S, Event]` — `applyEvent` folds events into
     new state via `copy`; delegate tx events to `ApplyTransactionEventFn` if account-owning.
6. **View + repo interface** — `entities/xxx/view/`.
   `XxxView(seqNr, xxx) extends View[XxxId]`; `trait XxxViewRepository extends
   DomainViewRepository[XxxId, XxxView]`; optional `functions.scala` with `GetXxxByIdFn` /
   `FindXxxByIdFn`.
7. **Reactors** — `domain/reactors/`.
   `XxxViewReactor(viewRepo) extends ViewReactor[Task, S, Event]` saving `XxxView(seqNr, state)`
   (the canonical DomainView reactor); add projection reactors modeled on `BrandAccountPathReactor`.

### B. Infra (`modules/infra`)

8. **Codecs** — `infra/serializers/.../domain/xxx/XxxSerializers.scala`. Circe `Codec`s for the
   state, the event ADT (discriminated via `DiscriminatorCodecHelper`, like `BrandEventCodec` /
   `EventCodec`), and nested types. Combine entity + transaction event codecs like Brand's
   `EventCodec` if account-owning. Use the **circe-codecs** skill, and **scala-specs** for the
   `verifyJson` round-trip test.
9. **View repo impl** — `infra/domain-adapters/.../xxx/XxxViewRepositoryImpl.scala`. Mirror
   `BrandViewRepositoryImpl`: `InMemViewRepository` + `XxxSnapshotViewRepository(session,
   snapshotEveryNEvents)` wrapped in `CommonViewRepository`; add a `make(...)` factory.
10. **EntityDefinitions** — `eventsourcing/entities/EntityDefinitions.scala`. Add
    `type XxxDef = EntityDefinition[XxxId, Xxx.S, Xxx.Command, Xxx.Event]`, the `xxx` field +
    `make` arg, and `private def xxxDef(timestampFn)` wiring the two handlers, `actorNameOf =
    _.valueUrlSafe`, `initialState = Xxx.S.initial`.
11. **EvEntities** — `eventsourcing/entities/EvEntities.scala`. Add `xxxGuardian: Actor` param;
    add `guardianOf` case `(_: XxxId, _: Xxx.Command[_]) => Task(xxxGuardian)`; in `make` build
    the guardian (`name = "XxxGuardian"`, `journals.xxx`, `sinks.xxx`), `actorOf(xxxG, "xxxs")`,
    and pass it through.
12. **JournalDefinitions** — `eventsourcing/common/JournalDefinitions.scala`. Add the `xxx`
    field to the case class, `make`, and `schemas` list; add `private def xxxDef(keyspace)` with
    `import XxxSerializers.given`, `JournalSchema(keyspace, "xxx_events", "xxx_snapshots",
    "xxx_ids")`, `idCodec = JournalCodec[XxxId](_.valueUrlSafe)(XxxId.ofUuidString)`,
    `stateCodec = Circe2Journal[Xxx.S]`, `eventCodec = Circe2Journal[Xxx.Event]`. (Cassandra
    tables auto-create from `schema.queries` at startup — no manual Cassandra DDL.)
13. **Journals** — `eventsourcing/common/Journals.scala`. Add the `xxx` field and
    `xxx = mkJournal(cassandra, definitions.xxx)`.
14. **ReactorUpdatesSinks** — `eventsourcing/reactors/ReactorUpdatesSinks.scala`. Add the `xxx`
    field and `xxx <- ReactorUpdatesSink.empty[Xxx.S, Xxx.Event]`.
15. **Reactors** — `eventsourcing/reactors/Reactors.scala`. Add the `xxx: List[ShardedReactor]`
    field, a `mkXxx: List[Make[Xxx.S, Xxx.Event]]` param, a
    `xxxGroup <- buildGroup("xxx", journals.xxx, entityDefs.xxx, mkXxx)` line, wire `xxx =
    xxxGroup._1`, and add `xxxGroup._2` to `journalTasks`.

### C. App (`modules/app/.../envs`)

16. **ActorSystemEnv** — no per-entity edit; it already calls `JournalDefinitions.make`,
    `Journals.make`, `ReactorUpdatesSinks.empty`. Just confirm it compiles.
17. **EntitiesEnv** — no new field; it calls `EntityDefinitions.make` + `EvEntities.make` (whose
    signature changed in step 11). Confirm it compiles.
18. **DomainReposEnv** — add `xxxViewRepo: XxxViewRepository`, build
    `xxxViewRepo <- XxxViewRepositoryImpl.make(jdbc, viewProps.snapshotEveryNEvents)`, pass through.
19. **ReactorsEnv** — instantiate `val xxxViewReactor = new XxxViewReactor(domainRepos.xxxViewRepo)`
    (+ projections); register via the new `mkXxx` param in `Reactors.make`, e.g.:
    ```scala
    mkXxx = List(
      entityView("domain.XxxViewReactor", xxxViewReactor, DomainViewSeqNrRepository[XxxId, XxxView](domainRepos.xxxViewRepo)),
      modelView("domain.XxxSomeProjectionReactor", xxxProjectionReactor),
      statelessFn("domain.XxxNotificationReactor", generalNotificationReactor[Xxx.S, Xxx.Event])
    )
    ```
    `entityView` → DomainView, `modelView` → DbProjection, `statefulFn`/`statelessFn` →
    Functions; recovery runs DomainView → DbProjection → StatefulFn. Add `_ <- sinks.xxx.add(reactors.xxx)`.

### D. Persistence

20. **Flyway** — `modules/app/src/main/resources/db/migration/app/V{next}__xxx_view.sql`. Add the
    Postgres snapshot/projection table(s) your snapshot repo and projection reactors use. Use the
    **flyway-migrations** skill. (Only Postgres needs SQL; Cassandra journal tables auto-migrate.)

## Verify & finish

- Compile with **sbt**, not IDE diagnostics: `sbt compile`. Format: `sbt scalafmtAll`.
- Add codec + handler specs (skills: **circe-codecs**, **scala-specs**); run `sbt unit-test`.
- `git add` new files immediately. **Never** `git commit`. Do not add anything under `.claude/`.

## Conventions (from CLAUDE.md)

- Always use imports, never fully-qualified type paths. Scala 2 brace syntax. Pure FP — no
  `var` / `null` / throw.
- `final case class`; prefer `given`/`using`; derive `CanEqual` (strictEquality is on).
- `enum` for fixed closed sets; `sealed trait` for caller-instantiated cases.
- Computational concepts: `trait` + companion `make` factory + anonymous impl, not a `class`.
- Keep comments minimal.
