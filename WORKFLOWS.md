# Workflows

## 1. Staff OTP login
```
Login.tsx → AuthContext.sendOtp(mobile) → POST /api/staff/auth/send-otp
  backend: normalise to 10 digits → regex ^[6-9]\d{9}$ → find staff (mobile | +91… | 91…), active,
           not resigned/retired → otp = TEST_MODE ? '123456' : random 6-digit
           → staff.updateMany({ lastOtp }) → sendOtp() via Airtel/Gupshup/WhatsApp (axios, 15s timeout)
→ user enters OTP → POST /verify-otp
  backend: find staff → compare lastOtp → clear lastOtp → institution active?
           → generateStaffToken → getStaffAccessibleModules → Set-Cookie staff_token (httpOnly, 7d)
→ AuthContext sets staff/institution/modules → joins socket room → redirect /dashboard
```

## 2. Page refresh (session restore)
`AuthProvider` mount → `GET /staff/auth/me` (cookie) → sets state or logs out. Any 401/403 from `services/api.request` fires the `auth:unauthorized` event, which clears state and runs `staffApi.util.resetApiState()`.

## 3. Logout
`POST /staff/auth/logout` (clears cookie) → state reset → `resetApiState()` on every slice → `/login`.
Note: the JWT itself stays valid until it expires (stateless). There is no server-side revocation.

## 4. Onboarding a new institution (Admin)
1. Admin creates master data (classes, years, sections, streams, modules) if not present.
2. `POST /api/admin/institutions` creates the institution, `InstitutionModule` rows for the modules, default roles, and the SUPERADMIN staff.
3. Admin sets **academic config** (current year + classes/sections/streams) and **attendance config**.
4. The school's SUPERADMIN logs into Staff ERP via OTP, configures roles and permissions, departments, shifts, form fields, then adds staff and students.

## 5. Adding a brand-new operational module (developer workflow)
1. **DB**: `prisma/schema/<x>.prisma` (tenant columns, soft delete, `@@map`), back-relation in `institution.prisma`, then `prisma db push`.
2. **Default fields**: add `default<X>Fields` to `config/defaultFields.config.js` and register it in `DEFAULT_MODULE_FIELDS['<moduleKey>']`.
3. **Backend**: validator, controller (5-step pipeline + CRUD + bulk + export), routes with `requirePermission('<moduleKey>', …)`, mount in `staff_side/routes/index.js`, `emitToInstitution` on mutations.
4. **Admin**: create the master Module (`moduleKey` matches) in Admin ERP, so it gets provisioned to institutions.
5. **Frontend**: `features/<x>/<x>ApiSlice.ts`, register in `app/store.ts` (reducer + middleware + socket invalidation) and `store/apiSlice.ts` (reset), page `pages/<X>.tsx` copying `ClassBatch.tsx` layout, entry in `config/moduleRegistry.ts`.
6. **Permissions**: SUPERADMIN gives roles access in Access Control.
7. `npx tsc --noEmit` in both frontends.

## 6. Student create
`StudentPage` drawer → `createStudent` mutation → `POST /staff/students` → `requirePermission('students','add')` → zod → controller:
resolve module fields → `validateModuleFields` (type/required/choices) → `validateModuleUniqueFields` → key = provided or `generate4DigitStudentKey` → `$transaction`: create `StudentInfo` + `StudentYearWise(currentYear)` → `emit student_updated` → 201. Every open Students page refetches through the socket invalidation.

## 7. Student promotion (year end)
`POST /students/promote {studentIds, sourceAcademicYear, targetAcademicYear, targetClass, targetSection?, targetStream?}` creates or updates a `StudentYearWise` row for the target year and updates the denormalised class/section on `StudentInfo`. The previous year's row stays as history.

## 8. Fee setup to collection
```
Bank Accounts → Fee Heads (priority, default bank) → Fee Structure (year, classes, admissionType, criteria, items)
→ optional per-student Concession / Installment plan → approve
→ Fee Collection: pick student → GET /fees/collection/student/:id  (obligation per head: structure − concession − paid)
→ POST /fees/collection/pay {studentId, academicYear, amountPaid, paymentMode, …}
   1 student in tenant? 2 previous-year dues? → block   3 resolve structure (feeStructureMatcher)
   4 past allocations per head   5 approved concessions per head   6 next receipt RCP-YYYY-NNNN
   7 $transaction: FeePayment → allocate by head priority → leftover to first head
                   → mark installments paid (oldest due first) → mark approved concessions applied
→ receipt modal (print) → emits fee_payment_created / fee_installment_updated / fee_concession_updated
→ later: POST /fees/collection/settle and /fees/reports/settlements/:id/reconcile
```
Cancel: `POST /fees/collection/cancel/:paymentId` sets status `CANCELLED` (excluded from paid totals).

## 9. Attendance
`GET /attendance/context` (tenant mode, classes this staff can mark) → `GET /roster?classBatchId&date…` → staff toggles absentees → `POST /submit` upserts the `AttendanceSession` and replaces `AttendanceAbsentee` rows. A submission older than `lockDaysLimit` is rejected for non-admins. `GET /monthly-register` builds the grid.

## 10. Real-time update loop
Controller mutation → `emitToInstitution(id, 'x_updated')` → each client in room `institution_<id>` → `store.ts` listener → `xApiSlice.util.invalidateTags([...])` → mounted queries refetch.

## 11. File upload
`ImageUpload` (crop) → `POST /staff/upload/image?module=students&subfolder=photos&prefix=student` → multer (5 MB, jpg/png/webp) → sharp → WebP ≤100 KB → R2 → returns `{url}` → saved on the entity (`profileImage` or `customFields[key]`). On replace or remove: `DELETE /staff/upload {url}`.

## 12. Git / release workflow observed
Branches `dev`, `roshan`, and personal branches merge into `dev`, then into `main`. Build commits are titled "staff build NN" and "admin build NN" (the `dist/` folders are committed). Before committing: `npx tsc --noEmit` in both frontends.
