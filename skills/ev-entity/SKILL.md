---
name: ev-entity
description: >
  Author (scaffold) a NEW event-sourced domain Entity — the full structure with no business
  logic: ID type, entity state, protocol (commands / events / replies), command & event handler
  stubs, syntax aliases, view model, view repository trait, view functions, view reactor, and
  Circe codecs — then automatically hand off to the wire-entity skill for the infra/app wiring.
  Use the existing Credentials entity as the structural reference. Covers PLAIN entities only
  (no money accounts). Trigger whenever the user wants to "create", "add", "scaffold",
  "author", "generate", or "make" a new entity, aggregate, or event-sourced entity — e.g.
  "create entity Sequencer" — or mentions a new EntityState, entity protocol, entity ID,
  StateId, or a new aggregate root. This is NOT for wiring an already-written entity (that is
  wire-entity alone) and NOT for processes (see wire-process). Trigger on: create entity,
  new entity, add entity, scaffold entity, event-sourced entity, new aggregate, ev-entity,
  entity protocol, EntityState, new StateId.
---

# Author a new event-sourced Entity

Scaffolds the complete structure of a new domain entity: every file, trait, object, and codec
in its final shape — but **no business logic**. Handler and view-factory bodies are `???`
stubs; the entity compiles and wires cleanly but is not invocable until implemented. After
scaffolding, the entity is wired into the runtime by following the **wire-entity** skill.

Mirror the existing **`Credentials`** entity (`entities/credentials/`) — it is the closest
plain-entity reference. Throughout, `Xxx` / `xxx` / `XxxId` stand for the new entity
(e.g. `Sequencer` / `sequencer` / `SequencerId`).

**Scope**: plain entities only. If the entity must hold money accounts (like Brand, Business,
Customer, Invoice — `HasAccountsStateId`, `HasAccount`, union `Command`/`Event` protocols),
STOP and tell the user account-holding entities are out of this skill's scope and are
hand-crafted from the Brand reference.

## Layout in one paragraph

All domain artifacts live in `modules/domain` under `entities/xxx/`; the ID in
`modules/domain-common` (`ids/StateId.scala`); codecs in `modules/infra/serializers`
(`domain/xxx/`); the reactor in `modules/domain/.../reactors/`. The **View** is the
read-model projection saved by the **ViewReactor** through the **ViewRepository**. The one
import every downstream layer uses is `import dev.fintech.domain.entities.xxx.syntax.*`
(or `syntax as Xxx`) — keep its alias letters exact; wiring and codecs rely on them.

## Before scaffolding, check

- **Entity or process?** If it's a long-running multi-step workflow / state machine
  (delays, retries, orchestration), it's a **process** — stop, this skill doesn't apply
  (authoring is hand-crafted from IbanAllocation; wiring is **wire-process**).
- **Accounts?** If it must hold money accounts → out of scope, tell the user (see above).
- **Name collision?** Grep `entities/` for the package name and `ids/StateId.scala` for
  `XxxId` before creating anything — a clash means asking the user, not renaming silently.

## Step 0 — Ask the user first (wait for answers, never default)

Ask these one by one and wait for the actual answer (see CLAUDE.md workflow rules):

1. **Reply style** — ask exactly: *"Select Reply style. Single Reply trait is enough for
   Small entities. Reply trait per command is suitable for large entities."*
   - **Single** → one `XxxReply` ADT shared by all commands (Credentials style).
   - **Per command** → a `sealed trait XxxReply` marker plus one reply ADT per command,
     e.g. `CreateXxxReply` (Brand style).

If anything else is ambiguous while scaffolding (naming, placement, an unexpected conflict),
stop and ask — never decide silently.

## Files to create — templates

The seed content is fixed: state carries **only the ID**, protocol has **one command
`Create` and one event `Created`**, all computational bodies are `???`. The user adds real
fields and logic later; `???` here is deliberate scaffolding, not a violation of the
no-exceptions rule.

### 1. ID type — `domain-common`, edit `ids/StateId.scala`

`StateId`/`EntityId` are sealed, so the new ID must live in
`modules/domain-common/src/main/scala/dev/fintech/domain/common/ids/StateId.scala`.
Mirror `GlobalSettingsId` (the plain-entity pattern):

```scala
final case class XxxId(value: UUID) extends EntityId {
  override def toString: String     = s"XxxId|${UuidToString.toString(value)}"
  override def valueUrlSafe: String = UuidToString.toString(value)
}
object XxxId {
  given CanEqual[XxxId, XxxId]                = CanEqual.derived
  def ofUuidString(x: String): Validated[XxxId] = Validated.validUuid("XxxId")(x).map(new XxxId(_))
}
```

### 2. State — `domain`, `entities/xxx/Xxx.scala`

```scala
package dev.fintech.domain.entities.xxx

import dev.fintech.domain.common.eventsourced.EntityState
import dev.fintech.domain.common.ids.XxxId

final case class Xxx(
    override val id: XxxId
) extends EntityState
object Xxx {

  given CanEqual[Xxx, Xxx] = CanEqual.derived

  def initial(id: XxxId): Xxx = Xxx(id = id)

}
```

