---
name: circe-codecs
description: >
  Create and modify Circe JSON codecs for domain types in the serializers module.
  Use when the user asks to write codecs, serializers, encoders, decoders, or JSON
  serialization for domain types. Covers both domain-common codecs (common/all/ traits)
  and domain-level codecs (events, states, commands). Includes discriminated ADT codecs
  with DiscriminatorCodecHelper, opaque type codecs, codec tests with JSON fixtures,
  and the verifyJson round-trip pattern.
  Trigger on: codecs, serializers, circe, json codec, encoder, decoder, serialization,
  json serialization, add codec, write codec, create serializer.
---

# Circe Codecs

## Before Writing Codecs, Ask Yourself

1. **Where does the type live?** `domain-common` types → `common/all/CommonXxxSerializers.scala` trait. `domain` types → `domain/xxx/XxxSerializers.scala` object.
2. **What kind of type?** Opaque type, case class, sealed trait/enum ADT, or wrapper?
3. **Does a codec for this type already exist?** Check existing `Common*Serializers` traits — the type may already be covered.
4. **What common codecs are needed?** Domain-level codecs depend on common codecs via `import CommonSerializers.given`.

## File Placement

| Type origin | Codec location | Pattern |
|-------------|---------------|---------|
| `domain-common` package `dev.fintech.domain.common.foo` | `common/all/CommonFooSerializers.scala` trait | Trait extending another common trait |
| `domain` module entity/process | `domain/xxx/XxxSerializers.scala` object | Object importing common `.given` |
| Shared finops types | `finops/common/FinOpCommonSerializers.scala` | Object with common finop codecs |

All codec files live under `modules/infra/serializers/src/main/scala/dev/fintech/infra/serializers/domain/`.

## Codec Patterns

### Opaque types (validated wrappers)

For types extending `Opaque[T, R]` from `dev.fintech.domain.common.validation`:

```scala
import dev.fintech.infra.serializers.domain.opaque.opaqueTypeCodec

given MyTypeCodec: Codec[MyType] = opaqueTypeCodec(MyType)
```

This handles validation on decode and unwrapping on encode. For key encoders/decoders on opaque types:

```scala
import dev.fintech.infra.serializers.domain.opaque.{opaqueTypeKeyEncoder, opaqueTypeKeyDecoder}

given MyTypeKeyEncoder: KeyEncoder[MyType] = opaqueTypeKeyEncoder(MyType)
given MyTypeKeyDecoder: KeyDecoder[MyType] = opaqueTypeKeyDecoder(MyType)
```

### Simple case classes (deriveCodec)

```scala
import io.circe.generic.semiauto.deriveCodec

given E_StartedCodec: Codec[E.Started] = deriveCodec
```

### Wrapper types (deriveUnwrapped / deriveUnwrappedE)

For types that wrap a single value:

```scala
import dev.fintech.libs.circe.utils.{deriveUnwrapped, deriveUnwrappedE}

// Infallible unwrap:
given CustomerIdCodec: Codec[CustomerId] = deriveUnwrapped(CustomerId.apply)(_.value)

// Fallible unwrap (with validation):
given ExchangeRateCodec: Codec[ExchangeRate] = deriveUnwrappedE(ExchangeRate.fromString)(_.value.underlying().toPlainString)
```

### Sealed trait / enum ADTs (DiscriminatorCodecHelper)

The project uses a `_t` discriminator field for tagged union serialization:

```scala
import dev.fintech.libs.circe.utils.DiscriminatorCodecHelper

given MyAdtCodec: Codec[MyAdt] = {
  val discriminator = DiscriminatorCodecHelper[MyAdt]("_t")
  val decoder: Decoder[MyAdt] = cursor => {
    discriminator.decode(cursor) {
      case "VariantA" => cursor.as[MyAdt.VariantA]
      case "VariantB" => cursor.as[MyAdt.VariantB]
    }
  }
  val encoder: Encoder[MyAdt] = {
    case v: MyAdt.VariantA => discriminator.encode(v, "VariantA")
    case v: MyAdt.VariantB => discriminator.encode(v, "VariantB")
  }
  Codec.from(decoder, encoder)
}
```

