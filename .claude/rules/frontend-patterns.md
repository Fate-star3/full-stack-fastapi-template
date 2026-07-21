---
paths: ["frontend/src/**/*.tsx", "frontend/src/**/*.ts"]
---

# Frontend Development Patterns

## API Calls

Always use the auto-generated SDK from `@/client`:

```ts
import { ItemsService, UsersService } from "@/client"

// Query (TanStack Query)
const { data, isLoading } = useQuery({
  queryKey: ["items"],
  queryFn: () => ItemsService.readItems({ query: { skip: 0, limit: 100 } }),
})

// Mutation
const mutation = useMutation({
  mutationFn: (data: ItemCreate) => ItemsService.createItem({ requestBody: data }),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ["items"] }),
})
```

Never hand-write `fetch` or `axios` calls. The generated client handles token injection, base URL, and error formatting automatically.

## Form Handling

Always use react-hook-form + zod:

```tsx
const schema = z.object({
  title: z.string().min(1, "Title is required").max(255),
  description: z.string().max(255).optional(),
})
type FormData = z.infer<typeof schema>

const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
  resolver: zodResolver(schema),
})
```

## Route File Structure

Each route file in `frontend/src/routes/_layout/` follows this pattern:

1. Imports (SDK client, hooks, components, UI primitives, zod, react-hook-form)
2. Zod schema for form validation
3. Route component with:
   - Query hooks for data fetching
   - Mutation hooks for data modification
   - Loading/error states using conditional rendering
   - Dialog state for create/edit forms
4. Toast notifications on success/error (use `sonner` toast)

## Styling

- Use **Tailwind CSS utility classes** directly in JSX
- Use **shadcn/ui** components from `@/components/ui/` (Button, Input, Dialog, Table, DropdownMenu, etc.)
- Icons from `lucide-react` (preferred) or `react-icons`
- Dark mode is automatic via `next-themes` — all components work in both themes
- Use `tw-animate-css` for animations

## Route Registration

Routes use TanStack Router's file-based routing:
- `frontend/src/routes/__root.tsx` — root layout
- `frontend/src/routes/_layout.tsx` — authenticated layout (sidebar)
- `frontend/src/routes/_layout/` — all authenticated pages
- `frontend/src/routes/login.tsx` — public, with `beforeLoad` redirect if logged in
- The route tree file `frontend/src/routeTree.gen.ts` is auto-generated, never edit it

## Error Handling

- 401/403 errors are handled globally in `main.tsx` — auto-redirect to `/login`
- Use `react-error-boundary` for component-level error boundaries
- Mutations should show error toasts on failure: `toast.error(error.message)`
