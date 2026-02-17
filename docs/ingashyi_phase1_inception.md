# Ingashyi Platform — Phase 1 Inception Pack

Date: 2026-02-17  
Prepared by: Development Partner (Greenfield build)
Primary Stack: Laravel 11 + PostgreSQL + Filament v3

## 1) Architecture Diagram (Components + Data Flow)

```text
┌───────────────────────────┐
│     Web Admin UI          │
│ (Admin/Mobiliser/Leaders) │
└──────────────┬────────────┘
               │ HTTPS
┌──────────────▼────────────┐
│     Laravel Backend API    │
│ - Auth + Session           │
│ - RBAC/Policy Scope        │
│ - Membership/Governance    │
│ - Cards + QR Scan          │
│ - Savings + Wallet Ledger  │
│ - Loans + Voting Engine    │
│ - Alerts + Reports         │
└───────┬───────────┬────────┘
        │           │
        │ SQL       │ Queue/Scheduler
┌───────▼───────┐   ▼
│ PostgreSQL    │  Laravel Scheduler/Queue jobs
│ - normalized  │  - generate alerts daily
│ - constraints │  - resolve loan vote timeout
└───────┬───────┘
        │
        │ file export artifacts
┌───────▼───────────────┐
│ Object/File Storage   │
│ - card print PDFs     │
│ - report exports      │
└───────────────────────┘
```

### Security and Integrity Controls
- RBAC and scope are enforced server-side for every query and mutation.
- QR payload carries only `card_uuid` (and optional signature), never member IDs.
- Savings uniqueness enforced in DB using unique index on `(member_id, week_key)`.
- Wallet transactions are append-only; corrections are reversal/adjustment entries.
- Sensitive actions are recorded in activity logs (issuance, revocation, export, reversals, loan decisions).

## Technology Baseline (Approved)
- Backend framework: **Laravel 11** (PHP 8.3+).
- Database: **PostgreSQL** with strict constraints/indexes.
- Admin/field UI: **Filament v3** (or equivalent Laravel admin stack if approved).
- AuthN/AuthZ: Laravel auth + policies + middleware for server-side RBAC scoping.
- Jobs: Laravel Queue + Scheduler for daily alerts and loan-voting timeout automation.
- Caching: Redis for dashboard/report metric caching.

## 2) Proposed MVP Database Schema / ERD (Final Draft for Approval)

### Core Entities
- `users` (platform users with role assignments).
- `roles`, `user_roles` (Admin, Mobiliser, Zone Leader, Group Leader, optional Member).
- `provinces`, `districts`, `sectors`, `cells`, `villages` (seeded location references).
- `zones` (linked to administrative units).
- `groups` (belongs to zone + optional admin unit linkage).
- `group_memberships` (historical member→group assignment periods).
- `members` (profile, status, location, youth fields, optional sensitive fields).
- `dependents` (optional member dependents).
- `leadership_assignments` (zone/group leaders with start/end dates).

### Identity + Scan Entities
- `cards` (separate from wallets) with lifecycle statuses:
  `generated | printed | issued | suspended | replaced | revoked`.
- `card_scans` (scan audit with context, actor if authenticated, and metadata).

### Operations + Financial Entities
- `wallets` (one per member).
- `wallet_transactions` append-only ledger with types:
  `saving | loan_disbursement | repayment | reversal | adjustment`.
- `savings` (weekly entries tied to `card_id`, `recorded_by`, `week_key`).
- `attendances` (scan-based attendance by meeting/session, used in vote eligibility).

### Loans + Governance Entities
- `loan_requests` with state machine:
  `requested -> voting -> approved -> disbursed -> repaid` OR `rejected`.
- `loan_votes` immutable votes (`yes|no`) by eligible members.
- `alerts` with `open|resolved`, severity, scoped visibility.
- `activity_logs` for auditable critical actions.

