# Security, Correctness and Performance Review

Reviewed 03-11-2026 at commit `ba5c5bf`. Every item below was confirmed by reading the code (file and line refs included). Severity is my estimate. Please discuss with the senior devs before fixing, because some may be known or intentional.

## ✅ What is done well (safety levels already in place)
| Layer | Control |
|---|---|
| Transport/headers | `helmet()` on all responses |
| CORS | Explicit allow-list from `CLIENT_URL`, `credentials: true` |
| Session | JWT in **httpOnly** cookie, `secure` in prod, `sameSite=lax`. The frontend never stores tokens and clears legacy localStorage keys |
| Brute force | `express-rate-limit` on login (100/15 min), OTP send (6/5 min per IP+mobile), OTP verify (5/5 min) |
| AuthZ | Module-level RBAC + per-staff override (`requirePermission`), SUPERADMIN role protected from creation, rename and assignment via the staff portal |
| Tenancy | `institutionId` always taken from the JWT; fetch-then-mutate with `{id, institutionId}` |
| Input | zod on most create/update routes; dynamic field validator (type, length, choices, uniqueness) |
| SQL injection | Raw SQL uses `$queryRaw` tagged templates and `Prisma.sql`; `Prisma.raw` only with whitelisted identifiers; sort direction mapped to fixed `Prisma.sql` fragments |
| Errors | Central handler; Prisma internals hidden on 500s |
| Uploads | MIME allow-list, size limits (5 MB images, 10 MB docs), in-memory processing, sanitised storage keys, images re-encoded by sharp (which strips payloads) |
| Data | Soft delete everywhere; money stored as `Decimal(12,2)`; multi-row writes in transactions |
| Logout | Resets every RTK Query cache, which prevents cross-tenant cache leaks |

---

## 🔴 Critical / High

### S-1. Any staff can delete ANY tenant's files in R2 (cross-tenant)
`backend/src/staff_side/controllers/upload.controller.js` `deleteFile`, and `services/storage/r2.service.js` `deleteR2ObjectByUrl`.
No permission check, and no check that the key starts with `uploads/<module>/<req.user.institutionId>/`. The fallback regex accepts almost any URL path. A logged-in staff member of School A can delete School B's logos, photos and documents.
**Fix:** derive the key, then require `key.split('/')[2] === req.user.institutionId` (the admin side can skip this). Drop the loose regex fallback.

### S-2. `superadminMiddleware` does not check the role
`staff_side/middlewares/superadmin.middleware.js` only checks that the staff member exists and is active. Every staff member (teacher, clerk) can therefore **create, edit or delete roles, departments, shifts, class timings and module form fields**.
**Fix:** `if (staff.role?.name !== 'SUPERADMIN' && staff.role?.roleKey !== 'superadmin') return sendError(res, 403, …)`. Also check `staff.institutionId === req.user.institutionId`.

### S-3. Settings / config routes have no permission checks
| Route | Risk |
|---|---|
| `PUT /staff/institution/profile`, `PATCH/POST/DELETE /logo` | Any staff can rename the school, change branding and the logo |
| `PUT /staff/modules/reorder`, `PUT /staff/modules/:moduleId/config` | Any staff can rename, reorder or **disable** modules for the whole school |
| `POST /staff/timetable/slots`, `DELETE /slots/:id` | Any staff can edit the timetable |
| `/staff/upload/*` | Any staff can upload into any `module`/`subfolder` name |
**Fix:** add `requirePermission('settings', 'edit')`, `requirePermission('timetable', 'edit')` and so on, matching the module keys.

### S-4. Socket.io has no auth; tenant events leak PII
`config/socket.js`: `cors: { origin: '*' }`, and `join_institution(id)` accepts any id with no token check. `fee_payment_created` broadcasts the **full payment including the student record**. Anyone who knows or obtains an institution UUID (it appears in API responses, R2 URLs and logo paths) can listen to that school's live events from any website.
**Fix:** a `io.use()` middleware that parses the `staff_token`/`admin_token` cookie and verifies the JWT; on join, only allow `institution_${socket.user.institutionId}`; restrict CORS to `CLIENT_URL`; emit ids only, never full records.

