# Product Requirements

## Problem Statement

Enterprise event management platforms like Cvent and Eventbrite are either prohibitively expensive for mid-market companies, overly complex for smaller teams, or lack the AI-driven personalization and real-time collaboration features modern event organizers demand. Mid-market businesses (50–500 employees) running 10–200 events per year are underserved: too large for consumer tools, too cost-sensitive for enterprise suites, and frustrated by disconnected toolchains for registration, venue sourcing, attendee engagement, and post-event analytics.

## Solution

A multi-tenant SaaS event management platform purpose-built for mid-market organizations. It unifies event creation, branded registration pages, attendee management, venue discovery, real-time check-in, session scheduling, sponsor management, email/SMS communications, and post-event analytics into a single cohesive product. AI-assisted agenda building, smart attendee matchmaking, and a white-label attendee portal differentiate it from legacy tools. Stripe-powered ticketing and subscription billing, Keycloak SSO, and a mobile-first check-in experience complete the offering.

## User Stories

### Event Organizer

- As a **Event Organizer**, I want to Create a new event with name, dates, location, capacity, and event type (in-person, virtual, hybrid), so that Quickly scaffold a complete event without navigating complex wizards
- As a **Event Organizer**, I want to Build a branded registration page with custom fields, logo, colors, and domain, so that Present a professional face to attendees without needing a developer
- As a **Event Organizer**, I want to Define multiple ticket tiers (free, paid, VIP, early-bird) with capacity limits and sale windows, so that Maximize revenue and control attendee mix
- As a **Event Organizer**, I want to Configure discount codes and group pricing rules, so that Drive registrations through promotional campaigns
- As a **Event Organizer**, I want to Build a session schedule with tracks, speakers, and room assignments, so that Give attendees a clear agenda and manage venue space efficiently
- As a **Event Organizer**, I want to Send targeted email and SMS campaigns to attendee segments, so that Increase attendance rates and keep attendees informed
- As a **Event Organizer**, I want to View real-time dashboard of registrations, revenue, and check-ins, so that Make data-driven decisions before and during the event
- As a **Event Organizer**, I want to Export attendee lists and analytics reports to CSV/PDF, so that Share results with stakeholders and integrate with CRM
- As a **Event Organizer**, I want to Manage a waitlist and automatically promote waitlisted attendees when capacity opens, so that Fill events to capacity without manual intervention
- As a **Event Organizer**, I want to Clone a past event as a template for recurring events, so that Save hours of setup time for repeat events

### Attendee

- As a **Attendee**, I want to Register for an event and purchase tickets via a branded checkout, so that Complete registration quickly with a trustworthy, familiar experience
- As a **Attendee**, I want to Receive a confirmation email with QR code and calendar invite, so that Have everything needed to attend in one place
- As a **Attendee**, I want to Build a personal agenda by selecting sessions from the event schedule, so that Plan the event experience in advance and avoid conflicts
- As a **Attendee**, I want to Update or cancel registration and receive an automatic refund per event policy, so that Manage plans without contacting support
- As a **Attendee**, I want to Access a personalized attendee portal with schedule, networking matches, and event info, so that Stay engaged and get maximum value from attendance
- As a **Attendee**, I want to Connect with other attendees via AI-powered matchmaking based on profile and interests, so that Make meaningful professional connections at the event
- As a **Attendee**, I want to Submit session feedback and event ratings post-event, so that Contribute to event improvement and feel heard

### Check-in Staff

- As a **Check-in Staff**, I want to Scan attendee QR codes via mobile browser or tablet app, so that Process check-ins in under 2 seconds without paper lists
- As a **Check-in Staff**, I want to Search attendees by name and manually check them in, so that Handle attendees who forgot their QR code without friction
- As a **Check-in Staff**, I want to View live check-in counts and flag VIP or special-needs attendees, so that Provide differentiated service at the door

### Speaker

- As a **Speaker**, I want to Accept a speaker invitation and submit bio, headshot, and session details via a self-service portal, so that Contribute event content without back-and-forth emails
- As a **Speaker**, I want to View assigned sessions, room, and time slot, so that Prepare effectively and arrive at the right place

### Sponsor

- As a **Sponsor**, I want to Purchase a sponsorship package and upload brand assets via a self-service portal, so that Complete sponsorship onboarding without manual coordination
- As a **Sponsor**, I want to View lead capture data from attendees who opted in to sponsor contact, so that Measure ROI from event sponsorship

### Organization Admin

- As a **Organization Admin**, I want to Invite team members and assign roles (admin, organizer, check-in staff, viewer), so that Delegate event management safely with role-based access
- As a **Organization Admin**, I want to Manage organization billing, subscription plan, and payment methods via Stripe portal, so that Control costs and upgrade/downgrade without contacting sales
- As a **Organization Admin**, I want to Configure SSO and SAML/OIDC for organization members via Keycloak, so that Enforce corporate identity policies and simplify login
- As a **Organization Admin**, I want to View aggregate analytics across all organization events, so that Understand event program performance at a glance
- As a **Organization Admin**, I want to Configure custom email sender domain and branding defaults for all events, so that Maintain brand consistency across all events
- As a **Organization Admin**, I want to Set organization-level feature flags and integrations (CRM, Slack, Zapier), so that Tailor the platform to the organization's workflow

### Super Admin (Platform)

- As a **Super Admin (Platform)**, I want to Manage all tenants, view platform-wide metrics, and impersonate organizations for support, so that Operate and support the platform efficiently
- As a **Super Admin (Platform)**, I want to Configure global feature flags via Unleash and roll out features progressively, so that Reduce deployment risk and test features with select tenants
- As a **Super Admin (Platform)**, I want to Monitor system health, error rates, and infrastructure costs, so that Maintain SLA commitments and control infrastructure spend

### Venue Manager

- As a **Venue Manager**, I want to List venue spaces with capacity, amenities, photos, and availability calendar, so that Attract event organizers searching for venues on the platform
- As a **Venue Manager**, I want to Receive and respond to venue inquiry requests from organizers, so that Convert platform traffic into bookings without a separate CRM

## Success Metrics

| Metric | Target |
|--------|--------|
| Monthly Recurring Revenue (MRR) | $50,000 MRR within 12 months of launch |
| Paying Organizations (Tenants) | 200 paying organizations within 12 months |
| Events Created Per Month | 1,000+ events/month by month 9 |
| Attendee Registrations Processed | 500,000 cumulative registrations within 18 months |
| Gross Ticket Volume (GTV) | $5M GTV processed through Stripe within 18 months |
| Registration Page Conversion Rate | >65% of page visitors complete registration |
| Check-in Processing Speed | QR scan to confirmed check-in under 2 seconds (p95) |
| Platform Uptime | 99.9% uptime SLA measured monthly |
| API Response Time | p95 < 300ms for all read endpoints |
| Net Promoter Score (NPS) | NPS > 45 from organizer surveys at 90-day cohort |
| Churn Rate | Monthly logo churn < 3% |
| Time to First Event Published | Median < 15 minutes from account creation to live registration page |
| Support Ticket Volume | < 0.5 tickets per event created (self-service effectiveness) |
| Trial-to-Paid Conversion | > 25% of trial accounts convert within 14-day trial period |