### 3. Protocol — `entities/xxx/XxxProtocol.scala`

Commands extend `EntityCommand[Reply]` and always carry `id`, `by: UpdateBy`,
`at: Timestamp`; events extend `EntityEvent` and always carry `by`, `at`. Single-reply
variant shown below; for the **per-command** style replace the reply section with:

```scala
  sealed trait XxxReply
  object XxxReply {
    given CanEqual[XxxReply, XxxReply] = CanEqual.derived
  }

  sealed trait CreateXxxReply extends XxxReply
  object CreateXxxReply {
    sealed trait Failure                     extends CreateXxxReply
    case object InvalidCommandInCurrentState extends Failure
    case object Success                      extends CreateXxxReply
  }
```

then type `Create` as `XxxCommand[CreateXxxReply]` (drop the `private type R`); the syntax
`R` alias still points at the marker `XxxReply`. Each later command gets its own
`<Verb>XxxReply` ADT beside it.

```scala
package dev.fintech.domain.entities.xxx

import dev.fintech.domain.common.UpdateBy
import dev.fintech.domain.common.eventsourced.{EntityCommand, EntityEvent}
import dev.fintech.domain.common.ids.XxxId
import dev.fintech.domain.common.time.Timestamp

object XxxProtocol {

  sealed trait XxxReply
  object XxxReply {
    given CanEqual[XxxReply, XxxReply] = CanEqual.derived

    sealed trait Failure                     extends XxxReply
    case object InvalidCommandInCurrentState extends Failure
    case object Success                      extends XxxReply
  }

  sealed trait XxxCommand[+A] extends EntityCommand[A] {
    override def id: XxxId
    def by: UpdateBy
    def at: Timestamp
  }
  object XxxCommand {
    given CanEqual[XxxCommand[_], XxxCommand[_]] = CanEqual.derived

    private type R = XxxReply

    final case class Create(id: XxxId, by: UpdateBy, at: Timestamp) extends XxxCommand[R]
  }

  sealed trait XxxEvent extends EntityEvent
  object XxxEvent {
    given CanEqual[XxxEvent, XxxEvent] = CanEqual.derived

    final case class Created(by: UpdateBy, at: Timestamp) extends XxxEvent
  }

}
```

### 4. Syntax aliases — `entities/xxx/syntax.scala`

The glue every layer imports. Keep the exact alias letters — wiring and codecs rely on them.

```scala
package dev.fintech.domain.entities.xxx

import dev.fintech.domain.entities.xxx.view.XxxView

object syntax {

  private val P: XxxProtocol.type = XxxProtocol

  type C[+A] = P.XxxCommand[A]
  type E     = P.XxxEvent
  type S     = Xxx
  type R     = P.XxxReply
  type V     = XxxView

  val C: P.XxxCommand.type = P.XxxCommand
  val E: P.XxxEvent.type   = P.XxxEvent
  val S: Xxx.type          = Xxx
  val R: P.XxxReply.type   = P.XxxReply
  val V: XxxView.type      = XxxView

}
```

### 5. Command handler — `entities/xxx/commandhandlers/XxxCommandHandler.scala`

Package layout is `commandhandlers/` + `eventhandlers/` (the majority convention; brand's
single `handlers/` is legacy — do not copy it). An `object` (no dependencies); take
dependencies via a `class` only when they appear later.

```scala
package dev.fintech.domain.entities.xxx.commandhandlers

import dev.fintech.domain.entities.xxx.syntax.*
import dev.fintech.libs.eventsourcing.model.{Effect, EntityCommandHandler}

object XxxCommandHandler extends EntityCommandHandler[S, C, E] {

  override def handleCommand[Reply](state: S, command: C[Reply]): Effect[E, S, Reply] = {
    command match {
      case cmd: C.Create => onCreate(state, cmd)
    }
  }

  private def onCreate(state: S, cmd: C.Create): Effect[E, S, R] = ???

}
```

When real logic is implemented later, the body follows the Credentials pattern:
`Effect.ofEither { for { _ <- assert(...)(R.SomeFailure) } yield Effect.persistOne(event, R.Success) }`.

### 6. Event handler — `entities/xxx/eventhandlers/XxxEventHandler.scala`

```scala
package dev.fintech.domain.entities.xxx.eventhandlers

import dev.fintech.domain.entities.xxx.syntax.*
import dev.fintech.libs.eventsourcing.model.EntityEventHandler

object XxxEventHandler extends EntityEventHandler[S, E] {

  override def applyEvent(state: S, event: E): S = ???

}
```

### 7. View — `entities/xxx/view/XxxView.scala`

Views project selected fields (not the whole state) and are built via
`of(seqNr, state): Option[View]` so not-yet-created state yields `None` — stubbed for now.

