---
name: nextjs-feature-based
description: >
  This skill should be used when the user asks to "work on the Next.js frontend", "implement a Server Component", "add a Client Component", "configure Zustand state", "validate with Zod", "use shadcn/ui", "write a Storybook story", "containerize with Docker", or "follow Next.js 15 architecture conventions".
---

# Next.js Feature-Based Architecture Expert

Expert in building scalable Next.js 15 frontends using a feature-based architecture. Features own their components, services, state, and tests. The `app/` directory is pure routing and data orchestration — all business logic lives in `src/features/`. The build artifact is always a Docker image using `output: 'standalone'`.

> **Documentation principle:** Feature CLAUDE.md files, the root CLAUDE.md Module/Feature Map, and ADRs MUST be updated whenever this skill is applied. Code changes without documentation updates are incomplete.

## Core Principles

1. **Features are the unit of organization** — not pages, not component types. A feature = a user-facing capability.
2. **`app/` orchestrates, `features/` implements** — pages fetch data and compose feature components; features hold the logic.
3. **Server Components by default** — add `'use client'` only when the component needs browser APIs, event listeners, or React state hooks.
4. **Zod validates all external data** — never cast `response.json()` directly to a TypeScript type.
5. **shadcn/ui is the design system** — use and customize it; never install competing component libraries.
6. **Zustand over Context for state** — React Context only for third-party wrappers that require it.
7. **Selectors prevent re-renders** — always `useStore((s) => s.field)`, never `useStore()`.
8. **Docker is the deployable artifact** — `output: 'standalone'`, container serves via `node server.js`.
9. **Features don't import each other's internals** — only from `exports/server.ts` or `exports/client.ts`.
10. **Documentation in the same commit as code** — CLAUDE.md updates are not optional.

## Capabilities

- Designing feature boundaries from a PRD or user flow description
- Creating full feature structure at Nivel 1 / 2 / 3
- Implementing async Server Components with proper data fetching patterns
- Building interactive Client Components with hooks and Zustand
- Defining and enforcing Zod schemas for all API responses
- Installing and customizing shadcn/ui components
- Writing Vitest + React Testing Library tests (components, hooks, stores, schemas)
- Creating Storybook stories with `@storybook/nextjs` and `experimentalRSC: true`
- Configuring Docker multi-stage builds for Next.js standalone output
- Managing TypeScript strict mode — no `any`, proper type guards
- Structuring global vs. feature-scoped Zustand stores
- Applying Conventional Commits and feature branch conventions

## Requirements

- Next.js 15.x (latest stable) with App Router
- TypeScript 5.x — `"strict": true` in `tsconfig.json`
- Tailwind CSS (required by shadcn/ui)
- shadcn/ui — installed to `src/shared/ui/` via `components.json`
- Zustand — state management (no Redux, no MobX, no Jotai)
- Zod — runtime validation of external data
- Vitest + `@testing-library/react` + `@testing-library/user-event`
- Storybook `@storybook/nextjs` with `experimentalRSC: true`
- Docker — `output: 'standalone'` + multi-stage Dockerfile
- ESLint (Next.js recommended) + Prettier + `eslint-config-prettier`

## Patterns

### Feature Structure (Nivel 2 — Standard)
```
src/features/[feature]/
├── CLAUDE.md
├── components/
│   ├── [FeatureList].tsx          # Server Component (default)
│   └── [FeatureForm].tsx          # Client Component ('use client')
├── services/
│   └── get-[resource].ts          # Async functions — data fetching
├── hooks/
│   └── use-[feature].ts           # Custom hooks (Client only)
├── stores/
│   └── [feature].store.ts         # Zustand store (if state needed)
├── schemas/
│   └── [resource].schema.ts       # Zod schemas
├── __tests__/
│   └── [Component].test.tsx
├── stories/
│   └── [Component].stories.tsx
└── exports/
    ├── server.ts                   # Public API for Server Components of other features
    └── client.ts                   # Public API for Client Components of other features
```

### Server Component with Data Fetching
```typescript
// features/products/services/get-products.ts
import { ProductListSchema } from '../schemas/product.schema';

export async function getProducts() {
  const res = await fetch(`${process.env.API_URL}/api/products`, {
    cache: 'no-store', // or { next: { revalidate: 60 } } for ISR
  });
  if (!res.ok) throw new Error('Failed to fetch products');
  const data: unknown = await res.json();
  return ProductListSchema.parse(data); // Always validate
}

// app/products/page.tsx (orchestration only)
import { ProductList } from '@/features/products/exports/server';

export default async function ProductsPage() {
  const products = await getProducts();
  return <ProductList products={products} />;
}
```

