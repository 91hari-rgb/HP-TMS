# HP TMS — Configuration Model

Status: **draft, aligned with the reference solution configuration document (SCD)** supplied by the product owner for an existing LSP TMS deployment.
The reference document is confidential and is **not** stored in this repository. This file records only the configuration *concepts* HP TMS adopts and how they map onto our modules. Nothing here copies vendor screens, code or text.

The clickable demo (`demo/hp-tms-demo.html` → **Configuration**) implements every area below, for three sample tenants: UAE/KSA (AED), India (INR) and the EU (EUR, operating in Germany, the Netherlands and Poland). Areas marked **Live** change behaviour in the demo; **Table** areas are maintained there and used in a later phase.

## 1. Why this matters

A configuration-driven LSP TMS lets each tenant run its own processes without code changes (brief, design principle 1). The reference SCD shows what a mature LSP needs to configure, so HP TMS treats these as first-class, tenant-scoped master tables with audit, rather than hard-coded logic.

## 1a. Global masters: countries and currencies (Phase 0)

HP TMS is global. The UAE, India and the EU tenants in the demo are only the first **country packs**; nothing in the code is specific to a region.

- **Country master** (platform reference data, ISO 3166): ISO 2/3/numeric codes, region, currency, dial code, **weekend days**, public holidays, tax name, **standard rate and road-freight rate** (they differ in some countries: India GST on transport 12%, China 9%), e-invoicing regime, postcode format, driving side, distance unit, trade bloc. 41 countries ship in the demo across the Middle East, South Asia, South-East and East Asia, Oceania, Europe, Africa and the Americas.
  - A tenant **activates** the countries it operates in.
  - Service-level deadlines use the **destination country's** weekend and holidays (a Friday is a working day in the UAE, not in KSA).
  - Tax on freight follows the country pack:
    - **Domestic:** the road-freight rate.
    - **GCC and other international lanes:** zero-rated.
    - **EU B2B:** place of supply is the customer's country. Domestic customers pay home-country VAT; customers in another EU state get the **reverse charge**; customers outside the EU are outside the scope.
    - **India:** CGST + SGST within a state, IGST between states.
  - Tax and e-invoicing entries are indicative and must be confirmed with a tax adviser per country before go-live.
