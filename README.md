# 🤝 BénévolApp

A full-stack volunteer management platform connecting vulnerable beneficiaries with RSA-recipient volunteers under admin supervision. Built as a monorepo with a Next.js web app, an Expo mobile app, and an Express backend — all sharing a single PostgreSQL database through Supabase.

> This repository is public for presentation purposes only. The source code is hosted in a private repository.

---

## 📖 About the Project

BénévolApp is built around a model of **horizontal mutual aid**: volunteers are themselves in precarious situations (RSA recipients), and beneficiaries receive concrete, regular help from them. Administrators coordinate everything in less than 15 minutes a day.

The platform covers the full lifecycle of a volunteer engagement:

- **Account creation & admin validation** — 3 roles (admin, bénévole, bénéficiaire), approval workflow with schedulable appointments
- **Mission management** — draft → published → completed/cancelled, recurring schedules, waitlist with automatic PL/pgSQL cascade
- **QR-code attendance** — beneficiary permanent QR, HMAC anti-fraud tokens, offline-first MMKV queue, 6-digit fallback
- **Hours export** — CSV and signed PDF attestation for the RSA advisor
- **Admin dashboard** — real-time alerts (🔴🟠🟡) via Supabase Realtime subscriptions
- **Human inbox** — separated from system notifications (`is_human` boolean), direct admin → user messaging
- **Managed accounts** — admin proxy for non-autonomous beneficiaries, full audit trail
- **GDPR compliance** — JSON personal data export, right to erasure (anonymisation), append-only audit logs
- **Async fraud detection** — shared device fingerprints, rapid consecutive check-ins, IP analysis → `audit_logs`

### Key Architectural Decision

Supabase is called **directly** from web and mobile (RLS + Auth native). The Express backend is reserved solely for operations requiring `service_role`: PDF/CSV exports, fraud analysis, and GDPR anonymisation. This keeps Row Level Security as the single source of truth across all 18 tables and avoids duplicating access-control logic in a middleware layer.

---

## 🧰 Tech Stack

### Monorepo

| Technology | Usage |
|---|---|
| **Turborepo 2.5.0** | Monorepo orchestration, parallel pipelines |
| **pnpm 10.33.0** | Package manager (workspaces) |
| **TypeScript 5.x** | Strict mode across all packages |
| **GitHub Actions** | CI: install, prisma generate, typecheck, lint, security audit |

### Web — Next.js

| Technology | Version | Usage |
|---|---|---|
| **Next.js** | 16.2.2 | App Router, Server Components, API Routes |
| **React** | 19.2.4 | UI rendering |
| **Tailwind CSS** | v4 | Styling (utility-first, dark mode) |
| **TanStack Query** | v5 | Server state management |
| **React Hook Form + Zod** | RHF 7.72.1 | Forms and validation |
| **Supabase JS** | — | Auth + database client (RLS) |
| **Sentry** | — | Error monitoring |

### Mobile — Expo / React Native

| Technology | Version | Usage |
|---|---|---|
| **Expo SDK** | 54.0.33 | React Native framework |
| **React Native** | 0.81.5 | Native UI rendering |
| **Expo Router** | ~4.0.21 | File-based navigation |
| **NativeWind** | ^4.1.23 | Tailwind CSS for React Native |
| **React Hook Form + Zod** | — | Forms and validation |
| **Supabase JS** | — | Auth + database client |
| **expo-secure-store** | — | Secure session token storage |
| **react-native-mmkv** | ^4.3.0 | Offline queue for QR check-ins |
| **react-native-qrcode-svg** | ^6.3.21 | Beneficiary QR display |
| **expo-camera** | ~17.0.10 | QR code scanning |
| **expo-file-system + expo-sharing** | — | CSV/PDF export and native share sheet |
| **Sentry** | — | Error monitoring |

### Backend — Express