Discriminator values use simple class names (e.g., `"Started"` not `"E_Started"`).

### ADT subset codecs (iemap narrowing)

When you need a codec for a sealed subtrait of an existing ADT, reuse the parent ADT codec and narrow with `iemap`. This avoids duplicating the discriminator logic — the parent codec handles encoding/decoding, and `iemap` filters to the desired subtype on decode:

```scala
given ParentAdtCodec: Codec[ParentAdt] = { /* full discriminator codec */ }

given SubtraitCodec(using codec: Codec[ParentAdt]): Codec[ParentAdt.Subtrait] = {
  codec.iemap {
    case acc: ParentAdt.Subtrait => Right(acc)
    case other                   => Left(s"decoding failed '${other.getClass.getSimpleName}' to 'ParentAdt.Subtrait'")
  }(identity)
}
```

The `using codec: Codec[ParentAdt]` parameter resolves the parent codec from scope. The `identity` encoder works because the subtrait is already a valid `ParentAdt` — the parent encoder handles it directly.

Use this pattern whenever you need codecs for intermediate sealed traits in the type hierarchy (e.g., `FinOpAccount.External`, `FinOpAccount.Internal`, `FinOpAccount.BelongsToCustomer` all derived from the full `FinOpAccount` codec).

### Null-excluding codecs

For types where null JSON fields should be omitted:

```scala
import dev.fintech.libs.circe.utils.deriveCodecExcludeNull

given MyTypeCodec: Codec.AsObject[MyType] = deriveCodecExcludeNull
```

### Missing-or-null defaults

For backward-compatible deserialization of optional collections:

```scala
import dev.fintech.libs.circe.utils.MissingOrNull.ListAsEmpty.given  // List defaults to Nil
import dev.fintech.libs.circe.utils.MissingOrNull.MapAsEmpty.given   // Map defaults to Map.empty
```

## Naming Conventions

### Codec val names

Use a prefix matching the type alias from the process/entity `syntax` object:

| Type category | Prefix | Example |
|--------------|--------|---------|
| Process events | `E_` | `given E_StartedCodec: Codec[E.Started] = deriveCodec` |
| Process states | `S_` | `given S_NotStartedCodec: Codec[S.NotStarted] = deriveCodec` |
| Entity events | `XE_` (type alias) | `given CE_RegistrationStartedCodec: Codec[CE.RegistrationStarted]` |
| Entity states | `XS_` (type alias) | `given CS_InitialCodec: Codec[CS.Initial]` |
| Master ADT codec | Full name | `given CryptoDepositEventCodec: Codec[E]` |
| Common types | Type name | `given SubUnitsCodec: Codec[SubUnits]` |

Always append `Codec` suffix. For key encoders/decoders append `KeyEncoder`/`KeyDecoder`.

### Discriminator values in JSON

Use the simple class name without prefix: `"Started"`, `"NotStarted"`, `"Completed"` — never `"E_Started"`.

## Common Serializer Trait Structure

Common codecs are `trait`s that extend other common traits for dependency ordering:

```scala
package dev.fintech.infra.serializers.domain.common.all

trait CommonFooSerializers extends CommonBarSerializers {
  // codecs for types in dev.fintech.domain.common.foo
  given MyTypeCodec: Codec[MyType] = opaqueTypeCodec(MyType)
}
```

The aggregator `CommonSerializers` extends all individual common traits. Domain-level codecs import them via:

```scala
import dev.fintech.infra.serializers.domain.common.CommonSerializers.given
```

## Codec Ordering

Define codecs in dependency order — `deriveCodec` needs all field codecs defined *above* it in the same file. If `deriveCodec` fails with "could not find implicit", the field's codec is likely defined below or not imported.

Within a domain serializer object, follow: helper types → individual variants → master discriminator codec. Within common traits, base types before compound types.

## Domain Serializer Object Structure

Domain codecs are `object`s that import common codecs:

