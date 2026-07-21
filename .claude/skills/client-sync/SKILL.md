---
name: client-sync
description: Regenerate the TypeScript API client from the backend OpenAPI schema
---

# Regenerate Frontend API Client

Run this after any backend change that modifies API endpoints, models, or schemas.

## Steps

1. Ensure the Docker Compose stack is running (backend must be healthy):

```bash
docker compose up -d --wait backend
```

2. Run the generation script from the project root:

```bash
bash ./scripts/generate-client.sh
```

3. Verify the generated code compiles:

```bash
cd frontend && bun run build
```

4. Commit the updated `frontend/src/client/` directory.

## What Gets Generated

- `frontend/src/client/types.ts` — TypeScript interfaces matching Pydantic schemas
- `frontend/src/client/sdk.gen.ts` — Typed service functions per API tag (e.g., `UsersService`, `ItemsService`)
- `frontend/src/client/schemas.gen.ts` — JSON schemas for runtime validation
- `frontend/src/client/core/` — HTTP client, auth token injection, error handling

## Troubleshooting

If generation fails:
- Verify `http://localhost:8000/api/v1/openapi.json` returns valid JSON
- Check that all route response models use SQLModel/Pydantic classes (not `dict` or `Any`)
- Ensure `frontend/openapi-ts.config.ts` hasn't been modified incorrectly
