---
name: scala-specs
description: >
  Generate Scala 3 unit test specs following this project's conventions.
  Use when the user asks to write specs, unit tests, tests, add test coverage,
  create tests, or cover code with tests for any module in this codebase.
  Handles test style selection (AnyWordSpec vs AsyncFunSuite vs AsyncFreeSpec),
  dependency stubbing via anonymous trait implementations, event sourcing test
  patterns, and serialization specs. Trigger on: specs, unit tests, tests, spec,
  write tests, add tests, create tests, cover with tests, test coverage, add spec.
---

# Scala Specs

## Before Writing Tests, Ask Yourself

1. **Purity**: Does the target use `Task`/`IO`? This determines the test style.
2. **Test body size**: Will individual test cases be small (≤15 lines) or large (>15 lines)?
3. **Dependencies**: Are they traits/interfaces? If not, you cannot stub them — notify the user.
4. **Existing fixtures**: Check if `*Samples`, `*TestData` objects exist in the same test package before creating inline data.
5. **CanEqual**: Will `shouldBe` comparisons need `given CanEqual` derivations for the types involved?
6. **Validation types**: Does the target use validated/opaque types that need `.getUnsafe` or `unsafe.given`?

## Style Selection

| Condition | Extends | Key Mixins |
|-----------|---------|------------|
| Pure code, small test cases (≤15 lines each) | `AnyWordSpec` | `Matchers` |
| Pure code, large test cases (>15 lines) or large ADT transitions | `AsyncFunSuite` | `AsyncTaskSpec`, `Matchers` |
| Effectful code (`Task`/`IO`-based) | `AsyncFreeSpec` | `AsyncTaskSpec`, `Matchers` |

### Module-specific mixins

| Module | Mix in | Provides |
|--------|--------|----------|
| `domain-common` | `DomainSpec` | `given Conversion` for `Int/Long ↔ SubUnits`, `Double ↔ ExchangeRate` |
| `domain` | `DomainSpecExt` | `fixedTime: Timestamp`, `TimestampFn.fixed()/.clock()/.now()`, `random.customerId()/.brandId()/.iosUdid()/.pushNotificationId()`, `given Conversion` for `String → TxId`, `Int/Long ↔ SubUnits` |
| `domain` (processes) | `BaseProcessSpec[S, C, E]` | `specCase(...)`, `haveStep(n).as[EventType, StateType]` matcher — for testing event-sourced process state machines |
| `domain-adapters` (JSON codecs) | `BaseSerializersSpec` | `decodeJsonFile[A]()`, `decodeJson[A]()`, `readJsonFile()`, `printJson[A]()` — loads from test resources |

## Project-Specific Deviations from Standard ScalaTest

These are things Claude wouldn't know without reading this codebase:

- Import `dsl.*` and `dsl.spec.AsyncTaskSpec` — NOT `cats.effect.testing.scalatest.AsyncIOSpec` directly. `Task` is the project alias for `IO`.
- `strictEquality` is enabled. When `shouldBe` fails to compile, add: `given CanEqual[MyType, MyType] = CanEqual.derived`. Common types needing this: `FiniteDuration`, `LocalDateTime`, `ZonedDateTime`, `Instant`, `ProcessStatus`, custom case classes with type parameters like `EventSourcedResult[_]`, `Option[Preference[_]]`, `DummyCommand[A]`.
- `implicitConversions` is enabled — `DomainSpec`/`DomainSpecExt` traits provide ergonomic conversions (e.g., `SubUnits(1000)` instead of `SubUnits(1000L).getUnsafe`).
- For validated/opaque types, import `dev.fintech.domain.common.validation.getUnsafe` for the `.getUnsafe` extension, and `dev.fintech.domain.common.validation.unsafe.given` for given unsafe conversions.
- Test file naming: always `*Spec.scala` (not `*Test.scala`).
- Package mirrors source: `src/main/scala/dev/fintech/foo/Bar.scala` → `src/test/scala/dev/fintech/foo/BarSpec.scala`.

## Stubbing Dependencies

Create anonymous trait implementations — never use mocking frameworks or `MockableMethod`/`Answer` from `dev.fintech.domain.mocks`.

### Pattern 1: Simple stub returning fixed data

```scala
val stubProvider: CurrencyRatesProvider = new CurrencyRatesProvider {
  override def get(brandId: BrandId): Task[CurrencyRates] = Task.pure(rates)
}
```

### Pattern 2: Configurable factory function

```scala
def stubProvider(result: Option[ExchangePair]): CurrencyRatesProvider = new CurrencyRatesProvider {
  override def get(brandId: BrandId): Task[CurrencyRates] = Task {
    CurrencyRates.of(Timestamp.Epoch, Some(EUR.id), result.toList)
  }
}
```

### Pattern 3: Tracking invocations with `Ref`

When you need to verify a stub was called or count invocations, use `cats.effect.Ref`:

```scala
case class ProviderStub(rates: CurrencyRates, invocations: Ref[Task, Int]) extends FairCurrencyRatesProvider {
  override def get: Task[CurrencyRates] = invocations.update(_ + 1) *> Task(rates)
}
def providerStub(rates: CurrencyRates): Task[ProviderStub] = Ref[Task].of(0).map(ProviderStub(rates, _))
```

### Pattern 4: Partial implementation with `???`

When only some methods are exercised, use `???` for unneeded ones:

```scala
new CommissionCalculator {
  override def calculate(...): Task[Option[Commission]] = ???
  override def transferable(...): Task[MoneyAmount] = Task(MoneyAmount.PositiveAmount(SubUnits(10)))
}
```

### If a dependency is NOT a trait

If a dependency is a concrete class that cannot be anonymously implemented, tell the user:
> `SomeDependency` is a concrete class, not a trait — cannot create an anonymous stub. Consider extracting a trait or advise on how to proceed.

