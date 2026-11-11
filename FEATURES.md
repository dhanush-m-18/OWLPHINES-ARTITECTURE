# Features

## Admin portal (platform owner)
| Feature | What it does |
|---|---|
| OTP login | Mobile plus 6-digit OTP (`tbl_admins.lastOtp`) |
| Institutions | Create and edit tenants (name, unique shortName, type/category/board, contact, address, branding, logo, active dates), with Excel/PDF export. Creating a tenant provisions its modules, roles and first SUPERADMIN staff |
| Master Modules | Global module catalogue (`moduleKey`, name, icon, path, parent, priority, `defaultShow`). Icon changes sync to every tenant in real time |
| Master Academics | Global Classes, Years, Intake Years, Streams, Programs, Sections, Subject Types |
| Academic Config per tenant | For each academic year, chooses which classes, sections, streams and programs the school uses and which year is current |
| Attendance Config per tenant | DAY_WISE / SESSION_WISE / SUBJECT_WISE, lock-days limit, class-teacher override |
| Boards / Categories / Institution Types | Lookup masters |
| Admin users | Manage system admins (toggle status, soft delete, restore) |

## Staff portal (per institution)

### Platform features available to every module
- **Dynamic sidebar** from DB modules, filtered by `read` permission, with DB-driven icons.
- **Form customiser** (`ModuleFormCustomizerDrawer`): add, reorder, hide or require fields; set type (Textbox, Character, Number, Contact, Email, Date, Dropdown, MultiSelect, Toggle, Textarea, Upload, Attachment); set unique and min/max; per-year dropdown choices; show-in-listing and show-in-export flags.
- **Listing standard**: search across all configured fields (including date patterns), column filter row, sort, tabs (Active/Inactive/Alumni/Trash), pagination, bulk select actions, Excel/PDF export with a column picker, soft delete plus restore.
- **Bulk import** from Excel with a template download, client-side validation and server-side batch upsert (chunks of 500 inserts / 100 updates).
- **Real-time refresh** via sockets. **Light/dark theme** plus tenant colours.

### Modules
| Module (moduleKey) | Highlights |
|---|---|
| Dashboard | Stats cards and charts (`/dashboard/stats`) |
| Students (`students`) | 4-digit auto `studentKey`, year-wise class/stream/section, promote to next year, mark alumni, bulk status, teaching staff see only their classes |
| Staff (`staff`) | Staff ID plus 4-digit `staffKey`, role, department, shift, teaching flag, resigned/retired (blocks login), per-staff permission overrides, subject assignment, profile view |
| Access Control (`accesscontrol`) | Roles × modules × {read, add, edit, delete, approve, report} matrix, plus per-staff overrides |
| Subjects (`subjects`) + Subject Types | Code unique per tenant, classes it applies to, lab/theory/elective |
| Class & Batch (`classandbatch`) | Class + stream + section per year, class teacher and section coordinator |
| Class Timings | Periods and breaks, days, ordering |
| Timetable (`timetable`) | Weekly matrix per class: assign subject + staff + room per slot, teacher schedule view, clash checks |
| Attendance (`attendance`) | Mode from tenant config (day/session/subject-wise); roster, mark absentees, lock after N days, monthly register, exports |
| Enrollment Forms (`enrollmentforms`) | Lists modules with `show.form = true` and renders their dynamic form |
| Settings | Institution profile, logo (R2), About Us (TinyMCE rich text), Modules management (rename, icon, reorder, enable), Departments, Shifts, Roles |
| Fees: Bank Accounts | Accounts that fee heads and items deposit into |
| Fees: Fee Heads | Tuition, Transport, etc.; priority (allocation order), default bank, refundable flag |
| Fees: Fee Structures | Per year + classes + admission type (New/Regular/Both) + criteria (`scopeConditions`: stream, quota, category…); items per head with amount, bank split %, installments, due date |
| Fees: Concessions | Per student/year/head amount; approval workflow (Pending, Approved, Rejected); applied on the next payment |
| Fees: Installments | Custom payment plans per student, with approval |
| Fees: Collection / Pay Now | Find student, compute obligation (structure − concessions − paid), blocks collection if previous-year dues exist, records payment, auto-allocates across heads by priority, generates receipt `RCP-YYYY-NNNN`, cancel receipt |
| Fees: Reports & Settlements | Financial dashboard, collection report, defaulters, student ledger, gateway/bank settlements and reconciliation, all exportable |
| Calendar / Events | Calendar UI. ⚠ events are stored in **localStorage** (`calendarEventsSlice`), not the backend |
| Reports | Generic report pages for modules with `show.report = true` |
| Generic dynamic modules | Any module without a dedicated page uses `ModuleRecord` JSONB rows |
