---
paths: ["backend/app/api/**/*.py", "backend/app/crud.py", "backend/app/models.py"]
---

# API Endpoint Conventions

Follow these patterns when creating or modifying API endpoints.

## Permission Patterns

There are exactly two patterns. Choose based on the use case:

**Route-level guard** — use when *every* access to this endpoint requires the same permission:

```python
@router.get("/", dependencies=[Depends(get_current_active_superuser)])
def read_users(session: SessionDep, skip: int = 0, limit: int = 100) -> Any:
```

**Inline check** — use when behavior differs by role or ownership:

```python
@router.get("/{id}")
def read_item(session: SessionDep, current_user: CurrentUser, id: uuid.UUID) -> Any:
    item = session.get(Item, id)
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    if not current_user.is_superuser and (item.owner_id != current_user.id):
        raise HTTPException(status_code=403, detail="Not enough permissions")
    return item
```

Don't mix both — pick one.

## Route Function Signatures

- Use keyword-only arguments after `*` for functions that have injected dependencies and input models
- Always declare `session: SessionDep` first
- Use `current_user: CurrentUser` for auth-required endpoints
- Input models come last: `user_in: UserCreate`
- Return type annotation is `Any` for routes that return different shapes

## Response Types

- Single object: `ItemPublic`
- List: `ItemsPublic` (with `data` array + `count` int)
- Action result: `Message` (with `message: str`)
- Never return raw SQLModel `table=True` objects — use the `*Public` schema

## Error Handling

- 404: Resource not found → `"Not found"`
- 400: Validation error (e.g., duplicate email) → `"The ... already exists"`
- 403: Permission denied → `"Not enough permissions"`
- 409: Conflict → `"User with this email already exists"`

Always check existence before permission — prevents leaking which IDs exist.

## Route Registration

All routes go in `backend/app/api/routes/` as separate files. Register in `backend/app/api/main.py`:

```python
from app.api.routes import myservice
api_router.include_router(myservice.router)
```

## Model Validation Convention

For creating: `Item.model_validate(item_in, update={"owner_id": current_user.id})`
For updating: `item_in.model_dump(exclude_unset=True)` then `item.sqlmodel_update(update_dict)`

Never use `**dict` unpacking — use `model_validate` and `model_dump` instead.

## CRUD Layer Convention

Functions in `backend/app/crud.py`:
- Use keyword-only arguments (`*` after no defaulted params)
- Take `session` as an explicit parameter (never global)
- Never touch HTTP — routes handle status codes and responses
- Return ORM model instances, not Pydantic schemas
- Name pattern: `create_*`, `get_*_by_*`, `update_*`, `delete_*`