## Process Test Pattern (BaseProcessSpec)

For testing event-sourced process state machines (multi-step workflows with orchestration), use `BaseProcessSpec[S, C, E]` from `dev.fintech.domain.processes.spec`.

### How it works

`BaseProcessSpec` provides two things:
- `specCase(commandHandler, eventHandler, orchestration)` — creates a `ProcessSpecCase` that drives the process without an ActorSystem
- `haveStep(index).as[EventType, StateType]` — ScalaTest matcher for verifying the event and resulting state at each step

`ProcessSpecCase` has two run modes:
- `runOneStep(state, command)` — executes one command, returns `EvReport`
- `runOrchestrated(state, commands)` — drives the full process lifecycle. Before consuming each command from the list, it checks the orchestration decision: if `Running`, it automatically sends `ProcessCommonCommand.Continue` instead of consuming the next command. If `Idle`/`Paused`, it consumes the next user command. If `Completed`, it stops.

Each `EvStep` in the report holds a single `event` and the `state` after applying that event. One command producing 3 events yields 3 steps.

### Structure

```scala
class MyProcessSpec extends AsyncFreeSpec with AsyncTaskSpec with Matchers with DomainSpecExt
    with BaseProcessSpec[S, Command, Event] {

  // 1. Import syntax aliases (E, FE, FC, C, S, PC) from the process package
  import myprocess.syntax.*

  // 2. Build stubs — prefer .fixed() on companion objects for simple cases,
  //    anonymous trait implementations for state-dependent logic
  private val myFnStub = MyFn.fixed(Some(FE.SomeEvent(fixedTime)))
  private val complexStub = new ComplexFn {
    override def apply(state: SomeState): Task[List[SomeEvent]] = {
      state match {
        case s: SomeState.CaseA => Task.pure(List(FE.EventA(fixedTime)))
        case _                  => Task.pure(List(FE.EventB(fixedTime)))
      }
    }
  }

  // 3. Wire up the ProcessSpecCase
  private val victim = specCase(
    commandHandler = new MyCommandHandler(MyCommandHandler.Dependencies(
      timestampFn = TimestampTaskFn.mk(TimestampFn.fixed()),
      myFn        = myFnStub,
      complexFn   = complexStub
      // ...
    )),
    eventHandler  = new MyEventHandler,
    orchestration = MyOrchestration
  )

  // 4. Test scenarios using runOrchestrated
  "Scenario: happy path" - {
    "should complete successfully" in {
      val commands: List[Command] = List(
        C.Start(...),
        FC.SomeCommand(...)
      )
      for {
        report <- victim.runOrchestrated(initialState, commands)
      } yield {
        report should haveStep(0).as[E.Started, S.Initializing]
        report should haveStep(1).as[FE.SomeEvent, S.Processing]
        // ...
        report.state shouldBe a[S.Completed]
      }
    }
  }
}
```

### Key conventions

- **Use syntax prefixes** for all events and commands: `E.Started`, `FE.TxHashUpdated`, `FC.UpdateTxHash`, `C.Start` — import them from the process's `syntax` object.
- **Validated/opaque types**: add explicit type annotations and rely on `unsafe.given` implicit conversions instead of calling `.getUnsafe`:
  ```scala
  private val depositId: CryptoDepositId = CryptoDepositId.make(FinOpProviderRef("ref-1"))
  private val txHash: CryptoTransactionHash.Ethereum = CryptoTransactionHash.Ethereum("0x" + "a" * 64)
  ```
- **Stub functions** with `.fixed()` when they produce a constant event regardless of input. Use anonymous implementations when the stub needs to inspect state or input to decide which event to return.
- **`TimestampTaskFn`**: wrap `TimestampFn.fixed()` with `TimestampTaskFn.mk(TimestampFn.fixed())` — the command handler expects `TimestampTaskFn`, not `TimestampFn`.

Reference implementation: `CryptoDepositSpec` in `modules/domain/src/test/scala/dev/fintech/domain/processes/finops2/cryptodeposit/`.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `shouldBe` won't compile | `strictEquality` — missing `CanEqual` | Add `given CanEqual[X, X] = CanEqual.derived` |
| `value is not a member` | Opaque type — underlying type methods hidden | Use the opaque type's own API (e.g., `.unwrap`, `.value`) |
| `no implicit argument of type Conversion` | Need validation imports | Add `import dev.fintech.domain.common.validation.unsafe.given` |
| Test compiles but runtime `ClassCastException` | Incorrect `CanEqual` covariance | Use exact type parameters in `CanEqual`, e.g., `CanEqual[Option[Preference[_]], ...]` |
| `getUnsafe` not found | Missing extension import | Add `import dev.fintech.domain.common.validation.getUnsafe` |

## NEVER

- **NEVER use mocking frameworks** (Mockito, ScalaMock) or `MockableMethod`/`Answer` — this project uses anonymous trait implementations exclusively. Mocking frameworks hide behaviour and break when trait signatures change.
- **NEVER use `AnyFlatSpec`** — not used anywhere in this project; `AnyWordSpec` is the standard for synchronous specs.
- **NEVER import `cats.effect.testing.scalatest.AsyncIOSpec` directly** — use `dsl.spec.AsyncTaskSpec` instead, which wraps it with the project's `Task` type alias.
- **NEVER use `var`, `null`, or throw exceptions** in test code — pure FP applies to tests too. Use `Either`, `Option`, `Task.raiseError`, or `.getUnsafe` for test convenience.
- **NEVER create `*Samples` or `*TestData` objects** unless the data is shared across 3+ spec files — inline test data instead.
- **NEVER use indentation-based Scala 3 syntax** — this project uses Scala 2 syntax with braces.