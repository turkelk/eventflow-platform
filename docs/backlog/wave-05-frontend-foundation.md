# Wave 5: Frontend Foundation

## 5.1 Implement app shell, sidebar navigation, command palette, and theme system

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Build the core application shell per 04-FRONTEND-ARCHITECTURE.md sections 8 and 6.

**Key files:**
- `frontend/src/app/layouts/AppLayout.tsx`
- `frontend/src/shared/components/AppSidebar.tsx` (collapsible, icon-only mode, keyboard shortcuts)
- `frontend/src/shared/components/CommandPalette.tsx` (Cmd+K, shadcn Command, debounced search)
- `frontend/src/shared/components/TopHeader.tsx`
- `frontend/src/app/providers/ThemeProvider.tsx` (light/dark/system, localStorage persistence)
- `frontend/src/shared/hooks/useCommandPalette.ts`
- `frontend/src/app/router.tsx` (all routes with lazy loading)

**Acceptance criteria:**
- [ ] Sidebar collapses to 56px icon-only at lg breakpoint, full 240px expanded
- [ ] `Cmd/Ctrl+K` opens command palette globally from any page
- [ ] Command palette shows recent items, quick actions, and live search results
- [ ] Dark mode toggled via user menu, persisted to localStorage, respects OS preference on first load
- [ ] Mobile bottom navigation renders for screens < 768px with 5-item layout

---

## 5.2 Build shared UI component library with design system tokens

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Create all shared UI components from the design system per 01-UX-RESEARCH.md section 5 and 04-FRONTEND-ARCHITECTURE.md section 6.

**Key files:**
- `frontend/src/shared/components/ui/` (Button, Input, Select, Sheet, Dialog, Command, DataTable, Badge)
- `frontend/src/shared/components/ui/SkeletonCard.tsx`
- `frontend/src/shared/components/ui/EmptyState.tsx`
- `frontend/src/shared/components/ui/ErrorState.tsx`
- `frontend/src/shared/components/ui/ConfirmDialog.tsx`
- `frontend/src/shared/components/InlineEditField.tsx` (click-to-edit with 800ms debounce auto-save)
- `frontend/src/styles/globals.css` (all CSS custom properties for both themes)

**Acceptance criteria:**
- [ ] All components use design system tokens (CSS vars) — no hardcoded colors
- [ ] Dark mode variants work for all components via `dark:` Tailwind prefix
- [ ] `InlineEditField` shows `ring-2 ring-indigo-500` on edit, saves on blur/Enter, cancels on Escape
- [ ] All interactive elements have visible focus rings meeting WCAG 2.1 AA (4.5:1 contrast)
- [ ] Skeleton screens match exact dimensions of loaded content to prevent layout shift

---

## 5.3 Setup TanStack Query, Zustand stores, and API client layer

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:m`, `agent:frontend`
**Status:** Pending

Implement state management architecture per 04-FRONTEND-ARCHITECTURE.md sections 5 and 10.

**Key files:**
- `frontend/src/app/providers/QueryProvider.tsx` (TanStack Query v5, staleTime 60s, retry logic)
- `frontend/src/shared/lib/query-keys.ts` (type-safe query key factory for all entities)
- `frontend/src/shared/lib/api-client.ts` (ky, Bearer interceptor, 401→refresh, correlation ID)
- `frontend/src/store/ui.store.ts` (Zustand: sidebar, command palette, notification drawer)
- `frontend/src/store/checkin.store.ts` (Zustand: live counter, offline queue)
- `frontend/src/store/event-editor.store.ts` (Zustand: active event, collaborators, unsaved)

**Acceptance criteria:**
- [ ] Query keys are typed constants — no magic strings in `useQuery` calls
- [ ] `apiClient` retries 3x with exponential backoff on 5xx, never on 4xx
- [ ] Global mutation error handler shows toast for all failed mutations
- [ ] Optimistic update pattern implemented for event/session inline edits (rollback on error)
- [ ] Zustand stores have no cross-store dependencies

---

## 5.4 Implement error boundaries, toast notifications, and loading states

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:m`, `agent:frontend`
**Status:** Pending

Implement the error handling strategy from 04-FRONTEND-ARCHITECTURE.md section 9.

