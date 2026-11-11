# Holiday Module – Implementation Plan

> Personal notes (Dhanush), 07-11-2026. Plan only – nothing built yet.
> Read `01_ROLES_AND_PERMISSIONS.md` first.

## ✅ Decisions confirmed (07-11-2026) – these OVERRIDE the details below

| # | Decision | Effect on the plan |
|---|---|---|
| 1 | Staff → by **role** only; Students → by **class** only | Drop `staffIds` and the "specific staff" dropdown and `/audience-options/staff` endpoint |
| 2 | Class-based first | Drop `studentSections` (can be added later) |
| 3 | Holiday types are **added by the admin** in the Holidays section | `holidayType` = **Dropdown** dynamic field with **no fixed choices** – the admin adds/edits the types via **Modify Form** (dropdown choices). No hard-coded list. |
| 4 | Calendar integration later | Phase 5 (Calendar, `/my` endpoint) postponed – focus is the Holidays module |
| 5 | Master module | Explained in `01_ROLES_AND_PERMISSIONS.md` §3; created via Admin_ERP → Master Modules |

**Final table columns:** `title, startDate, endDate, holidayType, description, forStaff, forStudents, staffRoleIds[], studentClasses[], academicYear, customFields, isActive, isDeleted, createdBy, updatedBy, timestamps`.

**Final "Applies To" section:** `[✓] Staff → Roles (multi-select, empty = all roles)` and `[✓] Students → Classes (multi-select, empty = all classes)`.

---

## 1. Goal

A new **Holidays** module, set up exactly like the other modules (master module → enabled per institution → Staff_ERP page with dynamic fields), where the person creating a holiday chooses **who it applies to**:

- **Staff** → pick **roles** from a dropdown (e.g. Principal, Teacher), then optionally pick **specific staff** from those roles.
- **Students** → pick **classes** (e.g. Pre-KG, I, II), optionally sections.
- Leaving a list empty means **everyone** in that group.

Examples:
| Holiday | Staff | Students |
|---|---|---|
| Diwali | All roles | All classes |
| Exam prep leave (students only) | – (not for staff) | II PUC |
| Staff training day | Teacher role | – (students not affected… or all, as chosen) |
| Pre-KG annual-day break | – | Pre-KG |

---

## 2. Step 0 – "Add Holiday at the initial" (master module setup)

Same path every module took. **No code** – done through the apps:

1. **Admin_ERP → Master Modules → Add** (System Admin login, e.g. 6361249205 / OTP 123456):
   | Field | Value |
   |---|---|
   | Module key | `holidays` |
   | Name | `Holidays` |
   | Default path | `/holidays` |
   | Default icon | `FaUmbrellaBeach` |
   | Default priority | `12` (after Events = 11) |
   | Parent | none |
2. **Admin_ERP → Institutions → Edit school → Modules → enable Holidays → Save.**
   This creates the institution module, seeds the default fields (from step 3.2 below – so do this **after** the backend code is deployed/pulled) and gives SUPERADMIN full permission.
3. **Staff_ERP → Access Control** (as SUPERADMIN): give other roles `read` / `add` / `edit` / `delete` on Holidays as needed.

⚠️ This writes to the **shared** Neon DB – confirm with the lead who creates the master module (you or them) so it isn't created twice.

---

## 3. Backend

### 3.1 New table `tbl_holidays` – `backend/prisma/schema/holiday.prisma` (new)

