# Data Model

> 23 entities defined. All tables are tenant-scoped via `organizationId` with Row-Level Security.

## Entity Definitions

### Organization

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| name VARCHAR(255) | `string` |  |
| slug VARCHAR(100) UNIQUE | `string` |  |
| plan_id UUID FK | `string` |  |
| stripe_customer_id VARCHAR(255) | `string` |  |
| stripe_subscription_id VARCHAR(255) | `string` |  |
| keycloak_realm_id VARCHAR(255) | `string` |  |
| custom_domain VARCHAR(255) | `string` |  |
| email_sender_name VARCHAR(255) | `string` |  |
| email_sender_address VARCHAR(255) | `string` |  |
| logo_url TEXT | `string` |  |
| primary_color VARCHAR(7) | `string` |  |
| timezone VARCHAR(100) | `string` |  |
| is_active BOOLEAN DEFAULT TRUE | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |
| updated_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- has many Events
- has many OrganizationMembers
- has many EmailTemplates
- has one SubscriptionPlan (via plan_id)

### OrganizationMember

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| user_id VARCHAR(255) (Keycloak subject) | `string` |  |
| email VARCHAR(255) | `string` |  |
| full_name VARCHAR(255) | `string` |  |
| role ENUM(admin, organizer, check_in_staff, viewer, billing_admin) | `string` |  |
| invited_at TIMESTAMPTZ | `string` |  |
| accepted_at TIMESTAMPTZ | `string` |  |
| is_active BOOLEAN | `string` |  |

**Relationships:**
- belongs to Organization
- created by User (Keycloak)

### SubscriptionPlan

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| name VARCHAR(100) (Starter, Growth, Enterprise) | `string` |  |
| stripe_price_id VARCHAR(255) | `string` |  |
| monthly_price_cents INTEGER | `string` |  |
| max_events_per_month INTEGER | `string` |  |
| max_attendees_per_event INTEGER | `string` |  |
| ticket_fee_percentage DECIMAL(5,4) | `string` |  |
| features JSONB | `string` |  |
| is_active BOOLEAN | `string` |  |

**Relationships:**
- has many Organizations

### Event

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| title VARCHAR(500) | `string` |  |
| slug VARCHAR(255) | `string` |  |
| description TEXT | `string` |  |
| event_type ENUM(in_person, virtual, hybrid) | `string` |  |
| status ENUM(draft, published, cancelled, completed) | `string` |  |
| start_at TIMESTAMPTZ | `string` |  |
| end_at TIMESTAMPTZ | `string` |  |
| timezone VARCHAR(100) | `string` |  |
| cover_image_url TEXT | `string` |  |
| venue_id UUID FK NULLABLE | `string` |  |
| virtual_meeting_url TEXT | `string` |  |
| max_capacity INTEGER | `string` |  |
| registration_opens_at TIMESTAMPTZ | `string` |  |
| registration_closes_at TIMESTAMPTZ | `string` |  |
| waitlist_enabled BOOLEAN DEFAULT FALSE | `string` |  |
| custom_registration_schema JSONB | `string` |  |
| is_private BOOLEAN DEFAULT FALSE | `string` |  |
| private_access_code VARCHAR(50) | `string` |  |
| cloned_from_event_id UUID NULLABLE | `string` |  |
| created_by_user_id VARCHAR(255) | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |
| updated_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Organization
- has many TicketTiers
- has many Sessions
- has many Registrations
- has many Speakers
- has many Sponsors
- has many EmailCampaigns
- belongs to Venue (optional)

### Venue

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK NULLABLE (null = platform venue listing) | `string` |  |
| name VARCHAR(500) | `string` |  |
| address_line1 VARCHAR(255) | `string` |  |
| address_line2 VARCHAR(255) | `string` |  |
| city VARCHAR(100) | `string` |  |
| state VARCHAR(100) | `string` |  |
| country VARCHAR(100) | `string` |  |
| postal_code VARCHAR(20) | `string` |  |
| latitude DECIMAL(10,8) | `string` |  |
| longitude DECIMAL(11,8) | `string` |  |
| capacity INTEGER | `string` |  |
| amenities JSONB | `string` |  |
| photos JSONB | `string` |  |
| contact_email VARCHAR(255) | `string` |  |
| contact_phone VARCHAR(50) | `string` |  |
| venue_manager_user_id VARCHAR(255) | `string` |  |
| is_verified BOOLEAN DEFAULT FALSE | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- has many Events
- has many VenueSpaces

### VenueSpace

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| venue_id UUID FK | `string` |  |
| name VARCHAR(255) | `string` |  |
| capacity INTEGER | `string` |  |
| setup_styles JSONB | `string` |  |
| amenities JSONB | `string` |  |
| floor_plan_url TEXT | `string` |  |

**Relationships:**
- belongs to Venue
- has many Sessions (as room)

