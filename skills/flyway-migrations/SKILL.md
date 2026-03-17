---
name: flyway-migrations
description: Create or update Flyway SQL migrations for this repo. Use when adding a new view-model table, changing columns on projection tables, adding indexes, or resetting reactor projections in `modules/app/src/main/resources/db/migration/app`.
---

# Flyway Migrations

## Keywords
flyway, migration, sql, postgres, database, schema, index

## Rules

- File naming: `V{version}__{description}.sql`
- Use upper case SQL keywords.
- Write SQL in Postgres dialect.
- `BIGSERIAL PRIMARY KEY` for auto-incrementing primary keys.
- `TIMESTAMP` (not `TIMESTAMPTZ`).
- `BIGINT` for monetary amounts (stored in minor units).
- Index naming: `idx_{table}_{columns}`
- Constraint naming: `{table}_unique` or `unique_{table}_{cols}`
- `IF NOT EXISTS` / `IF EXISTS` guards for idempotency.
- `UNIQUE` constraints for idempotent upserts.

## VARCHAR Widths

| Width | Usage |
|-------|-------|
| 16 | account numbers |
| 30 | types, enums (`item_type`, `travel_rule_type`) |
| 50 | currency, swift, state names |
| 100 | status |
| 128 | hashes |
| 200 | names, addresses |
| 500 | crypto addresses, tx hashes |
| 1000 | purpose, reference |

## View-Model Table: Drop+Recreate Sequence

View-model tables are projections rebuilt by reactors. NEVER use `ALTER TABLE` — always drop and recreate so reactors re-project all data.

```sql
-- 1. Drop indexes first
DROP INDEX IF EXISTS idx_my_view_...;

-- 2. Drop the table
DROP TABLE IF EXISTS my_view_table;

-- 3. Clean reactor sequence numbers (CRITICAL — reactors won't re-project without this)
DELETE
FROM reactor_seq_nr
WHERE name IN ('MyViewReactor', 'AnotherRelatedReactor');

-- 4. Create the table
CREATE TABLE my_view_table
(
    id         BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    CONSTRAINT my_view_table_unique UNIQUE (some_id, other_id)
);

-- 5. Create indexes
CREATE INDEX idx_my_view_table_some_col ON my_view_table (some_id, created_at DESC);
```

## Decision: When to Drop+Recreate vs Additive

- **View-model / projection table** → always drop+recreate + reactor_seq_nr cleanup
- **Non-view table** (config, domain state) → additive only (`ALTER TABLE ADD COLUMN`, new indexes)

## NEVER

- NEVER use `TIMESTAMPTZ` — project uses `TIMESTAMP`
- NEVER use `NUMERIC` / `DECIMAL` for monetary amounts — use `BIGINT` (minor units)
- NEVER `ALTER TABLE` on view-model tables — drop and recreate
- NEVER recreate a view-model table without `DELETE FROM reactor_seq_nr` for its reactors
- NEVER modify an existing migration — always add a new version

## Migrations Location

- `modules/app/src/main/resources/db/migration/app`

## Reactor List

Reactor names registered in `ReactorsEnv.scala`.

### Entity views (entityView)
- `domain.CustomerViewReactor`
- `domain.BrandViewReactor`
- `domain.BusinessViewReactor`
- `domain.InvoiceViewReactor`
- `domain.GlobalSettingsViewReactor`

### Model views (modelView)
- `domain.CustomerAccountPathReactor`
- `domain.CustomerPendingActionViewReactor`
- `domain.CustomerRecurringTransferViewReactor`
- `view.CustomerPhoneContactsReactor`
- `domain.BrandAccountPathReactor`
- `domain.BusinessPendingActionViewReactor`
- `domain.InvoiceAccountPathReactor`
- `domain.CurrencyExchangeTxStatementReactor`
- `domain.CurrencyExchangeFinOpViewReactor`
- `domain.InternalTransferTxStatementReactor`
- `domain.InternalTransferFinOpViewReactor`
- `domain.SepaDepositTxStatementReactor`
- `domain.SepaDepositFinOpViewReactor`
- `domain.SepaWithdrawalTxStatementReactor`
- `domain.SepaWithdrawalFinOpViewReactor`
- `domain.CryptoDepositTxStatementReactor`
- `domain.CryptoDepositWalletSweepReactor`
- `domain.CryptoDepositFinOpViewReactor`
- `domain.InvoiceProcessEventReactor`
- `domain.CryptoWithdrawalTxStatementReactor`
- `domain.CryptoWithdrawalFinOpViewReactor`

### Side-effect only (function)
- `domain.CustomerEventReactor`
- `domain.CustomerNotificationReactor`
- `domain.BrandNotificationReactor`
- `domain.BusinessNotificationReactor`
- `domain.KycReviewNotificationReactor`
- `domain.IbanAllocationReactor`
- `domain.CurrencyExchangeNotificationReactor`
- `domain.InternalTransferNotificationReactor`
- `domain.SepaDepositNotificationReactor`
- `domain.SepaWithdrawalNotificationReactor`
- `domain.CryptoDepositNotificationReactor`
- `domain.CryptoWithdrawalNotificationReactor`
- `domain.InvoiceEventReactor`