```prisma
model Holiday {
  id              String      @id @default(uuid())
  institutionId   String
  institution     Institution @relation(fields: [institutionId], references: [id], onDelete: Cascade)
  title           String
  startDate       DateTime    @db.Date
  endDate         DateTime    @db.Date
  holidayType     String?                 // Public / Festival / Institution / Vacation / Other
  description     String?
  forStaff        Boolean     @default(true)
  forStudents     Boolean     @default(true)
  staffRoleIds    String[]    @default([])  // empty = all roles
  staffIds        String[]    @default([])  // empty = all staff in the chosen roles
  studentClasses  String[]    @default([])  // empty = all classes
  studentSections String[]    @default([])  // empty = all sections
  academicYear    String?                 // which academic-config year the classes came from
  customFields    Json?       @db.JsonB
  isActive        Boolean     @default(true)
  isDeleted       Boolean     @default(false)
  createdBy       String?
  updatedBy       String?
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  @@index([institutionId])
  @@index([institutionId, isDeleted])
  @@index([institutionId, startDate])
  @@index([institutionId, endDate])
  @@index([customFields], type: Gin)
  @@map("tbl_holidays")
}
```

Why **text arrays** and not join tables: the school's class list is already stored as arrays (`tbl_institution_academic_config.classes`) and students store class as text, so arrays match the existing design and keep queries simple:

```sql
-- holidays that apply to one staff member
WHERE "forStaff" AND (cardinality("staffRoleIds") = 0 OR $roleId  = ANY("staffRoleIds"))
                 AND (cardinality("staffIds")     = 0 OR $staffId = ANY("staffIds"))
-- holidays that apply to one student
WHERE "forStudents" AND (cardinality("studentClasses")  = 0 OR $class   = ANY("studentClasses"))
                    AND (cardinality("studentSections") = 0 OR $section = ANY("studentSections"))
```

Also edit `backend/prisma/schema/institution.prisma` → add `holidays Holiday[]`.

**Create the table safely** (same as tbl_events – never `db push`):
```bash
npx prisma migrate diff --from-url "$DATABASE_URL" --to-schema-datamodel prisma/schema --script > holiday.sql
# open holiday.sql, keep ONLY the CREATE TABLE "tbl_holidays" + its indexes + FK, delete everything else
npx prisma db execute --file holiday.sql --schema prisma/schema
npx prisma generate      # backend stopped
```

### 3.2 Default fields – `backend/src/config/defaultFields.config.js` (edit)

Add `defaultHolidayFields` + `holidays: defaultHolidayFields` in the registry:
| fieldKey | Label | Type | Required |
|---|---|---|---|
| title | Holiday Name | Textbox (2–150) | ✅ |
| startDate | From Date | Date | ✅ |
| endDate | To Date | Date | ✅ |
| holidayType | Holiday Type | Dropdown (Public, Festival, Institution, Vacation, Other) | – |
| description | Description | Textarea (≤1000) | – |
| isActive | Status | Toggle (Active/Inactive) | – |

The **audience** (staff roles / staff / classes) is **not** a dynamic field – it's a fixed section in the form (like the event colour picker), because it needs linked dropdowns.

### 3.3 Controller – `backend/src/staff_side/controllers/holiday.controller.js` (new)

