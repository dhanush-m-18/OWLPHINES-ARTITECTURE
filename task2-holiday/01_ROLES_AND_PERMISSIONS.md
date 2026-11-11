# OWLPHIN – Roles, Relationships & Permissions Overview

> Personal notes (Dhanush). Verified against the code and the live database on 07-11-2026.
> Do **not** commit this folder.

---

## 1. The four kinds of users

| User type | Stored in | Logs in to | Token `userType` | Scope |
|---|---|---|---|---|
| **System Admin** | `tbl_admins` | **Admin_ERP** (port 5173) | `SYSTEM_ADMIN` | Whole platform – all institutions |
| **SUPERADMIN** (institution owner) | `tbl_staff` + role `SUPERADMIN` in `tbl_roles` | **Staff_ERP** (port 5174) | `STAFF` (roleName `SUPERADMIN`) | One institution – full access |
| **Staff** (Principal, Teacher, Accountant…) | `tbl_staff` + any other role | **Staff_ERP** | `STAFF` | One institution – only what their role/overrides allow |
| **Student** | `tbl_student_info` (+ `tbl_student_year_wise`) | **Student_ERP** (Flutter app) – no student login API yet | – | Their own data |

Also in code: a `MANAGEMENT` token type (`generateManagementToken` in `backend/src/utils/jwt.js`). It is **not used** right now – there is no Management model in `prisma/schema` and nothing calls it. The permission middleware still lets `MANAGEMENT` through, so remember it exists.

Login for all of them = **mobile number + OTP** (`lastOtp` column). With `OTP_TEST_MODE=true` in `backend/.env`, OTP `123456` always works.

---

## 2. How the tables relate

```
                 ┌────────────────────────┐
                 │  tbl_system_modules    │  ← "MASTER MODULES" (global catalogue,
                 │  (Module)              │     managed by System Admin)
                 │  moduleKey, name,      │
                 │  defaultPath, icon,    │
                 │  parentModuleId ───┐   │  (parent/child menu, e.g. Fee Management → Fee Heads)
                 └─────────┬──────────┘   │
                           │ 1 : many      └── self-relation
                           ▼
┌──────────────┐   ┌──────────────────────────────┐
│tbl_institutions├──►│ tbl_institution_modules      │  ← the module "switched on" for ONE school
└──────┬───────┘   │ (InstitutionModule)           │     (own name, path, priority, isEnabled)
       │           │ institutionId + moduleId (uniq)│
       │           └───┬───────────┬──────────┬────┘
       │               │           │          │
       │               ▼           ▼          ▼
       │   tbl_institution_   tbl_role_    tbl_staff_
       │   module_fields      module_      module_
       │   (form fields)      permissions  permissions
       │                      (per ROLE)   (per STAFF override)
       │                          ▲             ▲
       ▼                          │             │
  ┌──────────┐  1 : many   ┌──────┴───┐  many:1 ┌┴─────────┐
  │tbl_roles │◄────────────┤          │◄────────┤tbl_staff │
  │(Role)    │             └──────────┘         │ roleId   │
  └──────────┘                                   └──────────┘
```

Key facts:
- **Every row is per institution** (`institutionId`) except the global master tables: `tbl_system_modules`, `tbl_classes`, `tbl_sections`, `tbl_years`, `tbl_streams`, `tbl_programs`, `tbl_boards`, `tbl_master_subject_types`, `tbl_institution_types`, `tbl_institution_categories`.
- A staff member has **one role** (`tbl_staff.roleId`), plus optional `departmentId` and `shiftId`.
- A role belongs to one institution. `roleKey` is unique per institution (e.g. `principal`). `SUPERADMIN` is created automatically with `isSystemRole = true` and cannot be deleted.
- Permissions are **six booleans**: `read`, `add`, `edit`, `delete`, `approve`, `report`.

---

## 3. What "Master Module" means

`tbl_system_modules` (Prisma model `Module`) is the **master catalogue of every feature** the ERP can offer – Students, Staff, Subjects, Events, Calendar, Fee Management, etc. It is **global**, not per school.

- Managed only by **System Admin** in **Admin_ERP → Master Modules** page (`Admin_ERP/src/pages/MasterModules.tsx`) → API `/api/admin/modules` (`backend/src/admin_side/routes/module.routes.js`: `getMasterModules`, `createMasterModule`, `updateMasterModule`, `toggleMasterModuleStatus`, `deleteMasterModule`, `restoreMasterModule`).
- Each master module has: `moduleKey` (unique code used everywhere in code, e.g. `events`), `name`, `defaultPath`, `defaultIcon`, `defaultPriority` (menu order), `parentModuleId` (to nest under another menu), `defaultShow`, `isActive`.
- A master module does **nothing on its own**. A school only gets it when the System Admin **enables it for that institution** (Admin_ERP → Institutions → create/edit → modules). That creates the `tbl_institution_modules` row.

Current master modules (live DB): dashboard, students, staff, attendance, accesscontrol, fees (+ fees_bank_accounts, fees_heads, fees_structures, fees_collection, fees_pay_now, fees_installments, fees_institution_installments, fees_concessions, fees_reports), subjects, timetable, classandbatch, reports, enrollmentforms, calendar, events, settings, extracurricular (inactive), `event` (soft-deleted junk entry).

There is also **"Master Academics"** (`/api/admin/academics`): global lists of Classes, Years, Intake Years, Streams, Programs, Sections and Subject Types that schools pick from (stored per school in `tbl_institution_academic_config`).

