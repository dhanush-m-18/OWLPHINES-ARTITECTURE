# Architecture

## 1. Big picture

```
 ┌──────────────┐        ┌──────────────┐
 │  Admin_ERP   │        │  Staff_ERP   │      React 19 SPAs (Vite)
 │ (platform    │        │ (per-tenant  │      RTK Query per feature
 │  owner)      │        │  staff)      │      httpOnly cookie auth
 └──────┬───────┘        └──────┬───────┘
        │ REST /api/admin/*     │ REST /api/staff/*      + socket.io (rooms)
        └───────────┬───────────┘
                    ▼
        ┌──────────────────────────┐
        │  Express backend (5001)  │  helmet · cors allow-list · cookie-parser
        │  routes → middlewares →  │  rate-limit (auth/OTP) · zod validation
        │  controllers → services  │  central errorHandler
        └──────┬─────────┬─────────┘
               │         │
     Prisma 5  ▼         ▼  AWS S3 SDK
   ┌─────────────────┐  ┌───────────────────┐
   │ PostgreSQL      │  │ Cloudflare R2     │  images → sharp → WebP ≤100KB
   │ (Neon)          │  │ uploads/<module>/ │  docs stored as-is
   └─────────────────┘  │ <instId>/...      │
                        └───────────────────┘
   External: SMS/WhatsApp OTP gateways (Airtel / Gupshup / WhatsApp) via axios
```

## 2. Multi-tenancy model (most important concept)

- **Shared database, shared schema, row-level tenant column.** Every operational table has `institutionId`.
- The tenant id comes **only from the JWT** (`req.user.institutionId`), never from the request body. This is what stops one school from reading another's data.
- **Master tables** (`tbl_classes`, `tbl_years`, `tbl_streams`, `tbl_sections`, `tbl_programs`, `tbl_intake_years`, `tbl_master_subject_types`, `tbl_system_modules`, `tbl_boards`, categories, types) are global. Only the Admin portal writes them.
- Institutions then pick from the master data. For example, `InstitutionAcademicConfig` stores which classes, sections and streams a tenant uses for each academic year (`String[]` arrays).

**Why this design:** it's cheap to operate (one DB, one deploy) and easy to onboard a tenant (insert one row, then seed modules). The trade-off is that one missing `institutionId` filter leaks data across schools. That's why the house rule is so strict.

## 3. Two-portal, two-side backend

Backend code is split by audience:

```
src/admin_side/   → /api/admin/*   (userType SYSTEM_ADMIN)
src/staff_side/   → /api/staff/*   (userType STAFF / MANAGEMENT)
src/services, utils, config, middlewares → shared
```

Each side has its **own** `auth.middleware.js` and `validate.middleware.js`. They're nearly identical, but the auth middlewares read different cookies (`admin_token` / `staff_token`).

**Why:** the two audiences have totally different powers. Admin manages tenants and master data. Staff manage data inside one tenant. Keeping the routers separate makes it hard to accidentally expose an admin endpoint to staff.

## 4. Request lifecycle (staff side)

```
HTTP request
 → helmet, cookieParser, cors(allow-list from CLIENT_URL), express.json(10mb)
 → app.use('/api', routes)
   → /staff/<resource> router
     → authMiddleware        token from cookie staff_token | token | ?token= | Bearer
                              → jwt.verify → req.user = { staffId, institutionId, roleId, roleName, userType }
     → staffMiddleware       userType must be STAFF or MANAGEMENT
     → requirePermission(moduleKey, action)
                              → resolveModulePermission(): InstitutionModule enabled?
                                 → StaffModulePermission override  ‖  RoleModulePermission
                                 → SUPERADMIN / MANAGEMENT bypass everything
                              → sets req.instModule, req.modulePermissions
     → validateRequest({ body, query, params })  (zod, replaces req.body with parsed)
     → controller (try/catch → next(err))
        → prisma queries scoped by institutionId
        → emitToInstitution(instId, '<event>')
        → sendSuccess(res, code, msg, data, extra)
 → errorHandler  (Zod → 400, JWT → 401, P2002 → 400 duplicate, P2025 → 404, P2003 → 400, Prisma → sanitized 500)
```

## 5. Authentication

| Portal | Methods | Token |
|---|---|---|
| Admin | Mobile OTP only | JWT `{adminId, mobile, name, userType:'SYSTEM_ADMIN'}` in cookie `admin_token` |
| Staff | Mobile OTP (+ email/password, currently broken) | JWT `{staffId, institutionId, roleId, roleName, userType:'STAFF'}` in cookie `staff_token` |
| Management (multi-institution) | Token generator exists (`generateManagementToken`), no login route yet | `userType:'MANAGEMENT'` bypasses all permissions |

- OTP is stored **on the user row** (`staff.lastOtp`, `admin.lastOtp`). There is no separate OTP table, by design. It's cleared on successful verify.
- Cookies are `httpOnly`, `sameSite=lax`, `secure` in production, with a 7-day lifetime. The frontend **never stores the token**. `AuthContext` calls `/staff/auth/me` on mount to restore the session and sets `token='cookie_authenticated'` as a flag.
- **Why cookies over localStorage:** XSS can't read httpOnly cookies. The frontend actively wipes legacy `localStorage` token keys on load.

## 6. Authorization (RBAC + per-staff override)

