# HP TMS — Build Prompt

## Part 1 — Role, vision and scope

**Your role.** You are a senior product architect and full-stack engineering lead. You have 15+ years building enterprise logistics software: Transport Management Systems (TMS), Warehouse Management Systems (WMS), freight audit and pay, and carrier networks. You know how logistics service providers (LSPs), 3PLs, shippers and carriers actually work. You write production-grade, secure, tested code and you explain design decisions briefly.

**What we are building.** HP TMS is a cloud-native, multi-tenant Transport Management System delivered as SaaS. It has three surfaces:

1. **Web platform** for planners, dispatchers, carrier managers, finance and admins.
2. **Driver mobile app** (iOS and Android) for job execution, proof of delivery and live tracking.
3. **Customer portal and customer mobile app** for booking, tracking, documents and invoices.

**Business model.** LSP (logistics service provider / 3PL), multi-tenant. The system must run the full order-to-cash and procure-to-pay cycle for transport: order intake, planning, carrier selection, execution, visibility, proof of delivery, freight cost settlement and customer billing.

**Target market.** Global from day one: multi-country, multi-currency, multi-language and multi-time-zone. Package country-specific rules (tax, e-invoicing, tolls, truck restrictions, address formats, holidays) as configurable country packs, so a new country is added by configuration. Build the UAE/GCC and India packs first; the country-specific examples in this brief come from those markets. Support road FTL/LTL first, with ocean and air in phase 2. Support any ISO 4217 currency, with USD, EUR, GBP, AED, SAR and INR set up at launch. Support English, Arabic (RTL) and Hindi at launch, with more languages added through translation files. Handle cross-border road freight and multi-leg international moves.

**Design principles.**

- Configuration over customisation: rates, workflows, statuses, documents and rules are set by tenant admins, not by code changes.
- API-first: every screen action is also available as a documented REST API.
- Event-driven: every status change emits an event that other modules, webhooks and integrations can consume.
- Mobile-first for field users, data-dense desktop UI for office users.
- Offline-tolerant driver app: drivers lose signal on highways and in border zones.
- Auditability: every change to an order, rate, invoice or user is logged with who, when, old and new value.

