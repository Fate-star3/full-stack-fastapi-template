---
name: db-migrate
description: Create and apply Alembic database migrations after model changes
---

# Database Migration Workflow

## Create a Migration

After modifying `backend/app/models.py`, generate a migration:

```bash
docker compose exec backend bash -c "alembic revision --autogenerate -m 'Your description'"
```

The new migration file appears in `backend/app/alembic/versions/`.

## Review Before Applying

Always read the generated migration file. Auto-generation catches most changes but can miss:
- Custom index names
- Data migration alongside schema changes
- `ALTER TYPE` for enum changes (requires manual SQL)

See `.claude/rules/db-conventions.md` for detailed migration conventions.

## Apply the Migration

```bash
docker compose exec backend bash -c "alembic upgrade head"
```

This also runs automatically via `scripts/prestart.sh` on next `docker compose up`.

## Rollback

To undo the last migration:

```bash
docker compose exec backend bash -c "alembic downgrade -1"
```

## Remove All Migrations (Fresh Start)

If you want to reset the database schema without migrations:

1. Delete all files in `backend/app/alembic/versions/`
2. Uncomment `SQLModel.metadata.create_all(engine)` in `backend/app/core/db.py`
3. Comment out `alembic upgrade head` in `scripts/prestart.sh`
4. Delete the database volume: `docker compose down -v`
