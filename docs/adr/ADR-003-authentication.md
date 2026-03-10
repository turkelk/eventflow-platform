# ADR-003: Authentication Architecture — Keycloak OIDC

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Product, Security, Engineering Leads  

---

## Context

EventFlow is a multi-tenant SaaS platform with the following authentication requirements:

- **Multiple user roles per tenant**: Admin, EventManager, Coordinator, Viewer, CheckInStaff.
- **SSO integration**: Enterprise mid-market customers require SAML 2.0 or OIDC SSO with their IdP (Okta, Azure AD, Google Workspace). This is a hard requirement from Persona 4 (James — IT Administrator) and is a competitive gap vs. Eventbrite.
- **SCIM provisioning**: Automated user lifecycle management (creation, deactivation) from enterprise IdPs.
- **Multi-tenant isolation**: Users belong to a specific tenant (organization). Cross-tenant access must be impossible.
- **Social login**: Sign in with Google, Sign in with Microsoft for self-service users.
- **Magic links**: Speaker invitation flow requires passwordless magic link authentication.
- **Secure token handling**: Tokens must not be stored in localStorage (XSS risk). Memory storage with refresh token rotation.
- **Short-lived access tokens**: 5-minute access tokens to minimize breach window. 30-minute refresh tokens with rotation.

**Decision Driver — Keycloak vs. Alternatives**:

| Option | Verdict | Reason |
|---|---|---|
| **Keycloak** | ✅ Chosen | Self-hosted, full OIDC/SAML/SCIM support, social login, multi-realm, active open source community, no per-MAU pricing |
| Auth0 | Rejected | Per-MAU pricing becomes prohibitive at scale; vendor lock-in; less flexible for enterprise SSO customization |
| Okta | Rejected | Enterprise pricing; overkill for initial deployment; vendor lock-in |
| AWS Cognito | Rejected | Limited SAML customization; Cognito user pools don't support custom OIDC flows well; AWS-specific lock-in |
| Custom JWT | Rejected | Security liability; reinventing the wheel; no SAML/SCIM support without enormous effort |

Keycloak provides everything required at zero licensing cost and runs in our existing Kubernetes infrastructure.

---

## Decision

**Keycloak 24+ as the identity provider. OIDC Authorization Code Flow with PKCE for the React SPA. JWT validation on every API request (stateless backend). Multi-tenant isolation via `org_id` JWT claim.**

---

## Keycloak Architecture

### Realm Structure

```
Keycloak Master Realm (admin only — never for application users)
    └── eventflow Realm
            ├── Client: eventflow-spa (public, PKCE, Authorization Code Flow)
            ├── Client: eventflow-api (confidential, client credentials for service-to-service)
            ├── Client: eventflow-admin (confidential, for tenant provisioning API calls)
            ├── Identity Providers:
            │   ├── Google (social login)
            │   ├── Microsoft (social login / Azure AD enterprise)
            │   └── SAML 2.0 (enterprise SSO — configured per tenant)
            ├── User Federation:
            │   └── SCIM 2.0 endpoint (via Keycloak SCIM extension)
            └── Mappers:
                ├── org_id → JWT claim (from user attribute)
                ├── roles → JWT claim (realm + client roles)
                └── tenant_name → JWT claim
```

### Token Configuration

```
Access Token Lifespan:        5 minutes
Refresh Token Lifespan:       30 minutes
Refresh Token Rotation:       Enabled (every refresh issues a new refresh token)
Refresh Token Reuse Policy:   Detect reuse — revoke entire refresh token chain on reuse detection
SSO Session Idle:             30 minutes
SSO Session Max:              10 hours
Offline Session Idle:         30 days (for "Remember me")
```

**Why 5-minute access tokens?** Short-lived tokens limit the blast radius of a token leak. The React frontend uses silent refresh (iframe or background fetch) to obtain new access tokens before expiry, so users never experience a disruption.

### JWT Claims Structure

