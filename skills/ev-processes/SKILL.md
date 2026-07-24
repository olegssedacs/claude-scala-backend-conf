---
name: ev-processes
description: >
  Author (scaffold) a NEW event-sourced domain Process — a long-running business workflow /
  state machine — the full minimal structure with no business logic: ID type, protocol
  (command / event), sealed state, command & event handler stubs, Fn-decomposed functions,
  orchestration, syntax aliases, and Circe codecs — then automatically hand off to the
  wire-process skill for the infra/app wiring. Covers BOTH finop processes (financial
  operations under processes/finops2/, reference: InternalTransfer) and non-finop processes
  (under processes/, reference: KybReview). Trigger whenever the user wants to "create",
  "add", "scaffold", "author", "generate", or "make" a new process, workflow, state machine,
  finop process, or event-sourced process — e.g. "create process WalletTopUp" — or mentions
  a new ProcessState, process protocol, FinOpProcessId, orchestration, or a new multi-step
  business workflow. This is NOT for wiring an already-written process (that is wire-process
  alone) and NOT for entities (see ev-entity). Trigger on: create process, new process,
  add process, scaffold process, event-sourced process, new finop, finop process, new
  workflow, state machine process, ev-processes, ProcessState, FinOpProcessId, orchestration.
---

# Author a new event-sourced Process

Scaffolds the complete **minimal** structure of a new domain process: every file, trait,
object, and codec in its final shape — but **no business logic and no lifecycle shape**.
The seed is deliberately tiny: two states (`NotStarted`, `Completed`), one command
(`Start`), one event (`Started`), all bodies with domain meaning are `???`. The user grows
the state machine by hand afterwards (finop lifecycle reference: `SepaWithdrawal`; funds-only:
`InternalTransfer`). After scaffolding, the process is wired into the runtime by following
the **wire-process** skill.