### Client Component with Zustand
```typescript
// features/cart/stores/cart.store.ts
import { create } from 'zustand';

interface CartState {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: number) => void;
}

export const useCartStore = create<CartState>((set) => ({
  items: [],
  addItem: (item) => set((s) => ({ items: [...s.items, item] })),
  removeItem: (id) => set((s) => ({ items: s.items.filter((i) => i.id !== id) })),
}));

// In a component — always use selectors:
const items = useCartStore((s) => s.items);         // ✅ selector
const addItem = useCartStore((s) => s.addItem);     // ✅ selector
// const store = useCartStore();                    // ❌ subscribes to everything
```

### Zod Schema
```typescript
// features/orders/schemas/order.schema.ts
import { z } from 'zod';

export const OrderSchema = z.object({
  id: z.number(),
  status: z.enum(['pending', 'confirmed', 'shipped', 'cancelled']),
  total: z.number().positive(),
  created_at: z.string().datetime(),
});

export const OrderListSchema = z.array(OrderSchema);
export type Order = z.infer<typeof OrderSchema>;
```

### Vitest + RTL Test
```typescript
// features/auth/components/__tests__/LoginForm.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from '../LoginForm';

it('shows validation error for invalid email', async () => {
  const user = userEvent.setup();
  render(<LoginForm />);

  await user.type(screen.getByLabelText(/email/i), 'not-valid');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  expect(screen.getByText(/invalid email/i)).toBeInTheDocument();
});
```

### Storybook Story
```typescript
// features/products/stories/ProductCard.stories.tsx
import type { Meta, StoryObj } from '@storybook/nextjs';
import { ProductCard } from '../components/ProductCard';

const meta: Meta<typeof ProductCard> = {
  component: ProductCard,
  tags: ['autodocs'],
};
export default meta;

export const Default: StoryObj<typeof ProductCard> = {
  args: { name: 'Premium Widget', price: 49.99, inStock: true },
};

export const OutOfStock: StoryObj<typeof ProductCard> = {
  args: { name: 'Basic Widget', price: 9.99, inStock: false },
};
```

### Docker Standalone Build
```dockerfile
# Dockerfile — multi-stage for minimal production image
FROM node:22-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]   # standalone — NOT next start
```

## Anti-Patterns

### ❌ Business Logic in `app/`
```typescript
// WRONG — filtering logic in page.tsx
export default async function ProductsPage() {
  const products = await getProducts();
  const available = products.filter((p) => p.stock > 0); // ❌ logic in app/
  return <ProductList products={available} />;
}
```

### ❌ `useEffect` as Primary Data Fetching
```typescript
// WRONG — use a Server Component or React Query instead
useEffect(() => {
  fetch('/api/products').then(r => r.json()).then(setProducts);
}, []);
```

### ❌ Raw API Response Cast
```typescript
// WRONG — trusts the network response
const data = await res.json() as Product[]; // ❌ no validation
```

### ❌ Cross-Feature Internal Import
```typescript
// WRONG — imports internal file from another feature
import { useAuthStore } from '@/features/auth/stores/auth.store'; // ❌
// CORRECT
import { useAuthStore } from '@/features/auth/exports/client'; // ✅
```

### ❌ Vercel-Specific Features
```typescript
// WRONG — locks to Vercel infrastructure
import { unstable_cache } from 'next/cache'; // ❌ Vercel-only
export const config = { runtime: 'edge' };   // ❌ Vercel Edge
```

### ❌ Full Store Subscription
```typescript
const store = useCartStore(); // ❌ re-renders on any store change
const items = useCartStore((s) => s.items); // ✅ only re-renders when items changes
```

## Related Skills

Works well with: `prd-analysis`, `frontend-feature-dev`, `api-contract-sync`, `project-orchestration`

## When to Use

This skill applies when working in the Next.js frontend repository — creating features, implementing components, writing tests, creating Storybook stories, configuring Docker, or designing state management patterns. For backend work, use `laravel-modular-monolith`. For cross-repo coordination, use `api-contract-sync`.

## Coexistence with Superpowers

This skill activates when working in the Next.js frontend repository.
Superpowers has no equivalent — it is technology-agnostic and does not define Next.js architecture conventions.
This skill complements any Superpowers phase by applying Next.js 15 Feature-Based conventions.

If Superpowers is not installed, this skill works identically.