**Key files:**
- `frontend/src/app/router.tsx` (errorElement: RouteErrorBoundary per route)
- `frontend/src/shared/components/RouteErrorBoundary.tsx`
- `frontend/src/app/providers/ToastProvider.tsx` (Sonner config: bottom-right desktop, bottom-center mobile)
- `frontend/src/shared/lib/error-classifier.ts` (ApiError class, status→UI mapping)

**Toast durations:** success 3s, error 8s, warning 5s, loading persistent until replaced

**Acceptance criteria:**
- [ ] Route-level error boundary shows error in content area while sidebar remains visible
- [ ] 422 validation errors mapped back to React Hook Form `setError()` field-level display
- [ ] 429 rate limit toast shows Retry-After countdown
- [ ] All async content shows skeleton screens (not spinners) while loading
- [ ] Empty states include actionable CTA button (never blank page)

---

## 5.5 Implement DiscountCode entity with validation, group pricing, and ticket window enforcement

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

## Goal
Support discount codes on the public registration page with configurable types and usage limits.

## Acceptance Criteria
- `DiscountCode` entity: `code`, `eventId`, `discountType` (Percent/Fixed), `discountValue`, `maxUses`, `usedCount`, `expiresAt`, `ticketTypeIds[]`
- CRUD: `GET|POST /api/v1/events/{eventId}/discount-codes`, `PUT|DELETE .../{codeId}`
- `POST /public/events/{orgSlug}/{eventSlug}/validate-discount` returns discount amount
- Applied discount recorded on `Registration.discountCodeId` and `amountPaid`
- Atomic increment of `usedCount` via distributed lock to prevent over-use

## Technical Notes
- EF Core migration for `discount_codes` table
- Validation in `CreateRegistrationCommand` checks code validity, expiry, and remaining uses

---

## 5.6 Implement public event registration page with discount code validation flow

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:m`, `agent:frontend`
**Status:** Pending

## Goal
Branded public registration checkout page with discount code input, validation, and Stripe payment — competitive differentiator vs Cvent.

## Acceptance Criteria
- `POST /public/events/{orgSlug}/{eventSlug}/validate-discount` called on code entry (debounced)
- Discount amount shown inline; total price updates reactively
- Invalid/expired codes show inline error message
- Applied code persisted through Stripe checkout flow
- White-label theming applied via `TenantThemeProvider`

## Technical Notes
- Extends existing `RegistrationPage.tsx` with `DiscountCodeInput` component
- Zod schema validates code format client-side before API call
- `useDiscountCode` hook manages validation state and debouncing

---

## 5.7 Implement public registration endpoints: GET public event, register, cancel, lookup

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Public (unauthenticated) API endpoints powering the white-labeled attendee registration experience.

## Acceptance Criteria
- `GET /public/events/{orgSlug}/{eventSlug}` returns public event detail (no auth)
- `GET /public/events/{orgSlug}/{eventSlug}/sessions` returns published sessions
- `POST /public/events/{orgSlug}/{eventSlug}/register` creates registration (rate limited 20/min/IP)
- `GET /public/registrations/{registrationNumber}` returns registration status
- `POST /public/registrations/{registrationNumber}/cancel` cancels with token verification

## Technical Notes
- Tenant resolved from `orgSlug` via `GetTenantBySlugQuery` (bypasses JWT auth)
- `registrationNumber` = `ConfirmationCode` (e.g., `EVF-2025-A7X3K`)
- Cancellation requires token sent in confirmation email (not just confirmation code)

---

## 5.8 Implement registration status update, refund, and waitlist promotion endpoints

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Fill gaps in registration management API: manual status changes, refund processing, and waitlist promotion.

## Acceptance Criteria
- `GET /api/v1/events/{eventId}/registrations/{registrationId}` returns full registration detail
- `PUT /api/v1/events/{eventId}/registrations/{registrationId}/status` allows manual status change (Pending→Confirmed, etc.)
- `POST /api/v1/events/{eventId}/registrations/{registrationId}/refund` initiates Stripe refund and updates `PaymentStatus`
- `GET /api/v1/events/{eventId}/waitlist` returns paginated waitlist
- `POST /api/v1/events/{eventId}/waitlist/promote` promotes next eligible waitlist entry and sends offer email

## Technical Notes
- Refund via `Stripe.RefundService.CreateAsync()` with `PaymentIntentId`
- Promotion uses `OfferWaitlistSpotCommand` — sets `OfferExpiresAt` and sends email
- Status change restricted: cannot manually set `Paid` status (payment-only)

---

