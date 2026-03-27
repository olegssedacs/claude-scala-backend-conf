# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workflow rules
- After creating new files, always `git add` them immediately.

## Build & Test Commands

This is an SBT (Scala Build Tool) project using Scala 3.3.3 on JVM 25.

```bash
sbt compile              # Compile all modules
sbt unit-test            # Run unit tests across all testable modules
sbt itest                # Run integration tests (requires Cassandra, PostgreSQL, Redis)
sbt itestLocal           # Integration tests with local config (debug logging)
sbt itestDocker          # Integration tests with Docker config (info logging)
sbt app/run              # Run the main application
sbt app/jibImageBuild    # Build Docker image
sbt scalafmtAll          # Format all Scala sources
sbt scalafmtCheckAll     # Check formatting without modifying
```

To run a single test class:
```bash
sbt "testOnly *MyTestClassName"
```

Code formatting: `.scalafmt.conf` — Scala 3 dialect, max 160 columns.

## Architecture

### Overview

Fintech backend application using **event sourcing** with a **CQRS** pattern:
- **Cassandra** stores the event journal (append-only event log)
- **PostgreSQL** stores view snapshots and configuration (migrated via Flyway)
- **Redis** for caching
- **MinIO** for object/file storage (S3-compatible)

The application is built with pure functional Scala using **Cats Effect** (IO monad), **HTTP4s** (HTTP server), **Circe** (JSON), **Doobie** (SQL), and **PureConfig** (configuration).

### Module Structure

- `modules/libs/dsl2cats` - Domain monad DSL for Cats Effect IO monad.
- **`modules/libs/`** — Reusable infrastructure libraries (eventsourcing, caching, database drivers, blockchain support, HTTP utilities)
- **`modules/domain-common/`** — Shared domain value objects, validation, and types
- **`modules/domain/`** — Core business logic organized by subdomain:
  - `finops` — Financial operations (core)
  - `entities` — Accounts, businesses, customers
  - `kyc` / `kyt` — KYC/KYT compliance
  - `transfers` — Money transfers
  - `trading` — Trading operations
  - `processes` — Long-running business workflows (state machines)
  - `reactors` — Event-driven side-effect handlers
  - `facades` — Service interfaces for complex operations
  - `providers` — Pluggable external provider integrations
- **`modules/infra/`** — Infrastructure layer:
  - `api` — HTTP4s REST API endpoints, JWT auth middleware, error handling
  - `auth` — JWT/JWS token management
  - `domain-adapters` — Adapters implementing domain interfaces
  - `serializers` — Circe JSON serialization
- **`modules/apis/`** — External API definitions (fintech-api, fireblocks-api, management-api)
- **`modules/app/`** — Application entry point, configuration loading, environment setup, Flyway migrations
- **`modules/control-panel/`** — Admin/control panel
- **`modules/simulators/`** — Local simulators for ClearJunction, currency rates, Fireblocks
- **`modules/i-tests/`** — Integration tests

### Key Patterns

- **Event Sourcing**: Domain events stored in Cassandra journal, state reconstructed by replaying events. Periodic snapshots written to PostgreSQL for read views.
- **Reactors**: Event-driven handlers that produce side effects in response to domain events.
- **Processes**: State machines for multi-step business workflows (e.g., payouts).
- **Facades**: Domain service interfaces abstracting complex multi-aggregate operations.
- **Providers**: Pluggable implementations for external services (KYC, notifications, currency rates).

### Layering

Domain layer is isolated from infrastructure. Dependency direction: `domain` ← `infra/domain-adapters` ← `infra/api` ← `app`. The domain defines interfaces; adapters implement them.

### Configuration

Main config: `modules/app/src/main/resources/application.conf` (HOCON format). Environment variables override defaults via `${?VAR_NAME}` pattern. PureConfig loads typed case classes.

The app runs on port **9090** by default. Health check endpoint: `/app/health/liveness`.

### CI/CD

GitLab CI (`.gitlab-ci.yml`):
- MR builds: compile + unit-test + itest
- Branch builds: compile + Docker image build (sbt-jib)
- Deployment: `dev` branch → dev env, `stage` → staging, `master` → production
- Build image: `scala-sbt:graalvm-community-25.0.0_1.11.6_3.7.3`

### Database Migrations

Flyway SQL migrations in `modules/app/src/main/resources/db/migration/app/` (V1–V35+).

## Code style
- use Scala 2 syntax with braces.
- Pure FP - no `var`, no `null`, no throwing exceptions.
- prefer `given` and `using` over `implicit`.
- `implicitConversions` is enabled
- `strictEquality` is enabled, derive `CanEqual` when necessary.
- prefer `opaque type` over `AnyVal` classes
- Prefer `enum` when all cases are instantiated inside the enum itself (fixed, closed set of known instances)
- Prefer `sealed trait` when cases are instantiated by callers (case classes with caller-supplied parameters)
- case classes must be final.
- For computational logic, prefer a trait with a factory method and anonymous impl 
  over a class. If a concept is essentially a function or a small set of functions, 
  define it as a trait, hide the implementation anonymously, and expose construction 
  via a factory method in the companion object:
  ```scala
    trait MyFn {
      def apply(params: Params): Task[Result]
    }
    object MyFn {
      def make(dep: Dependency): MyFn = new MyFn {
        override def apply(params: Params): Task[Result] = ???
      }
    } 
  ```
  Avoid: 
  ```scala 
    class MyFn(dep: Dependency) { def apply(params: Params) = ??? }
  ```
- keep comments minimal, add comments only for non-obvious logic or contract clarification.