```scala
package dev.fintech.infra.serializers.domain.xxx

object XxxSerializers {
  import dev.fintech.infra.serializers.domain.common.CommonSerializers.given
  // additional domain-specific imports as needed

  // 1. Helper type codecs
  given SomeHelperCodec: Codec[SomeHelper] = deriveCodec

  // 2. Individual event codecs
  given E_StartedCodec: Codec[E.Started]     = deriveCodec
  given E_CompletedCodec: Codec[E.Completed] = deriveCodec

  // 3. Master event discriminator codec
  given EventCodec: Codec[E] = { /* DiscriminatorCodecHelper pattern */ }

  // 4. Individual state codecs
  given S_InitialCodec: Codec[S.Initial]     = deriveCodec
  given S_ProcessingCodec: Codec[S.Processing] = deriveCodec

  // 5. Master state discriminator codec
  given StateCodec: Codec[S] = { /* DiscriminatorCodecHelper pattern */ }
}
```

## Testing Codecs

### Test file structure

- **Spec file**: `src/test/scala/.../XxxEventsSerializersSpec.scala` extends `AnyFunSuite with BaseSerializersSpec` (or a domain-specific base like `BaseFinOpsSerializersSpec`)
- **JSON fixtures**: `src/test/resources/xxx/events/XxxEvent.VariantName.json`

### Test pattern

```scala
class MyEventsSerializersSpec extends AnyFunSuite with BaseSerializersSpec {
  import MySerializers.given

  test("MyEvent.Started") {
    val obj: E = E.Started(details = ..., at = Samples.fixedTime)
    verifyJson(obj, "xxx/events/MyEvent.Started.json")
  }
}
```

`verifyJson` does a full round-trip: encodes the Scala object to JSON, reads the fixture file, decodes the fixture JSON back to Scala, then asserts both the JSON and object representations match.

### JSON fixture format

```json
{
  "_t": "Started",
  "details": { ... },
  "at": 1000000
}
```

- `_t` field for discriminated ADTs
- Timestamps as epoch milliseconds (Long)
- UUIDs as strings
- Opaque types as their underlying representation
- Nested discriminated types have their own `_t` fields

### Fixture file naming

`{DomainType}.{VariantName}.json` — e.g., `CryptoDepositEvent.Started.json`, `Customer.FullCustomer.json`

Organize under `src/test/resources/` mirroring the domain: `events/` for events, `states/` for states.

## Adding a New ADT Variant

When adding a new event or state variant to an existing ADT:

1. Add individual codec: `given E_NewVariantCodec: Codec[E.NewVariant] = deriveCodec`
2. Add decoder case: `case "NewVariant" => cursor.as[E.NewVariant]`
3. Add encoder case: `case e: E.NewVariant => discriminator.encode(e, "NewVariant")`
4. Add test: construct the object, write JSON fixture, call `verifyJson`

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `could not find implicit value for parameter encoder: Encoder[X]` | Missing codec for field type `X` | Add/import the codec for `X` — check if it belongs in a `Common*Serializers` trait |
| `diverging implicit expansion for type Codec[X]` | Circular codec dependency | Define one codec manually with `Codec.from(decoder, encoder)` to break the cycle |
| `No given instance of type Codec[X] was found` | Common codec not imported | Add `import CommonSerializers.given` or the specific serializer's `.given` |
| `deriveCodec` compiles but produces wrong JSON field names | Case class field names differ from expected | Check if the case class uses `override val` — Circe derives from constructor params |
| `verifyJson` test fails with JSON mismatch | Field ordering or null handling | Use `deriveCodecExcludeNull` if nulls should be omitted, check field order in fixture |

## NEVER

- **NEVER use `_t` values with prefixes** — discriminator values are simple class names (`"Started"` not `"E_Started"`)
- **NEVER put domain-common codecs in domain-level serializer objects** — they belong in `common/all/` traits so all domain serializers can reuse them
- **NEVER forget to add the new codec as `given`** — Scala 3 semi-auto derivation resolves field codecs via `given` scope. A missing `given` causes `deriveCodec` to fail with a cryptic "could not find implicit" error pointing at the parent type, not the missing field codec.
- **NEVER use `Encoder.AsObject` or manual JSON building** when `deriveCodec` works — manual codecs drift from case class fields when fields are added/renamed, causing silent serialization bugs.
- **NEVER skip the test** — codec regressions are silent at compile time but cause runtime deserialization failures when reading from the Cassandra event journal or PostgreSQL snapshots. A `verifyJson` test catches these before they reach production.
