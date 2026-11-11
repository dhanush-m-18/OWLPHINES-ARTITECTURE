# Database (PostgreSQL via Prisma 5)

- Schema lives in `backend/prisma/schema/` with **one model per file**, plus `base.prisma` (generator with `previewFeatures = ["prismaSchemaFolder"]` and the postgres datasource).
- All ids are `String @id @default(uuid())`. Every model has `createdAt` / `updatedAt`, and almost all have `isActive` / `isDeleted`.
- ⚠ `prisma/migrations/` holds only an old `init` migration with **outdated table names** (`institutions`, `module_records`, `passwordHash` columns). The live DB is synced with **`npx prisma db push`**. Don't trust `migration.sql` as documentation.

## Table name map (use for raw SQL!)

| Model | Table (`@@map`) | Tenant-scoped |
|---|---|---|
| Admin | `tbl_admins` | no |
| Institution | `tbl_institutions` | (is the tenant) |
| InstitutionType | `tbl_institution_types` | no (master) |
| InstitutionCategory | `tbl_institution_categories` | no |
| Board | `tbl_boards` | no |
| Module | `tbl_system_modules` | no |
| InstitutionModule | `tbl_institution_modules` | yes |
| InstitutionModuleField | `tbl_institution_module_fields` | via institutionModuleId |
| ModuleRecord | `tbl_module_records` | yes |
| Role | `tbl_roles` (plural) | yes |
| RoleModulePermission | `tbl_role_module_permissions` | via role |
| StaffModulePermission | `tbl_staff_module_permissions` | via staff |
| Department | `tbl_department` (singular) | yes |
| Shift | `tbl_shift` (singular) | yes |
| Staff | `tbl_staff` | yes |
| StudentInfo | `tbl_student_info` | yes |
| StudentYearWise | `tbl_student_year_wise` | yes |
| Subject | `tbl_subjects` | yes |
| SubjectType | `tbl_subject_types` | yes |
| MasterSubjectType | `tbl_master_subject_types` | no |
| ClassBatch | `tbl_class_batches` | yes |
| StaffSubjectAssignment | `tbl_staff_subject_assignments` | yes |
| ClassTiming | `tbl_class_timings` | yes |
| TimetableSlot | `tbl_timetable_slots` | yes |
| InstitutionAcademicConfig | `tbl_institution_academic_config` | yes |
| InstitutionAttendanceConfig | `tbl_institution_attendance_configs` | yes (1:1) |
| AttendanceSession | `tbl_attendance_sessions` | yes |
| AttendanceAbsentee | `tbl_attendance_absentees` | yes |
| BankAccount | `tbl_bank_accounts` | yes |
| FeeHead | `tbl_fee_heads` | yes |
| FeeStructure | `tbl_fee_structures` | yes |
| FeeStructureItem | `tbl_fee_structure_items` | via structure |
| StudentFeeConcession | `tbl_student_fee_concessions` | yes |
| StudentFeeInstallment | `tbl_student_fee_installments` | yes |
| FeePayment | `tbl_fee_payments` | yes |
| FeePaymentAllocation | `tbl_fee_payment_allocations` | via payment |
| FeeSettlement | `tbl_fee_settlements` | yes |
| Class / Year / Stream / Section / Program / IntakeYear | `tbl_classes`, `tbl_years`, `tbl_streams`, `tbl_sections`, `tbl_programs`, `tbl_intake_years` | no (master) |

## Domain groups and relations

### Platform and tenancy
```
Institution ─┬─ InstitutionType, InstitutionCategory, Board (master FKs)
             ├─ branding Json (logoUrl, colours, aboutUs, websiteUrl, addressDetails…)
             ├─ InstitutionModule ─┬─ Module (system)
             │                     ├─ InstitutionModuleField[]  (dynamic form definition)
             │                     ├─ RoleModulePermission[]
             │                     ├─ StaffModulePermission[]
             │                     └─ ModuleRecord[] (generic JSONB rows)
             └─ every operational table below (onDelete: Cascade)
```

### People
- **Staff**: `staffId` (human code), `staffKey` (4-digit, unique per institution), `roleId`, `departmentId`, `shiftId`, `mobile`, `email`, `lastOtp`, `isTeachingStaff`, `isResigned/resignDate`, `isRetired/retiredDate`, `customFields` JSONB (GIN index).
  - ⚠ `email` and `mobile` are **not unique** (not even per institution). OTP login looks staff up by mobile across **all** tenants.
  - ⚠ **No `passwordHash` column**, yet `loginStaff` uses it.
