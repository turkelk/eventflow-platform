# Wave 9: Infrastructure

## 9.1 Create Helm charts for API service with HPA, PDB, NetworkPolicy, and Sealed Secrets

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:l`, `agent:devops`
**Status:** Pending

Implement the core API Helm chart per 06-INFRASTRUCTURE-DESIGN.md sections 3.3-3.8.

**Key files:**
- `helm/eventflow-api/Chart.yaml`
- `helm/eventflow-api/templates/deployment.yaml` (non-root user, readOnlyRootFilesystem, resource limits)
- `helm/eventflow-api/templates/hpa.yaml` (min:2, max:20, CPU 60%, Memory 70%)
- `helm/eventflow-api/templates/pdb.yaml` (minAvailable: 1)
- `helm/eventflow-api/templates/networkpolicy.yaml` (deny-all + explicit allow to PG/Redis/Keycloak)
- `helm/eventflow-api/templates/secret.yaml` (Sealed Secret template)
- `helm/eventflow-api/values.yaml`, `values.staging.yaml`, `values.production.yaml`

**Acceptance criteria:**
- [ ] Pod runs as non-root (UID 1001), `readOnlyRootFilesystem: true`, all capabilities dropped
- [ ] HPA scales up 2 pods per 60s, scale down stabilized 300s
- [ ] NetworkPolicy: ingress only from ingress-nginx namespace, egress only to PG/Redis/Keycloak
- [ ] PDB ensures at least 1 pod available during node drain
- [ ] `kubeseal` instructions documented for secret rotation

---

## 9.2 Create Helm charts for frontend, Keycloak, PostgreSQL, Redis, and supporting services

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:l`, `agent:devops`
**Status:** Pending

Implement remaining Helm charts and ArgoCD application manifests per 06-INFRASTRUCTURE-DESIGN.md.

**Key files:**
- `helm/eventflow-frontend/templates/deployment.yaml`
- `helm/eventflow-worker/templates/deployment.yaml` (HPA on Redis stream pending count)
- `helm/eventflow-ingress/templates/ingress.yaml` (TLS, HSTS, WebSocket, rate limiting annotations)
- `helm/eventflow-ingress/templates/certificate.yaml` (cert-manager wildcard for white-label)
- `argocd/applications/eventflow-api.yaml` (auto-sync, prune, selfHeal)
- `argocd/projects/eventflow.yaml`
- `helm/eventflow-monitoring/templates/` (Prometheus ServiceMonitor, Seq StatefulSet)

**Acceptance criteria:**
- [ ] ArgoCD auto-syncs on image tag change committed by CI pipeline
- [ ] cert-manager issues Let's Encrypt wildcard cert for `*.eventflow.io`
- [ ] Worker HPA scales on external metric `redis_stream_pending_count > 100`
- [ ] Ingress annotations set all security headers from NFR section 2.5
- [ ] `argocd app list` shows all applications Healthy and Synced after deployment

---

## 9.3 Implement CI/CD Docker build, push, and ArgoCD deployment pipeline

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:m`, `agent:devops`
**Status:** Pending

Implement the full CD pipeline from 06-INFRASTRUCTURE-DESIGN.md section 5.1.

**Key files:**
- `.github/workflows/ci-cd.yml` (security scan + backend + frontend + docker + deploy-staging + deploy-production)
- `.github/workflows/release.yml` (tag-based production release)

**Pipeline stages:** security-scan || backend || frontend → docker-build → deploy-staging → E2E on staging → deploy-production

**Acceptance criteria:**
- [ ] Multi-platform Docker builds (linux/amd64, linux/arm64) with SBOM and provenance attestation
- [ ] Image pushed to GHCR with `{branch}-{sha8}` tag format
- [ ] Snyk container scan runs on every built image, blocks on HIGH+ CVE
- [ ] `concurrency: cancel-in-progress: true` prevents redundant pipeline runs
- [ ] Slack notification sent on successful production deployment

---

## 9.4 Configure backup strategy, disaster recovery runbook, and nightly PostgreSQL backup

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:m`, `agent:devops`
**Status:** Pending

Implement backup automation and DR procedures per 06-INFRASTRUCTURE-DESIGN.md section 7.

**Key files:**
- `helm/eventflow-infra/templates/postgres-backup-cronjob.yaml` (nightly 2AM UTC S3 backup)
- `docs/RUNBOOK.md` (DR procedures, point-in-time recovery steps, estimated RTO 30min)
- `scripts/smoke-test.sh` (post-DR verification script)
- `infra/prometheus/alerting-rules.yml` (disk usage > 80%, backup job failure alerts)

