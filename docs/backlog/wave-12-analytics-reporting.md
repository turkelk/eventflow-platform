# Wave 12: Analytics & Reporting

## 12.1 Implement cross-event portfolio analytics dashboard and ROI reporting

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement the executive analytics features per API Specification section 17 and Persona 2 (David - Marketing Director).

**Key files:**
- `src/EventFlow.Application/Analytics/Queries/GetTenantDashboardQuery.cs` (role-aware, cached 5min)
- `src/EventFlow.Application/Analytics/Queries/GetCrossEventReportQuery.cs`
- `src/EventFlow.Application/Analytics/Queries/GetROIReportQuery.cs`
- `src/EventFlow.Application/Analytics/Queries/GetRegistrationTrendQuery.cs`
- `src/EventFlow.Application/Analytics/Queries/GetSessionAttendanceQuery.cs`
- `frontend/src/pages/analytics/AnalyticsPage.tsx`
- `frontend/src/features/analytics/components/CrossEventMetrics.tsx`

**Acceptance criteria:**
- [ ] `GetTenantDashboardQuery` returns different `role`-aware payload for EventManager vs Executive vs CheckInStaff
- [ ] `GetCrossEventReportQuery` aggregates across events: total registrations, revenue, attendance rate, avg NPS
- [ ] `GetROIReportQuery` returns revenue, cost inputs, estimated pipeline attribution
- [ ] `GetRegistrationTrendQuery` supports Hourly/Daily/Weekly granularity
- [ ] Analytics queries route to read replica via `IDbContextFactory<ReadDbContext>` to avoid OLTP impact

---

## 12.2 Build analytics pages: event analytics, registration trends, session attendance charts

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the analytics frontend pages per 04-FRONTEND-ARCHITECTURE.md and Persona 1 (Maya) and Persona 2 (David) requirements.

**Key files:**
- `frontend/src/pages/events/EventAnalyticsTab.tsx`
- `frontend/src/features/analytics/components/RevenueChart.tsx` (Recharts LineChart)
- `frontend/src/features/analytics/components/AttendanceChart.tsx` (Recharts BarChart)
- `frontend/src/features/analytics/components/EventROICard.tsx`
- `frontend/src/features/analytics/components/CrossEventMetrics.tsx`
- `frontend/src/features/analytics/hooks/useAnalytics.ts`

**Acceptance criteria:**
- [ ] Registration trend chart shows daily/hourly/weekly granularity toggle
- [ ] Session attendance table sortable by attendance count, click-through to session detail
- [ ] Revenue chart shows rolling 12-month trend for Executive dashboard
- [ ] All charts have accessible table fallback with `role="table"` for screen readers
- [ ] Chart data cached via TanStack Query staleTime 5 minutes

---

## 12.3 Implement event metrics snapshots background job and campaign analytics

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the analytics data collection infrastructure per Domain Model section 2.7.

**Key files:**
- `src/EventFlow.Application/Analytics/Queries/GetEventAnalyticsQuery.cs`
- `src/EventFlow.Application/Analytics/Queries/GetCampaignAnalyticsQuery.cs`
- `src/EventFlow.Infrastructure/BackgroundServices/EventMetricsSnapshotJob.cs` (every 15min for live events)
- `src/EventFlow.Infrastructure/BackgroundServices/EventStreamConsumer.cs` (analytics-processor consumer group)

**Snapshot fields:** TotalRegistrations, ConfirmedRegistrations, CheckedInCount, TotalRevenue, TotalPageViews, ConversionRate, EmailOpenRate

**Acceptance criteria:**
- [ ] `EventMetricsSnapshotJob` writes `EventMetricsSnapshot` record every 15 minutes for live/upcoming events
- [ ] Analytics consumer group processes `checkin.completed`, `attendee.registered`, `campaign.delivery_updated` events
- [ ] `GetCampaignAnalyticsQuery` returns: totalSent, delivered, opens, uniqueOpens, clicks, bounces, openRate, clickRate
- [ ] Analytics queries use read replica connection to avoid impacting OLTP throughput
- [ ] `EventMetricsSnapshot` append-only (no updates) for accurate trend analysis

---

