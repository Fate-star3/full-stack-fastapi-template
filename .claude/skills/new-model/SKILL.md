---
name: new-model
description: Scaffold a complete CRUD resource — model, migration, CRUD, API routes, frontend SDK, and frontend pages
---

# Scaffold a New CRUD Model

Follow this exact checklist when adding a new data model (e.g., Tag, Category, Comment, Post).

## 1. Add SQLModel Classes in `backend/app/models.py`

Follow the existing pattern — Base class, Create, Update, DB model, Public:

```python
# Shared properties
class TagBase(SQLModel):
    name: str = Field(min_length=1, max_length=255, unique=True)

class TagCreate(TagBase):
    pass

class TagUpdate(SQLModel):
    name: str | None = Field(default=None, min_length=1, max_length=255)

class Tag(TagBase, table=True):
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    created_at: datetime | None = Field(
        default_factory=get_datetime_utc,
        sa_type=DateTime(timezone=True),
    )

class TagPublic(TagBase):
    id: uuid.UUID
    created_at: datetime | None = None

class TagsPublic(SQLModel):
    data: list[TagPublic]
    count: int
```

Key conventions:
- UUID primary key always: `id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)`
- `created_at` uses `get_datetime_utc` factory with timezone-aware `DateTime`
- If the model belongs to a user, add `owner_id` with foreign key and `Relationship`
- `Base` class has shared fields (no `id`, no `created_at`)
- `Create` extends Base with required creation fields
- `Update` re-declares fields as `| None` with `default=None` for partial updates
- `*Public` is the response shape — always includes `id`

## 2. Create Database Migration

```bash
docker compose exec backend bash -c "alembic revision --autogenerate -m 'Add Tag model'"
```

Review the generated migration in `backend/app/alembic/versions/` to verify it's correct.

## 3. Add CRUD Functions in `backend/app/crud.py`

```python
def create_tag(*, session: Session, tag_in: TagCreate) -> Tag:
    db_obj = Tag.model_validate(tag_in)
    session.add(db_obj)
    session.commit()
    session.refresh(db_obj)
    return db_obj

def get_tag_by_name(*, session: Session, name: str) -> Tag | None:
    statement = select(Tag).where(Tag.name == name)
    return session.exec(statement).first()
```

See [api-conventions.md](../../rules/api-conventions.md) for CRUD naming and conventions.

## 4. Create Route File in `backend/app/api/routes/`

Pattern for owner-scoped resources (`items.py` is the template):

```python
router = APIRouter(prefix="/tags", tags=["tags"])

@router.get("/", response_model=TagsPublic)
def read_tags(session: SessionDep, skip: int = 0, limit: int = 100) -> Any:
    count_statement = select(func.count()).select_from(Tag)
    count = session.exec(count_statement).one()
    statement = select(Tag).offset(skip).limit(limit)
    tags = session.exec(statement).all()
    return TagsPublic(data=[TagPublic.model_validate(t) for t in tags], count=count)
```

Permission patterns to use:
- **Public read**: No auth dependency
- **Auth-required write**: Declare `current_user: CurrentUser` in params
- **Admin-only**: Add `dependencies=[Depends(get_current_active_superuser)]` to decorator
- **Owner-only**: Check `if obj.owner_id != current_user.id:` inline → 403

For partial updates, use `model_dump(exclude_unset=True)` + `sqlmodel_update()`.

## 5. Register the Router in `backend/app/api/main.py`

```python
from app.api.routes import tags
api_router.include_router(tags.router)
```

## 6. Regenerate Frontend SDK

```bash
bash ./scripts/generate-client.sh
```

## 7. Create Frontend Pages

Create files under `frontend/src/routes/_layout/` following existing patterns:

- List page: TanStack Table with delete confirmation dialog
- Create page: react-hook-form + zod validation, useMutation for POST
- Edit page: Same form, pre-populated via useQuery, useMutation for PUT
- Use `import { TagsService } from "@/client"` for API calls (name matches `tags` from the OpenAPI schema)