### S-5. Stored XSS via "About Us"
`Staff_ERP/src/components/institution/AboutInstitution.tsx:141` uses `dangerouslySetInnerHTML={{ __html: content }}`. The backend stores `aboutUs` HTML unsanitised, and (per S-3) **any staff** can set it. Script in that content runs for every staff member who opens it. Cookies are httpOnly, but the attacker can still make authenticated API calls as the victim (for example a SUPERADMIN).
**Fix:** sanitise on the server (e.g. `sanitize-html` with an allow-list) and also run DOMPurify before rendering.

### S-6. OTP weaknesses
`services/otp.service.js`, `staff_side/services/auth.service.js`, `admin_side/services/auth.service.js`
- `Math.random()` is not a CSPRNG. Use `crypto.randomInt(100000, 1000000)`.
- **OTP never expires** (no timestamp). It's valid until used or replaced.
- Stored in plain text (could be hashed).
- The verify limiter is keyed by **IP + mobile**, so rotating IPs bypasses it. There's no per-account failed-attempt counter or lockout.
- `OTP_TEST_MODE=true` makes `123456` valid for **every** account and returns the OTP in the response. A wrong production env var would be a full auth bypass. Add a startup guard: refuse to start if `NODE_ENV==='production' && OTP_TEST_MODE`.
- User enumeration: send-otp returns 404 "No active staff account…" or 403 "Resigned".
- Staff `mobile` is not unique across tenants. `updateMany` writes the same OTP to all matching staff, and verify logs into an **arbitrary** `findFirst` match.

### S-7. JWT secret fallback
`config/env.js`: `JWT_SECRET || 'fallback_secret_key_change_in_production'`. If the env var is missing in a deploy, anyone can forge SYSTEM_ADMIN tokens. **Fix:** throw at startup if it's missing.

---

## 🟠 Medium

| ID | Issue | Where | Fix |
|---|---|---|---|
| S-8 | Token accepted from `?token=` query string (leaks into logs, history, Referer) | both `auth.middleware.js` | Remove query-token support |
| S-9 | Both auth middlewares fall back to a generic `token` cookie; same secret for admin and staff | `auth.middleware.js`, `jwt.js` | Separate secrets or an `aud` claim per portal |
| S-10 | Stateless 7-day JWT; role, permission and deactivation changes don't apply until expiry (`requirePermission` trusts `roleName` from the token; `staffMiddleware` doesn't re-check `isActive`) | jwt / middlewares | Shorter TTL plus refresh, or check `isActive`/role from the DB (cache it) |
| S-11 | Fee concession and installment **approve/reject use `edit` permission**, not `approve`, so the creator can approve their own concession | `feeConcession.routes.js`, `feeInstallment.routes.js` | Use `requirePermission([...], 'approve')` and block self-approval |
| S-12 | `bankAccountId` in fee payment (and probably fee head and structure) isn't verified to belong to the tenant | `feePayment.controller.js:772` | `findFirst({id, institutionId})` before use |
| S-13 | Teaching-staff scope applies only to **reads**; `PUT/DELETE /students/:id` works on any student in the tenant if the role has edit/delete | `student.controller.js` | Apply `buildTeachingStaffWhere` in mutation lookups as well |
| S-14 | `limit` is unbounded (`limit=1000000`), and exports have no cap, so it's a DoS vector | list controllers | Clamp `limit` to ≤ 500 |
| S-15 | `express.json({limit:'10mb'})` globally | `app.js` | 1 MB default; bigger only on bulk-import routes |
| S-16 | Uploads trust the client MIME type; attachments keep the original extension (`.html` with a `text/plain` MIME, served from the R2 public domain) | `upload.middleware.js` / controller | Derive the extension from a MIME allow-list; sniff magic bytes; set `Content-Disposition: attachment` for documents |
| S-17 | CSRF: cookie auth + `sameSite=lax` + CORS. Lax blocks cross-site POST from forms, so the risk is mostly mitigated; still no CSRF token | — | Consider `sameSite=strict` or a double-submit token |
| S-18 | `zod .passthrough()` everywhere; safe only while controllers destructure fields | validators | Never spread `req.body` into Prisma `data` |
| S-19 | `getDynamicRecords` field lookup doesn't filter `isDeleted` | `record.service.js` | Add `isDeleted:false` |

---

## 🐞 Functional bugs found

