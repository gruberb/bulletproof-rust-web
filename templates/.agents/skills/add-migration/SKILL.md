# Skill: Add Migration

Add a database migration safely.

## Steps

1. **Create the migration file.**
   ```
   cargo sqlx migrate add <description>
   ```
   This creates a timestamped `.sql` file in `migrations/`.

2. **Write the SQL.** Follow these conventions:
   - `UUID PRIMARY KEY DEFAULT gen_random_uuid()` for IDs.
   - `TIMESTAMPTZ NOT NULL DEFAULT NOW()` for timestamps.
   - Add indexes on columns used in WHERE/JOIN.
   - Add the `update_updated_at` trigger if the table has `updated_at`.
   - Consider case-insensitive uniqueness (citext or ICU collation) for user-facing text fields.

3. **Check backward compatibility.** Before applying:
   - Will the current running application still work against the new schema?
   - If adding a NOT NULL column, does it have a DEFAULT?
   - If renaming or dropping a column, is the old application still deployed?
   - For destructive changes, use expand/contract: add the new structure first, migrate data, then remove the old structure in a later migration.

4. **Apply and prepare.**
   ```
   cargo sqlx migrate run
   cargo sqlx prepare
   ```

5. **Update queries.** If any `query_as!` macros reference changed columns or tables, update them now.

6. **Verify.**
   ```
   cargo sqlx prepare --check
   cargo test --workspace
   ```

7. **Commit.** Commit both the migration file and the updated `.sqlx/` metadata directory.