- **StudentInfo**: the master student row. `studentKey` (4-digit, `@@unique([institutionId, studentKey])`), `name`, `gender`, `dob`, `studentStatus`, `isActive`, `isAlumni`, `intakeYear`, current `class/stream/section` (denormalised copy), `customFields` JSONB (GIN).
- **StudentYearWise**: one row per student per academic year (`@@unique([institutionId, studentKey, currentYear])`) holding that year's `class/stream/section/isDiscontinued`. **Why:** promotions create a new year row and keep the history. Fees and attendance are computed for a given year.

### Academics
- **InstitutionAcademicConfig** (`yearName`, `isCurrent`, `classes[]`, `sections[]`, `streams[]`, `programs[]`): which master values a tenant uses per year.
- **ClassBatch**: class + stream + batch(section) + academicYear + classTeacher + sectionCoordinator. Stored as **strings**, not FKs to master tables.
- **Subject** (`subjectCode` unique per tenant, `classes String[]`, `hasLab/hasTheory/isElective`) and **SubjectType**.
- **StaffSubjectAssignment**: staff × subject × classBatch × year.
- **ClassTiming**: periods and breaks (`startTime/endTime` as strings, `days String[]`).
- **TimetableSlot**: classBatch × timing × dayOfWeek → subject + staff (+ room).

### Attendance
- **InstitutionAttendanceConfig** (1:1): `attendanceType` (DAY_WISE / SESSION_WISE / SUBJECT_WISE), `lockDaysLimit`, `allowClassTeacherSubjectOverride`.
- **AttendanceSession**: one per class × date × type × session × subject × timing, with counts and `takenById`.
- **AttendanceAbsentee**: only **absent** students are stored (present = roster minus absentees). This keeps storage small.
- ⚠ The unique key includes nullable columns (`session`, `subjectId`, `timingId`). In Postgres, `NULL`s are distinct, so the DB **does not** prevent duplicate DAY_WISE sessions. The controller must upsert by `findFirst`.

### Fees
```
BankAccount ◀── FeeHead.defaultBankAccount
FeeStructure (year, classes[], admissionType, scopeConditions Json, totalAmount)
   └─ FeeStructureItem (feeHead, amount, bankAccount(+secondary split %), installments Json, dueDate)
StudentFeeConcession (student, year, feeHead?, amount, approvedStatus 0/1/2, paymentStatus 0/1, status)
StudentFeeInstallment (student, year, amount, dueDate, approvedStatus, isPaid, paidAmount)
FeePayment (receiptNumber unique per tenant "RCP-YYYY-0001", amountPaid Decimal(12,2), mode, status PAID/CANCELLED,
            settlementId, settlementStatus 0/1)
   └─ FeePaymentAllocation (feeHead, feeStructureItem, allocatedAmount)   ← how a payment splits across heads
FeeSettlement (bank/gateway settlement batch: gross, fee, net, status, reconciledCount)
```
Money columns are `Decimal(12,2)`, but the controllers do arithmetic with JS `Number`/`parseFloat` (see SECURITY P-3).

## Index strategy observed
- `@@index([institutionId])` and `@@index([institutionId, isDeleted])` on almost every tenant table.
- Extra composite indexes for common filters (`[institutionId, academicYear]`, `[institutionId, paymentDate]`, …).
- GIN on `customFields` for Staff and StudentInfo (supports `path`/`equals` JSON filters).
- Missing: StudentInfo has no `[institutionId, isDeleted, isActive, isAlumni]` composite, which the tab counts use.

## Rules when changing the schema
1. New file `schema/<model>.prisma`, add `@@map("tbl_<name>")`, add `institutionId` + relation `onDelete: Cascade`, `isActive`, `isDeleted`, `createdAt`, `updatedAt`, `createdBy?`, `updatedBy?`.
2. Add the back-relation array in `institution.prisma`.
3. Add `@@index([institutionId])`, `@@index([institutionId, isDeleted])`, and uniqueness **per tenant** (`@@unique([institutionId, code])`).
4. `npx prisma validate`, then `npx prisma db push` (dev), then `npx prisma generate`.
5. If you add raw SQL, use the `@@map` name.