Copy the structure of `event.controller.js` (resolver → field fetcher → value extractor → search builder → export – the project's 5-step standard) and add the audience parts:

| Handler | Route | Notes |
|---|---|---|
| `getHolidays` | `GET /` | list with search, filters, sort, `from`/`to` overlap, plus filters `forStaff`, `forStudents`, `roleId`, `class` |
| `getHolidayById` | `GET /:id` | |
| `createHoliday` | `POST /` | validate dynamic fields + **audience** (below) |
| `updateHoliday` | `PUT /:id` | merge customFields; no-change handled on frontend |
| `deleteHoliday` / `bulkDeleteHolidays` | `DELETE /:id`, `POST /bulk-delete` | soft delete (`isDeleted: true, isActive: false`) |
| `exportHolidayExcel/Pdf` | `GET /export/excel|pdf` | add "Applies To" column (role names / classes) |
| **`getHolidayAudienceOptions`** | `GET /audience-options` | returns `{ roles:[{id,name,staffCount}], classes:[...], sections:[...], academicYear }` for this institution (roles from `tbl_roles`, classes/sections from the **current** `tbl_institution_academic_config`) |
| **`getHolidayAudienceStaff`** | `GET /audience-options/staff?roleIds=a,b` | active, non-deleted, non-resigned/retired staff of those roles: `{id,name,staffId,roleId}` |
| **`getMyHolidays`** | `GET /my?from&to` | holidays that apply to the **logged-in staff** (role + id rule above) – used by Calendar |

**Audience validation (server side, never trust the UI):**
- at least one of `forStaff` / `forStudents` must be true
- every `staffRoleIds` id must be a non-deleted role of **this** institution
- every `staffIds` id must be staff of **this** institution **and** have one of the chosen roles (if roles were chosen)
- every `studentClasses` / `studentSections` value must exist in the current academic config
- if `forStaff` is false → clear staff arrays; if `forStudents` is false → clear student arrays
- every query: `institutionId: req.user.institutionId, isDeleted: false`

Why separate `audience-options` endpoints: `GET /class-batches` needs `classandbatch` read permission and staff lists need `staff` read. A user who only has **Holidays** permission would get **403** on those. Serving the options from the holidays route (guarded by `holidays` add/edit) avoids cross-module permission problems.

Emit `emitToInstitution(institutionId, 'holidays_updated')` after create/update/delete.

### 3.4 Validator – `backend/src/staff_side/validators/holiday.validator.js` (new)
`createHolidaySchema`, `updateHolidaySchema` (`forStaff`, `forStudents` booleans; `staffRoleIds`, `staffIds`, `studentClasses`, `studentSections` = arrays of strings, max ~500 items), `bulkDeleteHolidaySchema`.

### 3.5 Routes – `backend/src/staff_side/routes/holiday.routes.js` (new) + `routes/index.js` (edit: `router.use('/holidays', holidayRoutes)`)
- `requirePermission('holidays', 'read')` for list/get/export
- `requirePermission('holidays', ['add','edit'])` for `audience-options`
- `GET /my` → any logged-in staff whose institution has Holidays enabled, or allow `calendar` read too (same OR-guard idea as `requireEventsOrCalendarRead` in `event.routes.js`)

---

## 4. Frontend (Staff_ERP)

| File | New/Edit | What |
|---|---|---|
| `src/features/holidays/holidaysApiSlice.ts` | new | `getHolidays`, `getHolidayById`, `create`, `update`, `delete`, `bulkDelete`, `getAudienceOptions`, `getAudienceStaff`, `getMyHolidays`; tag `Holidays`; all queries `refetchOnMountOrArgChange: true` |
| `src/app/store.ts` | edit | add slice to `combineSlices` + middleware; `socket.on('holidays_updated')` → invalidate `Holidays` |
| `src/store/apiSlice.ts` | edit | add `holidaysApiSlice.util.resetApiState()` on logout |
| `src/config/moduleRegistry.ts` | edit | `holidays: HolidaysPage` |
| `src/pages/Holidays.tsx` | new | listing page in the **Unified Card Standard** (copy `EventsDynamic.tsx`): header + Modify Form + Add, search, filter row, export, bulk delete, pagination; table columns: Holiday Name, Dates, Type, **Applies To**, Status |
| `src/components/holidays/HolidayAudienceSection.tsx` | new | the "Applies To" block in the Add/Edit drawer (below) |
| `src/components/holidays/AudienceBadges.tsx` | new | table cell: "Staff: Principal, Teacher" / "Students: I, II" / "Everyone" |
| `src/pages/Calendar.tsx` | edit (phase 2) | also load `getMyHolidays` and show them as red Holiday chips |

### "Applies To" section (in the drawer, under the dynamic fields)

```
Applies To
[✓] Staff                       [✓] Students
 ├ Roles:  [ All roles        ▾]   ├ Classes:  [ All classes   ▾]   (multi-select)
 └ Staff:  [ All staff in     ▾]   └ Sections: [ All sections  ▾]   (multi-select, optional)
            selected roles
```
- Reuse **`MultiSearchableSelect`** (`src/components/ui/MultiSearchableSelect.tsx`) for all four dropdowns.
- Roles + classes + sections come from `getAudienceOptions` (one call when the drawer opens).
- Staff dropdown loads from `getAudienceStaff?roleIds=…` **after** roles are chosen (skip the query when no role is selected → shows "All staff"); when roles change, drop staff that are no longer in the chosen roles.
- Unticking Staff/Students greys out and clears its dropdowns.
- Include the audience in `initialFormData` so the **"No changes found"** guard still works.
- Frontend checks: at least one of Staff/Students ticked.

---

## 5. Student side (later)

There is **no student API yet** (`backend/src` has only `admin_side` and `staff_side`; Student_ERP is a Flutter app). When the student app gets its API, add `GET /api/student/holidays` using the student rule from 3.1 (student's current `class`/`section` from `tbl_student_info` or the current `tbl_student_year_wise`).

Attendance could later skip dates that are holidays for that class/role – not part of this task.

---

## 6. Full file list

**Backend**
| File | New/Edit |
|---|---|
| `backend/prisma/schema/holiday.prisma` | new |
| `backend/prisma/schema/institution.prisma` | edit (+ `holidays Holiday[]`) |
| `backend/src/config/defaultFields.config.js` | edit (+ `defaultHolidayFields`, registry entry) |
| `backend/src/staff_side/controllers/holiday.controller.js` | new |
| `backend/src/staff_side/validators/holiday.validator.js` | new |
| `backend/src/staff_side/routes/holiday.routes.js` | new |
| `backend/src/staff_side/routes/index.js` | edit (mount `/holidays`) |

**Frontend (Staff_ERP)**
| File | New/Edit |
|---|---|
| `src/features/holidays/holidaysApiSlice.ts` | new |
| `src/app/store.ts` | edit |
| `src/store/apiSlice.ts` | edit |
| `src/config/moduleRegistry.ts` | edit |
| `src/pages/Holidays.tsx` | new |
| `src/components/holidays/HolidayAudienceSection.tsx` | new |
| `src/components/holidays/AudienceBadges.tsx` | new |
| `src/pages/Calendar.tsx` | edit (phase 2) |

**Database / setup (no code)**: create `tbl_holidays` (single CREATE only), `npx prisma generate`, master module `holidays` in Admin_ERP, enable per institution, Access Control permissions.

Shared files touched: only the **registration points** every module must touch (`routes/index.js`, `defaultFields.config.js`, `institution.prisma`, `store.ts`, `apiSlice.ts`, `moduleRegistry.ts`) – one-line additions, nothing existing changed.

---

## 7. Build order (phases)

1. **DB + backend core** – schema, table, default fields, controller CRUD, routes. Test with Postman/curl.
2. **Master module + enable** for XYZ (Admin_ERP) → sidebar shows Holidays.
3. **Frontend listing + drawer** with dynamic fields.
4. **Applies To section** – audience-options endpoints + role → staff and class pickers + validation.
5. **Calendar integration** – `/my` endpoint, red holiday chips.
6. Export "Applies To" column, polish, `npx tsc --noEmit`, commit.

---

## 8. Decisions to confirm with the lead

1. **Staff targeting depth** – roles only, or roles **+ specific staff** (planned)? Also by **department/shift**?
2. **Students** – classes only, or classes **+ sections** (planned)? Stream too (PUC)?
3. **Holiday types** – is "Public, Festival, Institution, Vacation, Other" OK?
4. **Calendar** – should holidays come from this new module and the old "Event type = Holiday" trick be dropped?
5. **Who creates the master module** `holidays` in Admin_ERP (shared DB) – you or the lead?
6. Should staff without Holidays permission still **see** holidays that apply to them (Calendar `/my`)? Planned: **yes**.