| Technology | Version | Usage |
|---|---|---|
| **Node.js** | 22 | JavaScript runtime |
| **Express** | 5.2.1 | HTTP server / REST API |
| **Prisma** | 7.6.0 | ORM + migrations (PostgreSQL) |
| **tsx** | — | Direct TypeScript execution (no build step) |
| **PDFKit** | ^0.15.0 | PDF generation for RSA attestation |
| **Helmet + express-rate-limit** | — | HTTP security headers + rate limiting |

### Database & Infrastructure

| Technology | Usage |
|---|---|
| **PostgreSQL 15** | Primary database |
| **Supabase** | Auth, RLS, Realtime — hosted EU West (Ireland, GDPR) |
| **Row Level Security** | 18 tables × 3 roles × multi-tenant — deny-all by default |
| **Vercel** | Web deployment |
| **Railway** | Backend deployment (Dockerfile + tsx) |
| **Expo EAS Build** | Mobile APK/IPA builds |

---

## 🗂️ Database — 18 Tables

| Table | Role |
|---|---|
| `organisations` | Multi-tenant root — all FKs anchor here |
| `profiles` | Public profile data (id mirrors auth.users.id) |
| `profiles_sensitive` | GDPR-isolated: email, phone, address, DOB, RSA number |
| `missions` | Status lifecycle: draft / published / cancelled / completed |
| `mission_schedules` | Recurrence rules and time slots |
| `mission_applications` | Applications + waitlist position |
| `mission_interventions` | Actual instances: planned / done / missed |
| `pointages` | Check-in/out — UNIQUE constraint per intervention |
| `beneficiary_qr` | Permanent beneficiary QR token (rotatable by admin) |
| `attendance_tokens` | Single-use HMAC anti-fraud tokens |
| `disponibilites` | Volunteer weekly availability |
| `types_service` | Service catalogue |
| `adresses` | Addresses for profiles and missions |
| `validation_appointments` | Admin account validation appointments |
| `mission_followups` | Admin follow-up notes per mission |
| `admin_notes` | Private admin notes (XOR: profile OR mission target) |
| `notifications` | System notifications + human inbox (`is_human` boolean) |
| `audit_logs` | Full action trace — append-only via service_role |

Row Level Security is enforced on every table using `SECURITY DEFINER` helper functions (`get_my_role()`, `get_my_org_id()`), keeping policy logic out of application code and preventing RLS recursion on the `profiles` table.

---

## 🔐 Backend API Endpoints

The Express backend exposes a small set of routes that require `service_role` access — operations that cannot be safely performed client-side.

### Export — auth via `X-Export-Secret` header

| Method | Route | Description | Rate limit |
|---|---|---|---|
| `GET` | `/api/export/heures/:benevole_id` | Volunteer hours as CSV | 20 req / 15 min |
| `GET` | `/api/export/pdf/:benevole_id` | RSA attestation as PDF | 20 req / 15 min |

### Fraud Detection

| Method | Route | Description | Rate limit |
|---|---|---|---|
| `POST` | `/api/fraud/check` | Async fraud analysis → writes to `audit_logs` | 60 req / 15 min |

### GDPR — auth via Bearer JWT

| Method | Route | Description | Rate limit |
|---|---|---|---|
| `POST` | `/api/rgpd/anonymize` | Anonymise profile + delete auth account | 5 req / hour |

---

## 🚢 Deployment

| Component | Platform | Status |
|---|---|---|
| Web (Next.js) | **Vercel** | ✅ Live |
| Backend (Express) | **Railway** | ✅ Live |
| Database / Auth | **Supabase** — EU West (Ireland) | ✅ Live |
| Mobile APK | **Expo EAS Build** | ⏳ Build not launched |

---

## 🎯 MVP Success Criteria

| Actor | Criterion |
|---|---|
| Volunteer | ≥ 1 mission/month, exportable hours history |
| Beneficiary | Regular concrete help, relational continuity |
| Admin | < 15 min/day, zero unhandled cancellations |
| Project | 50-person pilot, ≥ 90% missions filled |
| Technical | < 2–3s response time, offline-first mobile, GDPR compliant |

---

## 📄 Licence

Private project — source code is not publicly available.