---

## 4. What happens when a module is enabled for a school

(`backend/src/admin_side/controllers/institution.controller.js` → update, ~line 332; and `services/institution.service.js` → create)

1. **Upsert `tbl_institution_modules`** (institutionId + moduleId) – copies name, path, icon, priority, parent from the master module.
2. **Seed default form fields** – if the institution module is new, it inserts the module's defaults from `DEFAULT_MODULE_FIELDS` (`backend/src/config/defaultFields.config.js`) into `tbl_institution_module_fields`. (Later, `ensureDefaultFields` also auto-adds any new default fields when the page loads.)
3. **Give SUPERADMIN full permission** – upserts `tbl_role_module_permissions` for the SUPERADMIN role with all six flags `true`.
4. Emits socket event `institution_modules_updated` so open Staff_ERP sidebars refresh.

Other roles get **no permission** until the SUPERADMIN grants it in **Staff_ERP → Access Control**.

---

## 5. How a permission check works (runtime)

Backend route: `requirePermission('<moduleKey>', '<action>')` → `backend/src/staff_side/middlewares/permission.middleware.js` → `resolveModulePermission()` in `backend/src/staff_side/services/permission.service.js`.

Order of decision:

1. Token must be `STAFF` or `MANAGEMENT` → else **403**.
2. `MANAGEMENT` or role **`SUPERADMIN`** → **always allowed** (skips everything below).
3. Find the institution module (by id, moduleId or moduleKey). If missing, disabled, deleted, or the master module is inactive/deleted → **403 "Module disabled or not configured"**.
4. **Staff override** (`tbl_staff_module_permissions` for this staff + module) – if a row exists, **it wins completely**.
5. Else **role permission** (`tbl_role_module_permissions` for the staff's role + module).
6. Else → everything `false` → **403**.

So: **SUPERADMIN > staff override > role permission > nothing.**

Frontend mirrors this: `useModulePermission('<moduleKey>')` (`Staff_ERP/src/hooks`) and `moduleItem.permissions` hide Add/Edit/Delete buttons; the sidebar only lists modules with `read`.

Special routes:
- `superadminMiddleware` (`staff_side/middlewares/superadmin.middleware.js`) – role CRUD (`POST/PUT/DELETE /api/staff/roles`) is SUPERADMIN-only.
- `GET /api/staff/roles` – any logged-in staff (no module permission check).
- Admin routes (`/api/admin/*`) – `adminMiddleware` requires `SYSTEM_ADMIN`.

---

## 6. Who can do what – quick matrix

| Action | System Admin | SUPERADMIN | Staff (by role/override) |
|---|---|---|---|
| Create institution | ✅ | ❌ | ❌ |
| Create/edit **master modules** | ✅ | ❌ | ❌ |
| Enable a module for a school | ✅ | ❌ | ❌ |
| Master academics (classes, years…) | ✅ | ❌ | ❌ |
| Create/edit/delete **roles** | ❌ | ✅ | ❌ |
| Set role permissions (Access Control) | ❌ | ✅ | only if granted `accesscontrol` |
| Set per-staff override | ❌ | ✅ | only if granted `accesscontrol` |
| Use any enabled module | ❌ (no staff login) | ✅ everything | only the actions granted |
| Modify Form (dynamic fields) | ❌ | ✅ | needs that module's `edit` |

---

## 7. How students relate to classes (needed for Holiday)

- Student's class is **a text value**, not a foreign key: `tbl_student_info.class`, `.section`, `.stream` (current), and per year in `tbl_student_year_wise` (`currentYear`, `class`, `section`, `stream`).
- The **list of classes a school uses** = `tbl_institution_academic_config.classes` (text array) for the row with `isCurrent = true` (e.g. XYZ 2026-27 → `["Pre-KG","I","II"]`, sections `["A","B","C"]`).
- `tbl_class_batches` = actual class + batch (section) combos per academic year (`className`, `batchName`, `streamName`, `academicYear`, class teacher).
- Global master list of class names = `tbl_classes` (Pre-KG, I, II, IV … II PUC).

Live example – XYZ: roles **SUPERADMIN** (3 staff) and **Principal** (2 staff); class-batches **Pre-KG A**, **I B** (2026-27).

---

## 8. Files to remember

| Concern | File |
|---|---|
| Role / permission tables | `backend/prisma/schema/role.prisma`, `roleModulePermission.prisma`, `staffModulePermission.prisma` |
| Master + institution modules | `backend/prisma/schema/module.prisma`, `institutionModule.prisma`, `institutionModuleField.prisma` |
| Permission check | `backend/src/staff_side/middlewares/permission.middleware.js`, `services/permission.service.js` |
| SUPERADMIN-only guard | `backend/src/staff_side/middlewares/superadmin.middleware.js` |
| System-admin guard | `backend/src/admin_side/middlewares/admin.middleware.js` |
| Tokens | `backend/src/utils/jwt.js` |
| Master modules API | `backend/src/admin_side/routes/module.routes.js` |
| Enable modules for school | `backend/src/admin_side/controllers/institution.controller.js`, `services/institution.service.js` |
| Default fields | `backend/src/config/defaultFields.config.js`, `ensureDefaultFields` in `staff_side/controllers/moduleField.controller.js` |
| Roles API | `backend/src/staff_side/routes/role.routes.js` + `Staff_ERP/src/features/roles/rolesApiSlice.ts` |
| Module key → page | `Staff_ERP/src/config/moduleRegistry.ts` |