```json
{
  "sub": "usr_01HXYZ...",
  "iss": "https://auth.eventflow.com/realms/eventflow",
  "aud": "eventflow-api",
  "exp": 1704067800,
  "iat": 1704067500,
  "jti": "unique-token-id",
  "org_id": "ten_01HXYZ...",
  "org_name": "Acme Corporation",
  "email": "maya@acme.com",
  "email_verified": true,
  "name": "Maya Chen",
  "given_name": "Maya",
  "family_name": "Chen",
  "roles": ["event_manager"],
  "realm_access": {
    "roles": ["event_manager", "offline_access"]
  },
  "resource_access": {
    "eventflow-api": {
      "roles": ["event_manager"]
    }
  }
}
```

---

## Frontend (React SPA) Authentication

### Token Storage Strategy: In-Memory Only

```typescript
// DECISION: Tokens stored in memory (module-level variable), NOT localStorage, NOT cookies
// Rationale:
// - localStorage: vulnerable to XSS — any injected script can read tokens
// - HttpOnly cookies: viable for SSR apps; complex for SPA + API CORS setup; CSRF risk
// - In-memory: XSS cannot access memory variables; lost on page refresh → mitigated by silent refresh

// src/lib/auth/tokenStore.ts
let accessToken: string | null = null;
let refreshToken: string | null = null;

export const tokenStore = {
  getAccessToken: () => accessToken,
  getRefreshToken: () => refreshToken,
  setTokens: (access: string, refresh: string) => {
    accessToken = access;
    refreshToken = refresh;
  },
  clearTokens: () => {
    accessToken = null;
    refreshToken = null;
  },
};
```

### OIDC Authorization Code Flow with PKCE

```typescript
// src/lib/auth/authConfig.ts
import Keycloak from 'keycloak-js';

export const keycloak = new Keycloak({
  url: import.meta.env.VITE_KEYCLOAK_URL,
  realm: 'eventflow',
  clientId: 'eventflow-spa',
});

// PKCE is enabled by default in keycloak-js v24+
// Flow: browser → Keycloak login page → redirect back with auth code
//       → keycloak-js exchanges code for tokens (PKCE verifier sent)
//       → tokens stored in memory via tokenStore

// src/lib/auth/AuthProvider.tsx
export function AuthProvider({ children }: { children: ReactNode }) {
  const [initialized, setInitialized] = useState(false);

  useEffect(() => {
    keycloak
      .init({
        onLoad: 'check-sso',           // Silent SSO check on page load
        silentCheckSsoRedirectUri:      // Iframe-based silent token refresh
          window.location.origin + '/silent-check-sso.html',
        pkceMethod: 'S256',            // PKCE with SHA-256
        enableLogging: import.meta.env.DEV,
      })
      .then((authenticated) => {
        if (authenticated) {
          tokenStore.setTokens(
            keycloak.token!,
            keycloak.refreshToken!
          );
        }
        setInitialized(true);
      });

    // Auto-refresh access token 60 seconds before expiry
    keycloak.onTokenExpired = () => {
      keycloak
        .updateToken(60)
        .then((refreshed) => {
          if (refreshed) {
            tokenStore.setTokens(
              keycloak.token!,
              keycloak.refreshToken!
            );
          }
        })
        .catch(() => {
          // Refresh token expired → force re-login
          tokenStore.clearTokens();
          keycloak.login();
        });
    };
  }, []);

  if (!initialized) return <SplashScreen />;
  return <AuthContext.Provider value={keycloak}>{children}</AuthContext.Provider>;
}
```

### Axios Interceptor — Attach Bearer Token

```typescript
// src/lib/api/httpClient.ts
import axios from 'axios';
import { tokenStore } from '../auth/tokenStore';
import { keycloak } from '../auth/authConfig';

export const httpClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
});

httpClient.interceptors.request.use(async (config) => {
  // Ensure token is fresh (refresh if < 60s remaining)
  await keycloak.updateToken(60);
  const token = tokenStore.getAccessToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

httpClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token invalid or expired — force re-login
      tokenStore.clearTokens();
      keycloak.login();
    }
    return Promise.reject(error);
  }
);
```

---

## Backend (.NET 9) JWT Validation

### JWT Bearer Authentication Configuration

