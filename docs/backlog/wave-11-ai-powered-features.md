# Wave 11: AI-Powered Features

## 11.1 Implement AI Agenda Generation: streaming suggestion, apply, dismiss, and reasoning transparency

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement the AI agenda builder feature as a core competitive differentiator per API Specification section 18 and 01-UX-RESEARCH.md AI interaction patterns.

**Key files:**
- `src/EventFlow.Application/AI/Commands/GenerateAgendaSuggestionCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/AI/Commands/ApplyAgendaSuggestionCommand.cs`
- `src/EventFlow.Application/AI/Commands/DismissAgendaSuggestionCommand.cs`
- `src/EventFlow.Application/Common/Interfaces/IAiService.cs` (one method per AI operation)
- `src/EventFlow.Infrastructure/Adapters/AiService.cs` (OpenAI SDK, thin wrapper)
- `frontend/src/features/ai/components/AIStreamingResponse.tsx`
- `frontend/src/features/ai/components/AISuggestionCard.tsx`

**Acceptance criteria:**
- [ ] POST `/api/v1/ai/events/{id}/agenda` returns 202 Accepted with `pollUrl` immediately
- [ ] AI suggestion includes `reasoningText` ("Why?" transparency) and `confidence` (Low/Medium/High)
- [ ] `ApplyAgendaSuggestionCommand` creates sessions atomically, undoable via `Cmd+Z`
- [ ] `CostTrackingBehavior` records `TokensUsed` per tenant for monthly budget enforcement
- [ ] Streaming progress delivered via WebSocket `ai.agenda.progress` events to frontend

---

## 11.2 Implement AI Attendee Matchmaking: embedding generation, vector similarity, match suggestions

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement AI-powered attendee matchmaking using pgvector per 02-DOMAIN-MODEL.md AI context and API Specification section 18.

**Key files:**
- `src/EventFlow.Application/Attendees/Commands/GenerateAttendeeEmbeddingCommand.cs` (record + handler)
- `src/EventFlow.Application/AI/Commands/GenerateMatchSuggestionsCommand.cs`
- `src/EventFlow.Application/AI/Commands/SendMatchNotificationsCommand.cs`
- `src/EventFlow.Application/AI/Queries/GetMatchSuggestionsQuery.cs`
- `src/EventFlow.Infrastructure/Adapters/AiService.cs` (GenerateEmbeddingAsync method)
- `frontend/src/features/ai/hooks/useAIMatchmaking.ts`

**Vector search SQL:** `ORDER BY profile_embedding <=> $1 LIMIT 10` (cosine similarity with ivfflat index)

**Acceptance criteria:**
- [ ] `GenerateAttendeeEmbeddingCommand` creates OpenAI embedding from `MatchingProfile` JSONB, stores in `profile_embedding` column
- [ ] `GenerateMatchSuggestionsCommand` uses pgvector cosine similarity to find top matches per attendee
- [ ] `MatchSuggestion` stores `match_reasons` JSONB array: ["Both in SaaS", "Similar seniority"]
- [ ] Attendee must have `networking_opt_in = true` to participate in matchmaking
- [ ] ivfflat index on `attendees.profile_embedding` with `lists = 100` for performance

---

## 11.3 Implement AI Campaign Copy Generation and AI Insights Banner

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement AI-generated email copy and proactive dashboard insights per API Specification section 18.

**Key files:**
- `src/EventFlow.Application/AI/Commands/GenerateCampaignCopyCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/AI/Commands/GenerateInsightsBannerCommand.cs`
- `src/EventFlow.Application/AI/Commands/DismissInsightCommand.cs`
- `src/EventFlow.Application/AI/Queries/GetActiveInsightsQuery.cs` (cached 5min)
- `src/EventFlow.Infrastructure/BackgroundServices/AiInsightsGeneratorService.cs` (periodic)
- `frontend/src/features/ai/components/AIReasoningTooltip.tsx`

**Insight types:** RegistrationLagging, CapacityWarning, LowConversionRate, SessionLowSignups, CampaignOpportunity

**Acceptance criteria:**
- [ ] `AiInsightsGeneratorService` runs every 15 minutes for live events and every hour for upcoming events
- [ ] Each insight includes `actionUrl` deep-linking to the recommended action
- [ ] `DismissInsightCommand` marks insight as dismissed for the tenant (not regenerated until new data)
- [ ] `GetActiveInsightsQuery` cached 5 minutes; invalidated when new insight generated
- [ ] AI confidence indicator shown on all suggestion cards (Low=amber dot, Medium=blue, High=green)

---

## 11.4 Build AI Assistant page and AI suggestion card UI components

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the frontend AI features per 04-FRONTEND-ARCHITECTURE.md sections 12 and AI interaction patterns from 01-UX-RESEARCH.md section 9.8.

**Key files:**
- `frontend/src/pages/ai/AIAssistantPage.tsx`
- `frontend/src/features/ai/components/AIAssistantInput.tsx`
- `frontend/src/features/ai/components/AISuggestionCard.tsx` (reasoning transparency, confidence indicator)
- `frontend/src/features/ai/components/AIConfidenceIndicator.tsx` (Low/Medium/High colored dot)
- `frontend/src/features/ai/components/AIStreamingResponse.tsx` (SSE progress bar)
- `frontend/src/features/ai/hooks/useAIAgendaBuilder.ts`

**Acceptance criteria:**
- [ ] Every AI suggestion shows "Why?" expandable reasoning tooltip
- [ ] Low-confidence suggestions labeled: "Not sure about this one — review carefully"
- [ ] AI-applied suggestions immediately undoable via `Cmd+Z` (calls dismiss endpoint)
- [ ] Streaming response updates content in real-time via WebSocket `ai.agenda.progress`
- [ ] AI generation cost indicator shows token usage for power users (opt-in setting)

---

## 11.5 Implement AI-assisted event creation in onboarding with "describe your event" input

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:m`, `agent:frontend`
**Status:** Pending

Implement the primary AI onboarding path from 01-UX-RESEARCH.md section 7 (Option A: AI-Assisted).

**Key files:**
- `src/EventFlow.Application/AI/Commands/GenerateEventFromDescriptionCommand.cs` (record + handler + validator)
- `src/EventFlow.Infrastructure/Adapters/AiService.cs` (GenerateEventFromDescriptionAsync)
- `frontend/src/pages/onboarding/OnboardingFirstEventPage.tsx` (AI tab as primary CTA)

**AI output:** Event name, description, suggested agenda structure, recommended session types, suggested ticket types

**Acceptance criteria:**
- [ ] "Describe your event in a sentence" input sends prompt to AI, fills event form fields
- [ ] AI-generated fields are all editable inline immediately after generation
- [ ] Example placeholder: "Two-day SaaS product conference for 300 attendees in Austin, March 2025"
- [ ] If AI call fails, gracefully falls back to blank manual form with toast notification
- [ ] Generated event creation flow completes in < 4 minutes (measured in E2E test)

---