**Out of scope for now:** warehouse operations (put-away, picking), yard management beyond gate check-in/out, and customs filing (integrate, don't build).

## Part 2 — Users, personas and permissions

Build role-based access control (RBAC) with these default roles. Tenant admins can clone and edit roles down to the action level (view, create, edit, approve, delete, export) per module.

| Role | Surface | Main jobs |
| --- | --- | --- |
| Platform super-admin | Web | Create tenants, manage subscriptions, feature flags, global master data |
| Tenant admin | Web | Users, roles, business units, workflows, numbering, notification templates |
| Customer service / order desk | Web | Create and amend orders, quotes, handle exceptions and customer queries |
| Transport planner | Web | Build loads, optimise routes, assign vehicles and carriers |
| Dispatcher / control tower | Web | Live monitoring, exception handling, driver communication, ETA management |
| Carrier / procurement manager | Web | Onboard carriers, contracts, rate cards, spot tenders, scorecards |
| Fleet manager | Web | Own vehicles, drivers, maintenance, documents and compliance expiries |
| Finance / billing | Web | Freight audit, carrier settlement, customer invoicing, accruals, disputes |
| Management | Web, mobile | Dashboards, KPIs, profitability by lane, customer and carrier |
| Driver | Driver app | Accept jobs, navigate, capture POD, report delays and expenses |
| Carrier user (subcontractor) | Carrier portal | Accept tenders, assign own trucks and drivers, upload POD and invoices |
| Customer user | Customer portal and app | Book, track, download documents, approve quotes, view invoices |

**Data scoping rules.** Users see only data for their tenant, and within it only the business units, branches, customers or carriers they are assigned to. A carrier user sees only shipments tendered to that carrier. A customer user sees only their own company's shipments.

**Authentication.** Email + password with MFA, SSO via SAML 2.0 and OpenID Connect (Azure AD, Okta, Google), and phone OTP login for drivers.

## Part 3 — Core TMS modules (web platform)

Build each module below. For every module, deliver: data model, REST APIs, list and detail screens with filters and bulk actions, validation rules, events emitted, and unit + integration tests.

### 3.1 Master data

- Customers (bill-to, ship-to, consignee, notify party), with credit limit, payment terms, tax registration (TRN for UAE VAT, GSTIN for India) and contacts.
- Locations: addresses with geocodes, geofence radius, operating hours, dock count, loading constraints and access notes. Support UAE Makani numbers and plus codes where street addresses are weak.
- Carriers and subcontractors: legal entity, trade licence, insurance, bank details (masked), service areas, modes, equipment types, compliance documents with expiry alerts.
- Fleet: vehicles, trailers, containers, equipment types (curtain-sider, flatbed, reefer, lowbed for ODC, tipper), capacity in kg, m³ and pallets.
- Drivers: licence class and expiry, passport/visa/Emirates ID expiry (store document references only), medical and training certificates.
- Products/commodities with HS codes, dangerous goods class (UN number, ADR class), temperature range and stackability.
- Units of measure, currencies with daily exchange rates, tax codes, holiday calendars, reason codes.

### 3.2 Order management

- Order intake channels: manual entry, bulk Excel/CSV upload, customer portal, API, EDI 940/204, and email-to-order (parse PDF/email with AI, then human review).
- Order types: FTL, LTL/groupage, multi-drop distribution, multi-pick consolidation, container haulage (import, export, empty return), cross-border, project/ODC cargo.
- Order lines with packages, weight, volume, dimensions, declared value, special handling.
- Quote-to-order: generate a quote from rate cards, send for customer approval, convert to order.
- Order lifecycle statuses (configurable): Draft → Confirmed → Planned → Tendered → Accepted → Dispatched → In transit → Delivered → POD received → Invoiced → Closed, plus Cancelled and On hold.
- Amendments with version history and change-reason capture.

### 3.3 Planning and optimisation

- Planning board: map + Gantt + list views of unplanned orders, vehicles and drivers.
- Drag-and-drop load building with live utilisation bars (weight, volume, pallets, axle limits).
- Automatic load consolidation and route optimisation (vehicle routing problem with time windows, capacity, driver hours, vehicle restrictions and multi-depot). Use Google OR-Tools or an equivalent solver, run asynchronously, return the plan with cost and km comparison.
- Multi-leg and multi-modal shipments: pickup leg, linehaul, cross-dock, last-mile, with a different carrier per leg.
- Constraints: truck entry bans and timings (e.g., Dubai and Sharjah truck movement restrictions), hazmat routing, border crossing times, customer delivery windows.
- Planner can lock parts of a plan and re-optimise the rest.

### 3.4 Rating and freight cost engine

- Rate cards per customer (sell) and per carrier (buy): by lane, zone, distance band, weight break, pallet, container size, or flat per trip.
- Accessorials: waiting/detention, multi-drop, tail-lift, ODC escort, permits, fuel surcharge (index-linked table), border fees, toll (Salik, Darb), weekend/holiday surcharge.
- Contract validity dates, currency, minimum charges, rounding rules.
- Instant quote API: given origin, destination, cargo and date, return sell price, buy cost and margin.
- Margin guardrails: warn or block when margin falls below a tenant-set threshold.

### 3.5 Carrier management and tendering

- Carrier onboarding workflow with document upload, compliance check and approval.
- Carrier selection: rank by contract rate, performance score, capacity and lane coverage.
- Tendering modes: direct assignment, sequential waterfall (offer to carrier 1, then 2 after timeout), broadcast spot tender, and reverse auction.
- Tender acceptance via carrier portal, mobile, email link or EDI 990.
- Carrier scorecard: on-time pickup %, on-time delivery %, tender acceptance %, POD turnaround days, damage/claim rate, invoice accuracy.

### 3.6 Execution and dispatch

- Trip creation from planned loads, assignment of vehicle, trailer and driver, push to driver app.
- Trip sheet and delivery note generation (PDF, bilingual English/Arabic).
- Milestones: arrived at pickup, loaded, departed, border in/out, arrived at delivery, unloaded, POD captured.
- Exception management: delay, breakdown, accident, refused delivery, short/over/damaged cargo, temperature excursion. Each exception has owner, SLA and resolution log.
- Driver and vehicle hours tracking; alerts before limits are breached.
- Trip expenses: fuel, tolls, parking, cash advances, with receipt photos and approval.

### 3.7 Visibility and control tower

- Live map of all active trips from driver app GPS and telematics feeds, with geofence-based auto milestones.
- Predictive ETA using live position, traffic, historical lane times and remaining stops; recalculate every 5 minutes or on event.
- Control tower view: at-risk shipments ranked by severity, with one-click actions (call driver, notify customer, re-assign).
- Customer-facing tracking link (no login) showing status, ETA and POD, with tenant branding.
- Notifications by email, SMS, WhatsApp Business API and push, triggered by configurable rules.

### 3.8 Documents

- Generate: delivery note, consignment note/CMR, trip sheet, loading list, invoice, credit note, proof of delivery.
- Store: POD, customs documents, bill of lading, airway bill, packing list, photos, with OCR and auto-tagging to the shipment.
- Templates editable by tenant admin; bilingual and RTL layout support; QR code linking to the tracking page.

### 3.9 Freight audit and carrier settlement

- Accrue expected carrier cost at dispatch from the rate engine.
- Match carrier invoices (uploaded, emailed, EDI 210 or portal) against accruals: 3-way match of order, POD and invoice.
- Auto-approve within tolerance (e.g., ±2% or AED 50); route the rest to a dispute workflow.
- Self-billing option: generate the carrier's invoice from completed trips.
- Payment batches exported to the ERP; no direct payment execution.

### 3.10 Customer billing

- Invoice on POD, per shipment, or consolidated weekly/monthly per customer.
- Tax handling: UAE VAT 5% with TRN on invoice, KSA VAT and ZATCA e-invoicing readiness, India GST. Keep tax rules configurable per country.
- Credit notes, debit notes, revenue recognition dates, aging report.
- Profitability per shipment, customer, lane and month (revenue minus carrier and trip cost).

## Part 4 — Driver mobile app

Build one cross-platform app (iOS and Android) for own-fleet drivers and subcontractor drivers. Many drivers have basic phones and limited literacy, so the UI must use large buttons, icons, colour-coded statuses and minimal typing. Support English, Arabic, Hindi, Urdu, Malayalam and Bengali.

**Screens and features**

1. **Login:** phone number + OTP; device binding; optional PIN for quick re-entry.
2. **Today's jobs:** list of assigned trips with stop sequence, time windows and cargo summary. Accept or reject with reason.
3. **Trip detail:** stops, contacts (tap to call or WhatsApp), special instructions, documents to carry.
4. **Navigation:** hand-off to Google Maps or Waze with truck-safe route where available.
5. **Milestone buttons:** one tap for Arrived, Loading started, Loaded, Departed, Arrived at delivery, Unloaded. Auto-trigger by geofence, with driver confirmation.
6. **Proof of delivery:** recipient name, e-signature, photos (cargo, seal, damage), barcode/QR scan of packages, quantity delivered vs expected, OTP from consignee as optional secure POD.
7. **Proof of pickup:** same as POD, plus seal number and container number capture with camera OCR.
8. **Exceptions:** report delay, breakdown, accident, refused, damaged or short cargo with photo, voice note and reason code.
9. **Expenses:** fuel, tolls, parking, with receipt photo and amount; cash advance balance.
10. **Vehicle inspection:** pre-trip checklist (tyres, lights, straps, fire extinguisher) configurable per vehicle type.
11. **Messages:** chat with dispatcher, push notifications for new jobs and changes.
12. **Earnings (subcontractor drivers):** completed trips and payable amounts.

**Technical requirements**

- Offline-first: queue all actions, photos and signatures locally; sync automatically when online; show sync status clearly.
- Background GPS every 30–60 seconds while on a trip, adaptive to save battery; stop tracking when the trip ends (driver privacy).
- Compress photos to under 300 KB before upload; keep originals until synced.
- Every event carries device timestamp, server timestamp and GPS coordinates.
- Works on Android 9+ and iOS 15+, on low-end devices with 2 GB RAM.

## Part 5 — Customer portal, carrier portal and customer app

**Customer portal (web) and customer app (iOS/Android)** share one backend and one design system.

- Self-service booking: pick saved addresses, cargo templates, pickup date; get an instant quote; confirm.
- Bulk booking by Excel upload with row-level validation errors.
- Shipment list with search by order number, reference, container or AWB, and status filters.
- Live tracking map, ETA, milestone timeline, and POD download.
- Documents: download delivery notes, PODs, invoices; upload commercial invoices and packing lists.
- Invoices and statement of account; raise a dispute on an invoice line.
- Notification preferences per user (email, SMS, WhatsApp, push).
- Customer KPIs: on-time %, shipments by status, spend by lane and month.
- White-label: tenant logo, colours and custom domain (e.g., track.tenantname.com).

**Carrier portal (web, mobile-responsive)** for subcontractors:

- View and accept/reject tenders and bid on spot loads.
- Assign own truck and driver to accepted loads; the driver gets the driver app invitation.
- Upload POD and invoices; see payment status and remittance.
- Maintain own compliance documents and see expiry warnings.

## Part 6 — Integrations

Build an integration layer with a connector framework, so new partners are added by configuration and mapping, not new code. Every inbound and outbound message is logged with payload, status, retries and error, and failed messages can be reprocessed from a monitoring screen.

| Area | Integrate with | Format |
| --- | --- | --- |
| EDI (road/US-style) | Customers and carriers | X12 204 load tender, 990 response, 214 status, 210 freight invoice |
| EDI (international) | Forwarders, shipping lines | EDIFACT IFTMIN, IFTSTA, INVOIC |
| ERP | SAP S/4HANA, Oracle NetSuite, Microsoft Dynamics 365, Odoo, Tally (India) | REST/OData, flat file via SFTP |
| WMS | Tenant's WMS | Order release in, dispatch confirmation out |
| Telematics and GPS | Teltonika, Samsara, Geotab, Wialon and local GCC providers | REST, webhooks, MQTT |
| Maps and routing | Google Maps Platform, HERE (truck routing), Mapbox | REST |
| Ocean and air tracking | Container and AWB tracking aggregators | REST, webhooks |
| Messaging | WhatsApp Business API, SMS gateway, email (SES/SendGrid), push (FCM/APNs) | REST |
| E-invoicing | ZATCA (KSA), India GST IRP, UAE e-invoicing (when mandated) | Per authority spec |
| Customs and ports | Dubai Trade, Mirsal, ULIP (India), port community systems | Integrate only; no filing logic |
| Identity | Azure AD, Okta, Google Workspace | SAML 2.0, OIDC |

**Public API.** Publish an OpenAPI 3.1 spec for: orders, shipments, quotes, tracking events, documents, invoices, master data. Use OAuth 2.0 client credentials, rate limiting per client, idempotency keys on create calls, and outbound webhooks with signed payloads (HMAC) and retry with exponential backoff.

## Part 7 — Data model

Design a normalised relational schema with these core entities. Every table has `tenant_id`, `id` (UUID), `created_at`, `created_by`, `updated_at`, `updated_by` and soft-delete. Produce an ER diagram and migration files before writing services.

- **Tenant** → BusinessUnit → Branch
- **Party** (customer, carrier, consignee, vendor) → Contact, Address, Document, RateCard
- **Location** (geocode, geofence, hours, constraints)
- **Order** → OrderLine → Package/HandlingUnit
- **Shipment** (the physical movement; one order can split into many shipments, many orders can consolidate into one)
- **Leg** (one shipment has one or more legs, each with mode and carrier)
- **Load / Trip** → Stop (pickup or delivery, sequence, time window) → StopActivity
- **Vehicle, Trailer, Equipment, Driver**, with assignments per trip
- **Tender** → TenderOffer (carrier, price, status, response time)
- **TrackingEvent** (event code, timestamp, location, source: app, telematics, EDI, manual)
- **Exception** (type, severity, owner, SLA, resolution)
- **Document** (type, file reference, linked entity, OCR text)
- **RateCard** → RateLine → Accessorial, FuelSurchargeTable
- **Charge** (expected cost or revenue line on a shipment, buy or sell side)
- **CarrierInvoice** → InvoiceLine → MatchResult, Dispute
- **CustomerInvoice** → InvoiceLine, CreditNote, Payment reference
- **AuditLog** (entity, field, old value, new value, user, timestamp)
- **IntegrationMessage** (direction, partner, format, payload reference, status, retries)

Keep Order, Shipment and Trip as separate entities. This separation is what lets the system handle consolidation, splits and multi-leg moves without hacks.

## Part 8 — Architecture and tech stack

Use the stack below. Justify any change you make.

Architecture: all four surfaces (web platform, driver app, customer/carrier portals, partner systems) call one API gateway (OAuth 2.0, SSO, rate limits, webhooks). Behind it sits a modular monolith of domain modules that share data through domain events on an event bus, not direct database calls. The data layer is PostgreSQL (PostGIS, RLS), Redis + OpenSearch, object storage, and a Python OR-Tools optimiser service.

| Layer | Choice |
| --- | --- |
| Web front end | React + TypeScript, Next.js, Tailwind CSS, a component library with RTL support, TanStack Query, AG Grid for data-dense tables |
| Mobile apps | React Native (Expo) or Flutter; SQLite/WatermelonDB for offline storage |
| Backend | Modular monolith first (NestJS/TypeScript or Java Spring Boot), split into services only where load demands it |
| Database | PostgreSQL with PostGIS; row-level security for tenant isolation |
| Cache and queues | Redis; Kafka or RabbitMQ for domain events |
| Search | OpenSearch for shipment search and logs |
| Optimisation | Python service with Google OR-Tools, called asynchronously |
| Files | S3-compatible object storage with signed URLs |
| Real-time | WebSockets for live map and control tower updates |
| Infra | Docker, Kubernetes, Terraform; AWS or Azure, multi-region for data residency |
| CI/CD | GitHub Actions with lint, tests, security scans and preview environments |
| Observability | OpenTelemetry, Prometheus, Grafana, Sentry |

**Module boundaries** (separate packages with clear interfaces, even inside the monolith): Identity & tenancy, Master data, Orders, Planning, Rating, Carriers & tendering, Execution, Visibility, Documents, Finance (audit, settlement, billing), Integrations, Notifications, Analytics.

**Rules for the codebase**

- Domain events for every state change (e.g., `shipment.delivered`), published through an outbox table to avoid lost events.
- No business logic in controllers; keep it in domain services with unit tests.
- All money as decimal with currency code, never floats.
- All timestamps in UTC; display in the location's time zone.
- Feature flags per tenant for every major module.

## Part 9 — Non-functional requirements

| Area | Requirement |
| --- | --- |
| Multi-tenancy | Shared schema with `tenant_id` + Postgres row-level security; option for a dedicated database for large tenants |
| Performance | API p95 under 300 ms for reads, 800 ms for writes; shipment list of 100k rows filters in under 2 s; tracking map handles 5,000 live vehicles per tenant |
| Scale | 1 million shipments per month across tenants; 50 GPS pings per second sustained |
| Availability | 99.9% monthly uptime; RPO 15 minutes, RTO 4 hours; daily backups with point-in-time recovery |
| Security | OWASP Top 10 and OWASP MASVS for mobile; encryption at rest (AES-256) and in transit (TLS 1.2+); secrets in a vault; MFA for admins; session timeout; IP allow-listing per tenant |
| Privacy | Comply with UAE PDPL (Federal Decree-Law 45/2021), India DPDP Act 2023 and GDPR for EU customers; driver location only during active trips; data-retention settings per tenant |
| Data residency | Deploy in several cloud regions (e.g., Middle East, India, EU, US); each tenant picks a home region and its data stays there |
| Localisation | English and Arabic with full RTL mirroring; Hindi and others via translation files; local date, number and currency formats; Hijri date display option |
| Accessibility | WCAG 2.1 AA for web; large-touch-target mode on mobile |
| Audit | Immutable audit log for orders, rates, invoices, users and permissions; exportable for auditors |
| Testing | 80% unit-test coverage on domain logic; API contract tests; Playwright end-to-end tests for key flows; load tests on tracking ingestion |

## Part 10 — Analytics, dashboards and AI features

**Standard dashboards** (role-based, filter by date, branch, customer, carrier, lane, mode):

- Operations: shipments by status, on-time pickup and delivery %, open exceptions by age, POD pending.
- Fleet: vehicle utilisation %, empty km %, km per trip, fuel cost per km, maintenance due.
- Carriers: scorecard ranking, tender acceptance %, cost per lane vs benchmark.
- Finance: revenue, cost, gross margin by customer and lane, unbilled shipments, carrier invoice match rate, receivables aging.
- Customer service: order lead time, query volume, first-response time.

All dashboards export to Excel and PDF, and can be scheduled by email. Expose a read-only analytics database or data export for Power BI and Tableau.

**AI features** (each must show its reasoning or confidence, and let a human override):

1. Email and PDF order capture: extract orders from customer emails and attachments into draft orders for review.
2. Document OCR: read POD, CMR, bill of lading and carrier invoices; auto-link to the shipment and flag mismatches.
3. Predictive ETA and delay risk score per shipment.
4. Rate suggestion for spot quotes based on lane history, season and market index.
5. Carrier recommendation per load, ranked by cost, reliability and acceptance likelihood.
6. Anomaly detection on carrier invoices (duplicate, overcharge, unusual accessorials).
7. Natural-language assistant for office users: "Show me all late deliveries for customer X this week" returns a filtered list or chart.
8. Demand forecasting by lane to plan fleet and carrier capacity.

## Part 11 — Delivery phases, outputs and acceptance criteria

Build in this order. Finish, test and demo each phase before starting the next. Build scope: MVP first (Phases 0–1), then the remaining phases in order.

| Phase | Scope | Done when |
| --- | --- | --- |
| 0. Foundation | Repo structure, CI/CD, auth, tenancy, RBAC, audit log, design system (LTR + RTL), master data | A tenant admin can log in, create users and roles, and load customers, locations, carriers and vehicles |
| 1. MVP order-to-delivery | Orders, manual planning, trip dispatch, driver app (jobs, milestones, POD, offline), basic tracking, customer tracking link | An order goes from creation to POD with photos and signature, fully offline-tolerant |
| 2. Commercial | Rate engine, quotes, carrier tendering, carrier portal, customer billing with VAT, carrier invoice matching | A shipment is auto-rated, tendered, accepted, delivered, invoiced to the customer and matched to the carrier invoice |
| 3. Optimisation and visibility | Route optimisation, load building, control tower, predictive ETA, notifications (WhatsApp, SMS, push) | Planner optimises 200 orders in under 60 s; at-risk shipments surface automatically |
| 4. Integrations | Public API, webhooks, EDI, ERP and telematics connectors, monitoring screen | A customer creates orders by API and receives status webhooks; a telematics feed drives auto milestones |
| 5. Intelligence | Dashboards, AI order capture, OCR, invoice anomaly detection, NL assistant | Management sees margin by lane; an emailed order becomes a draft order without retyping |

**What I expect from you in every phase**

1. A short plan first: modules, entities, APIs and screens for the phase. Wait for my approval.
2. Database migrations and seed data for a realistic demo tenant (UAE LSP with 20 trucks, 3 subcontractors, 10 customers, 200 orders across Dubai, Abu Dhabi, Sharjah and Riyadh).
3. Working backend, web and mobile code with tests.
4. OpenAPI spec updated.
5. A README with setup steps, environment variables and how to run tests.
6. A list of assumptions you made and open questions for me.

**General acceptance criteria**

- [ ] No hard-coded tenant, currency, tax rate or language anywhere.
- [ ] Every list screen has search, filters, column chooser, sorting, pagination and Excel export.
- [ ] Every state change writes an audit record and emits a domain event.
- [ ] Arabic RTL layout renders correctly on web and mobile.
- [ ] Driver app completes a full trip with the phone in airplane mode and syncs afterwards without data loss.
- [ ] Tenant A can never read or change Tenant B's data (prove it with automated tests).
- [ ] Security scan shows no high or critical findings.

Before writing any code, ask me up to 5 clarifying questions about anything in this brief that is ambiguous or that would change the architecture.

---

**Start now:** Do not write code yet. Ask me your clarifying questions, then give me the Phase 0 plan: repo structure, data model with an entity diagram, and auth/tenancy/RBAC design. Save this brief in the repo as `docs/BRIEF.md` so every later session can read it.
