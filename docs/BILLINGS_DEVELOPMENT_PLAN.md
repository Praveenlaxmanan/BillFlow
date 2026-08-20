# Billings — SAP-Style Automatic Billing Platform

**Document:** Development plan with activity-level day estimates  
**Location:** `D:\Cursor\Billings`  
**Status:** Plan only — implementation not started  
**Last updated:** 2026-08-20

---

## 1. Product summary

**Billings** is an orchestrated billing platform where **customer requests** are validated against **master data and business rules**, **priced and taxed automatically**, converted into **billing documents** (invoices, credit memos), and **delivered to the customer**. Exceptions route to staff; **LLM is optional** and limited to unstructured intake and customer messaging—not pricing or tax math.

### One-line definition

> Customer request in → validated, priced, invoiced, sent out → or routed to exception queue with a clear reason.

### Terminology (aligned with Manufacturer project)

| Term | Meaning in Billings |
|---|---|
| **Orchestration** | Workflow engine: plan steps, pass evidence, retries, audit |
| **Module / worker** | Single capability: intake, pricing, invoice PDF, etc. |
| **Agent** | Optional LLM + tools (email parse, exception reply drafts) |
| **Mode** | Runtime behavior: `AUTO`, `DRAFT_ONLY`, `LLM_ASSISTED_INTAKE` |

---

## 2. Scope reference (SAP-like areas)

| Area | In scope (phased) |
|---|---|
| Customer / product master | Yes — Phase 0–1 |
| Pricing & contracts | Yes — Phase 1–2 |
| Tax determination | Yes — Phase 2 (rules); Phase 5 (external API optional) |
| Billing documents (invoice, credit memo) | Yes — Phase 1–4 |
| Output (PDF, email, portal) | Yes — Phase 2–3 |
| Accounts receivable & dunning | Partial — Phase 4–5 |
| GL / payment gateway / EDI | Phase 5 |

---

## 3. Assumptions for estimates

| Assumption | Value |
|---|---|
| Team size | **1 full-time developer** (backend + basic UI) |
| Working day | **1 day = 6–7 focused hours** |
| Tech stack | Python or Node API, PostgreSQL/SQL Server, workflow/orchestration layer |
| LLM | Optional; not on critical pricing/tax path |
| SAP integration | Out of scope for v1; design hooks only |
| Buffer | **+15%** recommended for unknowns (not included in table totals) |

**Calendar total (single developer):** ~**88 working days** (~4.5 months) for Phases 0–5 below.

---

## 4. Architecture overview

```text
Channels (Portal · API · Email · EDI)
              │
              ▼
┌─────────────────────────────────────────────┐
│  ORCHESTRATION — Billing workflow            │
│  received → validated → priced → billed → sent│
└───┬──────┬──────┬──────┬──────┬────────────┘
    ▼      ▼      ▼      ▼      ▼
 Intake  Master Pricing Document Output
         data   + Tax  module   module
 module  module module
                            ▼
                     Exception queue
                     (human tasks)
```

### Core modules

| Module | Responsibility | LLM? |
|---|---|---|
| Intake | Accept & normalize requests | Optional (email) |
| Master data | Customer, product, terms lookup | No |
| Pricing | Price lists, discounts, contracts | No |
| Tax | Jurisdiction rules | No |
| Credit / risk | Limit checks, blocks | No |
| Billing document | Invoice / credit memo lifecycle | No |
| Output | PDF, email, portal | No |
| AR / payments | Open items, allocation | No |
| Exception | Task queue, reason codes | Optional (draft replies) |

---

## 5. End-to-end automatic flow

```text
1. Customer submits request (API / portal / email)
2. Intake validates schema + idempotency
3. Master data resolves customer & products
4. Credit module approves or holds
5. Pricing module calculates line amounts
6. Tax module applies rates
7. Billing document module creates POSTED invoice
8. Output module generates PDF + sends email
9. On any failure → Exception module + audit log
```

---

## 6. Development phases — summary

| Phase | Name | Duration | Cumulative |
|---|---|---:|---:|
| **0** | Design & data model | **8 days** | 8 |
| **1** | Core billing (draft path) | **18 days** | 26 |
| **2** | Full automatic processing | **14 days** | 40 |
| **3** | Customer channels & portal | **12 days** | 52 |
| **4** | Extended billing types & AR | **18 days** | 70 |
| **5** | Enterprise integration | **18 days** | 88 |

---

## 7. Phase 0 — Design & data model (8 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 0.1 | Requirements workshop: request types, statuses, roles | 1 | Scope doc |
| 0.2 | ERD: customers, products, contracts, requests, invoices | 2 | ERD diagram + table list |
| 0.3 | Billing request / invoice JSON schemas (OpenAPI draft) | 1 | `schemas/` spec |
| 0.4 | Workflow state machine design (statuses, transitions) | 1 | State diagram |
| 0.5 | Rule catalog v1 (validation, pricing, credit, auto-post) | 1 | Rule matrix |
| 0.6 | Non-functional: idempotency, audit, security, numbering | 1 | NFR checklist |
| 0.7 | Seed data plan (sample customers, products, prices) | 1 | `DATA.md` equivalent |

