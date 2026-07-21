---
paths: ["backend/app/alembic/**/*"]
---

# Database Migration Conventions

## Migration Files

- Naming: `verb_noun` (e.g., `add_user_avatar_field`, `create_tag_table`)
- One migration per logical change — don't bundle unrelated schema changes
- Always review auto-generated migrations before applying; auto-generation can miss:
  - Custom index names (use explicit names, not auto-generated ones)
  - `ALTER TYPE` for enum changes (requires manual SQL — Alembic cannot detect these)
  - Data migrations alongside schema changes (split into separate revisions)

## Workflow

```bash
# Create a migration after modifying models.py
docker compose exec backend bash -c "alembic revision --autogenerate -m 'Your description'"

# Review the generated file in backend/app/alembic/versions/

# Apply
docker compose exec backend bash -c "alembic upgrade head"

# Rollback last migration
docker compose exec backend bash -c "alembic downgrade -1"
```

Migrations also apply automatically via `scripts/prestart.sh` on `docker compose up`.

## Fresh Start (Dev Only)

To reset the database completely:

1. Delete all files in `backend/app/alembic/versions/`
2. Uncomment `SQLModel.metadata.create_all(engine)` in `backend/app/core/db.py`
3. Comment out `alembic upgrade head` in `scripts/prestart.sh`
4. `docker compose down -v`