### Mandatory Constraints (MVP)
- `UNIQUE (member_id, week_key)` on `savings`.
- `UNIQUE (card_uuid)` on `cards`; UUID v4 required.
- Partial unique to prevent >1 active issued card per member.
- `UNIQUE (loan_request_id, voter_member_id)` on `loan_votes`.
- Borrower cannot vote constraint (`loan_votes.voter_member_id != loan_requests.borrower_member_id`).
- Immutable ledger guardrails (no update/delete on `wallet_transactions` at app policy level + DB permissions strategy).

## 3) UI Sitemap and Key Workflow Summary

## Primary Navigation
- Dashboard
- Members
- Zones
- Groups
- Cards
- Savings
- Loans
- Alerts
- Reports
- Settings (Admin)

## Key Screens
1. **Members List + Filters** (zone/group/status/card-status/youth tags).
2. **Register Member** (duplicate checks + optional dependents/profiling).
3. **Cards Workbench** (generate, print queue, issue, suspend, replace).
4. **Scan Endpoint UX**
   - Public: valid/invalid minimal response.
   - Authenticated: scoped details + quick actions.
5. **Savings Scan-first Capture** (scan → verify active card → save).
6. **Loan Request + Voting Board** (attendance-gated eligibility, progress meter).
7. **Alerts Action Required** (role-scoped queue + resolution notes).
8. **Reports Export Center** (admin-controlled exports).

## Workflow Commitments
- Registration enforces required Rwanda-localized fields.
- Card lifecycle is first-class and traceable.
- Savings can only be recorded against active issued cards.
- Vote outcomes are automatic by threshold/timebox rules.
- All critical actions produce audit log records.

## 4) Delivery Timeline + Resourcing Plan

## Team Shape (Laravel Delivery)
- 1 Product/Technical Lead
- 1 Backend Engineer
- 1 Full-stack Engineer (UI + API integration)
- 1 QA Engineer (shared)

## Milestone Plan (10 weeks)
1. **Week 1–2:** Architecture, schema, auth, RBAC scoping.
2. **Week 3–4:** Membership, zones/groups, import/export baseline.
3. **Week 5–6:** Cards, QR scan endpoint, card lifecycle + print output.
4. **Week 7:** Savings + wallet ledger + weekly uniqueness.
5. **Week 8:** Loans + voting engine + attendance gating + timeout job.
6. **Week 9:** Dashboards, alerts engine, reports/export controls.
7. **Week 10:** UAT hardening, performance checks, docs, deployment package.

## 5) Cost Breakdown Template (Aligned to Deliverables)

- D1 Repository + project setup: **10%**
- D2 Migrations/seeders/reference data: **15%**
- D3 MVP modules + role dashboards: **40%**
- D4 Card print/export + replacement/revocation: **10%**
- D5 Documentation + admin guide + schema dictionary: **10%**
- D6 Automated tests (required scenarios): **10%**
- D7 Deployment package + release notes: **5%**

> Final commercial values to be inserted after hosting/SLA assumptions are confirmed.

## 6) Risk Register + Mitigation Plan

| Risk | Impact | Likelihood | Mitigation |
|---|---|---:|---|
| Poor source data quality during onboarding | High | Medium | Duplicate detection + import validation with partial-failure reporting. |
| Scope creep before MVP stabilization | High | High | Formal change control and release gating against TOR acceptance criteria. |
| Performance degradation on dashboards | Medium | Medium | Caching + pre-aggregations + pagination from sprint 1. |
| Misconfigured role scoping causing data leakage | High | Medium | Policy tests per role and tenant-scope query guards. |
| Financial integrity violations from manual edits | High | Low | Append-only ledger design + reversal-only corrections + audit logs. |
| Field connectivity constraints for scan workflows | Medium | Medium | Offline/PWA planned for Phase 2; efficient web flows for MVP. |

## 7) Immediate Next Actions (Kickoff)
1. Approve this inception pack (architecture/schema/sitemap/timeline/risk).
2. Confirm database engine choice (PostgreSQL preferred).
3. Confirm Phase 1 auth model (email/password + optional OTP later).
4. Confirm acceptance test data set for UAT.
5. Start Milestone 1 implementation on approved branch.
