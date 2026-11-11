# Complex Implementations (deep dives)

## 1. Permission resolution: `staff_side/services/permission.service.js` + `middlewares/permission.middleware.js`
```js
requirePermission(moduleKey | [keys] | null, action | [actions])
```
- `null` key falls back to `req.params.moduleKey || req.params.moduleId` (used by generic `/modules/:moduleId/records`).
- An array of keys tries each one (e.g. `['fees_heads','fees']`), so a school can enable either a parent "Fees" module or the specific sub-module.
- `resolveModulePermission` finds the InstitutionModule by `id`, `moduleId` **or** `module.moduleKey` (lowercased). If it's disabled or deleted, it denies. Otherwise it checks the **staff override** (`staffId_institutionModuleId`), then the **role permission** (`roleId_institutionModuleId`), then all false.
- SUPERADMIN (by `roleName` in the JWT) and `userType: MANAGEMENT` skip everything.
- On success: `req.instModule`, `req.modulePermissions`.
- ⚠ Bug: the 403 message uses an undefined variable `targetKey`, which throws a ReferenceError, so a **denied** request returns **500** instead of 403. The deny still happens, but the status code is wrong. Fix: use `targetKeys[0]`.
- ⚠ `roleName` comes from the JWT, so a role change only takes effect after re-login (token lifetime 7 days).

`getStaffAccessibleModules()` does the same resolution for **all** enabled modules in 3 queries (modules, overrides, role perms, then Maps), avoiding N+1. It powers the sidebar.

## 2. Dynamic fields and validation: `utils/fieldValidator.js`, `config/defaultFields.config.js`
- `InstitutionModuleField.options` JSON holds `minLength`, `maxLength`, `choices[]`, `yearChoices{year:[…]}`, `isYearWise`, `adminOnly`, `showInListing`, `showInExport`, `trueLabel/falseLabel`, `defaultValue`, `currentYear`.
- `validateSingleField(field, value, year)` switches on `fieldType`: contact/phone (exactly 10 digits), number, character (letters and spaces), text (length), email (regex), dropdown (value ∈ choices or yearChoices[year], case-insensitive), date (`parseFlexibleDate`), and so on. Auto keys (`studentKey`, `staffKey`) skip the required check.
- `validateModuleFields(fields, body, year, {isUpdate})` looks in both `body[key]` and `body.customFields[key]`. On update, only fields that are present get validated.
- `validateModuleUniqueFields(prismaDelegate, instId, fields, body, excludeId)` checks `isUnique` fields against system columns or the JSONB path.
- `ensureDefaultFields(instModuleId, moduleKey)` seeds `DEFAULT_MODULE_FIELDS[moduleKey]` the first time a module's fields are requested. That's why new tenants "just work".

## 3. Student list engine: `student.controller.js#getStudentList`
1. In parallel: teaching-staff scope (`getTeachingStaffClassBatches`) and visible fields.
2. A teaching staff member with 0 classes gets an immediate empty response.
3. Build `where`: tenant + tab (active/inactive/alumni/trash) + teaching scope (`OR` of class/stream/section on StudentInfo **or** `yearWise.some`) + `currentYear` (`yearWise.some`) + column filters + dynamic search `OR`.
4. In parallel: `count(where)` plus 4 tab counts.
5. Sorting has 3 strategies:
   - `sortBy=studentKey`: **raw SQL** natural numeric sort (`CASE WHEN key ~ '^[0-9]+$' THEN key::BIGINT …`) with the same filters rebuilt as `Prisma.sql` fragments (`buildStudentSqlFilterConditions`, `buildStudentSqlSearchCondition`, `buildStudentSqlTeachingStaffCondition`). It fetches ids, then `findMany({id in})`, then restores the order with a Map.
   - System columns (`name`, `gender`, `dob`, `createdAt`): Prisma `orderBy` + skip/take.
   - Custom JSONB field or year-wise column: **fetches all matching rows**, sorts in JS with `localeCompare(…, {numeric:true})`, then slices the page. ⚠ This is O(N) memory per request (see PERF P-1).
6. `buildStudentDynamicFilterOptions` returns the distinct values for the filter dropdowns.

The same design is copied into Staff (`staff.controller.js`, sort by `staffId`/`staffKey` with whitelisted `Prisma.raw` column), Subjects, ClassBatch and the Fee modules.

