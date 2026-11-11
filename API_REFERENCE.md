# API Reference

Base: `http://localhost:5001/api`. Auth is the cookie (`staff_token` / `admin_token`) or `Authorization: Bearer <jwt>`.
Health: `GET /health`, which returns `{ uptime, timestamp }`.

## Response envelope
```json
// success
{ "success": true, "message": "Student list retrieved successfully", "data": [...],
  "pagination": { "page": 1, "limit": 10, "total": 120, "totalPages": 12 },
  "meta": { "pagination": {...}, "filterOptions": {...}, "tabCounts": {...} } }
// error
{ "success": false, "message": "Validation Error", "errors": ["body.name: Required"] }
```
Status codes: `400` validation / duplicate (P2002) / FK (P2003), `401` no or invalid token, `403` permission, `404` not found (P2025), `429` rate limit, `500` server (Prisma details sanitised).

## Common list query params (operational modules)
`page`, `limit` (`'all'` = no pagination), `search`, `sortBy`, `sortOrder` (`asc|desc`), `tab` (`active|inactive|alumni|trash`), `currentYear`, plus **any field key as a column filter** (`?class=I,LKG&gender=Male`). Multi-values are comma-separated.

Exports: `GET …/export/excel` and `…/export/pdf` with the same filters plus selected columns.

---

## Admin side `/api/admin/*` (all use `authMiddleware + adminMiddleware` except auth)

| Mount | Endpoints |
|---|---|
| `/auth` | `POST /send-otp` (otpRateLimiter), `POST /verify-otp` (otpVerifyRateLimiter), `POST /logout`, `GET /me` |
| `/categories` | CRUD for institution categories |
| `/boards` | CRUD for boards (CBSE, State…) |
| `/institution-types` | `GET /`, `POST /`, `PUT /:id`, `DELETE /:id` |
| `/modules` | `GET /`, `GET /export/excel`, `GET /export/pdf`, `POST /`, `PUT /:id`, `PATCH /:id/status`, `PATCH /:id/restore`, `DELETE /:id`. Updating an icon propagates to all tenants and emits `master_modules_updated` / `institution_modules_updated` |
| `/academics` | `GET /overview`; CRUD for `/classes`, `/years`, `/intake-years`, `/streams`, `/programs`, `/sections`, `/subject-types` |
| `/institutions` | `GET /`, `GET /export/excel`, `GET /export/pdf`, `GET /:id`, `POST /` (creates the tenant, seeds modules, roles and the SUPERADMIN staff), `PUT /:id` |
| `/institutions/:institutionId/academic-config` | `GET /`, `PUT /` |
| `/institutions/:institutionId/attendance-config` and `/attendance-configs` | `GET`, `POST`, `PUT`, `DELETE` |
| `/staff` | admin users: `GET /`, `POST /`, `PUT /:id`, `PATCH /:id/toggle-status`, `DELETE /:id`, `PATCH /:id/restore` |
| `/upload` | `POST /image`, `DELETE /` |

---

## Staff side `/api/staff/*` (all routers do `router.use(authMiddleware, staffMiddleware)` except `/auth`)

### Auth `/auth`
| Method | Path | Notes |
|---|---|---|
| POST | `/login` | `{email,password}` zod-validated. ⚠ broken (no `passwordHash` column) |
| POST | `/send-otp` | `{mobile, via?}`, 6 per 5 min per IP+mobile. In test mode the response includes `otp` |
| POST | `/verify-otp` | `{mobile, otp}`, 5 per 5 min. Sets the `staff_token` cookie and returns `{token, staff, institution, modules}` |
| POST | `/logout` | clears the cookie |
| GET | `/me` | session restore: `{staff, institution, modules[]}` |

### Operational modules (pattern = permission `moduleKey`)