**Phase 0 exit criteria:** Signed-off ERD, API contract for “submit request”, workflow states, rule catalog.

---

## 8. Phase 1 — Core billing, draft path (18 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 1.1 | Project scaffold: repo, DB migrations, config | 2 | Runnable skeleton |
| 1.2 | Customer master CRUD + lookup APIs | 2 | `customers` module |
| 1.3 | Product catalog CRUD + lookup APIs | 2 | `products` module |
| 1.4 | Price list storage + simple “base price” lookup | 2 | `pricing` module (basic) |
| 1.5 | Intake module: POST billing request, validation | 2 | Intake API |
| 1.6 | Orchestration skeleton: state store + step runner | 3 | Workflow engine v1 |
| 1.7 | Billing document module: DRAFT invoice header/lines | 3 | Draft invoice API |
| 1.8 | Unit tests for intake + master + draft invoice | 1 | Test suite v1 |
| 1.9 | Seed script: 10 customers, 20 products, 3 price lists | 1 | `init_db` script |

**Phase 1 exit criteria:** API accepts request → returns **draft invoice** with priced lines (fixed price list, no tax/credit yet).

---

## 9. Phase 2 — Full automatic processing (14 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 2.1 | Contract / customer-specific pricing rules | 2 | Pricing rules engine |
| 2.2 | Discount & surcharge rules (volume, promo codes) | 2 | Extended pricing |
| 2.3 | Tax module: region → rate, line tax amounts | 2 | Tax calculation |
| 2.4 | Credit module: limit, open AR, block flags | 2 | Credit decision API |
| 2.5 | Invoice number range (sequential, concurrency-safe) | 1 | Number service |
| 2.6 | Auto-post rules: when DRAFT → POSTED | 1 | Rule engine hook |
| 2.7 | Output module: PDF template + file storage | 2 | Invoice PDF |
| 2.8 | Email delivery (SMTP / SendGrid) | 1 | Send invoice job |
| 2.9 | Idempotency: duplicate request → same invoice | 1 | Idempotency keys |

**Phase 2 exit criteria:** Happy-path request **fully automated**: validate → price → tax → post → PDF → email, no manual step.

---

## 10. Phase 3 — Customer channels & portal (12 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 3.1 | Public REST API hardening (auth, rate limits) | 2 | Secured API |
| 3.2 | Customer portal: submit order, view invoices | 4 | Simple web UI |
| 3.3 | Internal clerk UI: search cases, view audit trail | 3 | Ops UI |
| 3.4 | Exception queue UI: list, assign, approve/reject | 2 | Exception module UI |
| 3.5 | Webhook / callback on invoice posted | 1 | Event notifications |

**Phase 3 exit criteria:** Customer can submit via portal/API and receive invoice; clerk can see exceptions.

---

## 11. Phase 4 — Extended billing types & AR (18 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 4.1 | Credit memo flow (return authorization link) | 3 | Credit memo module |
| 4.2 | Debit memo / invoice correction (delta billing) | 3 | Correction module |
| 4.3 | Recurring billing scheduler (subscriptions) | 4 | Scheduler + jobs |
| 4.4 | AR open items: post invoice to receivables ledger | 2 | AR sub-ledger |
| 4.5 | Payment recording & allocation to invoices | 3 | Payment module |
| 4.6 | Dunning: overdue rules + reminder letters | 2 | Dunning job |
| 4.7 | Reporting: billing run summary, exception KPIs | 1 | Reports API |

**Phase 4 exit criteria:** Credit/recurring paths work; basic AR and dunning operational.

---

## 12. Phase 5 — Enterprise integration (18 days)

| # | Activity | Days | Deliverable |
|---|---|---:|---|
| 5.1 | GL export (CSV / journal entry format) | 3 | FI export |
| 5.2 | Payment gateway integration (Stripe/Razorpay/etc.) | 4 | Online pay link |
| 5.3 | EDI / flat-file outbound invoices (optional format) | 4 | EDI adapter |
| 5.4 | External tax API adapter (Avalara-style hook) | 3 | Tax provider plug-in |
| 5.5 | Optional LLM intake: parse email → structured request | 3 | LLM agent module |
| 5.6 | Observability: structured logs, metrics, case tracing | 1 | Dashboards |

**Phase 5 exit criteria:** Finance export works; optional channels and LLM intake documented and gated by mode.

---

## 13. Orchestration implementation options

| Option | Dev effort | Best for |
|---|---|---|
| **Custom state machine** | Included in Phase 1 (3 days) | Full control, SAP-like statuses |
| **LangGraph ReWOO** | +2 days on top of 1.6 | Fixed pipelines with evidence `#E1…#En` |
| **Temporal / Camunda** | +5 days setup | Long-running, human tasks, retries |

**Recommendation for v1:** Custom state machine in Phase 1; add ReWOO-style evidence logging for audit (similar to Manufacturer project).

---

## 14. Automatic processing rule catalog (v1)

