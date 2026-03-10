# Client Module

## Component Organization
- `src/pages/` — Page-level components (one per route)
- `src/components/` — Reusable UI components
- `src/hooks/` — Custom React hooks for data fetching and shared logic
- `src/lib/` — Utility functions, API client, constants
- `src/types/` — TypeScript interfaces and types

## State Management
- Use React Query (TanStack Query) for ALL server state (API data).
- Use Zustand for UI-only state (sidebar open, theme, etc.).
- Use React hooks (`useState`, `useReducer`) for local component state.

## Styling
- Use TailwindCSS 4 utility classes exclusively. Avoid custom CSS.
- Use shadcn/ui components as the base — customize with Tailwind.
- Every color must have a `dark:` variant. Test both modes.
- Responsive: `sm:` (640px), `md:` (768px), `lg:` (1024px), `xl:` (1280px).
- Animations: use motion/react (NOT framer-motion).

## Forms
- Use React Hook Form + Zod for all forms.
- Validate on submit, show inline errors.

## Accessibility
- Semantic HTML: use `button` for actions, `a` for navigation, headings in order.
- `aria-label` on icon-only buttons and non-text interactive elements.
- Keyboard navigable: all interactive elements reachable via Tab.
- Focus management: trap focus in modals, restore on close.
- Color contrast: minimum 4.5:1 ratio for text.

## Performance
- Lazy-load pages and heavy components.
- Optimize images.
- Avoid unnecessary re-renders — memoize expensive computations.
- Keep bundle size small — check imports for tree-shaking.