```
Module (system, global)  ──1:N──▶ InstitutionModule (tenant enables/renames/reorders)
                                          │
                     ┌────────────────────┼─────────────────────┐
                     ▼                                          ▼
         RoleModulePermission                       StaffModulePermission
       (role → read/add/edit/delete/approve/report)  (per staff override; wins over role)
```
Resolution order (`permission.service.js`): **SUPERADMIN role / MANAGEMENT → all true**, then **staff override**, then **role permission**, then **all false**. A module the tenant disabled means 403 for everyone except the bypass roles.

The frontend mirrors this. Login and `/me` return `modules[]` with `permissions`. The sidebar shows only `read:true` modules, and `useModulePermission('students')` hides buttons. **The frontend checks are UX only. The backend `requirePermission` is the real gate.**

A second, data-level restriction applies to **teaching staff**. `getTeachingStaffClassBatches()` limits student lists to the class/stream/section combos they teach (class teacher or subject assignment). Admins and non-teaching staff are unrestricted (it returns `null`).

## 7. Dynamic modules and dynamic fields (core differentiator)

- Admin creates **system modules** (`tbl_system_modules`: `moduleKey`, default icon, path, parent).
- Each institution gets **InstitutionModule** rows (it can rename, re-icon, reorder, enable or disable them).
- Each InstitutionModule has **InstitutionModuleField** rows: the form definition (`fieldKey`, `label`, `fieldType`, `required`, `isUnique`, `visible`, `options` JSON).
  - `isSystemField: true` fields map to real columns (e.g. `StudentInfo.name`).
  - Custom fields live in the entity's `customFields` **JSONB** column (GIN-indexed on staff and students).
  - Defaults come from `config/defaultFields.config.js` (`DEFAULT_MODULE_FIELDS[moduleKey]`). They're auto-seeded on first access (`ensureDefaultFields`).
- Fully generic modules (no dedicated table) store rows in `ModuleRecord.data` (JSONB).
- The frontend renders forms with `DynamicFormField` and table cells with `DynamicTableCell` based on `fieldType`.

**Why:** every school wants slightly different student and staff forms. JSONB custom fields let a school add "Aadhaar No" or "Blood Group" without a schema migration.

## 8. Real-time sync (Socket.io)

- The server keeps a single `io`. Clients `emit('join_institution', id)` and join room `institution_<id>`. Admins join `superadmin_room`.
- After mutations, controllers call `emitToInstitution(id, 'student_updated')` and similar events.
- In the frontend, `app/store.ts` maps each event to `xxxApiSlice.util.invalidateTags([...])`, so every open screen in that school refetches. `AuthContext` also refreshes the profile and sidebar on permission or module events.
- **Why:** several staff edit the same data at the same time (fees counter, admin office), and screens stay consistent without polling.
- ⚠️ The socket server has **no authentication** and `cors: '*'` (see SECURITY S-4).

## 9. File storage (Cloudflare R2)

- `multer.memoryStorage()` keeps the file in memory. Images go through `sharp`, which compresses progressively to WebP ≤100 KB. Documents (pdf, docx, xlsx, csv, txt) are stored as-is.
- Key format: `uploads/<module>/<institutionId>/<subfolder>/<prefix>-<timestamp>-<rand>.<ext>` (`generateStorageKey` sanitises every segment).
- The DB stores **only the public URL**, never base64.
- When a file is replaced or removed, call `deleteR2ObjectByUrl(oldUrl)` to avoid orphans.

## 10. Frontend architecture

- **Routing is data-driven.** `App.tsx` has only `/login` plus `/:moduleParam` and `/:parent/:moduleParam`. `ModuleRecordsWrapper` finds the module in the user's `modules[]` (by id, key, path or name) and renders the component from `config/moduleRegistry.ts` (`CUSTOM_MODULE_REGISTRY[moduleKey]`). A module that's not assigned redirects to `/dashboard`.
- **State:** server state lives only in RTK Query (one `createApi` per feature, each with `credentials:'include'`). Local UI state uses `useState`. Contexts: `AuthContext`, `InstitutionContext` (branding and theme colours), `ThemeContext` (dark mode), `ToastContext`.
- **Theming:** tenant branding sets the CSS variables `--primary-color` and `--secondary-color`. Utility classes (`btn-primary`, `text-theme-primary`, and so on) use `color-mix()` for shades.
- **Logout and user switch:** `staffApi.util.resetApiState()` resets **every** feature slice, so no cached data from the previous tenant leaks.

## 11. Why things are the way they are (design rationale summary)

| Decision | Reason |
|---|---|
| Row-level `institutionId` | Simple SaaS ops, one DB |
| Soft delete everywhere | Schools need "Trash" and restore. Fee/attendance history must not lose referential data |
| JSONB `customFields` | Per-school form customisation with no migrations |
| `StudentInfo` + `StudentYearWise` split | Class, section and stream change every year, and history must be kept for promotions, fees and attendance |
| Strings for class/section names (not FKs) | Master data is chosen per year via `InstitutionAcademicConfig` arrays. Denormalised for flexibility (trade-off: renames don't cascade) |
| OTP on user row | Fewer tables. Only one active OTP per user is needed |
| RTK Query per feature + socket invalidation | Feature isolation and real-time freshness |
| `refetchOnMountOrArgChange: true` | Always fresh data plus a skeleton on module switch, which matters more than cache hits in an ERP |
| One controller file per entity with helpers inline | Senior-dev preference: everything about a module in one place (cost: 2,000+ line files) |