| Mount | moduleKey | Endpoints |
|---|---|---|
| `/students` | `students` | `GET /export/excel`, `/export/pdf`, `GET /`, `POST /bulk-import`(add), `/promote`(edit), `/bulk-alumni`(edit), `/bulk-status`(edit), `/bulk-restore`(delete), `POST /`(add), `GET /:id`, `PUT /:id`(edit), `DELETE /:id`, `POST /:id/restore`(delete) |
| `/staff` | `staff` | exports, `GET /`, `POST /bulk-import`, `/bulk-resigned`, `/bulk-retired`, `/bulk-restore`, `/bulk-delete`, `/:id/restore`, `POST /`, `PUT /:id`, `DELETE /:id`, `GET` and `PUT /:id/permissions` (`accesscontrol` read/edit) |
| `/subjects` | `subjects` | exports, `GET /`, `POST /bulk-import`, `/bulk-delete`, CRUD `/:id` |
| `/subject-types` | `subjects` | `GET /`, `GET /master`, CRUD |
| `/class-batches` | `classandbatch` | exports, `GET /`, `POST /bulk-delete`, CRUD |
| `/staff-subject-assignments` | `staff` | `GET /options`, `GET /staff/:staffId`, `POST /`, `DELETE /:id` |
| `/fees/bank-accounts` | `fees_bank_accounts` or `fees` | exports, list, bulk-delete, CRUD |
| `/fees/heads` | `fees_heads` or `fees` | exports, list, bulk-delete, CRUD |
| `/fees/structures` | `fees_structures` or `fees` | `POST /resolve`(read), exports, list, bulk-delete, CRUD |
| `/fees/concessions` | `fees_concessions` or `fees` | exports, `POST /export`, list, bulk-delete, `POST /`, `PUT /:id/approve`, `PUT /:id/reject` (⚠ uses `edit`, not `approve`), `PUT /:id`, `DELETE /:id` |
| `/fees/installments` | `fees_installments` or `fees` | same shape as concessions |
| `/fees/collection` | `fees_collection` / `fees_pay_now` / `fees` | exports, `GET /student/:studentId` (obligation), `GET /history/:studentId`, `POST /pay`(add), `GET /receipt/:paymentId`, `POST /cancel/:paymentId`(delete), `POST /settle`(edit), `GET /` |
| `/fees/reports` | fees reports | `GET /dashboard`, `/collections`, `/defaulters`, `/ledger/:studentId`, `/settlements` (+ excel/pdf each), `POST /settlements`, `POST /settlements/:id/reconcile` |
| `/modules/:moduleId/records` | `:moduleId` (dynamic) | `GET /`, `GET /:id`, `POST /`, `PUT /:id`, `DELETE /:id` (`requirePermission(null, action)` reads the module from params) |

### Configuration / settings routes (⚠ only `auth + staff`, NO permission check)

| Mount | Endpoints | Guard |
|---|---|---|
| `/institution` | `GET /profile`, `PUT /profile`, `PATCH`/`POST /logo`, `DELETE /logo` | none beyond login |
| `/modules` | `GET /`, `GET /institution`, `PUT /reorder`, `PUT /:moduleId/config` | none beyond login |
| `/modules/:moduleId/fields` | `GET /`, `POST /seed`, `PUT /bulk`, `POST /`, `PUT /reorder`, `PUT /:fieldId`, `DELETE /:fieldId` | `superadminMiddleware` (⚠ checks active staff only, not role) |
| `/roles` | `GET /`, `GET /:id`, `POST /`, `PUT /:id`, `DELETE /:id` | writes: `superadminMiddleware` |
| `/departments`, `/shifts` | CRUD | writes: `superadminMiddleware` |
| `/class-timings` | `GET /`, `GET /:id`, `POST /`, `POST /bulk`, `PUT /:id`, `DELETE /:id` | writes: `superadminMiddleware` |
| `/timetable` | `GET /classes`, `GET /slots`, `GET /class-staff-subjects/:classBatchId`, `GET /teacher-schedule/:staffId`, `POST /slots`, `DELETE /slots/:id` | none beyond login |
| `/attendance` | `GET /context`, `/subject-schedule`, `/roster`, `POST /submit`, `GET /monthly-register`, exports | access decided in `attendanceAccess.service.js` (class teacher / subject teacher / admin) |
| `/academic-config` | `GET /` | none |
| `/enrollment/forms` | `GET /`, `GET /:moduleId` | none |
| `/dashboard` | `GET /stats` | none |
| `/upload` | `POST /image?module=&subfolder=&prefix=`, `POST /attachment?…`, `DELETE /` `{url}` | none (⚠ delete has no tenant check) |

## Socket.io events (server → client, room `institution_<id>`)
`student_updated`, `staff_updated`, `staff_permissions_updated`, `role_permissions_updated`, `institution_modules_updated`, `institution_updated`, `departments_updated`, `shifts_updated`, `class_timings_updated`, `subjects_updated`, `subject_types_updated`, `class_batch_updated`, `staff_subject_assignment_updated`, `module_fields_updated`, `module_records_updated`, `academic_config_updated`, `master_academics_updated`, `bank_accounts_updated`, `fee_head_*`, `fee_structure_*`, `fee_installment_*`, `fee_concession_*`, `fee_payment_created`, `fee_payment_cancelled`, `fee:payment:settled`, `timetable_updated`, `attendance_updated`, `attendance_config_updated`.
Client → server: `join_institution(institutionId)`, `join_superadmin()`.