Both process kinds share the same style: **Fn decomposition** (one `*Fn` trait per unit of
work, built in the handler companion's `make`) and an **`Orchestration`** that drives the
machine by telling the guardian when to auto-send `ProcessCommonCommand.Continue`.
Processes are reply-less (`tell`-only) — there is no Reply ADT and no reply question.

Throughout, `Xxx` / `xxx` / `XxxId` stand for the new process (e.g. `WalletTopUp` /
`wallettopup` / `WalletTopUpId`).

## Step 0 — Ask the user first (wait for answers, never default)

Ask these one by one and wait for the actual answer (see CLAUDE.md workflow rules):

1. **Is the new process a finop?** Ask exactly: *"Is the new process a financial operation
   (finop)?"* This decides everything below:

   |                       | **Finop**                                    | **Non-finop**                    |
   |-----------------------|----------------------------------------------|----------------------------------|
   | Package               | `processes/finops2/xxx/`                     | `processes/xxx/`                 |
   | Reference             | `InternalTransfer`                           | `KybReview`                      |
   | ID parent             | `FinOpProcessId` (UUIDv5 from `FinOpId`)     | `ProcessId`                      |
   | State parent          | `FinOpState` (+ `FinOpState.*` markers)      | `ProcessState` (+ `ProcessState.*` markers) |
   | Command/Event parent  | `FinOpCommand` / `FinOpEvent`                | `ProcessCommand` / `ProcessEvent` (domain-common) |
   | Handler base          | `FinOpProcessCommandHandler[S, Command, Event]` | `ProcessCommandHandler[Task, S, C, E]` + local `ignoreF` |
   | Fn signature          | `Task[List[Event]]` (+ handler `maybePersist`) | `Task[Effect[E, S, Unit]]`       |
   | Common-command alias  | `PC`                                         | `CC`                             |
   | FinOpView             | connected (state extends `FinOpState`; wire-process registers `finOpViewReactor`) | NOT connected |
   | Serializers package   | `serializers/.../domain/finops2/xxx/`        | `serializers/.../domain/xxx/`    |

2. **Non-finop only — ID basis**: ask what the process ID is derived from — a UUIDv5 of
   some upstream identifier(s) (like `KybReviewId.make(kybId, relevanceTs)`) or a plain
   `UUID` (like `IbanAllocationId`). Finop IDs are always UUIDv5 derived from `FinOpId`
   — no question needed.

If anything else is ambiguous while scaffolding (naming, placement, an unexpected
conflict), stop and ask — never decide silently.

## Before scaffolding, check

- **Process or entity?** If it's an aggregate holding state mutated by external commands
  with replies (not a workflow that advances itself), it's an **entity** — stop, use
  **ev-entity** instead.
- **Name collision?** Grep `processes/` (and `processes/finops2/`) for the package name
  and `ids/StateId.scala` for `XxxId` before creating anything — a clash means asking
  the user, not renaming silently.

## Why the structure is shaped this way

The command handler is split into `handleCommand` (the process's own commands,
e.g. `Start`) and `handleCommonCommand` (`ProcessCommonCommand.Continue`, routed **by
current state** to the Fn that performs that state's work). The `Orchestration` inspects
the state after every event and decides `Idle` / `Running` / `Completed` / `Delayed` —
`Running` makes the guardian auto-send `Continue`, which is how the machine advances
without external input. Fns are one-job functions (`StartFn`, `InitFn`, …) taking their
dependencies via factory methods, so the handler stays a pure router and each unit of work
is independently testable (`fixed(...)` stubs).

Finop processes additionally plug into the shared FinOp machinery: `FinOpState` phase
markers (`Initializing`/`FundsProcessing`/`Executing`/`SettlementProcessing`/`Reverting`/
`Finalizing`/`Completed`), `FinOpCommonState` capability traits (`FundsProcessing`,
`AcceptsSettlement`, `AcceptsCryptoExecution`, `AcceptsKytTravelRuleProcessing`,
`AcceptsCompletion`), reusable `FinOpCommonEvent`s + common Fns (`ProcessFundsFn`,
`CompleteFn`, `SucceedSettlementFn`, …), and `BalanceUpdateChain` for internal funds
movement (reserve → approve → revert compensation). None of that is scaffolded — it is
what the user reaches for when implementing; the seed only fixes the file/type skeleton
so all of it can be added without restructuring.

## Files to create — finop variant

All in `modules/domain/.../processes/finops2/xxx/`, ID in `domain-common`, codecs in
`serializers`. Mirror **`InternalTransfer`** (`processes/finops2/internaltransfer/`) for
every naming/layout question the templates below don't answer.

### F1. ID type — edit `domain-common/.../ids/StateId.scala`

Add a namespace entry in `StaticNamespaces` — take the **next free** value in the existing
sequence (`...-800000000000`, `...-900000000000`, `...-110000000000`, …); do not renumber
existing ones (there is a known todo about the sequence — leave it):

```scala
  val XxxId: UUID = UUID.fromString("00000000-0000-0000-0000-<next-free>")
```

and the ID mirroring `SepaWithdrawalId`:

```scala
final case class XxxId(value: UUIDv5) extends FinOpProcessId {
  override def toString: String     = s"XxxId|$valueUrlSafe"
  override def valueUrlSafe: String = value.asString
}
object XxxId {
  given CanEqual[XxxId, XxxId] = CanEqual.derived
  def make(finOpId: FinOpId): Validated[XxxId] = {
    UUIDv5.fromStringInput(StaticNamespaces.XxxId, finOpId.value).map(XxxId.apply)
  }
  def ofUuidString(x: String): Validated[XxxId] = UUIDv5.fromUUIDv5String(x).map(apply)
}
```

Also add the Circe id codec to `serializers/.../domain/common/all/CommonIdsSerializers.scala`:

```scala
  given XxxIdCodec: Codec[XxxId] = deriveUnwrappedE(XxxId.ofUuidString)(_.valueUrlSafe)
```

### F2. Protocol — `XxxProtocol.scala`

`Event`/`Command` are **union type aliases** — the seed unions are single-membered; common
event families (`InitCommonEvent | FundsCommonEvent | ...`) and common command families
(`CommonFiatSettlementCommand`, …) get added to the unions during implementation.

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.common.ids.XxxId
import dev.fintech.domain.common.time.Timestamp
import dev.fintech.domain.processes.finops2.common.{FinOpCommand, FinOpEvent}

object XxxProtocol {

  type Event   = XxxEvent
  type Command = XxxCommand

  sealed trait XxxCommand extends FinOpCommand
  object XxxCommand {

    final case class Start(id: XxxId) extends XxxCommand with FinOpCommand.InitCommand

  }

  sealed trait XxxEvent extends FinOpEvent
  object XxxEvent {

    final case class Started(at: Timestamp) extends XxxEvent with FinOpEvent.InitEvent

  }

}
```

Real finop `Start` commands carry `details: FinOpDetails.XxxDetails` and
`txs: FinOpTransactions` — those (and the `FinOpDetails` case, the `Details` model in
`model.scala`) are added during implementation, not scaffolded.

### F3. State — `XxxState.scala`

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.common.finop.{CompletionStatus, FinOpId}
import dev.fintech.domain.common.ids.XxxId
import dev.fintech.domain.processes.finops2.common.FinOpState

sealed trait XxxState extends FinOpState {
  override def id: XxxId
}
object XxxState {

  def initial(id: XxxId): XxxState = NotStarted(id)

  final case class NotStarted(id: XxxId) extends XxxState with FinOpState.Initializing {
    override def maybeFinOpId: Option[FinOpId] = None
  }

  final case class Completed(id: XxxId, status: CompletionStatus) extends XxxState with FinOpState.Completed {
    override def maybeFinOpId: Option[FinOpId] = ???
  }

}
```

(`Completed.maybeFinOpId` becomes `Some(details.finOpId)` once the `Details` model exists.)

### F4. Syntax aliases — `syntax.scala`

Keep the exact alias letters — the finops2 convention is **`PC`** for
`ProcessCommonCommand` (never `CC` here; that is the non-finop/legacy alias). `FE`/`FC`
aliases join the file when common events/commands enter the unions.

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.common.time.Timestamp
import dev.fintech.libs.eventsourcing.model.ProcessCommonCommand

object syntax {

  type Event   = XxxProtocol.Event
  type Command = XxxProtocol.Command
  type E       = XxxProtocol.XxxEvent
  type S       = XxxState
  type PC      = ProcessCommonCommand

  val C: XxxProtocol.XxxCommand.type = XxxProtocol.XxxCommand
  val E: XxxProtocol.XxxEvent.type   = XxxProtocol.XxxEvent
  val PC: ProcessCommonCommand.type  = ProcessCommonCommand
  val S: XxxState.type               = XxxState

  extension (e: Event) {
    def at: Timestamp = e match {
      case e: E => e.at
    }
  }

}
```

### F5. Command handler — `XxxCommandHandler.scala`

A `class` over a `Dependencies` case class of Fns, with a `make` factory. The routing is
structure; the work is in the Fns.

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.processes.finops2.common.FinOpProcessCommandHandler
import dev.fintech.domain.processes.finops2.xxx.functions.StartFn
import dev.fintech.domain.processes.finops2.xxx.syntax.*
import dev.fintech.domain.utils.TimestampTaskFn
import dev.fintech.libs.eventsourcing.model.Effect
import dsl.Task

class XxxCommandHandler(
    deps: XxxCommandHandler.Dependencies
) extends FinOpProcessCommandHandler[S, Command, Event] {

  override protected def timestampFn: TimestampTaskFn = deps.timestampFn

  import deps.*

  override def handleCommand(state: S, command: Command): Task[Effect[Event, S, Unit]] = {
    command match {
      case c: C.Start => onStart(state, c)
    }
  }

  override def handleCommonCommand(state: S, command: PC): Task[Effect[Event, S, Unit]] = {
    command match {
      case PC.Continue => ignoreF
    }
  }

  private def onStart(state: S, command: C.Start): Task[Effect[Event, S, Unit]] = {
    startFn.apply(state, command).map(maybePersist)
  }

}
object XxxCommandHandler {

  final case class Dependencies(
      timestampFn: TimestampTaskFn,
      startFn: StartFn
  )

  def make(
      timestampFn: TimestampTaskFn
  ): XxxCommandHandler = new XxxCommandHandler(
    Dependencies(
      timestampFn = timestampFn,
      startFn     = StartFn(timestampFn)
    )
  )

}
```

When lifecycle states appear, `PC.Continue` grows a state match
(`case s: S.Initializing => initialize(s)` …) delegating to per-state Fns — see
`InternalTransferCommandHandler`; use the inherited `onState[S1]` / `onStateMatch` helpers
for command-vs-state guards.

### F6. Event handler — `XxxEventHandler.scala`

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.processes.finops2.xxx.syntax.*
import dev.fintech.libs.eventsourcing.model.ProcessEventHandler

object XxxEventHandler extends ProcessEventHandler[S, Event] {

  override def applyEvent(state: S, event: Event): S = ???

}
```

(Implemented later as one `mapIf[S.SomeState](state)(_.transition)` per event family.)

### F7. Orchestration — `XxxOrchestration.scala`

```scala
package dev.fintech.domain.processes.finops2.xxx

import dev.fintech.domain.processes.finops2.xxx.syntax.*
import dev.fintech.libs.eventsourcing.model.{Orchestration, OrchestrationDecision}

object XxxOrchestration extends Orchestration[S] {

  override def decide(state: S): OrchestrationDecision = {
    import OrchestrationDecision.*
    state match {
      case s: S.NotStarted => Idle
      case s: S.Completed  => Completed
    }
  }

}
```

Every intermediate state added later maps to `Running` (auto-`Continue`) or `Idle`
(waiting on an external command — e.g. a `Settling` state waiting for a webhook).

### F8. Functions — `functions/StartFn.scala`

Finop Fns return `Task[List[Event]]`; the handler persists via `maybePersist`. Every Fn
gets a `fixed(...)` stub factory for specs.

```scala
package dev.fintech.domain.processes.finops2.xxx.functions

import dev.fintech.domain.processes.finops2.xxx.syntax.*
import dev.fintech.domain.utils.TimestampTaskFn
import dsl.*

trait StartFn {
  def apply(state: S, command: C.Start): Task[List[Event]]
}
object StartFn {

  def fixed(events: List[Event]): StartFn = new StartFn {
    override def apply(state: S, command: C.Start): Task[List[Event]] = Task.pure(events)
  }

  def apply(timestampFn: TimestampTaskFn): StartFn = new StartFn {
    override def apply(state: S, command: C.Start): Task[List[Event]] = ???
  }

}
```

During implementation, reuse the common Fns from `finops2/common/functions/`
(`ProcessFundsFn`, `CompleteFn`, `SucceedSettlementFn`, `RejectSettlementFn`, KYT fns, …)
before writing new ones.

### F9. Codecs — `serializers/.../domain/finops2/xxx/XxxSerializers.scala`

Follow the **circe-codecs** skill (read it). The seed needs:

```scala
package dev.fintech.infra.serializers.domain.finops2.xxx

import dev.fintech.domain.processes.finops2.xxx.syntax.*
import dev.fintech.libs.circe.utils.*
import io.circe.*
import io.circe.generic.semiauto.deriveCodec

object XxxSerializers {

  import dev.fintech.infra.serializers.domain.common.CommonSerializers.given

  given E_StartedCodec: Codec[E.Started] = deriveCodec
  given XxxEventCodec: Codec[E] = {
    val discriminator = DiscriminatorCodecHelper[E]("_t")
    val decoder: Decoder[E] = cursor => {
      discriminator.decode(cursor) { case "Started" =>
        cursor.as[E.Started]
      }
    }
    val encoder: Encoder[E] = { case e: E.Started =>
      discriminator.encode(e, "Started")
    }
    Codec.from(decoder, encoder)
  }

  given S_NotStartedCodec: Codec[S.NotStarted] = deriveCodec
  given S_CompletedCodec: Codec[S.Completed]   = deriveCodec

  given XxxStateCodec: Codec[S] = {
    val discriminator = DiscriminatorCodecHelper[S]("_t")
    val decoder: Decoder[S] = cursor => {
      discriminator.decode(cursor) {
        case "NotStarted" => cursor.as[S.NotStarted]
        case "Completed"  => cursor.as[S.Completed]
      }
    }
    val encoder: Encoder[S] = {
      case s: S.NotStarted => discriminator.encode(s, "NotStarted")
      case s: S.Completed  => discriminator.encode(s, "Completed")
    }
    Codec.from(decoder, encoder)
  }

}
```

While `Event = E`, `XxxEventCodec` **is** the `Codec[Event]` — do NOT also define a
separate `EventCodec` (ambiguous givens). Once common events join the union, add the
orElse-combining union codec exactly as in `InternalTransferSerializers.EventCodec`
(own codec first, then `FinOpCommonEventSerializers.FinOpCommonEventCodec`).

### F10. No view files

Finop processes do **not** get a per-process view/repository/reactor. They project into
the shared `FinOpView` / `FinOpViewRepository` (`processes/finops2/views/`) automatically
because the state extends `FinOpState` — wire-process registers `finOpViewReactor` (plus
`txStatementReactor`, `cpTransactionsReactor`, `generalNotificationReactor`) in
`ReactorsEnv`. Nothing to author here.

## Files to create — non-finop variant

**MANDATORY — if Step 0 answered "non-finop", READ
[`references/non-finop.md`](references/non-finop.md) ENTIRELY** and follow its templates
for ID, protocol, state, syntax (`CC` alias), handler, and Fns — they override the finop
templates F1–F5/F8 above (F6/F7/F9 shapes are shared and referenced from there).
**Do NOT load it for a finop process** — the finop templates above are self-contained.

## NEVER

- **NEVER define the ID outside `ids/StateId.scala`** — `StateId`/`ProcessId` are sealed.
- **NEVER scaffold lifecycle states, `Details` models, `FinOpDetails` cases, or
  `BalanceUpdateChain` plumbing** — the seed is `NotStarted`/`Completed` + `Start`/`Started`
  only; lifecycle shape is implementation, not scaffolding.
- **NEVER implement the `???` stubs or invent domain fields** — structure only; a
  "helpfully" filled-in Fn injects unreviewed business logic.
- **NEVER mix the alias conventions** — finop syntax uses `PC`, non-finop uses `CC`;
  wiring and future edits pattern-match on these.
- **NEVER give a process a Reply ADT or `EntityCommand` parents** — processes are
  tell-only (`ProcessCommand extends Command[Any]`); replies are an entity concept.
- **NEVER add a per-process view for a finop process** — `FinOpView` is shared; a custom
  view duplicates the projection.
- **NEVER define both `Codec[E]` and `Codec[Event]` while `Event = E`** — ambiguous
  givens; the union codec appears only when the union gains a second member.
- **NEVER answer the Step 0 questions yourself or proceed on a timeout** — wait for the
  user's actual answer, then scaffold.

## Finish

1. `git add` every new file immediately (never anything under `.claude/`). `StateId.scala`
   and `CommonIdsSerializers.scala` are edits, not new files.
2. `sbt compile` (never trust IDE diagnostics alone).
3. Format only the touched files: `sbt "<module>/scalafmtOnly <path> ..."`.
4. **Hand off to wiring**: invoke the **wire-process** skill now and follow it end to end —
   it creates the `ProcessDefinition` (with `orchestration = Some(XxxOrchestration)`),
   guardian, journal definition (UUIDv5 ids use
   `PostgresJournalIdCodec.uuid[XxxId](_.value.value)(ofUUIDv5(XxxId.apply))`), reactor
   sinks/groups, and `*Env` wiring. All wire-process preconditions are satisfied by the
   files above.
5. Remind the user of the deferred implementation surface: lifecycle states + orchestration
   `Running` cases, `Details` model (`model.scala`), for finops the `FinOpDetails.XxxDetails`
   case + `txs: FinOpTransactions` on `Start`, common event/command unions, and the actual
   sender of `Start` (finops: usually `FinOpConfirmService`; deposits: webhook services;
   sweeps: a reactor).