### TicketTier

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| name VARCHAR(255) | `string` |  |
| description TEXT | `string` |  |
| price_cents INTEGER | `string` |  |
| currency VARCHAR(3) DEFAULT 'USD' | `string` |  |
| quantity_total INTEGER | `string` |  |
| quantity_sold INTEGER DEFAULT 0 | `string` |  |
| quantity_reserved INTEGER DEFAULT 0 | `string` |  |
| sale_starts_at TIMESTAMPTZ | `string` |  |
| sale_ends_at TIMESTAMPTZ | `string` |  |
| tier_type ENUM(general, vip, early_bird, group, free) | `string` |  |
| max_per_order INTEGER DEFAULT 10 | `string` |  |
| is_hidden BOOLEAN DEFAULT FALSE | `string` |  |
| stripe_price_id VARCHAR(255) | `string` |  |
| sort_order INTEGER | `string` |  |

**Relationships:**
- belongs to Event
- has many RegistrationTickets

### DiscountCode

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| code VARCHAR(50) | `string` |  |
| discount_type ENUM(percentage, fixed_amount) | `string` |  |
| discount_value DECIMAL(10,2) | `string` |  |
| max_uses INTEGER NULLABLE | `string` |  |
| uses_count INTEGER DEFAULT 0 | `string` |  |
| valid_from TIMESTAMPTZ | `string` |  |
| valid_until TIMESTAMPTZ | `string` |  |
| applicable_tier_ids JSONB | `string` |  |
| is_active BOOLEAN DEFAULT TRUE | `string` |  |

**Relationships:**
- belongs to Event
- has many Registrations (applied_discount_code_id)

### Registration

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| attendee_id UUID FK | `string` |  |
| status ENUM(pending, confirmed, cancelled, waitlisted, checked_in) | `string` |  |
| registration_number VARCHAR(50) UNIQUE | `string` |  |
| qr_code_token VARCHAR(255) UNIQUE | `string` |  |
| total_amount_cents INTEGER | `string` |  |
| discount_code_id UUID FK NULLABLE | `string` |  |
| discount_amount_cents INTEGER DEFAULT 0 | `string` |  |
| stripe_payment_intent_id VARCHAR(255) | `string` |  |
| stripe_checkout_session_id VARCHAR(255) | `string` |  |
| custom_field_responses JSONB | `string` |  |
| checked_in_at TIMESTAMPTZ | `string` |  |
| checked_in_by_user_id VARCHAR(255) | `string` |  |
| waitlist_position INTEGER NULLABLE | `string` |  |
| refunded_at TIMESTAMPTZ | `string` |  |
| refund_amount_cents INTEGER | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |
| updated_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Event
- belongs to Attendee
- has many RegistrationTickets
- has many SessionRegistrations

### RegistrationTicket

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| registration_id UUID FK | `string` |  |
| ticket_tier_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| quantity INTEGER | `string` |  |
| unit_price_cents INTEGER | `string` |  |
| subtotal_cents INTEGER | `string` |  |

**Relationships:**
- belongs to Registration
- belongs to TicketTier

### Attendee

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| user_id VARCHAR(255) NULLABLE (Keycloak subject, null for guest checkout) | `string` |  |
| email VARCHAR(255) | `string` |  |
| first_name VARCHAR(255) | `string` |  |
| last_name VARCHAR(255) | `string` |  |
| company VARCHAR(255) | `string` |  |
| job_title VARCHAR(255) | `string` |  |
| phone VARCHAR(50) | `string` |  |
| bio TEXT | `string` |  |
| avatar_url TEXT | `string` |  |
| linkedin_url TEXT | `string` |  |
| interests JSONB | `string` |  |
| networking_opt_in BOOLEAN DEFAULT FALSE | `string` |  |
| sponsor_contact_opt_in BOOLEAN DEFAULT FALSE | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- has many Registrations
- has many SessionRegistrations
- has many AttendeeMatches (as source or target)

### Session

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| title VARCHAR(500) | `string` |  |
| description TEXT | `string` |  |
| track VARCHAR(255) | `string` |  |
| session_type ENUM(keynote, breakout, workshop, panel, networking, break) | `string` |  |
| starts_at TIMESTAMPTZ | `string` |  |
| ends_at TIMESTAMPTZ | `string` |  |
| venue_space_id UUID FK NULLABLE | `string` |  |
| virtual_room_url TEXT | `string` |  |
| max_capacity INTEGER NULLABLE | `string` |  |
| is_featured BOOLEAN DEFAULT FALSE | `string` |  |
| sort_order INTEGER | `string` |  |
| recording_url TEXT | `string` |  |
| slide_deck_url TEXT | `string` |  |

**Relationships:**
- belongs to Event
- has many SessionSpeakers
- has many SessionRegistrations
- belongs to VenueSpace (optional)

### Speaker

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| first_name VARCHAR(255) | `string` |  |
| last_name VARCHAR(255) | `string` |  |
| email VARCHAR(255) | `string` |  |
| bio TEXT | `string` |  |
| headshot_url TEXT | `string` |  |
| company VARCHAR(255) | `string` |  |
| job_title VARCHAR(255) | `string` |  |
| linkedin_url TEXT | `string` |  |
| twitter_handle VARCHAR(100) | `string` |  |
| invitation_token VARCHAR(255) | `string` |  |
| invitation_accepted_at TIMESTAMPTZ | `string` |  |
| status ENUM(invited, confirmed, declined) | `string` |  |