```csharp
// Api/Extensions/ServiceCollectionExtensions.cs
public static IServiceCollection AddKeycloakAuthentication(
    this IServiceCollection services,
    IConfiguration configuration)
{
    services
        .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.Authority = configuration["Keycloak:Authority"];
            // e.g., https://auth.eventflow.com/realms/eventflow

            options.Audience = "eventflow-api";

            options.RequireHttpsMetadata = true;

            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ClockSkew = TimeSpan.FromSeconds(30),
                // Short clock skew — access tokens are 5 min, we don't want 5-min skew tolerance
            };

            // Map Keycloak roles claim to ASP.NET Core roles
            options.MapInboundClaims = false;
            options.TokenValidationParameters.NameClaimType = "name";
            options.TokenValidationParameters.RoleClaimType = "roles";
        });

    return services;
}
```

### Tenant Resolution Middleware

```csharp
// Api/Middleware/TenantResolutionMiddleware.cs
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;

    public TenantResolutionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context, ITenantContext tenantContext)
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            // Extract org_id from JWT claim
            var orgIdClaim = context.User.FindFirst("org_id");
            if (orgIdClaim == null || !Guid.TryParse(orgIdClaim.Value, out var tenantId))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Missing or invalid org_id claim.");
                return;
            }

            // Populate TenantContext — available to all MediatR handlers
            tenantContext.TenantId = tenantId;
            tenantContext.UserId = Guid.Parse(context.User.FindFirst("sub")!.Value);
            tenantContext.UserEmail = context.User.FindFirst("email")?.Value;
            tenantContext.Roles = context.User.FindAll("roles").Select(c => c.Value).ToList();

            // Set PostgreSQL RLS context variable for this connection
            // Done via EF Core interceptor — see ADR-006
        }

        await _next(context);
    }
}
```

### Role-Based Authorization

```csharp
// Domain roles mapped to ASP.NET Core policy names
// Api/Extensions/AuthorizationExtensions.cs
public static IServiceCollection AddEventFlowAuthorization(
    this IServiceCollection services)
{
    services.AddAuthorization(options =>
    {
        options.AddPolicy("RequireAdmin",
            policy => policy.RequireRole("admin"));

        options.AddPolicy("RequireEventManager",
            policy => policy.RequireRole("admin", "event_manager"));

        options.AddPolicy("RequireCoordinator",
            policy => policy.RequireRole("admin", "event_manager", "coordinator"));

        options.AddPolicy("RequireAuthenticated",
            policy => policy.RequireAuthenticatedUser());
    });

    return services;
}

// Usage in controller (only role-gating — business logic stays in handler)
[HttpDelete("{id:guid}")]
[Authorize(Policy = "RequireEventManager")]
public async Task<IActionResult> DeleteEvent(Guid id, CancellationToken ct)
{
    await _mediator.Send(new DeleteEventCommand(id), ct);
    return NoContent();
}
```

### Resource-Based Authorization in Handlers

```csharp
// Application/Features/Events/Commands/UpdateEvent/UpdateEventHandler.cs
// Handler checks tenant ownership BEFORE operating on data
public class UpdateEventHandler : IRequestHandler<UpdateEventCommand, EventDto>
{
    private readonly EventFlowDbContext _db;
    private readonly ITenantContext _tenantContext;

    public async Task<EventDto> Handle(
        UpdateEventCommand command,
        CancellationToken ct)
    {
        // EF Core global filter ensures this query is already tenant-scoped
        // But we explicitly verify ownership for defense in depth
        var @event = await _db.Events
            .FirstOrDefaultAsync(e => e.Id == command.EventId, ct)
            ?? throw new NotFoundException($"Event {command.EventId} not found.");

        // Explicit ownership check — belt AND suspenders
        if (@event.TenantId != _tenantContext.TenantId)
            throw new ForbiddenException("Access denied to this event.");

        // Business logic here...
        @event.Update(command.Name, command.Description, command.StartDateUtc);

        await _db.SaveChangesAsync(ct);
        return @event.ToDto();
    }
}
```

---

## Multi-Tenant SSO Configuration