## 4. Natural sort for padded keys
```sql
ORDER BY CASE WHEN s."studentKey" ~ '^[0-9]+$' THEN s."studentKey"::BIGINT ELSE 999999999999 END ASC,
         s."studentKey" ASC, s."createdAt" DESC
```
**Why:** keys are strings (`'0001'`, `'10000'`), and plain text order puts `'10000'` before `'2'`.

## 5. Key generation: `generate4DigitStudentKey`
Loads **all** keys of the tenant (including deleted ones, because of the `@@unique`) and returns the first free number padded to 4 digits. It fills gaps.
⚠ It runs **outside** the transaction, so two concurrent creates can pick the same key and the second fails with P2002 ("already exists"). It's also O(N) per create. Better: retry on P2002, or use a per-tenant counter row.

## 6. Date search: `utils/date.js`
- `parseFlexibleDate` handles DD-MM-YYYY, YYYY-MM-DD and `/` variants, builds `Date.UTC`, and refuses bare year or month-year (so "1995" isn't treated as 01-01-1995). ⚠ It doesn't reject impossible dates (31-02-2020 rolls over to 02-03-2020).
- `getDateSearchVariants(q)` expands a user query into all storage formats (`15-05-1995` → `1995-05-15`, `05-1995` → `1995-05`, `15-05` → `-05-15`).
- `buildDateSearchConditions(key, q, isSystemField)`: for system DateTime columns it builds `gte/lt` ranges (day, month or year). For JSONB string dates it builds `string_contains` for each variant. Raw SQL uses `TO_CHAR(col,'DD-MM-YYYY') ILIKE`.

## 7. Fee structure matching: `utils/feeStructureMatcher.js` + `resolveMatchingFeeStructure`
Candidate structures: same tenant + academicYear, active, not deleted, where `classes` contains the student's class. Then filter by `admissionType` (NEW_ADMISSION when the intake year = academic year, REGULAR otherwise, BOTH always), then by `scopeConditions` (JSON criteria such as stream, quota or category compared to student fields/customFields). The best match comes first by specificity and priority.

## 8. Fee payment allocation: `feePayment.controller.js#collectFeePayment`
- Per head: `pending = max(0, item.amount − approvedConcession(head) − alreadyAllocated(head))`.
- Heads are sorted by `feeHead.priority`, then `item.priority`. The payment is poured into them in order (waterfall).
- Overpayment remainder is added to the **first** head (advance).
- Installments: walk approved unpaid installments oldest-first and mark them paid or partially paid.
- Concessions: all approved, unapplied concessions for that year get `paymentStatus=1`.
- Receipt `RCP-<yyyy>-<seq 4-digit>` comes from the latest receipt by `createdAt`.
- See SECURITY_AND_PERFORMANCE.md (L-1 to L-4) for race conditions and precision issues.

## 9. Bulk import (Students/Staff/Subjects)
1. Pre-fetch fields and all existing records (lookup maps by key and unique fields).
2. Validate each row (`validateSingleField`, dropdown choices, duplicates within the file and against the DB) and collect `{row, message}` errors.
3. New rows: `createMany` in chunks of 500. Existing rows (matched by key): non-destructive partial updates in `$transaction([...])` chunks of 100. Then upsert year-wise rows.
4. Return a summary `{created, updated, failed, errors[]}` and emit once.

## 10. Image compression: `services/image/imageCompression.service.js`
`sharp(buffer).rotate()` (EXIF), resize to fit, then a WebP quality loop that steps down (and scales dimensions down) until ≤100 KB. It keeps alpha. It's used for logos, avatars, student photos and dynamic `Upload` fields.

## 11. Frontend data-driven routing: `App.tsx` + `config/moduleRegistry.ts`
URL `/:moduleParam` is matched against the user's `modules[]` by id, key, path or slugified name. Not found redirects to `/dashboard` (this works as a soft guard). Found: `getCustomModuleComponent(moduleKey)` tries the key, the singular form and the plural form. **Why:** paths and names are editable per tenant, but `moduleKey` is stable, so pages bind to `moduleKey`.

## 12. Socket to RTK cache bridge: `Staff_ERP/src/app/store.ts`
Module-level `socket.on(...)` calls register **once** on import and dispatch `invalidateTags`. Combined with `refetchOnMountOrArgChange: true`, this means screens are fresh both on navigation and on remote changes.