**Relationships:**
- belongs to Event
- has many SessionSpeakers

### SessionSpeaker

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| session_id UUID FK | `string` |  |
| speaker_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| is_moderator BOOLEAN DEFAULT FALSE | `string` |  |

**Relationships:**
- belongs to Session
- belongs to Speaker

### SessionRegistration

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| session_id UUID FK | `string` |  |
| registration_id UUID FK | `string` |  |
| attendee_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| registered_at TIMESTAMPTZ | `string` |  |
| feedback_rating INTEGER NULLABLE | `string` |  |
| feedback_comment TEXT NULLABLE | `string` |  |

**Relationships:**
- belongs to Session
- belongs to Registration
- belongs to Attendee

### Sponsor

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| company_name VARCHAR(255) | `string` |  |
| logo_url TEXT | `string` |  |
| website_url TEXT | `string` |  |
| tier ENUM(platinum, gold, silver, bronze, media_partner) | `string` |  |
| description TEXT | `string` |  |
| contact_email VARCHAR(255) | `string` |  |
| lead_capture_enabled BOOLEAN DEFAULT FALSE | `string` |  |
| leads_count INTEGER DEFAULT 0 | `string` |  |
| portal_access_token VARCHAR(255) | `string` |  |
| stripe_payment_intent_id VARCHAR(255) | `string` |  |
| package_price_cents INTEGER | `string` |  |
| sort_order INTEGER | `string` |  |

**Relationships:**
- belongs to Event
- has many SponsorLeads

### SponsorLead

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| sponsor_id UUID FK | `string` |  |
| attendee_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| opted_in_at TIMESTAMPTZ | `string` |  |
| notes TEXT | `string` |  |

**Relationships:**
- belongs to Sponsor
- belongs to Attendee

### EmailCampaign

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| name VARCHAR(255) | `string` |  |
| subject VARCHAR(500) | `string` |  |
| body_html TEXT | `string` |  |
| campaign_type ENUM(announcement, reminder, confirmation, post_event, custom) | `string` |  |
| status ENUM(draft, scheduled, sending, sent, cancelled) | `string` |  |
| recipient_filter JSONB | `string` |  |
| scheduled_at TIMESTAMPTZ | `string` |  |
| sent_at TIMESTAMPTZ | `string` |  |
| sent_count INTEGER DEFAULT 0 | `string` |  |
| open_count INTEGER DEFAULT 0 | `string` |  |
| click_count INTEGER DEFAULT 0 | `string` |  |
| created_by_user_id VARCHAR(255) | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Event
- has many EmailDeliveries

### EmailDelivery

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| campaign_id UUID FK | `string` |  |
| attendee_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| status ENUM(queued, sent, delivered, opened, clicked, bounced, unsubscribed) | `string` |  |
| sent_at TIMESTAMPTZ | `string` |  |
| opened_at TIMESTAMPTZ | `string` |  |
| provider_message_id VARCHAR(255) | `string` |  |

**Relationships:**
- belongs to EmailCampaign
- belongs to Attendee

### AttendeeMatch

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| event_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| attendee_a_id UUID FK | `string` |  |
| attendee_b_id UUID FK | `string` |  |
| match_score DECIMAL(5,4) | `string` |  |
| match_reasons JSONB | `string` |  |
| status ENUM(suggested, accepted_by_a, accepted_by_b, mutual, dismissed) | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Event
- belongs to Attendee (a)
- belongs to Attendee (b)

### VenueInquiry

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| venue_id UUID FK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| event_id UUID FK NULLABLE | `string` |  |
| requested_by_user_id VARCHAR(255) | `string` |  |
| event_date TIMESTAMPTZ | `string` |  |
| expected_attendees INTEGER | `string` |  |
| message TEXT | `string` |  |
| status ENUM(pending, responded, closed) | `string` |  |
| responded_at TIMESTAMPTZ | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Venue
- belongs to Organization

### WebhookEndpoint

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| url TEXT | `string` |  |
| secret VARCHAR(255) | `string` |  |
| events JSONB | `string` |  |
| is_active BOOLEAN DEFAULT TRUE | `string` |  |
| last_triggered_at TIMESTAMPTZ | `string` |  |
| failure_count INTEGER DEFAULT 0 | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Organization
- has many WebhookDeliveries

### AuditLog

| Field | Type | Constraints |
|-------|------|-------------|
| id UUID PK | `string` |  |
| org_id UUID FK (RLS key) | `string` |  |
| user_id VARCHAR(255) | `string` |  |
| action VARCHAR(255) | `string` |  |
| entity_type VARCHAR(100) | `string` |  |
| entity_id UUID | `string` |  |
| changes JSONB | `string` |  |
| ip_address VARCHAR(45) | `string` |  |
| user_agent TEXT | `string` |  |
| created_at TIMESTAMPTZ | `string` |  |

**Relationships:**
- belongs to Organization

