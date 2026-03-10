# UX Research & Design Intelligence

> **EventFlow** — Mid-Market Event Management SaaS
> Document Version: 1.0 | Status: Approved for Implementation

---

## Table of Contents

1. [Research Methodology](#1-research-methodology)
2. [Competitor UX Analysis](#2-competitor-ux-analysis)
   - 2.1 Cvent
   - 2.2 Eventbrite
   - 2.3 Hopin / RingCentral Events
   - 2.4 Notion (Adjacent: Collaboration & Structure)
   - 2.5 Linear (Adjacent: Modern SaaS UX Benchmark)
3. [Competitive Gap Analysis](#3-competitive-gap-analysis)
4. [Target User Personas](#4-target-user-personas)
5. [Design System Decisions](#5-design-system-decisions)
6. [Navigation Architecture](#6-navigation-architecture)
7. [Onboarding Flow Design](#7-onboarding-flow-design)
8. [Dashboard Layout Strategy](#8-dashboard-layout-strategy)
9. [Key Interaction Patterns](#9-key-interaction-patterns)
10. [Mobile Responsiveness Strategy](#10-mobile-responsiveness-strategy)
11. [Dark Mode & Theming](#11-dark-mode--theming)
12. [Accessibility Standards](#12-accessibility-standards)
13. [Implementation Recommendations Summary](#13-implementation-recommendations-summary)

---

## 1. Research Methodology

### Approach
This document synthesizes UX analysis of five SaaS products — three direct competitors in event management and two adjacent-domain benchmarks chosen for their exemplary modern SaaS UX patterns. For each product, we analyzed:

- Public product screenshots, demo videos, and marketing materials
- App Store / G2 / Capterra user reviews focusing on UX complaints and praise
- Publicly available case studies and design blog posts
- Heuristic evaluation against Nielsen's 10 usability principles
- Competitor weakness data from the BRD competitive intelligence section

### Why Adjacent Benchmarks Matter
Event management tools (Cvent, Eventbrite) are notoriously behind the curve on UX. Including Notion and Linear — which serve knowledge workers and engineering teams respectively — establishes the UX quality bar that modern mid-market SaaS buyers now expect. These tools have trained users to expect fast, keyboard-navigable, collaborative, and aesthetically refined interfaces.

---

## 2. Competitor UX Analysis

### 2.1 Cvent

**Product**: Enterprise event management platform. $1,500–$25,000+/year.

#### Visual Design Language
- **Colors**: Corporate navy (#1B3A6B), white, orange accent (#FF6600). Dated palette that reads as early-2010s enterprise software.
- **Typography**: Proprietary font stack, heavy use of medium-weight sans-serif. Text density is high; walls of text common in dashboards.
- **Spacing**: Cramped. Information density prioritized over breathing room. Tables are the primary UI metaphor everywhere.
- **Iconography**: Functional but inconsistent — mix of flat icons and older skeuomorphic elements. No unified icon system.
- **Overall Aesthetic**: Feels like enterprise ERP software. Functional but visually intimidating.

#### Navigation Patterns
- **Structure**: Top navigation bar with mega-menus + left sidebar for sub-navigation. Two competing navigation systems create cognitive overhead.
- **Depth**: Users report needing 4–6 clicks to reach common tasks (e.g., exporting an attendee list).
- **Search**: Basic global search, not a command palette. No keyboard shortcut support.
- **Breadcrumbs**: Present but inconsistent; users frequently report being "lost" in the product.

#### Onboarding Flow
- **Approach**: Sales-led. Most users access Cvent after a lengthy sales process and formal onboarding by a Cvent CSM.
- **Self-Service**: Minimal. No meaningful interactive product tour.
- **Time to First Value**: Days to weeks. Requires training sessions before users can create their first event.
- **Documentation**: Extensive but overwhelming — hundreds of help articles with no clear learning path.

#### Dashboard Layout
- **Primary View**: A tabular list of events with status columns. No visual overview of pipeline health.
- **Metrics Shown**: Registration counts, event dates. Advanced analytics are buried 3+ levels deep.
- **Personalization**: None. Every user sees the same dashboard regardless of role or recent activity.
- **Activity Feed**: Absent. No real-time signal of what's happening across events.

#### Key Interactions
- **Forms**: Multi-step wizard forms that are long and unforgiving. Validation errors appear only on submit.
- **Modals**: Heavy use of full-page forms rather than contextual drawers or inline editing.
- **Drag-Drop**: Limited. Session scheduling requires navigating to a separate "Agenda Builder" module.
- **Inline Editing**: Mostly absent. Nearly everything requires opening a separate edit form.
- **Real-Time Collaboration**: None. Two users editing the same event simultaneously will overwrite each other.

#### Mobile Responsiveness
- **Organizer App**: Separate mobile app for check-in ("Cvent Check-in"). Inconsistent experience from web.
- **Web (Responsive)**: The admin portal is not meaningfully responsive. Not usable on mobile for event management.
- **Attendee Experience**: A separate attendee hub app. Three different apps (admin web, check-in mobile, attendee hub) creates fragmentation.

#### Dark Mode / Theming
- **Dark Mode**: Not available.
- **White-Labeling**: Available for attendee-facing pages but not the organizer backend.
- **Theming**: Organizer UI is fixed. Brand customization is registration-page-only.

#### G2/Capterra UX Complaints (Representative Quotes)
- *"The UI is from 2008 and hasn't really changed. Everything takes too many clicks."*
- *"I dread using Cvent. My team actively avoids it until they have to."*
- *"The learning curve is brutal. We paid for two weeks of onboarding and still feel lost."*
- *"No dark mode, no keyboard shortcuts, no way to quickly find anything."*

---

### 2.2 Eventbrite

**Product**: Self-service event ticketing and discovery platform. Freemium to ~2.5% + $1.79/ticket.

#### Visual Design Language
- **Colors**: Orange primary (#F05537), white backgrounds, gray secondary text. Consumer-grade palette designed for broad appeal rather than professional tooling.
- **Typography**: Clean sans-serif (custom "EBDisplay"), reasonable hierarchy. Better typographic system than Cvent.
- **Spacing**: Generous on consumer-facing pages; admin backend is more cramped.
- **Overall Aesthetic**: Consumer-app feel. Works well for individual event creators; feels too casual for corporate event managers running 50+ events.

#### Navigation Patterns
- **Structure**: Top navigation bar only. Admin panel uses a flat left sidebar.
- **Depth**: Shallower than Cvent — most tasks are 2–3 clicks. Achieves simplicity by offering fewer features.
- **Search**: Events searchable but no command palette in organizer admin.
- **Context Switching**: Easy between events but no multi-event dashboard overview.

#### Onboarding Flow
- **Approach**: Product-led. "Create your first event" CTA prominently on sign-up.
- **Time to First Value**: 15–20 minutes to a published event page. Excellent for simple events.
- **Guided Tour**: Minimal tooltips, relies on intuitive layout.
- **Limitation**: The simple onboarding breaks down when users need multi-session events, sponsor management, or CRM integration — the platform simply doesn't support these well.

#### Dashboard Layout
- **Primary View**: "Your Events" card grid sorted by date. Visually pleasant.
- **Metrics**: Registration count and revenue per event shown on cards. Clean.
- **Missing**: No cross-event analytics, no attendee journey view, no capacity planning.
- **Suitability**: Works for 1–10 events/year. Starts to feel inadequate for 50+ events.

#### Key Interactions
- **Event Creation**: Linear wizard. Simple and effective for basic events.
- **Inline Editing**: Available for event descriptions. Not available for tickets or attendees.
- **Check-In**: Separate Eventbrite Organizer app. Clean mobile UI but limited customization.
- **Real-Time Collaboration**: None. Single-user assumption baked into the product.

#### Mobile Responsiveness
- **Organizer Web**: Responsive but feature-limited on mobile. Core flows work; advanced settings don't.
- **Mobile App**: "Eventbrite Organizer" is well-designed for check-in but not for event management.
- **Attendee Experience**: Consumer app is polished. Organizer experience lags.

#### Dark Mode / Theming
- **Dark Mode**: Not available in organizer admin.
- **Event Pages**: Brand color customization available. Templates are limited.

#### G2/Capterra UX Complaints
- *"Great for simple events. The moment you need anything complex, you hit walls everywhere."*
- *"No multi-user collaboration. My team can't work on the same event at the same time."*
- *"Missing analytics. I can't see which sessions were most popular or build a post-event report."*

---

### 2.3 Hopin / RingCentral Events

**Product**: Virtual and hybrid event platform. Acquired by RingCentral in 2023. ~$1,500–$15,000/year.

#### Visual Design Language
- **Colors**: Dark-mode-first design (#111827 background, purple accents #7C3AED). Modern and confident.
- **Typography**: Inter font family. Excellent hierarchy; readable in dense information contexts.
- **Spacing**: Generous whitespace in content areas. Sidebar components are denser by necessity.
- **Overall Aesthetic**: The most visually modern of the direct competitors. Clearly built post-2020 with contemporary design sensibilities.

#### Navigation Patterns
- **Structure**: Left sidebar (collapsible) + content area. Single navigation system. Clean.
- **Sections**: Dashboard, Events, Speakers, Sponsors, Analytics. Logical grouping.
- **Event Context**: When inside an event, context stays persistent in sidebar. Users always know where they are.
- **Keyboard Support**: Some keyboard navigation but no full command palette.

#### Onboarding Flow
- **Approach**: Product-led with sales assist for enterprise.
- **Interactive Demo**: Available. Lets prospective users explore a pre-built event.
- **Guided Setup**: "Event Setup Checklist" progress bar is a standout pattern. Shows % complete.
- **Time to First Value**: ~30 minutes for a virtual event. Longer for hybrid.

#### Dashboard Layout
- **Primary View**: Upcoming events with capacity and registration progress bars. More informative than Eventbrite.
- **Analytics Preview**: Shows total registrations, event count, and a revenue trend sparkline on the main dashboard.
- **Recent Activity**: Shows recent registrations and check-ins. Useful signal for organizers.

#### Key Interactions
- **Session Builder**: Drag-and-drop timeline. Most sophisticated of the competitors.
- **Speaker Management**: Invite flow is smooth — generates a magic link. Good pattern.
- **Sponsor Tiers**: Visual tier builder is intuitive. Drag sponsors between tiers.
- **Real-Time Collaboration**: Limited. Shows "X is editing" indicator but no true concurrent editing.

#### Mobile Responsiveness
- **Web Admin**: Responsive with clear breakpoints. Usable on tablets for event monitoring.
- **Attendee App**: Purpose-built mobile app. Good networking features.
- **Weakness**: Advanced features (analytics, custom reporting) not accessible on mobile.

#### Dark Mode / Theming
- **Dark Mode**: Default and only option in some views. Inconsistently applied.
- **White-Label**: Available at higher tiers. Custom domain + brand colors.

#### G2/Capterra UX Complaints
- *"Post-RingCentral acquisition, the product feels less focused. Features feel bolted on."*
- *"Strong for virtual, weaker for in-person. Hybrid feels like two different products stitched together."*
- *"No AI features. I have to manually match attendees and build agendas myself."*

---

### 2.4 Notion

**Product**: Collaborative workspace / knowledge management. Adjacent domain. $8–$15/user/month.

> **Why Included**: Notion represents the gold standard for mid-market SaaS UX that this target audience uses daily. Event managers at 50–500 person companies almost certainly use Notion. Our product must feel as familiar and modern.

#### Visual Design Language
- **Colors**: Deliberately minimal. Near-white (#FFFFFF / #F7F7F5) backgrounds. Black text. Accent colors only for status indicators.
- **Typography**: Custom "Notion" font + system fallbacks. Exceptional readability at all sizes.
- **Spacing**: Generous. Content-first philosophy. Navigation chrome is minimal.
- **Density**: User-controlled. Compact, Default, and Comfortable modes.
- **Overall Aesthetic**: Clean, confident, focused. Feels like a premium tool without being ostentatious.

#### Navigation Patterns
- **Structure**: Collapsible left sidebar with nested tree navigation. Industry-standard for knowledge tools.
- **Keyboard**: Extensive keyboard shortcuts. `/` command for block insertion is iconic.
- **Command Palette**: `Cmd/Ctrl+K` global search and command palette. Essential for power users.
- **Breadcrumbs**: Top bar shows full path. Persistent context.
- **Sidebar Sections**: Favorites, Workspaces, Private pages. Role-based visibility.

#### Onboarding Flow
- **Template Gallery**: New users choose a template to start. Immediate value without a blank-canvas problem.
- **Interactive Tooltip Tour**: Non-intrusive. Tooltips appear contextually, not as a forced linear tour.
- **Workspace Setup**: Team name, invite teammates, choose use case. 3 steps. ~5 minutes.
- **Time to First Value**: Under 10 minutes for personal use. Under 20 minutes for team setup.

#### Dashboard Layout
- **Concept**: No fixed "dashboard" — the user's home page is customizable.
- **Home View**: Recent pages, pinned favorites, team updates. Personalized from day one.
- **Lesson**: Users want to see *their* data, not generic platform metrics.

#### Key Interactions
- **Inline Editing**: Everything is editable inline. No separate "edit mode."
- **Block-Based Editing**: Drag-and-drop blocks. Intuitive and flexible.
- **Mentions/Comments**: `@mention` anywhere. Contextual comments on any block.
- **Multi-Select**: Bulk actions on multiple items. Expected by power users.
- **Real-Time Collaboration**: Multiplayer cursors. Industry benchmark for collaboration.

#### Mobile Responsiveness
- **Approach**: Separate but consistent mobile app experience. Same information architecture.
- **Reading vs. Writing**: Reading content is excellent on mobile. Complex editing is intentionally simplified.

#### Dark Mode / Theming
- **Dark Mode**: System-aware + manual toggle. Well-implemented.
- **Custom Fonts**: Serif, Sans-Serif, Mono options.
- **Page Covers/Icons**: Personalization that delights without overwhelming.

---

### 2.5 Linear

**Product**: Issue tracking and project management for software teams. Adjacent domain. $8–$16/user/month.

> **Why Included**: Linear is the modern SaaS UX benchmark for professional tools. It is frequently cited by mid-market SaaS product teams as the UX standard they aspire to. Its interaction design philosophy directly informs our approach.

#### Visual Design Language
- **Colors**: Near-black backgrounds in default (dark) mode (#0F0F0F, #1A1A1A). Purple accent (#5E6AD2). Light mode is equally refined.
- **Typography**: Inter. Exceptional typographic system with clear hierarchy.
- **Spacing**: Dense but breathable. Information-rich without feeling cluttered — achieved through exceptional grid discipline.
- **Motion**: Purposeful micro-animations. Page transitions are instant (no loading spinners for cached data). Status changes animate smoothly.
- **Overall Aesthetic**: The benchmark for "power tool that is also beautiful." Feels fast before you even measure performance.

#### Navigation Patterns
- **Structure**: Narrow left sidebar (icons only at minimum width) + second-level navigation panel + content area. Triptych layout.
- **Command Palette**: `Cmd/Ctrl+K` opens a fuzzy-search command palette. Can execute any action without touching a mouse.
- **Keyboard-First**: Every feature has a keyboard shortcut. Keyboard shortcut guide accessible from help menu.
- **Context**: Current team/project always visible in sidebar. Never lost.

#### Onboarding Flow
- **Approach**: "Create your first issue in 60 seconds" — radical simplicity for onboarding.
- **Progressive Disclosure**: Advanced features (cycles, roadmaps, integrations) hidden until user is comfortable with core.
- **No Forced Tour**: The product is so intuitive that a guided tour feels unnecessary. Hints appear inline.

#### Dashboard Layout
- **My Issues**: Personalized view of work assigned to the current user. Role-relevant immediately.
- **Inbox**: Notifications, mentions, and updates. Separates noise from signal.
- **Views**: Multiple built-in views (Board, List, Timeline). User picks their preferred.

#### Key Interactions
- **Triage Shortcut**: Press `T` to triage unprocessed issues. Single-key actions throughout.
- **Inline Editing**: Every field editable on click. No separate edit screens.
- **Drag-Drop**: Kanban columns, timeline reordering. Smooth and responsive.
- **Bulk Actions**: Checkbox multi-select with floating action bar.
- **Real-Time**: Changes reflect instantly across all connected clients.

#### Mobile Responsiveness
- **Philosophy**: "Core actions on mobile, full power on desktop."
- **Mobile App**: Native iOS/Android. Not just a responsive web view.
- **Scope Management**: Mobile app focuses on viewing and updating issues, not complex configuration.

#### Dark Mode / Theming
- **Dark Mode**: Default and most polished state of the app.
- **Light Mode**: Available and equally refined.
- **System Aware**: Respects OS preference on first load.

---

## 3. Competitive Gap Analysis

This section maps directly to the BRD's competitive weaknesses and defines how EventFlow's UX addresses each gap.

### Gap Matrix

| UX Dimension | Cvent | Eventbrite | Hopin | EventFlow Opportunity |
|---|---|---|---|---|
| **Time to First Event** | Days/Weeks | 15–20 min | 30–45 min | **< 10 minutes** with AI-assisted setup |
| **Navigation Complexity** | High (dual nav systems) | Low (but shallow) | Medium | **Linear-inspired sidebar**: single navigation system, collapsible, keyboard-navigable |
| **Real-Time Collaboration** | None | None | Partial | **Notion-inspired**: multiplayer cursors, presence indicators, live field updates |
| **Mobile Organizer UX** | Fragmented (3 apps) | Fragmented (2 apps) | Partial | **Unified**: single responsive web app + PWA; same capabilities on all devices |
| **AI Features** | None | None | None | **Core differentiator**: AI agenda builder, attendee matchmaking, smart communications |
| **Dashboard Personalization** | None | None | None | **Role-aware home**: different defaults for Event Manager vs. Executive vs. Check-in Staff |
| **Dark Mode** | Absent | Absent | Inconsistent | **System-aware dark mode** from day one |
| **Command Palette** | Absent | Absent | Absent | **Cmd/Ctrl+K** global command palette — power user multiplier |
| **Inline Editing** | Rare | Partial | Partial | **Everything editable inline** — no separate edit screens |
| **Keyboard Navigation** | Minimal | Minimal | Minimal | **Comprehensive keyboard shortcuts** throughout |
| **Multi-Event Overview** | Complex/Reports-only | Absent | Basic | **Portfolio dashboard**: across-event metrics, pipeline health |
| **Visual Design Quality** | Dated (2010s enterprise) | Consumer-grade | Modern | **Linear-quality aesthetics** — the benchmark mid-market expects |

### The Three Core UX Bets

**Bet 1: Speed of Mastery**
Every competitor fails on onboarding. Cvent requires days. EventFlow targets < 10 minutes to a published event. This is achieved through: AI-assisted event creation, progressive disclosure of advanced features, a template library, and a setup checklist with visual progress.

**Bet 2: Collaboration as a First-Class Citizen**
No competitor offers true real-time multi-user collaboration on event setup. EventFlow treats this as a core feature, not an afterthought. Multiple team members can edit the same event simultaneously with live presence indicators. This directly addresses the "disconnected toolchains" frustration in the BRD.

**Bet 3: AI That's Visible and Trustworthy**
AI features in SaaS often feel like magic boxes users don't trust. EventFlow surfaces AI reasoning: "I suggested this agenda order because attendees with [profile A] tend to engage better in morning sessions." Transparency builds adoption.

---

## 4. Target User Personas

### Persona 1: Maya — The Event Program Manager
**Age**: 32 | **Company Size**: 200 employees | **Events/Year**: 40–80
**Tech Comfort**: High. Uses Linear, Notion, Slack, Figma daily.
**Primary Goal**: Run a flawless conference without managing 6 disconnected tools.
**Key Frustrations with Competitors**:
- "Cvent makes me feel stupid. It shouldn't be this hard."
- "I can't collaborate with my co-organizer in real time. We're always stepping on each other."
- "Building an agenda takes me 4 hours. I know AI could do a first draft in minutes."

**EventFlow UX Implications**:
- Command palette is essential for Maya's speed
- AI agenda builder directly solves her biggest time sink
- Real-time collaboration with her co-organizer is table stakes
- She expects dark mode — she's using Linear all day

### Persona 2: David — The Marketing Director
**Age**: 44 | **Company Size**: 120 employees | **Events/Year**: 10–20
**Tech Comfort**: Medium. Comfortable with HubSpot, Google Analytics, basic SaaS.
**Primary Goal**: Prove event ROI to the CFO. Run branded, professional events.
**Key Frustrations with Competitors**:
- "I can't get a clear picture of which events drove pipeline."
- "Cvent is $15K/year. I can't justify that to the board for 15 events."
- "I need our events to look like us, not like an Eventbrite page."

**EventFlow UX Implications**:
- Executive dashboard with cross-event ROI metrics is David's home screen
- White-label registration pages with brand color/font customization are essential
- Pricing must be self-evident (no "contact sales" friction)
- Dashboard should not require training — immediate comprehension

### Persona 3: Sarah — The Event Coordinator (Day-Of Staff)
**Age**: 26 | **Company Size**: 80 employees | **Events/Year**: 30+ (in supporting role)
**Tech Comfort**: Medium-High. Comfortable with consumer apps, basic SaaS.
**Primary Goal**: Check in 500 attendees in 20 minutes without chaos.
**Key Frustrations with Competitors**:
- "The check-in app crashes or lags. It's embarrassing at the door."
- "I can't see who's checked in from my phone in real time."
- "I have to use three different apps on event day."

**EventFlow UX Implications**:
- Check-in view must be a PWA that works offline (with sync when reconnected)
- Single URL / single app on event day — no app installs required
- Large touch targets, high-contrast UI for hectic venue environments
- Real-time check-in counter visible to all staff simultaneously

### Persona 4: James — The IT Administrator
**Age**: 38 | **Company Size**: 350 employees | **Events/Year**: N/A (platform admin)
**Tech Comfort**: Very High. Manages SSO, SCIM, security policies.
**Primary Goal**: Ensure EventFlow integrates with company SSO and meets data governance requirements.
**Key Frustrations with Competitors**:
- "Cvent has no SCIM provisioning. I manage 200 user accounts manually."
- "Eventbrite doesn't support SSO. Security team rejected it."

**EventFlow UX Implications**:
- Admin portal must surface Keycloak SSO configuration clearly
- SCIM/SAML setup should be self-service with a guided wizard
- Audit logs must be easily filterable and exportable
- Role management UI must be intuitive (no reading a 40-page manual)

---

## 5. Design System Decisions

### Technology: TailwindCSS 4 + shadcn/ui

**Rationale**: shadcn/ui provides unstyled-but-structured components (Radix UI primitives) that give us accessibility and behavior for free while TailwindCSS 4 handles all visual styling. This is the most widely adopted pattern in the React/TypeScript ecosystem as of 2024–2025 and aligns with what mid-market SaaS developers expect to maintain.

### Color System

```
Primary Palette:
  Brand Primary:    #6366F1  (Indigo 500 — modern, professional, not overused)
  Brand Secondary:  #8B5CF6  (Violet 500 — gradient companion)
  Brand Dark:       #4338CA  (Indigo 700 — hover/active states)

Neutral Palette (Light Mode):
  Background:       #FFFFFF
  Surface:          #F8FAFC  (Slate 50)
  Border:           #E2E8F0  (Slate 200)
  Muted Text:       #64748B  (Slate 500)
  Body Text:        #1E293B  (Slate 800)
  Heading Text:     #0F172A  (Slate 900)

Neutral Palette (Dark Mode):
  Background:       #0C0E14
  Surface:          #161B27
  Border:           #1E2736
  Muted Text:       #94A3B8  (Slate 400)
  Body Text:        #CBD5E1  (Slate 300)
  Heading Text:     #F1F5F9  (Slate 100)

Semantic Colors:
  Success:          #10B981  (Emerald 500)
  Warning:          #F59E0B  (Amber 500)
  Error:            #EF4444  (Red 500)
  Info:             #3B82F6  (Blue 500)

Event Status Colors:
  Draft:            #94A3B8  (Slate 400)
  Published:        #10B981  (Emerald 500)
  Live:             #EF4444  (Red 500 — live events need urgency)
  Completed:        #6366F1  (Indigo 500)
  Cancelled:        #6B7280  (Gray 500)
```

**Why Indigo/Violet?** The primary competitor palette (Cvent navy/orange, Eventbrite orange, Hopin purple) all anchor on single-hue schemes. Indigo-to-violet gradient reads as: modern, trustworthy, slightly premium. It's used by Stripe, Linear, and other "best-in-class" SaaS brands. It avoids the corporate rigidity of navy and the casualness of orange.

### Typography

```
Font Family:
  Display/Headings:  'Inter', system-ui, sans-serif (weight: 600, 700)
  Body:              'Inter', system-ui, sans-serif (weight: 400, 500)
  Monospace:         'JetBrains Mono', 'Fira Code', monospace
  (Inter via Google Fonts CDN; self-hosted for GDPR compliance)

Scale (TailwindCSS base):
  xs:     12px / 16px line-height   (metadata, timestamps)
  sm:     14px / 20px line-height   (secondary text, table cells)
  base:   16px / 24px line-height   (body copy)
  lg:     18px / 28px line-height   (card titles)
  xl:     20px / 28px line-height   (section headers)
  2xl:    24px / 32px line-height   (page titles)
  3xl:    30px / 36px line-height   (dashboard stats)
  4xl:    36px / 40px line-height   (hero/onboarding)
```

**Rationale**: Inter is the industry standard for modern SaaS. Used by Linear, Vercel, Figma, Loom, and hundreds of high-quality products. Exceptional legibility at small sizes in data-dense interfaces. No licensing concerns.

### Spacing System

Strict adherence to TailwindCSS's default 4px base grid. All spacing in components uses `p-2` (8px), `p-4` (16px), `p-6` (24px), `gap-3` (12px), etc. No arbitrary spacing values outside of the grid. This creates visual consistency without a manual design system audit.

### Shadow System
```
Card shadow (light):    0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.06)
Card shadow (hover):    0 4px 6px rgba(0,0,0,0.07), 0 2px 4px rgba(0,0,0,0.06)
Modal shadow:           0 20px 25px rgba(0,0,0,0.15)
Dropdown shadow:        0 4px 6px rgba(0,0,0,0.07)
(Dark mode: remove box shadows, use border instead — borders read better on dark backgrounds)
```

### Border Radius
```
Buttons:         rounded-md  (6px)
Cards:           rounded-xl  (12px)
Input fields:    rounded-md  (6px)
Modals:          rounded-2xl (16px)
Badges/Pills:    rounded-full
Large containers: rounded-2xl (16px)
```

---

## 6. Navigation Architecture

### Decision: Left Sidebar + Command Palette

**Chosen Pattern**: Collapsible left sidebar (Linear/Notion-style) with `Cmd/Ctrl+K` command palette overlay.

**Rejected Patterns**:
- **Top nav with mega-menus** (Cvent): Creates dual navigation cognitive overhead. Rejected.
- **Top nav only** (Eventbrite): Too shallow for a feature-rich platform. Rejected.
- **Pure tab-based**: Doesn't scale to EventFlow's feature depth. Rejected.

### Sidebar Structure

```
Left Sidebar (240px expanded, 56px icon-only collapsed)
│
├── [Logo / Workspace Switcher]  ← Click to switch tenants
│
├── OVERVIEW
│   ├── 🏠 Home (personalized dashboard)
│   ├── 📅 Events
│   └── 📊 Analytics
│
├── MANAGE
│   ├── 👥 Attendees
│   ├── 📍 Venues
│   ├── 🎤 Speakers
│   └── 🏢 Sponsors
│
├── COMMUNICATE
│   ├── ✉️ Campaigns
│   └── 🤖 AI Assistant
│
├── ORGANIZATION
│   ├── ⚙️ Settings
│   ├── 👤 Team
│   └── 🔗 Integrations
│
└── [User Avatar + Name]  ← Profile, billing, logout
```

### Navigation Behavior Rules

1. **Active State**: Current section highlighted with `bg-indigo-50 text-indigo-700` (light) or `bg-indigo-950 text-indigo-300` (dark). Left border indicator `border-l-2 border-indigo-600`.

2. **Collapse Behavior**: Sidebar collapses to icon-only on `lg` breakpoint if user has toggled it, or automatically at `md` breakpoint. Tooltip shows section name on icon hover when collapsed.

3. **Context Sidebar (Event Detail View)**: When inside a specific event (`/events/{id}`), the main sidebar remains visible and a secondary context panel appears in the content area showing the event sub-navigation (Overview, Registration, Sessions, Speakers, Sponsors, Communications, Analytics, Settings). This is NOT a nested sidebar — it's an in-content tab strip.

4. **Keyboard Navigation**: `Cmd+1` through `Cmd+7` for main sidebar sections. Arrow keys navigate within lists. Tab moves between interactive elements.

5. **Notifications Bell**: In the top-right header area (not in the sidebar). Shows a badge count. Opens a Hopin-inspired notification drawer from the right.

### Command Palette Design (`Cmd/Ctrl+K`)

Implemented via shadcn/ui `<Command>` component (Radix UI `cmdk` primitive).

```
Command Palette — full-width centered modal overlay
┌─────────────────────────────────────────────────────────┐
│  🔍  Search events, attendees, actions...                │
├─────────────────────────────────────────────────────────┤
│  RECENT                                                   │
│  📅  Q4 Sales Kickoff 2025          Event               │
│  📅  Product Launch Webinar         Event               │
│  👤  Maya Chen                      Attendee            │
├─────────────────────────────────────────────────────────┤
│  QUICK ACTIONS                                            │
│  ➕  Create new event               ⌘N                  │
│  📧  Send campaign                                       │
│  📊  View analytics                                      │
│  ⚙️  Open settings                  ⌘,                  │
├─────────────────────────────────────────────────────────┤
│  NAVIGATE TO                                              │
│  🏠  Home                                                │
│  📅  Events                                              │
│  👥  Attendees                                           │
└─────────────────────────────────────────────────────────┘
```

Search results are fetched via debounced API calls. Results include: events (with status badge), attendees (with avatar), venues, speakers. Actions execute immediately (e.g., "Create new event" navigates to the creation flow).

---

## 7. Onboarding Flow Design

### Design Goal: < 10 Minutes to Published Event

This is EventFlow's primary competitive advantage over Cvent (days) and Hopin (30–45 min). Achieving this requires ruthless prioritization of the onboarding critical path.

### Step 1: Account Creation & Workspace Setup (2 min)
```
Screen: Sign Up
- Email + Password  OR  [Sign in with Google] [Sign in with Microsoft]
- After auth: "Let's set up your workspace"
  - Organization name (pre-filled from email domain if recognizable)
  - Industry (dropdown: Technology, Finance, Healthcare, Education, Other)
  - Events per year (slider: 1–10, 10–50, 50–200, 200+)
  - [Continue →]
- Keycloak handles auth; workspace info flows to tenant provisioning
```

### Step 2: Invite Your Team (Optional, Deferrable) (30 sec)
```
Screen: "EventFlow works best with your team"
- Email invite field (up to 5 emails)
- [Invite Team →] or [Skip for now, I'll do this later]
- Non-blocking: skipping doesn't penalize the user
- Progress: Step 2 of 3
```

### Step 3: Create Your First Event (5–7 min)
```
Screen: "Create your first event — let AI help"

Option A — AI-Assisted (Primary CTA):
  - Text input: "Describe your event in a sentence"
  - Example: "Two-day SaaS product conference for 300 attendees in Austin, March 2025"
  - [Generate with AI →]
  - AI fills: Event name, description, suggested agenda structure, recommended session types
  - User reviews and adjusts inline — all fields editable immediately
  - Estimated time: 3–4 minutes

Option B — Manual (Secondary CTA):
  - Traditional form: Name, Date, Location, Capacity
  - Simple, no AI involvement
  - Estimated time: 5–7 minutes

Option C — From Template (Tertiary CTA):
  - Template gallery: "Annual Conference", "Team Offsite", "Product Launch", "Webinar", "Workshop"
  - Each template pre-fills appropriate fields and session structures
  - Estimated time: 2–3 minutes
```

### Step 4: Event Setup Checklist (Progressive)
```
After first event is created, user lands on the Event Overview page with a:
"Setup Checklist" component (collapsible, pinned to the top initially)

□ ✅ Event created                          (done)
□    Customize registration page            → [Customize]
□    Add your first session                 → [Add Session]
□    Set ticket types & pricing             → [Set Up Tickets]
□    Invite speakers                        → [Invite Speakers]
□    Publish your event                     → [Publish]

Progress bar: ██░░░░  1 of 5 complete

This checklist uses the Hopin pattern (the best pattern observed) but extends it
with inline action buttons that launch focused task drawers without leaving the page.
Dismissable after 3 tasks are complete. Accessible from help menu afterward.
```

### Anti-Patterns to Avoid
- ❌ **Forced product tours** that overlay UI before the user can explore
- ❌ **Video embeds** that autoplay in onboarding (intrusive, not scannable)
- ❌ **"Complete your profile"** nag screens before the user has created any data
- ❌ **Email verification gates** that block access before first value (verify async)
- ❌ **Feature flooding** — showing all navigation sections before the user has context for them

### Progressive Feature Disclosure

Advanced features (AI Matchmaking, Sponsor Management, Custom Integrations, White-Label Domains) are hidden from the sidebar by default and revealed progressively:
- After 1st event created: Sponsor Management unlocks
- After 1st event published: White-Label settings unlocks
- After 10 attendees registered: AI Matchmaking feature flag tooltip appears
- After 1st event completed: Advanced Analytics unlocks

Implemented via Unleash feature flags + user property checks in the frontend `useFeatureFlags` hook.

---

## 8. Dashboard Layout Strategy

### Philosophy: Role-Aware Personalized Home

No competitor offers a personalized dashboard. EventFlow's home screen adapts based on the user's role and recent activity. This is a direct response to the G2 complaints about generic dashboards.

### Layout: Event Manager Home Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│ HEADER: Good morning, Maya ☀️    [+ Create Event]  [Quick Actions ▾]│
├─────────────────────────────────────────────────────────────────────┤
│ STAT CARDS (4-column grid)                                           │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────────┐│
│ │ Upcoming     │ │ Registered   │ │ Open Tasks   │ │ Revenue MTD ││
│ │ Events: 7    │ │ This Month:  │ │ 12 pending   │ │ $24,500     ││
│ │ ↑2 vs last  │ │ 1,240 ↑18%  │ │ 3 overdue ⚠ │ │ ↑34% MoM   ││
│ └──────────────┘ └──────────────┘ └──────────────┘ └─────────────┘│
├─────────────────────────────────────────────────────────────────────┤
│ TWO-COLUMN SPLIT (60/40)                                             │
│                                                                      │
│ LEFT: Upcoming Events (next 30 days)     RIGHT: Activity Feed        │
│ ┌───────────────────────────────────┐   ┌───────────────────────┐  │
│ │ [Event Card] Q4 Sales Kickoff     │   │ 🟢 LIVE NOW           │  │
│ │  Mar 15 · Austin TX · 280 reg     │   │ DevSummit 2025        │  │
│ │  ████████░░ 280/350 capacity      │   │ 47 checked in / 120   │  │
│ │  [Manage] [View Page]             │   ├───────────────────────┤  │
│ ├───────────────────────────────────┤   │ RECENT ACTIVITY       │  │
│ │ [Event Card] Product Webinar      │   │ • Sarah K. registered │  │
│ │  Mar 22 · Virtual · 89 reg        │   │   for Q4 Kickoff      │  │
│ │  ████░░░░░░ 89/200 capacity       │   │   2 min ago           │  │
│ │  [Manage] [View Page]             │   │ • New speaker invite  │  │
│ └───────────────────────────────────┘   │   accepted: Dr. Kim   │  │
│                                         │   15 min ago          │  │
│                                         │ • Campaign sent to    │  │
│                                         │   450 attendees       │  │
│                                         │   1 hour ago          │  │
│                                         └───────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│ BOTTOM: AI Insights Banner (dismissable)                             │
│ 🤖 "Registration for Q4 Sales Kickoff is tracking 23% below this    │
│    time last year. Consider sending a reminder campaign." [Do it →] │
└─────────────────────────────────────────────────────────────────────┘
```

### Layout: Executive (Marketing Director) Home Dashboard

```
Different from Event Manager:
- Top row: Revenue MTD, Pipeline Attributed, Total Attendees YTD, Avg NPS
- Main chart: Revenue and attendance trend over rolling 12 months
- Event performance table: All events sorted by ROI
- No "activity feed" — executives need summary, not stream of micro-events
```

### Layout: Check-In Staff Home Dashboard

```
Radically simplified for day-of use:
- Full-screen: "[Event Name] Check-In"
- Large counter: "142 / 350 Checked In" with live updating
- Search bar (huge, centered): Scan QR or type name
- Recent check-ins list
- No sidebar navigation (focused mode)
- High contrast, large touch targets for venue/glare conditions
```

### Dashboard Data Loading Strategy

- **Skeleton screens** (not spinners) for all card components. Skeleton matches the exact dimensions of the loaded content to prevent layout shift.
- **Optimistic updates**: Check-in increments immediately on the counter before the server confirms.
- **Stale-while-revalidate**: Dashboard stats show cached data instantly, refresh in background. Staleness indicator: "Updated 2 min ago" with a subtle refresh icon.
- **Real-time via WebSocket** (Redis Streams backed): Live check-in counter, activity feed. Reconnection handled automatically.

---

## 9. Key Interaction Patterns

### 9.1 Event Creation Flow

**Pattern**: Right-side drawer (800px width on desktop, full-screen on mobile) for creating new events.

**Rationale**: Full-page navigation for event creation breaks context. Users should be able to create an event while referencing another event in the background. Drawer keeps the main content visible.

**Implementation**: shadcn/ui `<Sheet>` component with `side="right"`. Animated slide-in from right. ESC to close with unsaved changes warning.

```
Create Event Drawer:
┌────────────────────────────────────────────────────────┐
│ ✕  Create Event              [Save Draft] [Publish →]  │
├────────────────────────────────────────────────────────┤
│ 🤖 "Describe your event for AI setup"                  │
│ [_________________________________________________]    │
│ [Generate Agenda] [Skip]                               │
├────────────────────────────────────────────────────────┤
│ Event Name *                                           │
│ [________________________________]                     │
│                                                        │
│ Event Type     [Conference ▾]  Format  [In-Person ▾]  │
│                                                        │
│ Start Date/Time            End Date/Time              │
│ [📅 Mar 15, 2025 9:00 AM]  [📅 Mar 16, 2025 6:00 PM] │
│                                                        │
│ Venue                          Capacity               │
│ [🔍 Search venues...]          [____]                 │
│                                                        │
│ Description (Markdown supported)                      │
│ [______________________________________________]      │
│ [______________________________________________|      │
└────────────────────────────────────────────────────────┘
```

### 9.2 Inline Editing

**Pattern**: Click-to-edit on all text fields in detail views.

**Implementation**:
- Single-click on any text field in a detail view activates inline edit mode
- Field gets a subtle border highlight (`ring-2 ring-indigo-500`)
- `Enter` or click-away to save; `Esc` to cancel
- Auto-save with debounce (800ms after last keystroke)
- Save indicator: brief checkmark animation on successful save
- Conflict detection: if another user has modified the same field since the current user loaded it, show a conflict resolution modal

**Affected Views**: Event detail header, session titles, attendee notes, speaker bio, sponsor tier names.

### 9.3 Drag-and-Drop Patterns

**Library**: `@dnd-kit/core` (preferred over react-beautiful-dnd which is unmaintained).

**Use Cases**:
1. **Session Scheduler**: Drag sessions onto a timeline grid. Supports resize (drag bottom edge to change duration). Conflict detection highlights overlapping sessions in red.
2. **Sponsor Tier Builder**: Drag sponsor logos between tier rows (Platinum, Gold, Silver, Bronze). Reorder within a tier.
3. **Agenda Builder**: Drag agenda items to reorder. Parent/child relationship for multi-track sessions.
4. **Speaker Lineup**: Drag speakers to session slots.

**Accessibility**: All drag-and-drop operations have keyboard equivalents (focus element → arrow keys to move → Enter to confirm placement).

### 9.4 Modals vs. Drawers vs. Inline — Decision Framework

| Scenario | Pattern | Reason |
|---|---|---|
| Creating a new entity (event, session, speaker) | **Right drawer** | Large form, needs space, preserves context |
| Confirming a destructive action (delete event) | **Center modal** | Small, focused, requires explicit decision |
| Editing a single field | **Inline** | Minimum disruption |
| Viewing attendee details | **Right drawer** | Medium form, preserve event context |
| AI generation in progress | **Center modal with progress** | Focus attention on async operation |
| Bulk action confirmation | **Bottom bar** (floating) | Non-blocking, shows count of affected items |
| Filters panel | **Left panel slide-in** | Doesn't obscure data being filtered |
| Notification details | **Right notification drawer** | Consistent with notification bell location |

### 9.5 Form Validation Pattern

**Standard**: React Hook Form + Zod (client-side schema mirrors FluentValidation server-side).

**Rules**:
- Validation runs on-blur for individual fields (not on-change — too aggressive)
- Submit validation runs on all fields simultaneously
- Error messages appear below the field with a red indicator
- First error field receives focus on submit attempt
- Server-side validation errors (from FluentValidation 422 responses) are mapped back to specific form fields — not just a generic toast
- Required fields marked with `*` — explained once at the top of each form: "Required fields are marked with an asterisk"

### 9.6 Bulk Actions

**Pattern**: Checkbox multi-select with floating action bar.

```
When 1+ rows are selected:
┌──────────────────────────────────────────────────────────────┐
│  ✓ 34 attendees selected   [Email] [Export] [Cancel Ticket]  │
│                            [Add Tag] [Remove] [✕ Deselect]   │
└──────────────────────────────────────────────────────────────┘
(Fixed-position bar at the bottom of the viewport)
```

### 9.7 Toast Notification System

**Library**: shadcn/ui `<Sonner>` (Sonner toast library — lightweight, accessible).

**Positioning**: Bottom-right on desktop, bottom-center on mobile.

**Types**:
- **Success** (green): "Event published successfully" — 3s auto-dismiss
- **Error** (red): "Failed to send campaign — check email settings" — 8s, with action button
- **Warning** (amber): "Event capacity is 95% full" — 5s
- **Info** (blue): "AI is building your agenda... (this takes ~15s)" — no auto-dismiss, replaced by success toast
- **Loading** (with spinner): For async operations. Replaced by success/error when complete.

### 9.8 AI Interaction Patterns

AI features are a core differentiator. They need unique interaction patterns:

**AI Suggestion Cards**:
```
┌─────────────────────────────────────────────────────────────┐
│ 🤖 AI Suggestion                                      [✕]   │
│                                                              │
│ "Based on 847 similar tech conferences, I suggest adding    │
│  a 30-minute networking break at 10:30 AM. Attendee         │
│  satisfaction scores 23% higher with mid-morning breaks."  │
│                                                              │
│ [Apply Suggestion ✓]  [See Alternatives]  [Ignore]         │
└─────────────────────────────────────────────────────────────┘
```

**Reasoning Transparency**: Every AI suggestion includes a "Why?" link that expands a brief explanation. This builds user trust and adoption.

**AI Confidence Indicator**: Low/Medium/High confidence shown as a colored dot. Low-confidence suggestions are labeled: "Not sure about this one — review carefully."

**Undo AI Actions**: All AI-applied suggestions are immediately undoable via `Cmd+Z` or the "Undo" toast action.

---

## 10. Mobile Responsiveness Strategy

### Philosophy: Progressive Enhancement + Role-Optimized Mobile Views

**Decision**: Single codebase responsive web app (PWA) rather than separate native apps.

**Rationale**:
- Cvent's 3-app fragmentation is a top user complaint. We eliminate this entirely.
- Mid-market IT teams don't want to manage an MDM policy for a native event app.
- PWA with service workers provides the offline capability needed for check-in.
- React 19 + TailwindCSS 4's responsive utilities make this feasible with excellent quality.

### Breakpoint Strategy

```
TailwindCSS Breakpoints:
  sm:   640px   (large phones landscape)
  md:   768px   (tablets portrait)
  lg:   1024px  (tablets landscape / small laptops)
  xl:   1280px  (laptops)
  2xl:  1536px  (large desktops)

Layout Behavior:
  < md:   Single column. Sidebar becomes bottom navigation bar (5 items).
          All drawers become full-screen modals.
          Tables become card lists.
          Charts become simplified sparklines.

  md–lg:  Two column where applicable. Sidebar collapses to icon-only.
          Drawers are 90% width.

  ≥ lg:   Full sidebar + content area. All desktop patterns active.
```

### Mobile Bottom Navigation (< md)

```
┌──────────────────────────────────────────────────────┐
│                  (Content Area)                       │
│                                                       │
├──────────────────────────────────────────────────────┤
│  🏠 Home  │  📅 Events  │  ➕  │  👥 Attendees  │  ☰ More  │
└──────────────────────────────────────────────────────┘
         Center ➕ button = quick-create action sheet
```

### Check-In PWA Mode

The check-in experience (`/events/{id}/checkin`) is a specialized PWA mode:

```
PWA Check-In Mode Features:
- Install prompt shown proactively for check-in staff
- Service worker caches the full attendee list for the specific event
- Offline mode: check-ins stored locally, synced when connectivity restored
- Conflict resolution: if same attendee checked in on 2 devices offline, deduplicate by earliest timestamp
- Camera API: QR code scanner uses device camera (no extra app needed)
- Manifest: Standalone display mode (no browser chrome)
- Theme color: Brand primary (#6366F1) for native-feeling status bar
```

### Touch Target Standards

- Minimum touch target: 44×44px (Apple HIG standard)
- Tap spacing: 8px minimum between adjacent targets
- Swipe gestures: Swipe left on list items to reveal delete/archive actions (standard mobile pattern)
- Long press: Opens context menu on list items (alternative to right-click)

---

## 11. Dark Mode & Theming

### Dark Mode Implementation

**Approach**: CSS custom properties + TailwindCSS `dark:` variant. System-aware by default.

```typescript
// Theme implementation in layout root
// tailwind.config.ts
export default {
  darkMode: 'class', // Enables class-based dark mode toggling
  // ...
}

// ThemeProvider.tsx — wraps entire app
// Uses localStorage for persistence
// Respects prefers-color-scheme on first load
// Options: 'light' | 'dark' | 'system'
```

**Toggle Location**: User menu (bottom of sidebar) with `Sun / Moon / Monitor` icon options.

**Dark Mode Design Principles**:
1. **No pure black** (#000): Use dark slate (#0C0E14) — easier on eyes and shows depth
2. **Elevation via lightness**: Cards lighter than background, modals lighter than cards
3. **Reduced saturation**: Colored elements (status badges, buttons) slightly desaturated in dark mode to reduce glare
4. **Border over shadow**: In dark mode, use `border border-slate-700` instead of box shadows for card delineation
5. **Consistent semantic colors**: Green stays green in dark mode (adjusted brightness), red stays red — don't invert semantics

### White-Label Theming

For the white-label feature (a core differentiator over competitors), tenant-level theming is stored in the database and applied via CSS custom properties:

```css
/* Applied to :root at the tenant level */
--brand-primary: {tenant.primaryColor};
--brand-secondary: {tenant.secondaryColor};
--brand-font: {tenant.fontFamily};
--brand-logo: url({tenant.logoUrl});
--brand-favicon: url({tenant.faviconUrl});
```

**Scope**: White-label theming applies to:
- Registration pages (attendee-facing)
- Email templates
- Event schedule pages (public)
- Check-in kiosk view

**NOT** white-labeled: The organizer admin backend (EventFlow brand is retained). This is industry standard and acceptable per BRD.

**Implementation**:
- `TenantThemeProvider` React context wraps attendee-facing routes
- CSS custom properties set dynamically based on `useCurrentTenant()` hook
- Contrast ratio validation: Auto-adjust text color to meet WCAG AA if tenant brand colors have poor contrast

### Accessibility & Theming

All color choices validated against:
- WCAG 2.1 AA minimum (4.5:1 contrast for normal text, 3:1 for large text)
- WCAG 2.1 AAA target for primary navigation elements
- Validated in both light and dark modes
- Validated against tenant white-label color overrides (validation run at theme save time)

---

## 12. Accessibility Standards

### Target: WCAG 2.1 AA

All components from shadcn/ui (Radix UI primitives) are accessible by default. EventFlow's standards:

**Focus Management**:
- Visible focus ring on all interactive elements: `ring-2 ring-offset-2 ring-indigo-500`
- Focus trapped within modals/drawers when open
- Focus restored to trigger element when modal/drawer closes
- Skip-to-content link at the top of every page (visually hidden, shown on focus)

**ARIA**:
- All icons used as buttons have `aria-label`
- Dynamic content (live check-in counter) marked with `aria-live="polite"`
- Chart data has accessible table fallback
- Status badges use `role="status"` with descriptive text (not just color)

**Keyboard**:
- All interactive features reachable and operable by keyboard alone
- Drag-and-drop has keyboard equivalent (as specified in section 9.3)
- Date pickers support keyboard date entry (not just calendar click)

**Reduced Motion**:
```css
@media (prefers-reduced-motion: reduce) {
  /* All transitions and animations disabled */
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

**Screen Reader Testing**: All flows tested with VoiceOver (macOS/iOS) and NVDA (Windows) before release.

---

## 13. Implementation Recommendations Summary

This section consolidates the actionable design decisions for the engineering team.

### Must-Have for v1 (MVP)

| # | Decision | Rationale | Competitor Gap |
|---|---|---|---|
| 1 | **Left sidebar navigation** (Linear-style, collapsible) | Single navigation system eliminates Cvent's dual-nav confusion | Cvent weakness |
| 2 | **`Cmd/Ctrl+K` command palette** | Power user multiplier; zero competitors have this | All competitors |
| 3 | **< 10 min onboarding** with AI-assisted event creation | Primary conversion advantage; Cvent requires days | Cvent weakness |
| 4 | **Setup checklist** (Hopin-inspired) with inline action buttons | Guides users to activation without a forced tour | Hopin pattern, improved |
| 5 | **Inline editing** on all detail view fields | Eliminates round-trip to edit forms; modern user expectation | All competitors |
| 6 | **System-aware dark mode** from day one | Requested by power users (Maya persona); zero competitors offer this | All competitors |
| 7 | **Skeleton screens** for all async data | Perceived performance critical for adoption | All competitors |
| 8 | **Role-aware home dashboard** | Event Manager vs. Executive vs. Check-in views | All competitors |
| 9 | **PWA check-in mode** with offline support | Eliminates fragmented app problem; Sarah persona's top need | Cvent weakness |
| 10 | **Right-side drawer** for create/edit flows | Context preservation; standard modern SaaS pattern | All competitors |

### Must-Have for v2 (Post-MVP)

| # | Decision | Rationale |
|---|---|---|
| 11 | **Real-time collaboration** (multiplayer cursors) | Notion-pattern; top G2 complaint across all competitors |
| 12 | **AI suggestion cards** with reasoning transparency | Core differentiator; builds trust in AI features |
| 13 | **White-label theming** for attendee-facing pages | David persona's requirement; competitive with Hopin |
| 14 | **Bulk action floating bar** | Power user need for managing 200+ attendee lists |
| 15 | **Mobile bottom navigation** for tablet/phone | Field staff usability (Sarah persona) |

### Technology Alignment

| UX Decision | Frontend Implementation | Backend Requirement |
|---|---|---|
| Command palette search | shadcn/ui `<Command>` + debounced fetch | `GET /api/search?q=` — unified search endpoint |
| Real-time activity feed | WebSocket via Redis Streams subscription | Redis Streams event publishing from all write handlers |
| AI agenda builder | Streaming response UI with progress indicator | MediatR handler calling AI service with SSE response |
| Inline editing auto-save | React Hook Form + debounce + optimistic update | PATCH endpoints for all editable resources |
| Offline check-in | Service Worker + IndexedDB for attendee cache | Sync endpoint: `POST /api/events/{id}/checkin/sync` |
| Tenant theming | CSS custom properties set by `TenantThemeProvider` | `GET /api/tenant/theme` — served before page render |
| Feature flags | `useUnleash()` hook gating component render | Unleash client initialized in `main.tsx` |
| Dark mode persistence | `localStorage` + `class` strategy in Tailwind | None (client-side only) |

### Design Review Checklist

Before any screen ships to production:

- [ ] Contrast ratio ≥ 4.5:1 for all text (verified in both light and dark modes)
- [ ] All interactions reachable by keyboard alone
- [ ] Skeleton screens implemented for all async data loading states
- [ ] Empty states designed (not blank pages) with actionable CTAs
- [ ] Error states designed (API failures shown gracefully, not as blank screens)
- [ ] Mobile layout tested at 375px (iPhone SE) and 768px (iPad)
- [ ] Loading indicators use skeleton (not spinner) for content areas
- [ ] All form fields have associated `<label>` elements
- [ ] Destructive actions require confirmation with a modal
- [ ] AI features show reasoning transparency ("Why?" link)
- [ ] Feature flag gates tested in both enabled and disabled states

---

*Document maintained by: Product Design Team*
*Last Updated: See git history*
*Next Review: After first 50 user interviews on beta*