- **Currency master** (ISO 4217): code, numeric code, name, symbol and **minor units**. KWD, BHD, OMR and JOD have 3 decimals; JPY, KRW and VND have none. Amounts are formatted and rounded to the currency's own decimals.
  - Exchange rates are held per USD and cross-converted to each tenant's base currency. A daily agent imports them (UAE central bank, RBI, ECB, … per tenant).
  - **Divisions** can have their own base currency (for example EuroLink's Polish entity in PLN).
  - **Customers** can have a billing currency; invoices show the converted total and the rate date.
- **Tenant profile:** every tenant-specific value lives in data, not code:
  - home country, domain, label printers, tax-ID validation pattern;
  - registration names, insurers, number formats, toll product, e-invoice format, fiscal year start;
  - hubs, zones and carrier-selection rules.

  Adding a country or a tenant is configuration only.

## 2. Organisation hierarchy (Phase 0)

```
Tenant (company) ─┬─ Division (legal entity: tax registration, base currency, contra account)
                  │     └─ Branch (4-digit, location, cost centre)
                  │            └─ Department (operations / invoice / fixed internal)
                  │                   └─ File type = department + handling code
                  └─ shared master data (relations, locations, equipment, codes)
```

- **Division = legal entity.** Each division has its own tax registration and base currency. A contra account balances cross-division postings, for example when one entity's truck runs another entity's shipment.
  - In India, each state GST registration (GSTIN) is its own division, because GST registration is per state.
- **Branch.** Every order is registered at the branch of its pickup location. The branch decides the cost centre on postings and limits which data users can see (see RBAC scopes in the Phase 0 plan).
- **Department.** Orders and trips belong to a department. A separate invoice department splits operations work from finance work. A fixed internal department (`SYS`) holds system files.
- **File type** (department + handling code) is the main switch that steers how an order is processed:
  - category (shipment / trip / general);
  - default service level, planning method and document set;
  - charge generation;
  - file closing and archiving windows. The defaults are:
    - minimum 90 days before a file can close;
    - an error is logged after 95 days still open;
    - a closed file can be reopened within 3 days;
    - a closed file is archived 9 days after closing.

**Data model additions:** `division`, `department`, `file_type`, and `file_type_setting` (per-file-type key/value settings with validity dates).

## 3. Finance configuration (Phase 2)

| Entity | Purpose |
| --- | --- |
| `ledger_account` | Chart of accounts used for postings (debtors and creditors control accounts, revenue, cost, VAT/GST, accruals, inter-division contra). |
| `cost_centre` | Taken from the file's branch on every posting line. |
| `journal` | SI sales invoices, SC credit notes, PI purchase invoices, AC accruals at dispatch, BK bank. |
| `tax_code` | Per country. Rate, plus payable and receivable GL accounts. Chosen per lane: UAE 5% / zero-rated international, KSA 15%, India CGST + SGST (intra-state) or IGST (inter-state). |
| `payment_term` | Due date = invoice date + days. |
| `activity` | Cost and charge code (FRT, FSC, SVC, WAIT, …) with sales GL, purchase GL, VAT behaviour and cost centre rule. **Every charge line on a shipment is an activity.** |
| `activity_generation_rule` | Automatic activity generation: validity dates, branch, file type, type of file (order/trip), purchase/sales, charge-to relation (principal, carrier, internal relation, …), activity, condition, and the fallback when the relation is missing. Switching a rule off stops that charge being generated. |

- **Journal posting:** every invoice, credit note and purchase invoice produces a balanced journal. The ERP integration exports these postings; HP TMS does not keep a general ledger.
- **Accruals:** carrier cost is accrued at dispatch (Dr cost / Cr accrued carrier cost). When the purchase invoice is approved, the accrual is released and any difference posts to price variance.
- **Supplementary billing:** a closed file can be reopened (status 998) to add a missing charge. Only the new lines are billed, on a supplementary invoice. The original invoice is never changed.

## 4. Relations configuration (Phases 0–2)

- **Business relations** can be debtors, creditors, both, or prospects. They link to **relation groups**, which carry the control account.
- **Registration types** are numbered: 1–999 are standard, 1000+ are tenant-specific. They store trade licence, TRN/GSTIN and similar, plus flags such as "excluded from unrated-consignment check", instead of hard-coded carrier lists.
- **Con note types per carrier:** a prefix, a number range, the next number, and a mod-7 check digit. A con note is issued at dispatch, printed on labels and CMR, and searchable in Track & trace.
- **Shipping unit ranges (SSCC):** GS1 company prefix plus serial range, per relation. Every pallet label carries an 18-digit SSCC with a GS1 check digit.
- **Consignment carrier selection:** an ordered rule list. Each rule matches on branch, department, principal, loading zone, unloading zone, a weight operator and package code, and sets the carrier and service level during order entry. A carrier blocked by an expired compliance document falls back to own fleet.

## 5. Process flow (Phases 1–4)

- **Status codes per object**, numbered:
  - **File (order):** 100 draft, 150 on hold, 200 confirmed, 400 dispatched, 500 in transit, 600 delivered, 700 POD received, 800 invoiced, 900 cancelled, 998 reopened, 999 closed.
  - **Trip / leg:** 100–800.
  - Units, sales invoices and purchase invoices each have their own numbered list.
- **Track & trace codes** (see the code master in the demo), and **T&T code cascading**: a code logged on a trip or unit is taken over by its orders. For example:
  - PU-DEP on a trip sets file status 500 on every order on the trip;
  - DL-POD on every unit sets the order to DL-POD.
- **Events**: 6-digit numbers. The first 2 digits are the module (10 file, 20 trip, 30 unit, 50 invoice, 60 relation); 0000–4999 are standard and 5000–9999 tenant-specific. Each event links to a T&T code and a follow-up action (alert, email template, task, return message).
- **Process-flow expectations:** "after code A, expect code B within N hours, else take an action". For example, a POD expected within 2 hours of arrival. Breaches appear in the control tower.
- **Hold events** are reason codes that block planning and dispatch.
- **Agents** are background jobs with status codes (0 not activated, 1 activated, 2 stopping, 3 erroneous, 4 restart requested, 5 paused). The jobs are: ETA recalculation, expectations monitor, auto return messages, closing files, archiving files, exchange-rate import, and document-expiry alerts.

## 6. Connectivity (Phase 4)

- **Trigger auto return message:** T&T code + principal + file type → outbound message (X12 214, webhook, email, e-invoice), sent directly or by an agent.
- **EDI parameters** per partner: relation, department, message (ORDERS-IN, MANIFEST-IN, TRIP-PLAN, STATUS-OUT, INVOICE-OUT), format and type of setting. The "trip planning = Y" flag is set only for the trip-plan message.
- **Message monitor:** every inbound and outbound message keeps its payload, status, retries and error. A failed outbound message can be retried; an inbound message in error can be reprocessed after the mapping is fixed. This matches the brief's integration monitoring screen.

## 7. Transport configuration (Phases 1–3)

- **Trip debrief:** each delivery is debriefed with a status code (delivered OK, with remarks, partial, refused, return on documents) and a quality/qualification code. Refused and partial deliveries automatically create a **related return order**.
- **Distribution network** (Phase 3, multi-leg):
  - hubs and cross-docks;
  - postcode zones. The UAE has no postal codes, so zones there are by area or Makani number;
  - network connections between hubs (days, cut-off, arrival, priority);
  - fixed trip templates.
- **Equipment types / groups** (already in Phase 0): payload, volume, pallets, length, axles, temperature range, out-of-gauge flag, rate factor.
- **Cross-docking / RF:** out of scope, in line with the brief (warehouse operations).

## 7a. Master orders (order consolidation) and related orders (Phase 1)

- **Master order** (file kind `Master`, file status 250): groups compatible confirmed orders into one load.
  - **Checks before creating one:** same equipment; total weight and pallets within that equipment's capacity; no order already in another master. Warnings appear for mixed zones, different pickup days and tight service levels.
  - **Cost preview:** separate trips versus one shared trip.
  - **Billing and dispatch:** each child keeps its own customer, references, charges and invoice. The master carries no charges, and the trip cost is shared by weight. Dispatching the master dispatches all its children on one trip.
  - **Where you create one:** select orders in Orders and click *Consolidate into master*; use *Consolidate with similar orders* in an order's details; or use *Create master order* on a Planning → Order consolidation suggestion.
  - **Changing one:** add similar orders from the master's details, or remove a child. A master left with fewer than two orders is dissolved.
- **Related orders panel** on every order shows: its master, siblings and children (consolidation); original and return orders (from trip debrief); and **manual links** (same delivery, split from, replacement for, same customer PO, claim / damage).
- **T&T code `ORD-CON`** is logged on each child when it joins a master.
- **Data model:** `order.kind` (standard / master), `order.master_id`, and `order_link` (from, to, relation type).

## 8. General tables (Phases 0–1)

UN/LOCODE locations, delivery terms (Incoterms 2020), UN/ECE package codes, commodities (HS code, UN number, ADR class, temperature), tariff types, service levels, means of transport, reference types (mandatory per principal), types of date (monitored or not), amount types, transport instructions, quality codes, email templates, holiday calendars.

## 9. Coverage

`demo → Configuration → Coverage overview` lists 45 configuration areas, with where each lives in HP TMS, whether it is Live in the demo, and its build phase. Use it as the traceability matrix between the reference SCD's areas and HP TMS.

## 10. Open questions raised by the reference SCD

1. **Divisions per KSA entity.** Should KSA lanes invoice from a separate KSA division (SAR base currency, its own VAT registration), or from the UAE entity with zero-rating? This affects ZATCA e-invoicing in Phase 2.
2. **File numbering.** Per branch (as in the demo) or per department? The reference uses branch-level numbering.
3. **Activity generation vs rate cards.** Should automatic activity generation rules be maintained by tenant admins, or generated from rate cards? The demo uses rules for *which* charges appear and rate cards for *how much*.
4. **Agents.** Which background jobs need per-tenant schedules versus platform-wide ones?