Enterprise tenants can configure their own SAML 2.0 or OIDC identity provider via Keycloak's Identity Provider federation. Each tenant's SSO configuration is stored in Keycloak and activated for their organization.

```csharp
// Infrastructure/Adapters/KeycloakAdminService.cs
// Called by: ProvisionTenantHandler when SSO is enabled for a tenant
public async Task ConfigureTenantSsoAsync(
    Guid tenantId,
    SsoConfiguration ssoConfig,
    CancellationToken ct)
{
    // Single external call: Keycloak Admin REST API
    // Creates an Identity Provider federation for this tenant
    var response = await _keycloakAdminHttpClient.PostAsJsonAsync(
        $"/admin/realms/eventflow/identity-provider/instances",
        new KeycloakIdentityProviderRequest
        {
            Alias = $"tenant-{tenantId}",
            ProviderId = ssoConfig.Protocol, // "saml" or "oidc"
            Config = MapSsoConfig(ssoConfig),
        },
        ct);

    response.EnsureSuccessStatusCode();
}
```

---

## Magic Link Authentication (Speaker Invitations)

Speaker invitation uses a custom magic link flow:

```
1. InviteSpeakerHandler generates a secure random token (32 bytes, base64url encoded)
2. Token stored in speakers table with expiry (48 hours)
3. Email sent to speaker with link: https://app.eventflow.com/speaker-invite/{token}
4. Speaker clicks link → frontend calls POST /api/speakers/accept-invite/{token}
5. AcceptSpeakerInviteHandler validates token, creates Keycloak user if not exists,
   returns a short-lived OIDC login hint URL that auto-authenticates the speaker
6. Token marked as used (single-use)
```

---

## SCIM 2.0 Provisioning

For enterprise tenants requiring automated user lifecycle management (Persona 4 — James):

- Keycloak SCIM extension exposes `/scim/v2` endpoint
- Enterprise IdPs (Okta, Azure AD) can automatically provision and deprovision users
- User creation in Keycloak automatically triggers `UserProvisionedEvent` via domain event
- EventFlow assigns the appropriate default role based on the IdP group mapping configured by the tenant admin

---

## Security Considerations

| Threat | Mitigation |
|---|---|  
| XSS token theft | Tokens in memory only — not accessible to injected scripts |
| CSRF | SPA uses Authorization header (Bearer), not cookies — CSRF not applicable |
| Token replay | Short 5-min access token lifespan limits replay window |
| Refresh token theft | Rotation enabled + reuse detection revokes entire chain |
| Cross-tenant data access | org_id in JWT + EF Core global filter + PostgreSQL RLS (3 layers) |
| Brute force login | Keycloak built-in brute force detection (exponential backoff + account lockout) |
| Session fixation | PKCE code challenge prevents authorization code interception |
| JWT algorithm confusion | Backend specifies RS256 explicitly — rejects HS256 |

---

## Consequences

### Positive
- **Enterprise SSO**: SAML/OIDC federation meets James persona's hard requirement.
- **SCIM provisioning**: Eliminates manual user management for enterprise customers.
- **No per-MAU cost**: Keycloak is self-hosted; cost scales with infrastructure, not user count.
- **Social login**: Google/Microsoft sign-in out of the box — reduces onboarding friction for self-service users.
- **Multi-realm support**: Future: separate realm per major enterprise customer if needed.

### Negative
- **Operational overhead**: Keycloak requires maintenance, upgrades, and HA configuration for production.
  Mitigation: Use Keycloak Operator for Kubernetes deployment; configure active-passive HA with PostgreSQL backend.
- **Keycloak cold start**: Keycloak JVM startup is 30–60 seconds. Mitigation: Use Keycloak Quarkus distribution (fast startup), K8s readiness probe.
- **Custom branding**: Keycloak login page customization requires theme development.
  Mitigation: Custom Keycloak theme with EventFlow branding in the white-label scope.

---

## References
- [Keycloak 24 documentation](https://www.keycloak.org/documentation)
- [PKCE RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [keycloak-js library](https://www.npmjs.com/package/keycloak-js)
- [OWASP Token Storage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage)