| ID | Bug | Evidence |
|---|---|---|
| B-1 | **Staff email/password login is broken**: `loginStaff` compares against `staff.passwordHash`, but `Staff` has no such column, so bcrypt throws and the user gets a 500 | `staff_side/services/auth.service.js:36` vs `prisma/schema/staff.prisma` |
| B-2 | **Generic module record search is broken**: raw SQL queries `module_records`, but the table is `tbl_module_records`. Any search on a dynamic module returns a 500 (this is exactly the rule AGENTS.md warns about) | `staff_side/services/record.service.js:92,103,114` vs `moduleRecord.prisma @@map` |
| B-3 | `requirePermission` denial throws `ReferenceError: targetKey is not defined`, so the user gets a 500 instead of a 403 | `permission.middleware.js:48` |
| B-4 | Audit fields are always empty: code uses `req.user.id` and `req.user.name`, but the staff JWT has neither (`staffId` only) | `timetable.controller.js:545+`, `feePayment.controller.js:876+`, `staff.controller.js:1203` |
| B-5 | `prisma/migrations` is stale (old table names, `passwordHash`), so `prisma migrate deploy` on a fresh DB creates the wrong schema | `migrations/20260903115538_init` |
| B-6 | `npm test` points to the non-existent `tests/multiTenant.test.js` | `backend/package.json` |
| B-7 | Attendance unique key includes nullable columns, so Postgres allows duplicate DAY_WISE sessions | `attendanceSession.prisma` |
| B-8 | `parseFlexibleDate('31-02-2020')` silently becomes 02-03-2020 | `utils/date.js` |
| B-9 | Calendar events are stored only in localStorage (per browser, not shared, wiped on logout) | `features/calendarEvents/calendarEventsSlice.ts` |

## 💰 Fee-logic integrity

| ID | Issue |
|---|---|
| L-1 | Receipt number is generated **outside** the transaction from the "latest by createdAt", so concurrent payments get the same number and one fails with a P2002 error. Use a per-tenant sequence row updated inside the transaction (`SELECT … FOR UPDATE`) or retry on P2002 |
| L-2 | Past allocations and pending amounts are read outside the transaction, so two concurrent payments can both fill the same pending head. Use a Serializable transaction or lock the student row |
| L-3 | Money math uses JS floats (`parseFloat`, `Number`), so rounding drift is possible (0.1+0.2). Use `Prisma.Decimal` or integer paise |
| L-4 | All approved concessions get marked "applied" on **any** payment, even a ₹1 partial one |
| L-5 | Overpayment has no upper bound; the excess silently goes to the first head |

## ⚡ Performance

| ID | Issue | Fix |
|---|---|---|
| P-1 | Sorting by a custom or year-wise field loads **all** matching rows into memory, then slices | Sort in SQL (`customFields->>'key'`) |
| P-2 | `generate4DigitStudentKey` / staffKey loads every key per create | Counter table or `MAX()` + retry |
| P-3 | Each list request runs 5 count queries plus filter-option queries | Cache tab counts briefly; composite index `[institutionId, isDeleted, isActive, isAlumni]` |
| P-4 | `requirePermission` does 2–3 DB queries per request, uncached | Small LRU cache keyed by staff+module, invalidated by socket/permission updates |
| P-5 | Search uses `contains` / `ILIKE '%q%'` and `data::text LIKE`, which can't use btree indexes | `pg_trgm` GIN indexes on name columns |
| P-6 | Whole-file Excel/PDF export generated in memory | Stream with ExcelJS streaming writer |
| P-7 | Frontend `strict:false`, 2,000-line page components | Split pages into hooks plus sub-components over time |

## 🧹 Code quality summary
- Consistent and readable, with strong conventions (response helpers, tenant filter, soft delete, step comments).
- Weaknesses: very large files, duplicated helpers per module (contradicts the reusability rule), duplicated middlewares across sides, no tests, loose TS, `dist/` committed, stale migrations.
- Type-check: Staff_ERP ✅ 0 errors, Admin_ERP ✅ 0 errors. `prisma validate` ✅. Backend syntax ✅.

## Suggested fix order
1. S-2, S-3, S-1 (authorization holes: small, high impact)
2. S-4 socket auth, S-5 sanitise HTML, S-7 JWT secret guard
3. B-1, B-2, B-3 (quick bug fixes)
4. S-6 OTP hardening (expiry, crypto RNG, attempt counter, test-mode guard)
5. L-1, L-2, L-3 fee transaction integrity
6. Medium items, then performance
