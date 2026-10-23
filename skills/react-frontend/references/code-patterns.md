# React Code Patterns

## shared/api/http.ts

```ts
const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? '';

export interface FieldError {
  field: string;
  message: string;
}

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly title: string,
    public readonly detail: string,
    public readonly fieldErrors: FieldError[] = [],
  ) {
    super(detail || title);
  }

  static async from(res: Response): Promise<ApiError> {
    try {
      const body = await res.json();
      return new ApiError(res.status, body.title ?? res.statusText, body.detail ?? '', body.errors ?? []);
    } catch {
      return new ApiError(res.status, res.statusText, '');
    }
  }
}

let accessToken: string | null = null;
let onUnauthorized: () => void = () => {};

export const setAccessToken = (token: string | null) => { accessToken = token; };
export const setUnauthorizedHandler = (handler: () => void) => { onUnauthorized = handler; };

export async function http<T>(path: string, init: RequestInit = {}): Promise<T> {
  const headers = new Headers(init.headers);
  if (init.body !== undefined) headers.set('Content-Type', 'application/json');
  if (accessToken) headers.set('Authorization', `Bearer ${accessToken}`);

  const res = await fetch(`${BASE_URL}${path}`, { ...init, headers });

  if (res.status === 401 && accessToken) onUnauthorized();
  if (!res.ok) throw await ApiError.from(res);
  return (res.status === 204 ? undefined : await res.json()) as T;
}
```

## features/products/api.ts

```ts
import { http } from '../../shared/api/http';
import type { components } from '../../shared/api/schema';

export type Product = components['schemas']['ProductResponse'];
export type CreateProduct = components['schemas']['CreateProductRequest'];
export interface Page<T> { content: T[]; page: number; size: number; totalElements: number; totalPages: number }

export const listProducts = (page: number, size = 20) =>
  http<Page<Product>>(`/api/v1/products?page=${page}&size=${size}&sort=createdAt,desc`);

export const createProduct = (body: CreateProduct) =>
  http<Product>('/api/v1/products', { method: 'POST', body: JSON.stringify(body) });
```

## features/products/hooks.ts

```ts
import { keepPreviousData, useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { createProduct, listProducts } from './api';

export const productKeys = {
  all: ['products'] as const,
  list: (page: number) => [...productKeys.all, 'list', page] as const,
};

export function useProducts(page: number) {
  return useQuery({ queryKey: productKeys.list(page), queryFn: () => listProducts(page), placeholderData: keepPreviousData });
}

export function useCreateProduct() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: createProduct,
    onSuccess: () => qc.invalidateQueries({ queryKey: productKeys.all }),
  });
}
```

## List page with the four states

```tsx
export function ProductListPage() {
  const [page, setPage] = useState(0);
  const { data, isPending, isError, refetch } = useProducts(page);

  if (isPending) return <Spinner label="Loading products" />;
  if (isError) return <ErrorState message="Could not load products." onRetry={refetch} />;
  if (data.content.length === 0)
    return <EmptyState title="No products yet" action={<Link to="/products/new">Add product</Link>} />;

  return (
    <main>
      <h1>Products</h1>
      <ProductTable products={data.content} />
      <Pagination page={data.page} totalPages={data.totalPages} onChange={setPage} />
    </main>
  );
}
```

## Form with zod and server errors

```tsx
const schema = z.object({
  sku: z.string().trim().min(1, 'SKU is required').max(64),
  name: z.string().trim().min(1, 'Name is required').max(200),
  unitPrice: z.string().regex(/^\d+(\.\d{1,2})?$/, 'Enter a price like 19.99'),
  supplierId: z.coerce.number().int().positive('Choose a supplier'),
});
type FormValues = z.infer<typeof schema>;

export function ProductForm({ onSaved }: { onSaved: () => void }) {
  const { register, handleSubmit, setError, formState: { errors, isSubmitting } } =
    useForm<FormValues>({ resolver: zodResolver(schema) });
  const create = useCreateProduct();

  const onSubmit = handleSubmit(async (values) => {
    try {
      await create.mutateAsync(values);
      onSaved();
    } catch (e) {
      if (e instanceof ApiError) {
        e.fieldErrors.forEach((f) => setError(f.field as keyof FormValues, { message: f.message }));
        if (e.fieldErrors.length === 0) setError('root', { message: e.detail || e.title });
      }
    }
  });

  return (
    <form onSubmit={onSubmit} noValidate>
      <label htmlFor="sku">SKU</label>
      <input id="sku" {...register('sku')} aria-invalid={!!errors.sku} />
      {errors.sku && <p role="alert">{errors.sku.message}</p>}
      {/* ...other fields... */}
      {errors.root && <p role="alert">{errors.root.message}</p>}
      <button type="submit" disabled={isSubmitting}>Save</button>
    </form>
  );
}
```

## vite.config.ts dev proxy

```ts
export default defineConfig({
  plugins: [react()],
  server: { proxy: { '/api': 'http://localhost:8080' } },
  test: { environment: 'jsdom', setupFiles: './src/test/setup.ts' },
});
```
