# Add Migration

## Create Migration
```bash
dotnet ef migrations add <MigrationName> --project src/<DataProject>
dotnet ef database update --project src/<DataProject>
```

## Golden Rules
1. **Every migration must be reversible** — implement both Up and Down.
2. **Never drop columns or tables with data in production** — add new, migrate data, then remove old.
3. **Test both up and down** locally before committing.
4. **One concern per migration** — don't mix schema changes with data migrations.

## Risk Table

| Operation | Risk | Notes |
|-----------|------|-------|
| Add column (nullable) | Low | Safe, no data impact |
| Add column (non-null) | Medium | Requires default value |
| Add index | Low | May lock table briefly on large datasets |
| Rename column | High | Breaking change — update all code references first |
| Drop column | High | Data loss — ensure no code references remain |
| Drop table | Critical | Irreversible data loss — requires explicit approval |
| Change column type | High | May fail if data can't be cast |

## Verification
1. Run `dotnet ef migrations script` to review raw SQL.
2. Apply to a fresh database: `dotnet ef database update`.
3. Roll back: `dotnet ef database update <PreviousMigration>`.