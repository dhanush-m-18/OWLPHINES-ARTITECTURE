# OWLPHIN ERP: Developer Reference (dhanush notes)

My personal reference for the OWLPHIN codebase. It covers how the system works, why it is built this way, the conventions the senior developers follow, and what I found in the code-quality and security review.

> Snapshot: analysed on 03-11-2026 at commit `ba5c5bf` (branch `main`).

## What OWLPHIN is

OWLPHIN is a **multi-tenant School / PU-College ERP (SaaS)**. A single backend and database serve many institutions (tenants). There are three apps:

| App | Stack | Who uses it | Dev port |
|---|---|---|---|
| `backend/` | Node 24 + Express 4 + Prisma 5 + PostgreSQL (Neon) + Socket.io + Cloudflare R2 | Both portals | `5001` |
| `Staff_ERP/` | React 19 + Redux Toolkit 2 (RTK Query) + Tailwind 4 + Vite 8 | Each institution's staff (SUPERADMIN, teachers, accountants, etc.) | `5174` |
| `Admin_ERP/` | Same stack as Staff_ERP | OWLPHIN system admins (the platform owner) | `5173` |

## Index of this folder

| File | What's inside |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, request lifecycle, multi-tenancy, auth, real-time, storage, and the reasons behind each choice |
| [PROJECT_STRUCTURE.md](PROJECT_STRUCTURE.md) | Folder-by-folder map of all 3 apps |
| [CODE_CONVENTIONS.md](CODE_CONVENTIONS.md) | How the senior devs write code (backend + frontend) |
| [DATABASE.md](DATABASE.md) | Every Prisma model, table names (`@@map`), relations, and gotchas |
| [API_REFERENCE.md](API_REFERENCE.md) | All route groups, middleware chains, response format |
| [FEATURES.md](FEATURES.md) | Feature-by-feature description (students, staff, fees, attendance, etc.) |
| [WORKFLOWS.md](WORKFLOWS.md) | Step-by-step flows: login, add a module, fee collection, and others |
| [COMPLEX_IMPLEMENTATIONS.md](COMPLEX_IMPLEMENTATIONS.md) | Deep dives: dynamic fields, permission resolution, natural sort SQL, fee allocation, date search |
| [SECURITY_AND_PERFORMANCE.md](SECURITY_AND_PERFORMANCE.md) | **Review findings** (security bugs, logic bugs, perf), with severity and fixes |
| [CODE_TEMPLATES.md](CODE_TEMPLATES.md) | Copy-paste templates for new code that matches house style |
| [GLOSSARY.md](GLOSSARY.md) | Domain and code terms |

## How to run it locally

### 1. Prerequisites
- Node.js 20+ (verified on v24.12.0, npm 11.6.2)
- A PostgreSQL database (the team uses **Neon** serverless Postgres)
- Optional: Cloudflare R2 bucket (needed for image and file uploads), SMS gateway credentials (for real OTP)

### 2. Backend `.env` (`backend/.env`, git-ignored, so get the values from a senior dev)
```env
PORT=5001
DATABASE_URL=postgresql://USER:PASS@HOST/DB?sslmode=require
JWT_SECRET=<long random string>          # NEVER rely on the code fallback
JWT_EXPIRES_IN=7d
CLIENT_URL=http://localhost:5173,http://localhost:5174   # comma separated CORS allow-list
NODE_ENV=development

# OTP
OTP_TEST_MODE=true                       # dev only: OTP is always 123456 and no SMS is sent
DEFAULT_OTP_VIA=airtel
OTP_SMS_AIRTEL_URL=
OTP_WHATSAPP_URL=
OTP_SMS_GUPSHUP_URL=
SMS_API_USERNAME=
SMS_API_PASSWORD=

# Cloudflare R2
CLOUDFLARE_R2_PUBLIC_URL=
CLOUDFLARE_R2_S3_ENDPOINT=
CLOUDFLARE_R2_BUCKET_NAME=owlphin-staging
CLOUDFLARE_R2_ACCESS_KEY_ID=
CLOUDFLARE_R2_SECRET_ACCESS_KEY=
CLOUDFLARE_R2_REGION=auto
```

### 3. Frontend `.env` (optional)
```env
VITE_API_BASE_URL=http://localhost:5001/api   # default if unset
```

### 4. Commands
```bash
# backend
cd backend
npm ci                       # postinstall runs `prisma generate`
npx prisma db push           # sync schema to DB (see DATABASE.md: migrations folder is stale)
npm run prisma:seed          # seed master data (admin, modules, etc.)
npm run dev                  # nodemon src/server.js  -> http://localhost:5001/health

# staff portal
cd Staff_ERP
npm ci                       # postinstall copies TinyMCE assets to public/tinymce
npm run dev                  # http://localhost:5174

# admin portal
cd Admin_ERP
npm ci
npm run dev                  # http://localhost:5173
```

### 5. Logging in (dev)
- With `OTP_TEST_MODE=true`, use **mobile OTP login** with any seeded admin or staff mobile number and OTP `123456`.
- Staff email+password login is currently **broken** (see SECURITY_AND_PERFORMANCE.md, B-1).

## Verification I ran (03-11-2026)

| Check | Result |
|---|---|
| `npm ci` (all 3 apps) | OK |
| `npx tsc --noEmit -p tsconfig.app.json`, Staff_ERP | **0 errors** |
| `npx tsc --noEmit -p tsconfig.app.json`, Admin_ERP | **0 errors** |
| `npx prisma validate` | **valid** |
| `node --check` on every backend `.js` | **no syntax errors** |
| Backend boot (`node src/server.js`) | All modules import fine. It retries the DB connection 5x and then exits because I have no `.env` or DB locally. With a real `DATABASE_URL` it will start. |
| `oxlint` | Not run: the machine ran out of memory (OOM) when running checks in parallel |
| Automated tests | **None exist.** `npm test` points to `tests/multiTenant.test.js`, which is missing |

## Top 10 things to know before writing code
1. **Every query must be tenant-scoped**: `where: { institutionId: req.user.institutionId, isDeleted: false }`.
2. **Never hard delete.** Set `{ isDeleted: true, isActive: false }`.
3. Routes are `authMiddleware → staffMiddleware → requirePermission('<moduleKey>', '<action>') → validateRequest(zodSchema) → controller`.
4. Controllers wrap everything in `try { ... } catch (e) { return next(e) }` and respond with `sendSuccess` / `sendError`.
5. After a mutation, call `emitToInstitution(institutionId, '<entity>_updated')` so other staff screens refresh.
6. Raw SQL: **check `@@map` in the `.prisma` file** for the real table name (`tbl_shift` is singular but `tbl_roles` is plural).
7. Dates: backend parses with `parseFlexibleDate`; frontend shows `DD-MM-YYYY` via `formatDate`.
8. Frontend: one RTK Query `createApi` per feature, `refetchOnMountOrArgChange: true`, and register the slice in `app/store.ts` **and** `store/apiSlice.ts` (resetApiState).
9. UI: copy the layout of `ClassBatch.tsx` / `Subjects.tsx` exactly (one card, header, toolbar, table, pagination).
10. Edit forms: snapshot `initialFormData` and show `toast.info('No changes found')` when nothing changed.