```scala
package dev.fintech.domain.entities.xxx.view

import dev.fintech.domain.common.eventsourced.View
import dev.fintech.domain.common.ids.XxxId
import dev.fintech.domain.entities.xxx.Xxx
import dev.fintech.libs.eventsourcing.model.SeqNr

final case class XxxView(
    override val seqNr: SeqNr,
    override val id: XxxId
) extends View[XxxId]
object XxxView {

  def of(seqNr: SeqNr, s: Xxx): Option[XxxView] = ???

}
```

### 8. View repository trait — `entities/xxx/view/XxxViewRepository.scala`

Only the trait lives in the domain; the Postgres-backed impl is created in
`infra/domain-adapters` by **wire-entity** (its step 1).

```scala
package dev.fintech.domain.entities.xxx.view

import dev.fintech.domain.common.ids.XxxId
import dev.fintech.domain.entities.common.DomainViewRepository

trait XxxViewRepository extends DomainViewRepository[XxxId, XxxView]
```

### 9. View functions — `entities/xxx/view/functions.scala`

Function-shaped accessors downstream code depends on (structural delegation, implemented):

```scala
package dev.fintech.domain.entities.xxx.view

import dev.fintech.domain.common.ids.XxxId
import dsl.Task

object functions {

  trait FindXxxByIdFn extends (XxxId => Task[Option[XxxView]])
  object FindXxxByIdFn {
    def apply(repo: XxxViewRepository): FindXxxByIdFn = repo.findViewById(_)
  }

}
```

### 10. View reactor — `reactors/XxxViewReactor.scala`

Lives in `modules/domain/.../reactors/` next to `CredentialsViewReactor` (its exact shape —
structural delegation to the stubbed `V.of`):

```scala
package dev.fintech.domain.reactors

import dev.fintech.domain.entities.xxx.syntax.*
import dev.fintech.domain.entities.xxx.view.XxxViewRepository
import dev.fintech.libs.eventsourcing.model.{HandleUpdate, SeqNr}
import dsl.Task

class XxxViewReactor(
    viewRepo: XxxViewRepository
) extends ViewReactor[Task, S, E] {

  override def handleUpdate: HandleUpdate[Task, S, E] = { case (seqNr, s, _) => updateView(seqNr, s) }

  private def updateView(seqNr: SeqNr, s: S): Task[Unit] = {
    V.of(seqNr, s) match {
      case Some(v) => viewRepo.saveView(v)
      case None    => Task.unit
    }
  }

}
```

### 11. Codecs — `serializers`, `domain/xxx/XxxSerializers.scala`

Follow the **circe-codecs** skill (read it) for the details. The scaffold needs, in
`modules/infra/serializers/.../domain/xxx/XxxSerializers.scala`:

- `given XxxIdCodec: Codec[XxxId] = deriveUnwrapped(XxxId.apply)(_.value)` (mirror
  `GlobalSettingsIdCodec`)
- `given XxxCodec: Codec[Xxx] = deriveCodec` — state
- one `deriveCodec` per event case (`E_CreatedCodec`) and the discriminated
  `Codec[E]` via `DiscriminatorCodecHelper[E]("_t")` (mirror `CredentialsSerializers`)
- `given XxxViewCodec: Codec[V] = deriveCodec`

## NEVER

- **NEVER define the ID outside `ids/StateId.scala`** — `StateId`/`EntityId` are sealed;
  a new ID anywhere else won't compile.
- **NEVER copy the brand/globalsettings layout** — single `handlers/` package and
  wrap-the-state views are legacy; new entities use `commandhandlers/` + `eventhandlers/`
  and field-projecting views with `of(...): Option[V]`.
- **NEVER use Brand as the authoring template** — it is the *wiring* reference for
  wire-entity and carries account unions plus the legacy layout; the authoring reference
  is Credentials.
- **NEVER implement the `???` stubs or invent domain fields** — the scaffold contract is
  structure only; a "helpfully" filled-in handler injects unreviewed business logic.
- **NEVER skip `CanEqual` derivations** — strictEquality is on; downstream pattern matches
  and `==` fail to compile without them.
- **NEVER answer the Step 0 questions yourself or proceed on a timeout** — wait for the
  user's actual answer, then scaffold.
- **NEVER `sbt scalafmtAll`** — it reformats the whole repo and pollutes the diff; use
  `<module>/scalafmtOnly` on touched files only. Never `git commit`; never `git add`
  anything under `.claude/`.

## Finish

1. `git add` every new file immediately (never anything under `.claude/`). `StateId.scala`
   is an edit, not a new file.
2. `sbt compile` (never trust IDE diagnostics alone).
3. Format only the touched files: `sbt "<module>/scalafmtOnly <path> ..."`.
4. **Hand off to wiring**: invoke the **wire-entity** skill now and follow it end to end —
   it creates the `XxxViewRepositoryImpl`, the `EntityDefinition`, guardian, journal
   definition, reactor sinks/groups, `*Env` wiring, and the Flyway view table. All
   wire-entity preconditions are satisfied by the files above.
