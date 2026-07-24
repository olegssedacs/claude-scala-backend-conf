# Non-finop variant — templates

Loaded only when Step 0 answered **non-finop**. Same file set and style as the finop
variant, but under `processes/xxx/` and `serializers/.../domain/xxx/`. Mirror
**`KybReview`** (`processes/kyb/`) for anything the templates below don't answer.
Differences from the finop templates:

- **ID** (`StateId.scala`): `final case class XxxId(...) extends ProcessId` — UUIDv5 with
  a `make(...)` from the basis the user named in Step 0 (mirror `KybReviewId`; if the
  basis warrants a `StaticNamespaces` entry, add one like `BusinessKybId`), or plain
  `UUID` (mirror `IbanAllocationId`). Same `CommonIdsSerializers` codec line as F1.
- **Protocol**: `XxxCommand extends ProcessCommand` / `XxxEvent extends ProcessEvent`
  (from `dev.fintech.domain.common.eventsourced`), commands override `def id: XxxId`,
  events carry `at: Timestamp`. No union aliases needed — plain `C`/`E`.
- **State**: `sealed trait XxxState extends ProcessState`; intermediate states mix in
  `ProcessState.Running`, terminal mixes `ProcessState.Completed`; no `maybeFinOpId`;
  keep `def initial(id: XxxId): XxxState = NotStarted(id)`.
- **Syntax** — the non-finop alias for `ProcessCommonCommand` is **`CC`**:

  ```scala
  object syntax {

    private val P: XxxProtocol.type = XxxProtocol

    type C  = P.XxxCommand
    type CC = ProcessCommonCommand
    type E  = P.XxxEvent
    type S  = XxxState

    val C: P.XxxCommand.type          = P.XxxCommand
    val CC: ProcessCommonCommand.type = ProcessCommonCommand
    val E: P.XxxEvent.type            = P.XxxEvent
    val S: XxxState.type              = XxxState
  }
  ```
- **Command handler**: extends `ProcessCommandHandler[Task, S, C, E]` directly (there is
  no non-finop helper base); Fn dependencies as plain constructor params (kyb style) with
  a companion `make`; define locally:

  ```scala
  private def ignoreF: Task[Effect[E, S, Unit]] = Task.pure(Effect.ignore(()))
  ```

  `handleCommonCommand(state, command: CC)` matches `CC.Continue => ignoreF` in the seed.
- **Fns** return `Task[Effect[E, S, Unit]]` directly (persist via
  `Effect.persistOne(event, ())`), body `???` in the seed:

  ```scala
  trait StartFn {
    def apply(state: S, cmd: C.Start): Task[Effect[E, S, Unit]]
  }
  object StartFn {
    def apply(timestampFn: TimestampTaskFn): StartFn = new StartFn {
      override def apply(state: S, cmd: C.Start): Task[Effect[E, S, Unit]] = ???
    }
  }
  ```
- **Event handler / orchestration / codecs**: same shapes as F6/F7/F9 in SKILL.md (state
  codec cases for `NotStarted`/`Completed`, event codec for `Started`; no union-codec
  concern).
- **No FinOpView connection** — a non-finop process gets no domain view at all in the
  seed (reactors registered at wiring time are typically just
  `generalNotificationReactor`; the shared `process_status` table tracks lifecycle).