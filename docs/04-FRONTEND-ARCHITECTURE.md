# EventFlow — Frontend Architecture

> Document Version: 1.0 | Status: Approved for Implementation  
> Stack: React 19 + TypeScript (strict) + Vite 7 + TailwindCSS 4 + shadcn/ui

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [Project Structure](#2-project-structure)
3. [Page Tree & Route Definitions](#3-page-tree--route-definitions)
4. [Component Hierarchy](#4-component-hierarchy)
5. [State Management Architecture](#5-state-management-architecture)
6. [Design System Tokens](#6-design-system-tokens)
7. [Authentication Flow](#7-authentication-flow)
8. [Navigation Architecture](#8-navigation-architecture)
9. [Error Handling Strategy](#9-error-handling-strategy)
10. [API Client Layer](#10-api-client-layer)
11. [Real-Time & WebSocket Layer](#11-real-time--websocket-layer)
12. [AI Feature Patterns](#12-ai-feature-patterns)
13. [PWA & Offline Strategy](#13-pwa--offline-strategy)
14. [Performance Strategy](#14-performance-strategy)
15. [Testing Architecture](#15-testing-architecture)

---

## 1. Technology Stack

```
Runtime:         React 19 (concurrent features enabled)
Language:        TypeScript 5.x (strict mode: true)
Build Tool:      Vite 7
Styling:         TailwindCSS 4 + shadcn/ui (Radix UI primitives)
Routing:         React Router 7 (file-based routing)
Server State:    TanStack Query v5 (React Query)
UI State:        Zustand 5
Forms:           React Hook Form 7 + Zod 3
Drag & Drop:     @dnd-kit/core + @dnd-kit/sortable
Real-Time:       native WebSocket (reconnecting-websocket wrapper)
Charts:          Recharts 2
Icons:           Lucide React
Toasts:          Sonner
Date/Time:       date-fns 3
HTTP Client:     ky (lightweight fetch wrapper with retry)
PWA:             vite-plugin-pwa (Workbox)
Testing:         Vitest + React Testing Library + Playwright
Code Quality:    ESLint (flat config) + Prettier + TypeScript strict
Feature Flags:   @unleash/proxy-client-react
```

---

## 2. Project Structure

```
src/
├── app/                          # App shell, providers, router
│   ├── App.tsx                   # Root component, provider composition
│   ├── router.tsx                # Route definitions
│   ├── providers/
│   │   ├── AuthProvider.tsx      # OIDC session management
│   │   ├── QueryProvider.tsx     # TanStack Query client
│   │   ├── ThemeProvider.tsx     # Dark/light/system mode
│   │   ├── TenantThemeProvider.tsx # White-label CSS vars
│   │   ├── FeatureFlagProvider.tsx # Unleash context
│   │   ├── WebSocketProvider.tsx # WS connection + reconnect
│   │   └── ToastProvider.tsx     # Sonner configuration
│   └── layouts/
│       ├── AppLayout.tsx         # Authenticated shell (sidebar + header)
│       ├── AuthLayout.tsx        # Login/callback pages
│       ├── PublicLayout.tsx      # Registration pages (white-labeled)
│       ├── CheckInLayout.tsx     # Full-screen check-in mode
│       └── AdminLayout.tsx       # Platform admin shell
│
├── pages/                        # Route page components (thin)
│   ├── dashboard/
│   │   └── DashboardPage.tsx
│   ├── events/
│   │   ├── EventsPage.tsx        # Events list / portfolio view
│   │   ├── EventDetailPage.tsx   # Event detail shell
│   │   ├── EventOverviewTab.tsx
│   │   ├── EventRegistrationTab.tsx
│   │   ├── EventSessionsTab.tsx
│   │   ├── EventSpeakersTab.tsx
│   │   ├── EventSponsorsTab.tsx
│   │   ├── EventCommunicationsTab.tsx
│   │   ├── EventAnalyticsTab.tsx
│   │   └── EventSettingsTab.tsx
│   ├── attendees/
│   │   └── AttendeesPage.tsx
│   ├── venues/
│   │   ├── VenuesPage.tsx
│   │   └── VenueDetailPage.tsx
│   ├── speakers/
│   │   └── SpeakersPage.tsx
│   ├── sponsors/
│   │   └── SponsorsPage.tsx
│   ├── campaigns/
│   │   ├── CampaignsPage.tsx
│   │   └── CampaignBuilderPage.tsx
│   ├── analytics/
│   │   └── AnalyticsPage.tsx
│   ├── ai/
│   │   └── AIAssistantPage.tsx
│   ├── settings/
│   │   ├── SettingsPage.tsx
│   │   ├── SettingsTeamTab.tsx
│   │   ├── SettingsBillingTab.tsx
│   │   ├── SettingsIntegrationsTab.tsx
│   │   ├── SettingsWhiteLabelTab.tsx
│   │   └── SettingsSecurityTab.tsx
│   ├── checkin/
│   │   └── CheckInPage.tsx       # PWA check-in mode
│   ├── public/
│   │   ├── RegistrationPage.tsx  # Attendee registration (white-labeled)
│   │   ├── SchedulePage.tsx      # Public event schedule
│   │   └── SpeakerPortalPage.tsx # Speaker self-service portal
│   ├── auth/
│   │   ├── LoginPage.tsx
│   │   ├── CallbackPage.tsx      # OIDC callback handler
│   │   └── LogoutPage.tsx
│   ├── onboarding/
│   │   ├── OnboardingWorkspacePage.tsx
│   │   ├── OnboardingInvitePage.tsx
│   │   └── OnboardingFirstEventPage.tsx
│   └── admin/                    # Platform admin (separate access)
│       ├── AdminDashboardPage.tsx
│       ├── AdminTenantsPage.tsx
│       ├── AdminUsersPage.tsx
│       └── AdminBillingPage.tsx
│
├── features/                     # Feature-scoped components and hooks
│   ├── events/
│   │   ├── components/
│   │   │   ├── EventCard.tsx
│   │   │   ├── EventStatusBadge.tsx
│   │   │   ├── CreateEventDrawer.tsx
│   │   │   ├── EventSetupChecklist.tsx
│   │   │   ├── EventCapacityBar.tsx
│   │   │   └── EventContextNav.tsx   # In-event tab strip
│   │   ├── hooks/
│   │   │   ├── useEvents.ts
│   │   │   ├── useEventDetail.ts
│   │   │   ├── useCreateEvent.ts
│   │   │   └── useUpdateEvent.ts
│   │   └── schemas/
│   │       └── event.schema.ts       # Zod schemas
│   ├── sessions/
│   │   ├── components/
│   │   │   ├── SessionScheduler.tsx  # DnD timeline
│   │   │   ├── SessionCard.tsx
│   │   │   ├── SessionConflictAlert.tsx
│   │   │   └── CreateSessionDrawer.tsx
│   │   └── hooks/
│   │       ├── useSessions.ts
│   │       └── useSessionScheduler.ts
│   ├── attendees/
│   │   ├── components/
│   │   │   ├── AttendeeTable.tsx
│   │   │   ├── AttendeeCard.tsx
│   │   │   ├── AttendeeDrawer.tsx
│   │   │   ├── BulkActionBar.tsx
│   │   │   └── AttendeeImportModal.tsx
│   │   └── hooks/
│   │       ├── useAttendees.ts
│   │       └── useBulkActions.ts
│   ├── checkin/
│   │   ├── components/
│   │   │   ├── CheckInCounter.tsx
│   │   │   ├── QRScanner.tsx
│   │   │   ├── AttendeeSearch.tsx
│   │   │   └── CheckInListItem.tsx
│   │   └── hooks/
│   │       ├── useCheckIn.ts
│   │       └── useOfflineCheckIn.ts  # IndexedDB sync
│   ├── ai/
│   │   ├── components/
│   │   │   ├── AIAssistantInput.tsx
│   │   │   ├── AISuggestionCard.tsx
│   │   │   ├── AIConfidenceIndicator.tsx
│   │   │   ├── AIStreamingResponse.tsx # SSE progress UI
│   │   │   └── AIReasoningTooltip.tsx
│   │   └── hooks/
│   │       ├── useAIAgendaBuilder.ts
│   │       └── useAIMatchmaking.ts
│   ├── communications/
│   │   ├── components/
│   │   │   ├── CampaignBuilder.tsx
│   │   │   ├── EmailTemplateEditor.tsx
│   │   │   └── CampaignMetricsCard.tsx
│   │   └── hooks/
│   │       └── useCampaigns.ts
│   ├── analytics/
│   │   ├── components/
│   │   │   ├── RevenueChart.tsx
│   │   │   ├── AttendanceChart.tsx
│   │   │   ├── EventROICard.tsx
│   │   │   └── CrossEventMetrics.tsx
│   │   └── hooks/
│   │       └── useAnalytics.ts
│   ├── sponsors/
│   │   ├── components/
│   │   │   ├── SponsorTierBuilder.tsx  # DnD tier management
│   │   │   ├── SponsorCard.tsx
│   │   │   └── SponsorInviteModal.tsx
│   │   └── hooks/
│   │       └── useSponsors.ts
│   └── venues/
│       ├── components/
│       │   ├── VenueCard.tsx
│       │   ├── VenueSearchFilters.tsx
│       │   └── VenueMapView.tsx
│       └── hooks/
│           └── useVenues.ts
│
├── shared/                       # Cross-feature shared components
│   ├── components/
│   │   ├── ui/                   # shadcn/ui re-exports + customizations
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Sheet.tsx
│   │   │   ├── Dialog.tsx
│   │   │   ├── Command.tsx       # Command palette base
│   │   │   ├── DataTable.tsx     # TanStack Table wrapper
│   │   │   ├── SkeletonCard.tsx
│   │   │   ├── EmptyState.tsx
│   │   │   ├── ErrorState.tsx
│   │   │   └── ConfirmDialog.tsx
│   │   ├── CommandPalette.tsx    # Global Cmd+K
│   │   ├── AppSidebar.tsx        # Primary navigation
│   │   ├── TopHeader.tsx         # Top bar with notifications
│   │   ├── NotificationDrawer.tsx
│   │   ├── UserMenu.tsx
│   │   ├── ThemeToggle.tsx
│   │   ├── PresenceIndicator.tsx # Multiplayer avatars
│   │   ├── InlineEditField.tsx   # Click-to-edit wrapper
│   │   └── FeatureGate.tsx       # Unleash flag gate component
│   ├── hooks/
│   │   ├── useAuth.ts            # Auth context consumer
│   │   ├── useCurrentTenant.ts
│   │   ├── useFeatureFlags.ts    # Unleash hook wrapper
│   │   ├── useCommandPalette.ts  # Global Cmd+K state
│   │   ├── useWebSocket.ts       # WS subscription hook
│   │   ├── useInlineEdit.ts      # Debounced auto-save
│   │   ├── useDarkMode.ts
│   │   ├── useMediaQuery.ts
│   │   └── useConfirm.ts         # Programmatic confirm dialog
│   ├── lib/
│   │   ├── api-client.ts         # ky HTTP client config
│   │   ├── query-keys.ts         # TanStack Query key factory
│   │   ├── auth-utils.ts         # Token utilities
│   │   └── format.ts             # Date, currency, number formatters
│   └── types/
│       ├── api.types.ts          # API response types
│       ├── domain.types.ts       # Domain entity types
│       └── common.types.ts
│
├── styles/
│   ├── globals.css               # TailwindCSS 4 directives + CSS vars
│   └── themes.css                # Dark mode + tenant theming vars
│
└── main.tsx                      # Entry point, Unleash init
```

---

## 3. Page Tree & Route Definitions

### Route Table

| Path | Component | Auth | Role | Description |
|------|-----------|------|------|-------------|
| `/login` | `LoginPage` | Public | Any | Keycloak redirect entry point |
| `/auth/callback` | `CallbackPage` | Public | Any | OIDC code exchange handler |
| `/logout` | `LogoutPage` | Public | Any | Post-logout landing |
| `/onboarding/workspace` | `OnboardingWorkspacePage` | Authenticated | Any | Step 1: org setup |
| `/onboarding/invite` | `OnboardingInvitePage` | Authenticated | Any | Step 2: team invite |
| `/onboarding/first-event` | `OnboardingFirstEventPage` | Authenticated | Any | Step 3: first event |
| `/` | `DashboardPage` | Authenticated | Any | Role-aware home |
| `/events` | `EventsPage` | Authenticated | Organizer+ | Event portfolio list |
| `/events/:id` | `EventDetailPage` | Authenticated | Organizer+ | Event shell with tab nav |
| `/events/:id/overview` | `EventOverviewTab` | Authenticated | Organizer+ | Checklist + stats |
| `/events/:id/registration` | `EventRegistrationTab` | Authenticated | Organizer+ | Ticket types + form config |
| `/events/:id/sessions` | `EventSessionsTab` | Authenticated | Organizer+ | Session scheduler |
| `/events/:id/speakers` | `EventSpeakersTab` | Authenticated | Organizer+ | Speaker management |
| `/events/:id/sponsors` | `EventSponsorsTab` | Authenticated | Organizer+ | Sponsor tier builder |
| `/events/:id/communications` | `EventCommunicationsTab` | Authenticated | Organizer+ | Email/SMS campaigns |
| `/events/:id/analytics` | `EventAnalyticsTab` | Authenticated | Organizer+ | Per-event analytics |
| `/events/:id/settings` | `EventSettingsTab` | Authenticated | Admin+ | Event config |
| `/events/:id/checkin` | `CheckInPage` | Authenticated | Staff+ | PWA check-in mode |
| `/attendees` | `AttendeesPage` | Authenticated | Organizer+ | Cross-event attendee list |
| `/venues` | `VenuesPage` | Authenticated | Organizer+ | Venue discovery |
| `/venues/:id` | `VenueDetailPage` | Authenticated | Organizer+ | Venue detail + booking |
| `/speakers` | `SpeakersPage` | Authenticated | Organizer+ | Global speaker roster |
| `/sponsors` | `SponsorsPage` | Authenticated | Organizer+ | Global sponsor list |
| `/campaigns` | `CampaignsPage` | Authenticated | Organizer+ | Cross-event campaigns |
| `/campaigns/new` | `CampaignBuilderPage` | Authenticated | Organizer+ | Campaign creation |
| `/analytics` | `AnalyticsPage` | Authenticated | Manager+ | Cross-event analytics |
| `/ai` | `AIAssistantPage` | Authenticated | Organizer+ | AI assistant interface |
| `/settings` | `SettingsPage` | Authenticated | Admin+ | Org settings shell |
| `/settings/team` | `SettingsTeamTab` | Authenticated | Admin+ | User/role management |
| `/settings/billing` | `SettingsBillingTab` | Authenticated | Owner | Billing + plan |
| `/settings/integrations` | `SettingsIntegrationsTab` | Authenticated | Admin+ | CRM/webhook integrations |
| `/settings/white-label` | `SettingsWhiteLabelTab` | Authenticated | Admin+ | Brand customization |
| `/settings/security` | `SettingsSecurityTab` | Authenticated | Admin+ | SSO/SCIM/audit config |
| `/admin` | `AdminDashboardPage` | Authenticated | PlatformAdmin | Platform overview |
| `/admin/tenants` | `AdminTenantsPage` | Authenticated | PlatformAdmin | Tenant management |
| `/admin/users` | `AdminUsersPage` | Authenticated | PlatformAdmin | Global user admin |
| `/admin/billing` | `AdminBillingPage` | Authenticated | PlatformAdmin | Billing overview |
| `/r/:slug` | `RegistrationPage` | Public | Any | Attendee registration (white-labeled) |
| `/e/:slug/schedule` | `SchedulePage` | Public | Any | Public event schedule |
| `/speaker-portal/:token` | `SpeakerPortalPage` | Public (magic link) | Any | Speaker self-service |
| `*` | `NotFoundPage` | Any | Any | 404 fallback |

### Route Guard Implementation

```typescript
// app/router.tsx
import { createBrowserRouter, Navigate } from 'react-router-dom';
import { AuthGuard } from './guards/AuthGuard';
import { RoleGuard } from './guards/RoleGuard';
import { OnboardingGuard } from './guards/OnboardingGuard';

// AuthGuard: checks token presence + expiry; redirects to /login
// RoleGuard: checks Keycloak realm roles from token claims
// OnboardingGuard: redirects to /onboarding/workspace if !tenant.onboardingComplete

const router = createBrowserRouter([
  {
    path: '/',
    element: <AuthGuard><OnboardingGuard><AppLayout /></OnboardingGuard></AuthGuard>,
    errorElement: <RouteErrorBoundary />,
    children: [
      { index: true, element: <DashboardPage /> },
      {
        path: 'events',
        children: [
          { index: true, element: <EventsPage /> },
          {
            path: ':id',
            element: <EventDetailPage />,
            children: [
              { index: true, element: <Navigate to="overview" replace /> },
              { path: 'overview', element: <EventOverviewTab /> },
              { path: 'registration', element: <EventRegistrationTab /> },
              { path: 'sessions', element: <EventSessionsTab /> },
              { path: 'speakers', element: <EventSpeakersTab /> },
              { path: 'sponsors', element: <EventSponsorsTab /> },
              { path: 'communications', element: <EventCommunicationsTab /> },
              { path: 'analytics', element: <EventAnalyticsTab /> },
              { path: 'settings', element: <RoleGuard roles={['admin', 'owner']}><EventSettingsTab /></RoleGuard> },
            ]
          },
          { path: ':id/checkin', element: <CheckInLayout><CheckInPage /></CheckInLayout> }
        ]
      },
      // ... remaining routes
    ]
  },
  { path: '/login', element: <AuthLayout><LoginPage /></AuthLayout> },
  { path: '/auth/callback', element: <CallbackPage /> },
  // Public routes (no auth guard)
  { path: '/r/:slug', element: <PublicLayout><RegistrationPage /></PublicLayout> },
  { path: '/e/:slug/schedule', element: <PublicLayout><SchedulePage /></PublicLayout> },
]);
```

---

## 4. Component Hierarchy

### Shared Components

```
AppLayout
├── AppSidebar (240px / 56px collapsed)
│   ├── WorkspaceSwitcher
│   ├── NavSection (Overview: Home, Events, Analytics)
│   ├── NavSection (Manage: Attendees, Venues, Speakers, Sponsors)
│   ├── NavSection (Communicate: Campaigns, AI Assistant)
│   ├── NavSection (Organization: Settings, Team, Integrations)
│   └── UserMenu (avatar, name, theme toggle, logout)
├── TopHeader
│   ├── BreadcrumbNav
│   ├── CommandPaletteButton (Cmd+K)
│   ├── NotificationBell (badge + drawer)
│   └── QuickCreateButton (+ dropdown)
└── <Outlet /> (page content)
```

### Feature Component Hierarchy — Events

```
EventsPage
├── PageHeader (title, Create Event button)
├── EventFilters (status, date range, search)
├── ViewToggle (Grid / List / Calendar)
├── EventGrid / EventList / EventCalendar
│   └── EventCard (×n)
│       ├── EventStatusBadge
│       ├── EventCapacityBar
│       └── EventCardActions (Manage, View Page, Duplicate, Delete)
└── CreateEventDrawer (Sheet, right side)
    ├── AIEventSetupInput
    ├── EventForm (React Hook Form)
    │   ├── EventBasicFields
    │   ├── VenueSearchField (async combobox)
    │   ├── DateTimeRangePicker
    │   └── CapacityField
    └── DrawerActions (Save Draft, Publish)

EventDetailPage
├── EventDetailHeader (inline editable title, status badge)
├── PresenceIndicator (who else is viewing/editing)
├── EventContextNav (tab strip: Overview | Registration | Sessions | Speakers...)
├── EventSetupChecklist (collapsible progress bar)
└── <Outlet /> (active tab content)

EventSessionsTab
├── SessionToolbar (Add Session, AI Suggest, View Toggle)
├── SessionScheduler (DnD timeline grid)
│   ├── TimeAxisColumn
│   ├── TrackColumns (×n)
│   │   └── SessionBlock (draggable, resizable)
│   │       ├── SessionConflictHighlight
│   │       └── SessionContextMenu
│   └── DragOverlay
└── CreateSessionDrawer

CheckInPage (PWA mode — CheckInLayout)
├── CheckInHeader (event name, live counter)
├── QRScanner (camera API)
├── AttendeeSearchInput (large, centered)
├── RecentCheckInsList
│   └── CheckInListItem (name, time, status)
└── OfflineBanner (shown when navigator.onLine === false)
```

### AI Feature Components

```
AISuggestionCard
├── AIConfidenceIndicator (Low/Medium/High dot)
├── SuggestionContent
├── AIReasoningTooltip ("Why?" expandable)
└── SuggestionActions
    ├── ApplyButton
    ├── AlternativesButton
    └── IgnoreButton

AIStreamingResponse
├── StreamingProgressBar
├── StreamingTextContent (updates via SSE)
└── CancelStreamButton
```

---

## 5. State Management Architecture

### Philosophy

- **Server state** (anything from the API): **TanStack Query v5** exclusively
- **UI state** (sidebar collapsed, drawer open, selected rows, theme): **Zustand 5** exclusively  
- **Form state**: **React Hook Form** (local to form component, not in Zustand)
- **URL state**: React Router search params for filters, pagination, active tab
- **No Redux**. No Context for data that changes frequently.

### TanStack Query Configuration

```typescript
// app/providers/QueryProvider.tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,           // 1 minute — data stays fresh
      gcTime: 5 * 60 * 1000,          // 5 minutes — cache retention
      retry: (failureCount, error) => {
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 3;
      },
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
    mutations: {
      onError: (error) => globalErrorHandler(error),
    },
  },
});
```

### Query Key Factory

```typescript
// shared/lib/query-keys.ts
export const queryKeys = {
  events: {
    all: () => ['events'] as const,
    list: (filters: EventFilters) => ['events', 'list', filters] as const,
    detail: (id: string) => ['events', 'detail', id] as const,
    analytics: (id: string) => ['events', 'analytics', id] as const,
  },
  attendees: {
    all: () => ['attendees'] as const,
    byEvent: (eventId: string, filters: AttendeeFilters) =>
      ['attendees', 'event', eventId, filters] as const,
    detail: (id: string) => ['attendees', 'detail', id] as const,
  },
  sessions: {
    byEvent: (eventId: string) => ['sessions', 'event', eventId] as const,
  },
  speakers: {
    all: () => ['speakers'] as const,
    byEvent: (eventId: string) => ['speakers', 'event', eventId] as const,
  },
  sponsors: {
    byEvent: (eventId: string) => ['sponsors', 'event', eventId] as const,
  },
  analytics: {
    portfolio: (filters: AnalyticsFilters) => ['analytics', 'portfolio', filters] as const,
    event: (eventId: string) => ['analytics', 'event', eventId] as const,
  },
  tenant: {
    current: () => ['tenant', 'current'] as const,
    theme: () => ['tenant', 'theme'] as const,
    team: () => ['tenant', 'team'] as const,
  },
  search: {
    global: (query: string) => ['search', query] as const,
  },
};
```

### Zustand Store Structure

```typescript
// Stores are scoped — no single god store

// store/ui.store.ts — global UI state
interface UIStore {
  sidebarCollapsed: boolean;
  toggleSidebar: () => void;
  commandPaletteOpen: boolean;
  openCommandPalette: () => void;
  closeCommandPalette: () => void;
  notificationDrawerOpen: boolean;
  toggleNotificationDrawer: () => void;
}

// store/checkin.store.ts — check-in page state (optimistic updates)
interface CheckInStore {
  checkedInCount: number;
  recentCheckIns: CheckInRecord[];
  pendingOfflineCheckIns: OfflineCheckIn[]; // IndexedDB queue
  incrementCount: () => void;
  addRecentCheckIn: (record: CheckInRecord) => void;
  addOfflineCheckIn: (record: OfflineCheckIn) => void;
  clearOfflineQueue: () => void;
}

// store/event-editor.store.ts — active event editing state
interface EventEditorStore {
  activeEventId: string | null;
  unsavedChanges: boolean;
  collaborators: Collaborator[]; // who else is editing
  setActiveEvent: (id: string) => void;
  setCollaborators: (collaborators: Collaborator[]) => void;
  markUnsaved: () => void;
  markSaved: () => void;
}

// store/session-scheduler.store.ts — DnD session state
interface SessionSchedulerStore {
  draggedSession: Session | null;
  conflicts: ConflictPair[];
  setDraggedSession: (session: Session | null) => void;
  setConflicts: (conflicts: ConflictPair[]) => void;
}
```

### Optimistic Updates Pattern

```typescript
// features/events/hooks/useUpdateEvent.ts
export function useUpdateEvent(eventId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: UpdateEventDto) => api.events.update(eventId, data),
    onMutate: async (data) => {
      await queryClient.cancelQueries({ queryKey: queryKeys.events.detail(eventId) });
      const previous = queryClient.getQueryData(queryKeys.events.detail(eventId));
      queryClient.setQueryData(queryKeys.events.detail(eventId), (old: EventDetail) => ({
        ...old,
        ...data,
      }));
      return { previous };
    },
    onError: (_err, _data, context) => {
      queryClient.setQueryData(queryKeys.events.detail(eventId), context?.previous);
      toast.error('Failed to save changes — reverted.');
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.events.detail(eventId) });
    },
  });
}
```

---

## 6. Design System Tokens

### CSS Custom Properties (globals.css)

```css
@layer base {
  :root {
    /* Brand */
    --brand-primary: 239 68% 60%;       /* #6366F1 indigo-500 (HSL) */
    --brand-secondary: 258 90% 66%;     /* #8B5CF6 violet-500 */
    --brand-dark: 243 75% 59%;          /* #4338CA indigo-700 */

    /* Backgrounds */
    --background: 0 0% 100%;            /* #FFFFFF */
    --surface: 210 40% 98%;             /* #F8FAFC slate-50 */
    --surface-hover: 210 40% 96%;

    /* Borders */
    --border: 214 32% 91%;              /* #E2E8F0 slate-200 */
    --border-strong: 215 20% 65%;       /* #94A3B8 slate-400 */

    /* Text */
    --foreground: 222 47% 11%;          /* #0F172A slate-900 */
    --foreground-body: 215 25% 27%;     /* #1E293B slate-800 */
    --foreground-muted: 215 16% 47%;    /* #64748B slate-500 */

    /* Semantic */
    --success: 160 84% 39%;             /* #10B981 emerald-500 */
    --warning: 38 92% 50%;              /* #F59E0B amber-500 */
    --error: 0 84% 60%;                 /* #EF4444 red-500 */
    --info: 217 91% 60%;                /* #3B82F6 blue-500 */

    /* Event Status */
    --status-draft: 215 16% 47%;        /* slate-400 */
    --status-published: 160 84% 39%;    /* emerald-500 */
    --status-live: 0 84% 60%;           /* red-500 — urgency */
    --status-completed: 239 68% 60%;    /* indigo-500 */
    --status-cancelled: 220 9% 46%;     /* gray-500 */

    /* Shadows (light mode) */
    --shadow-card: 0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.06);
    --shadow-card-hover: 0 4px 6px rgba(0,0,0,0.07), 0 2px 4px rgba(0,0,0,0.06);
    --shadow-modal: 0 20px 25px rgba(0,0,0,0.15);

    /* Radius */
    --radius-sm: 6px;
    --radius-md: 8px;
    --radius-lg: 12px;
    --radius-xl: 16px;

    /* Sidebar */
    --sidebar-width: 240px;
    --sidebar-collapsed-width: 56px;
  }

  .dark {
    --background: 222 47% 6%;           /* #0C0E14 */
    --surface: 222 35% 12%;             /* #161B27 */
    --surface-hover: 222 35% 15%;
    --border: 220 30% 18%;              /* #1E2736 */
    --border-strong: 217 19% 35%;
    --foreground: 213 31% 95%;          /* #F1F5F9 slate-100 */
    --foreground-body: 214 32% 81%;     /* #CBD5E1 slate-300 */
    --foreground-muted: 215 20% 65%;    /* #94A3B8 slate-400 */

    /* No box shadows in dark mode — use borders */
    --shadow-card: none;
    --shadow-card-hover: none;
    --shadow-modal: none;
  }
}
```

### TailwindCSS 4 Config

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';

export default {
  darkMode: 'class',
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: {
          primary: 'hsl(var(--brand-primary))',
          secondary: 'hsl(var(--brand-secondary))',
          dark: 'hsl(var(--brand-dark))',
        },
        background: 'hsl(var(--background))',
        surface: 'hsl(var(--surface))',
        border: 'hsl(var(--border))',
        foreground: 'hsl(var(--foreground))',
        muted: 'hsl(var(--foreground-muted))',
        success: 'hsl(var(--success))',
        warning: 'hsl(var(--warning))',
        error: 'hsl(var(--error))',
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'Fira Code', 'monospace'],
      },
      borderRadius: {
        DEFAULT: 'var(--radius-md)',
        sm: 'var(--radius-sm)',
        lg: 'var(--radius-lg)',
        xl: 'var(--radius-xl)',
      },
      boxShadow: {
        card: 'var(--shadow-card)',
        'card-hover': 'var(--shadow-card-hover)',
        modal: 'var(--shadow-modal)',
      },
    },
  },
} satisfies Config;
```

### Typography Scale

```typescript
// Applied via TailwindCSS utility classes — no custom component needed
const typography = {
  xs: 'text-xs leading-4',       // 12px/16px — metadata, timestamps
  sm: 'text-sm leading-5',       // 14px/20px — secondary text, table cells
  base: 'text-base leading-6',   // 16px/24px — body copy
  lg: 'text-lg leading-7',       // 18px/28px — card titles
  xl: 'text-xl leading-7',       // 20px/28px — section headers
  '2xl': 'text-2xl leading-8',   // 24px/32px — page titles
  '3xl': 'text-3xl leading-9',   // 30px/36px — dashboard stats
  '4xl': 'text-4xl leading-10',  // 36px/40px — hero/onboarding
};
```

---

## 7. Authentication Flow

### OIDC Flow with Keycloak

```
1. User navigates to protected route
2. AuthGuard detects no valid token in memory
3. Redirect to /login
4. LoginPage triggers Keycloak authorization_code PKCE flow
   → Redirect to https://auth.eventflow.io/realms/eventflow/protocol/openid-connect/auth
   → PKCE: generate code_verifier + code_challenge (SHA-256)
   → Store code_verifier in sessionStorage (not localStorage)
5. User authenticates with Keycloak (username/password, SSO, or social)
6. Keycloak redirects to /auth/callback?code=XXX&state=YYY
7. CallbackPage exchanges code for tokens:
   → POST /protocol/openid-connect/token
   → Receives: access_token, refresh_token, id_token
8. Tokens stored:
   → access_token: in-memory only (AuthProvider state / Zustand)
   → refresh_token: httpOnly cookie (set by BFF pattern or Keycloak direct)
   → id_token: in-memory for user profile
9. User profile decoded from id_token claims:
   → sub (user ID), email, name, tenant_id (custom claim), roles
10. Silent refresh:
    → useEffect with setInterval at (token_exp - 60s)
    → Uses refresh_token to get new access_token
    → On refresh failure: redirect to /login
11. Logout:
    → Clear in-memory tokens
    → Redirect to Keycloak logout endpoint with id_token_hint
    → Keycloak clears session and redirects to /login
```

### Auth Provider Implementation

```typescript
// app/providers/AuthProvider.tsx
interface AuthState {
  accessToken: string | null;
  user: UserProfile | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: () => void;
  logout: () => void;
  getToken: () => string | null; // for API interceptor
}

// Token storage strategy:
// - Access token: React state (memory) — cleared on page refresh → triggers silent auth
// - Refresh: Handled by Keycloak JS adapter or custom PKCE flow
// - Never localStorage for access tokens (XSS risk)

// Role extraction from token claims:
function extractRoles(token: string): string[] {
  const payload = parseJwt(token);
  return [
    ...(payload.realm_access?.roles ?? []),
    ...(payload.resource_access?.eventflow?.roles ?? []),
  ];
}
```

### API Request Authentication

```typescript
// shared/lib/api-client.ts
import ky from 'ky';
import { useAuthStore } from '../store/auth.store';

export const apiClient = ky.create({
  prefixUrl: '/api',
  hooks: {
    beforeRequest: [
      (request) => {
        const token = useAuthStore.getState().accessToken;
        if (token) {
          request.headers.set('Authorization', `Bearer ${token}`);
        }
        // Correlation ID for distributed tracing
        request.headers.set('X-Correlation-ID', crypto.randomUUID());
      },
    ],
    afterResponse: [
      async (_request, _options, response) => {
        if (response.status === 401) {
          // Attempt silent token refresh
          const refreshed = await refreshAccessToken();
          if (!refreshed) {
            useAuthStore.getState().logout();
          }
        }
      },
    ],
  },
  retry: {
    limit: 3,
    statusCodes: [408, 429, 500, 502, 503, 504],
    backoffLimit: 30000,
  },
  timeout: 30000,
});
```

---

## 8. Navigation Architecture

### Sidebar Implementation

```typescript
// shared/components/AppSidebar.tsx
// Width: 240px expanded, 56px collapsed
// Collapse: controlled by UIStore.sidebarCollapsed
// Auto-collapse: at < 1024px (lg breakpoint)

const navSections = [
  {
    label: 'OVERVIEW',
    items: [
      { label: 'Home', icon: Home, path: '/', shortcut: 'Cmd+1' },
      { label: 'Events', icon: Calendar, path: '/events', shortcut: 'Cmd+2' },
      { label: 'Analytics', icon: BarChart3, path: '/analytics', shortcut: 'Cmd+3',
        featureFlag: 'analytics-dashboard' },
    ],
  },
  {
    label: 'MANAGE',
    items: [
      { label: 'Attendees', icon: Users, path: '/attendees', shortcut: 'Cmd+4' },
      { label: 'Venues', icon: MapPin, path: '/venues', shortcut: 'Cmd+5' },
      { label: 'Speakers', icon: Mic, path: '/speakers' },
      { label: 'Sponsors', icon: Building2, path: '/sponsors',
        featureFlag: 'sponsor-management' },
    ],
  },
  {
    label: 'COMMUNICATE',
    items: [
      { label: 'Campaigns', icon: Mail, path: '/campaigns' },
      { label: 'AI Assistant', icon: Sparkles, path: '/ai',
        featureFlag: 'ai-assistant' },
    ],
  },
  {
    label: 'ORGANIZATION',
    items: [
      { label: 'Settings', icon: Settings, path: '/settings', shortcut: 'Cmd+,' },
      { label: 'Team', icon: UserCog, path: '/settings/team' },
      { label: 'Integrations', icon: Plug, path: '/settings/integrations' },
    ],
  },
];
```

### Command Palette

```typescript
// shared/components/CommandPalette.tsx
// Trigger: Cmd+K (global keydown listener in useCommandPalette hook)
// Base: shadcn/ui <Command> (cmdk primitive)

// Search: debounced 200ms → GET /api/search?q=&types=events,attendees,venues,speakers
// Results: grouped by type with icons and metadata
// Actions: typed as CommandAction — navigate | execute | open-drawer

// Keyboard shortcuts displayed inline
// Recent items: stored in localStorage (last 10 visited resources)
// Quick actions: hardcoded (Create Event, Send Campaign, View Analytics)

export const COMMAND_SHORTCUTS: Record<string, string[]> = {
  'create-event': ['meta', 'n'],
  'open-settings': ['meta', ','],
  'toggle-sidebar': ['meta', 'b'],
  'open-command-palette': ['meta', 'k'],
};
```

### Mobile Bottom Navigation (< 768px)

```typescript
// Replaces sidebar on mobile
// 5 items: Home, Events, [+] Create, Attendees, More
// Center "+" triggers an action sheet (shadcn Sheet from bottom)
// "More" opens a full-screen drawer with remaining nav items

const mobileNavItems = [
  { label: 'Home', icon: Home, path: '/' },
  { label: 'Events', icon: Calendar, path: '/events' },
  { label: 'Create', icon: Plus, action: 'create-event' }, // center CTA
  { label: 'Attendees', icon: Users, path: '/attendees' },
  { label: 'More', icon: Menu, action: 'open-more-drawer' },
];
```

---

## 9. Error Handling Strategy

### Error Boundary Hierarchy

```typescript
// Hierarchy: Route-level > Feature-level > Component-level

// 1. Root ErrorBoundary (catches unhandled errors)
<RootErrorBoundary>         // Shows full-page error with reload button
  <AuthGuard>
    <AppLayout>             // 
      <RouteErrorBoundary>  // Per-route: shows error in content area, sidebar remains
        <FeatureErrorBoundary> // Per-feature: inline error state with retry
          <ComponentLevel /> // graceful degradation
```

### Error Boundary Component

```typescript
// shared/components/RouteErrorBoundary.tsx
// Uses React 19's built-in error boundary integration with router
// errorElement in route definitions catches router-level errors

export function RouteErrorBoundary() {
  const error = useRouteError();
  return (
    <ErrorState
      title={isRouteErrorResponse(error) ? `${error.status} Error` : 'Something went wrong'}
      message={isRouteErrorResponse(error) ? error.statusText : 'An unexpected error occurred.'}
      action={{ label: 'Try again', onClick: () => window.location.reload() }}
    />
  );
}
```

### API Error Classification

```typescript
// shared/lib/api-client.ts — error classification
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    public message: string,
    public fieldErrors?: Record<string, string[]>, // 422 validation errors
    public correlationId?: string,
  ) { super(message); }
}

// Error → UI mapping:
// 400 Bad Request     → Toast: specific message from API
// 401 Unauthorized    → Silent token refresh attempt → /login
// 403 Forbidden       → Toast: "You don't have permission to do that"
// 404 Not Found       → Inline EmptyState component (not toast)
// 409 Conflict        → Inline conflict resolution UI
// 422 Validation      → React Hook Form field-level errors (setError())
// 429 Rate Limited    → Toast with retry-after countdown
// 500+ Server Error   → Toast: "Something went wrong on our end" + retry
// Network timeout     → Toast: "Connection lost — retrying..." (auto-retry)
```

### Toast Strategy (Sonner)

```typescript
// Configuration:
const toastConfig = {
  position: 'bottom-right' as const, // desktop
  richColors: true,
  closeButton: true,
  duration: {
    success: 3000,
    error: 8000,
    warning: 5000,
    info: 'persistent', // dismissed only by success/error or user action
    loading: 'persistent',
  },
};

// Usage patterns:
toast.promise(
  updateEvent(data),
  {
    loading: 'Saving...',
    success: 'Event saved',
    error: (err) => err.message,
  }
);
```

---

## 10. API Client Layer

### Domain API Modules

```typescript
// shared/lib/api/events.api.ts
export const eventsApi = {
  list: (filters: EventFilters) =>
    apiClient.get('events', { searchParams: toSearchParams(filters) }).json<PaginatedResponse<EventSummary>>(),
  detail: (id: string) =>
    apiClient.get(`events/${id}`).json<EventDetail>(),
  create: (data: CreateEventDto) =>
    apiClient.post('events', { json: data }).json<EventDetail>(),
  update: (id: string, data: UpdateEventDto) =>
    apiClient.patch(`events/${id}`, { json: data }).json<EventDetail>(),
  delete: (id: string) =>
    apiClient.delete(`events/${id}`).json<void>(),
  publish: (id: string) =>
    apiClient.post(`events/${id}/publish`).json<EventDetail>(),
  duplicate: (id: string) =>
    apiClient.post(`events/${id}/duplicate`).json<EventDetail>(),
  getAnalytics: (id: string) =>
    apiClient.get(`events/${id}/analytics`).json<EventAnalytics>(),
  // AI endpoints
  generateAgenda: (id: string, prompt: string) =>
    // Returns SSE stream — handled by AIStreamingResponse component
    fetch(`/api/events/${id}/ai/agenda`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${getToken()}` },
      body: JSON.stringify({ prompt }),
    }),
};
```

---

## 11. Real-Time & WebSocket Layer

```typescript
// app/providers/WebSocketProvider.tsx
// Single WS connection per authenticated session
// URL: wss://api.eventflow.io/ws?token={access_token}
// Reconnect: exponential backoff (1s, 2s, 4s, 8s, max 30s)

// Message types from server:
type WSMessage =
  | { type: 'checkin.updated'; payload: { eventId: string; count: number; attendeeId: string } }
  | { type: 'event.updated'; payload: { eventId: string; changes: Partial<EventDetail>; userId: string } }
  | { type: 'presence.joined'; payload: { eventId: string; user: UserPresence } }
  | { type: 'presence.left'; payload: { eventId: string; userId: string } }
  | { type: 'campaign.sent'; payload: { campaignId: string; count: number } }
  | { type: 'ai.agenda.progress'; payload: { jobId: string; progress: number; partial: string } }
  | { type: 'notification.new'; payload: Notification };

// Subscription hook:
export function useWebSocketSubscription<T extends WSMessage>(
  type: T['type'],
  handler: (payload: T['payload']) => void,
  deps: DependencyList = [],
) {
  const { subscribe, unsubscribe } = useWebSocket();
  useEffect(() => {
    const id = subscribe(type, handler);
    return () => unsubscribe(id);
  }, deps);
}

// Usage in CheckInPage:
useWebSocketSubscription('checkin.updated', ({ eventId, count }) => {
  if (eventId === currentEventId) {
    useCheckInStore.getState().setCount(count);
    // Also invalidate TanStack Query cache
    queryClient.invalidateQueries({ queryKey: queryKeys.attendees.byEvent(eventId, {}) });
  }
}, [currentEventId]);
```

---

## 12. AI Feature Patterns

### SSE Streaming for AI Responses

```typescript
// features/ai/components/AIStreamingResponse.tsx
// Handles Server-Sent Events from AI generation endpoints

export function AIStreamingResponse({ jobId, onComplete }: Props) {
  const [content, setContent] = useState('');
  const [progress, setProgress] = useState(0);
  const [isComplete, setIsComplete] = useState(false);

  useWebSocketSubscription('ai.agenda.progress', ({ jobId: id, progress: p, partial }) => {
    if (id !== jobId) return;
    setProgress(p);
    setContent(partial);
    if (p === 100) setIsComplete(true);
  }, [jobId]);

  return (
    <div>
      <Progress value={progress} className="mb-4" />
      <div className="prose dark:prose-invert">
        {content}
        {!isComplete && <BlinkingCursor />}
      </div>
      {isComplete && (
        <div className="flex gap-2 mt-4">
          <Button onClick={() => onComplete(content)}>Apply Agenda</Button>
          <Button variant="ghost" onClick={onRetry}>Regenerate</Button>
        </div>
      )}
    </div>
  );
}
```

### AI Suggestion Card

```typescript
// features/ai/components/AISuggestionCard.tsx
const confidenceConfig = {
  low: { color: 'text-warning', dot: 'bg-warning', label: 'Low confidence — review carefully' },
  medium: { color: 'text-info', dot: 'bg-info', label: 'Medium confidence' },
  high: { color: 'text-success', dot: 'bg-success', label: 'High confidence' },
};
```

---

## 13. PWA & Offline Strategy

### Service Worker Configuration

```typescript
// vite.config.ts — vite-plugin-pwa
VitePWA({
  registerType: 'autoUpdate',
  workbox: {
    globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
    runtimeCaching: [
      {
        urlPattern: /^\/api\/events\/[\w-]+\/attendees/,
        handler: 'NetworkFirst',
        options: {
          cacheName: 'checkin-attendees',
          networkTimeoutSeconds: 3,
          expiration: { maxEntries: 50, maxAgeSeconds: 24 * 60 * 60 }, // 24h
        },
      },
    ],
  },
  manifest: {
    name: 'EventFlow',
    short_name: 'EventFlow',
    theme_color: '#6366F1',
    background_color: '#0C0E14',
    display: 'standalone',
    icons: [
      { src: '/icons/pwa-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icons/pwa-512.png', sizes: '512x512', type: 'image/png', purpose: 'any maskable' },
    ],
  },
})
```

### Offline Check-In (IndexedDB)

```typescript
// features/checkin/hooks/useOfflineCheckIn.ts
// Uses idb library (IndexedDB with Promises)

const DB_NAME = 'eventflow-checkin';
const STORE_NAME = 'pending-checkins';

export function useOfflineCheckIn(eventId: string) {
  const isOnline = useOnlineStatus();

  const checkIn = async (attendeeId: string) => {
    const record: OfflineCheckIn = {
      id: crypto.randomUUID(),
      eventId,
      attendeeId,
      checkedInAt: new Date().toISOString(),
      synced: false,
    };

    if (isOnline) {
      // Optimistic: update counter immediately, sync in background
      await api.checkin.checkIn(eventId, attendeeId);
    } else {
      // Store in IndexedDB for later sync
      const db = await openDB(DB_NAME);
      await db.add(STORE_NAME, record);
      // Update local Zustand counter optimistically
      useCheckInStore.getState().addOfflineCheckIn(record);
      toast.info('Checked in offline — will sync when reconnected');
    }
  };

  // Sync when online status restored
  useEffect(() => {
    if (isOnline) syncOfflineCheckIns(eventId);
  }, [isOnline, eventId]);

  return { checkIn };
}
```

---

## 14. Performance Strategy

### Code Splitting

```typescript
// All page components are lazy-loaded
const DashboardPage = lazy(() => import('../pages/dashboard/DashboardPage'));
const EventDetailPage = lazy(() => import('../pages/events/EventDetailPage'));
const AnalyticsPage = lazy(() => import('../pages/analytics/AnalyticsPage'));
// Suspense boundary at route level with skeleton fallback

// Heavy features split separately:
const SessionScheduler = lazy(() => import('../features/sessions/components/SessionScheduler'));
const CampaignBuilder = lazy(() => import('../features/communications/components/CampaignBuilder'));
const VenueMapView = lazy(() => import('../features/venues/components/VenueMapView'));
```

### React 19 Concurrent Features

```typescript
// useTransition for non-urgent state updates (e.g., filter changes)
const [isPending, startTransition] = useTransition();
const handleFilterChange = (filters: EventFilters) => {
  startTransition(() => setFilters(filters));
};

// useDeferredValue for search input
const deferredQuery = useDeferredValue(searchQuery);

// Streaming SSR via Suspense boundaries:
// Each dashboard card is independently suspended with skeleton fallback
```

### Prefetching Strategy

```typescript
// Prefetch event detail on event card hover (200ms delay)
onMouseEnter={() => {
  setTimeout(() => {
    queryClient.prefetchQuery({
      queryKey: queryKeys.events.detail(event.id),
      queryFn: () => eventsApi.detail(event.id),
      staleTime: 30000,
    });
  }, 200);
}}
```

---

## 15. Testing Architecture

### Test Structure

```
src/
├── __tests__/
│   ├── unit/               # Pure function tests (utils, schemas, formatters)
│   ├── integration/        # Component + hook tests with MSW mocks
│   └── e2e/                # Playwright specs (in /e2e directory at root)

e2e/
├── auth.spec.ts            # Login, logout, token refresh
├── events.spec.ts          # Create, edit, publish event
├── checkin.spec.ts         # Check-in flow, offline mode
├── ai-agenda.spec.ts       # AI agenda generation flow
└── onboarding.spec.ts      # Full onboarding critical path
```

### MSW (Mock Service Worker) Setup

```typescript
// tests/mocks/handlers.ts — API mocks for Vitest
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/events', () => HttpResponse.json(mockEventList)),
  http.post('/api/events', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ ...mockEvent, ...body }, { status: 201 });
  }),
];
```

### Coverage Targets

```
Unit tests:       90% line coverage (business logic, utils, schemas)
Component tests:  85% coverage (all interactive components)
E2E tests:        All critical user journeys:
  - Onboarding (< 10 min to published event)
  - Create/publish event
  - Session scheduling with conflict detection
  - Check-in (online + offline simulation)
  - Campaign creation and send
  - AI agenda generation
  - White-label registration page
```