| Rule ID | Condition | Action | Module |
|---|---|---|---|
| R-001 | Required fields missing | Reject + reason code | Intake |
| R-002 | Unknown customer | Exception: master data | Master data |
| R-003 | Customer blocked | Reject | Master data |
| R-004 | Unknown product | Exception: catalog | Master data |
| R-005 | No price found | Exception: pricing | Pricing |
| R-006 | Credit exceeded | Hold → exception queue | Credit |
| R-007 | All checks pass | Auto-post invoice | Orchestration |
| R-008 | Duplicate idempotency key | Return existing invoice | Intake |
| R-009 | Tax jurisdiction unknown | Exception: tax setup | Tax |
| R-010 | Post success | Generate PDF + email | Output |

---

## 15. Data model (minimum entities)

| Entity | Purpose |
|---|---|
| `customers` | Bill-to, terms, credit limit, block flag |
| `products` | SKU, UoM, tax classification |
| `price_lists` / `contract_prices` | Effective-dated pricing |
| `billing_requests` | Incoming customer requests |
| `billing_request_lines` | Line items on request |
| `invoices` | Posted billing documents |
| `invoice_lines` | Priced + taxed lines |
| `invoice_numbers` | Sequence control |
| `payments` / `allocations` | AR (Phase 4) |
| `workflow_events` | Audit trail per case |
| `exceptions` | Human tasks |

**Phase 0 activity 0.2** produces the full ERD; seed data covers ~10 customers and ~50 invoice scenarios.

---

## 16. API sketch (Phase 1 target)

```http
POST /api/v1/billing-requests
Idempotency-Key: {uuid}
Content-Type: application/json

{
  "customer_id": "C-10001",
  "po_reference": "PO-7788",
  "currency": "USD",
  "lines": [
    { "product_id": "SKU-001", "quantity": 10, "unit": "EA" }
  ]
}
```

```http
201 Created

{
  "case_id": "BR-2026-00042",
  "status": "POSTED",
  "invoice_id": "INV-2026-001104",
  "pdf_url": "/artifacts/invoices/INV-2026-001104.pdf",
  "total_amount": 1250.00,
  "tax_amount": 125.00
}
```

---

## 17. LLM usage policy (modes)

| Mode | LLM used for | Phases |
|---|---|---|
| `AUTO` | Nothing on money path | 1–4 |
| `DRAFT_ONLY` | No post/send; preview only | 2+ |
| `LLM_ASSISTED_INTAKE` | Email → JSON request | 5 |
| `LLM_EXCEPTION_COMMS` | Draft clerk/customer emails | 5 (optional) |

**Never use LLM for:** tax rates, invoice totals, credit decisions, number assignment.

---

## 18. Milestones & demo schedule

| Milestone | Target (day) | Demo |
|---|---:|---|
| M0 — Design complete | 8 | Walkthrough ERD + API |
| M1 — Draft invoice | 26 | Postman: request → draft PDF data |
| M2 — Auto invoice emailed | 40 | End-to-end happy path |
| M3 — Portal live | 52 | Customer submits order in UI |
| M4 — Credit memo + recurring | 70 | Two billing types |
| M5 — GL export + pay link | 88 | Finance handoff package |

---

## 19. Risks & mitigations

| Risk | Impact | Mitigation | Extra days |
|---|---|---|---:|
| Pricing rules more complex than assumed | High | Phase 1 fixed price only; rules engine in 2.1 | +3–5 |
| Tax compliance | High | Start with static tables; external API in Phase 5 | +0 (phased) |
| Concurrency on invoice numbers | Medium | DB sequence + locking in 2.5 | Included |
| Unstructured email intake | Medium | Defer LLM to Phase 5; API first | Included |
| Scope creep (“full SAP”) | High | Strict phase gates per this doc | — |

---

## 20. Suggested team parallelization (if 2 developers)

| Track A (Core) | Track B (UI / integration) |
|---|---|
| Phases 0–2 modules | Portal mockups from day 10 |
| Orchestration + pricing | Phase 3 UI |
| Phase 4 AR | Phase 5 GL export |

**With 2 developers:** calendar time ≈ **50–55 working days** (~2.5–3 months).

---

## 21. Relationship to Manufacturer project

| Manufacturer | Billings |
|---|---|
| Analytic question → SQL → chart → PPT | Commercial request → price → invoice → PDF |
| LLM in presentation only | LLM in intake/comms only (optional) |
| ReWOO evidence `#E1…#E3` | Case evidence per workflow step |
| SQLite seed for demos | Master + transactional seed for billing scenarios |

Patterns reused: **orchestration folder**, **module per capability**, **seed data before agents/LLM**, **modes** for when AI is on.

---

## 22. Next steps (when you approve build)

1. Confirm Phase 0 scope (which request types in v1).
2. Choose database (PostgreSQL recommended) and API framework.
3. Run Phase 0 activities 0.1–0.7 (8 days).
4. Implement Phase 1 scaffold at `D:\Cursor\Billings` (separate from Manufacturer).

---

## 23. Document history

| Version | Date | Author | Notes |
|---|---|---|---|
| 1.0 | 2026-08-20 | Plan | Initial development plan with day estimates |