**RPO:** 1 hour (continuous WAL), **RTO:** 30 minutes

**Acceptance criteria:**
- [ ] Nightly backup CronJob runs pg_dump | gzip | aws s3 cp with SSE-KMS encryption
- [ ] Backup integrity verified via SHA-256 checksum logged after upload
- [ ] RUNBOOK.md documents step-by-step PITR recovery with estimated 25-30min total time
- [ ] Prometheus alert fires if backup CronJob fails for 2 consecutive days
- [ ] Backup retention: 30 days in S3 (lifecycle policy configured)

---

## 9.5 Implement VenueSpace entity with capacity, amenities, photos, and availability calendar

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Venue managers can list individual spaces within a venue with full details and availability.

## Acceptance Criteria
- `VenueSpace` entity: `venueId`, `name`, `capacity`, `amenities[]`, `photoUrls[]`, `setupStyles[]`, `isAvailable`
- CRUD endpoints under `GET|POST /api/v1/venues/{venueId}/spaces`
- Availability calendar: `GET /api/v1/venues/{venueId}/spaces/{spaceId}/availability?from=&to=`
- Cross-checks against events already using the space
- EF Core migration and configuration added

## Technical Notes
- `SetupStyle` enum: `Theatre`, `Classroom`, `Banquet`, `Reception`, `Cabaret`, `Boardroom`
- Extends existing `VenueRoom` concept or replaces it with richer `VenueSpace`
- Availability computed from `Session.VenueRoomId` assignments

---

## 9.6 Build venue spaces management UI with floor plan, setup styles, and availability calendar

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:l`, `agent:frontend`
**Status:** Pending

## Goal
Venue manager frontend for listing and managing venue spaces with rich detail and availability view.

## Acceptance Criteria
- Venue detail page includes **Spaces** tab
- Space card shows capacity, amenities chips, setup style badges, photo carousel
- Availability calendar view (month view) showing booked vs. available dates
- Add/edit space drawer with photo upload and amenity checkboxes
- Inquiry flow: organizer can send inquiry from space card

## Technical Notes
- Calendar component: `react-big-calendar` or custom TailwindCSS grid
- Photo upload via `POST /api/v1/files/upload-url` presigned URL pattern
- Integrates with `VenueSpace` API endpoints

---

## 9.7 Implement SponsorLead capture and sponsor portal lead export

**Labels:** `type:feature`, `layer:fullstack`, `priority:medium`, `size:m`, `agent:fullstack`
**Status:** Pending

## Goal
Sponsors can capture attendee leads via their portal and export them post-event.

## Acceptance Criteria
- `SponsorLead` entity wired to `EventSponsor` and `Registration`
- `POST /api/v1/events/{eventId}/sponsors/{eventSponsorId}/leads` creates a lead
- `GET /api/v1/sponsor-portal/{portalToken}/leads` returns leads for the sponsor
- Lead export as CSV downloadable from sponsor portal
- Lead score (`Hot`/`Warm`/`Cold`) editable by sponsor

## Technical Notes
- Sponsor portal token validated via `EventSponsor.PortalToken` (secure random, 48h expiry)
- CSV export via same async S3 pattern as attendee export
- Rate limit: 60/min on portal endpoints

---

## 9.8 Implement venue inquiries management endpoints for venue managers

**Labels:** `type:feature`, `layer:backend`, `priority:low`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Venue managers can view and respond to booking inquiries submitted by event organizers.

## Acceptance Criteria
- `VenueInquiry` entity: `id`, `venueId`, `tenantId`, `contactName`, `contactEmail`, `eventDate`, `attendeeCount`, `message`, `status` (New/Responded/Closed), `createdAt`
- `GET /api/v1/venues/{venueId}/inquiries` lists inquiries for a venue (RequireManager)
- `PUT /api/v1/venues/{venueId}/inquiries/{inquiryId}/respond` updates status and sends response email
- Inquiry submitted via venue detail page contact form (public or authenticated)
- EF Core migration for `venue_inquiries` table

## Technical Notes
- Response email sent via `IEmailService` with venue manager's reply text
- Inquiry creation triggered from venue discovery UI `InquiryForm` component
- Status transitions: `New` → `Responded` → `Closed`

---

