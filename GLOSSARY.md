# Glossary

| Term | Meaning |
|---|---|
| **Tenant / Institution** | One school or college. Every data row belongs to one via `institutionId` |
| **Admin / SYSTEM_ADMIN** | OWLPHIN platform operator using Admin_ERP (`tbl_admins`) |
| **SUPERADMIN** | The top role **inside** an institution (staff portal). Bypasses module permissions |
| **MANAGEMENT** | userType for a multi-institution management user (token generator exists, no login yet). Bypasses permissions |
| **Staff** | Any institution employee user (`tbl_staff`). Teaching or non-teaching |
| **Teaching staff** | `isTeachingStaff=true`. Sees only students of their assigned classes |
| **Module (system module)** | Global feature definition in `tbl_system_modules`, identified by `moduleKey` |
| **InstitutionModule** | A tenant's copy of a module (rename, icon, priority, enable, `show.form/report`) |
| **moduleKey** | Stable lowercase id (`students`, `staff`, `subjects`, `classandbatch`, `fees_heads`, `fees_collection`, `accesscontrol`, `attendance`, `timetable` …). Binds backend permissions, default fields and frontend page |
| **Module field / InstitutionModuleField** | One configurable form field of a module |
| **System field** | `isSystemField=true`; maps to a real DB column |
| **Custom field** | A field stored in the `customFields` JSONB column |
| **ModuleRecord** | Generic JSONB row for modules without a dedicated table |
| **Permission actions** | `read`, `add`, `edit`, `delete`, `approve`, `report` |
| **Role permission / Staff override** | `RoleModulePermission` (per role) / `StaffModulePermission` (per person, wins over role) |
| **Soft delete** | `isDeleted=true, isActive=false`; shown in the "Trash" tab; restorable |
| **studentKey / staffKey** | Auto 4-digit per-tenant ids (`0001`); naturally sorted |
| **staffId** | 1) the human staff code on the Staff row; 2) the JWT claim holding the Staff **uuid**. Don't confuse them |
| **StudentInfo / StudentYearWise** | Master student row / one row per academic year with that year's class, section and stream |
| **currentYear / academicYear** | Academic year label like `2026-2027` or `2026-27` (variants handled by `expandYearVariants`) |
| **Intake year** | Year the student joined; decides NEW_ADMISSION vs REGULAR fee structure |
| **Alumni** | Passed-out student (`isAlumni`); automatically inactive |
| **Promotion** | Moving students to the next year's class (new year-wise row) |
| **Class & Batch** | Class + stream + batch/section unit with a class teacher and section coordinator |
| **Stream** | PU/college specialisation (PCMB, Commerce …) |
| **Academic config** | Per-tenant per-year list of classes, sections, streams and programs (`InstitutionAcademicConfig`) |
| **Class timing** | A period or break definition used by timetable and attendance |
| **DAY_WISE / SESSION_WISE / SUBJECT_WISE** | Attendance modes (per day / morning and afternoon / per subject period) |
| **Absentee** | Only absent students are stored per attendance session |
| **Lock days** | Days after which attendance can't be edited (`lockDaysLimit`) |
| **Fee head** | A fee component (Tuition, Transport…). `priority` decides allocation order |
| **Fee structure / item** | Fee plan for year + classes + admission type + criteria / per-head amount inside it |
| **Criteria / scopeConditions** | JSON rules (stream, quota, category…) a student must match for a structure |
| **Concession** | Approved reduction of a head's amount for a student |
| **Installment** | Approved custom payment schedule for a student |
| **Allocation** | How one payment's amount is split across fee heads |
| **Receipt number** | `RCP-<YYYY>-<NNNN>`, unique per tenant |
| **Settlement / Reconciliation** | Matching collected payments to bank or gateway settlement batches |
| **Defaulters** | Students with pending dues |
| **Ledger** | Per-student fee history |
| **R2** | Cloudflare object storage (S3-compatible) for uploads |
| **OTP_TEST_MODE** | Dev flag: OTP is always `123456`, no SMS sent. Must never be on in production |
| **Unified Card Standard** | Mandatory page layout (header, toolbar, table, pagination in one card) |
| **No-Change Toast Guard** | Skip the API call and show "No changes found" when an edit form is unchanged |
| **RTK Query slice** | One `createApi` per feature in `features/<x>/<x>ApiSlice.ts` |
| **invalidateTags** | Marks cached queries stale so they refetch (triggered by socket events) |
| **resetApiState** | Clears all cached data (logout or user switch) |
| **Neon** | Serverless Postgres host; hence the connect-retry on boot (cold starts) |
