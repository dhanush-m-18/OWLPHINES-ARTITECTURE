# 6. Evaluation of the Subjects implementation

Each item was confirmed by reading the code (file:line) and, where marked **[DB]**, by checking the live staging data for ABC.

## ✅ What's good (keep this when replicating)
- **Config-driven UI:** table columns, form inputs, export columns and filters all come from field rows. No hard-coded column list in JSX.
- **Double validation:** the frontend validator mirrors the backend `validateModuleFields`, so the UX is fast and the server is still authoritative.
- **Tenant safety:** every query uses `institutionId` from the JWT. Updates and deletes find with `{id, institutionId, isDeleted:false}` first, and bulk delete puts `institutionId` inside `updateMany.where`.
- **Soft delete** everywhere, plus **revive on re-create** (works with the per-tenant unique code that includes deleted rows).
- **Raw SQL is parameterised** (`$queryRaw`, `Prisma.sql`, `Prisma.join`). Sort direction comes from fixed fragments. No injection found.
- **Natural sort** of codes in SQL. **Debounced search**, cached filter options, skeletons, no-change guards in both the form and the customiser.
- **Real-time:** `subjects_updated` / `module_fields_updated` keep every open screen in sync.
- **System fields protected:** they can't be deleted, the primary key field is forced visible, required and first, and admin-only choices can't be overwritten by the school.

## 🔴 High

### E-1. Anyone logged in can change the Subjects form layout
`moduleField.routes.js` protects writes with `superadminMiddleware`, which only checks that the staff member exists and is active (**no role check**). The UI hides "Modify Form" unless the user has `edit`, but a teacher with only `read` can call `PUT /api/staff/modules/subjects/fields/bulk` directly to hide or relabel fields, make fields required, or delete custom fields for the whole school.
**Fix:** check the SUPERADMIN role in `superadminMiddleware`, or use `requirePermission(moduleKey,'edit')` on the field routes.

## 🟠 Medium (functional bugs)

### E-2. Sorting by a custom-field column breaks the list
`Subjects.tsx` lets every header be clicked (`handleSort(field.fieldKey)`). The backend non-code path does `orderBy: [{ [sortBy]: … }]` (`subject.controller.js:150`). For a custom key (e.g. `creditHours`) Prisma throws a validation error, so the user gets a 400 "Invalid data provided" and the table shows nothing. System columns (including the `classes` array) sort fine **[DB-probed]**.
**Fix:** whitelist sortable columns; for custom keys sort in SQL with `customFields->>key`.

### E-3. `classFilter` ignored on the default sort
`classFilter` is added only to the Prisma `where` (`:70-77`), but the default `sortBy='subjectCode'` uses the raw SQL path, which never reads `classFilter`. Any caller using `?classFilter=I` with the default sort gets unfiltered results. (The page itself filters with `?classes=I`, which does work in both paths.)

### E-4. Bulk import bypasses the dynamic rules
`bulkImportSubjects` checks only code, name and duplicates:
- No `validateModuleFields` (required custom fields, lengths, types, choices) and no `validateModuleUniqueFields`.
- **Every unknown column becomes a `customFields` key**: typos and extra Excel columns pollute JSONB.
- `subjectType` is uppercased and not checked against `tbl_subject_types`; `subjectTypeId` is never set.
- `createMany({skipDuplicates:true})`: if a **soft-deleted** subject has the same code, the row is silently skipped (unique includes deleted rows) but is still counted in `importedCount`.
- No zod schema and no row limit.

### E-5. Custom fields can never be unique
`ModuleFormCustomizerDrawer.tsx:359`: `isUnique = (isInstitutionModule || !f.isSystemField) ? false : …`. The unique toggle is ignored for custom fields, so the backend's JSONB unique check in `validateModuleUniqueFields` is unreachable from the UI.

### E-6. Runtime data saved into field options **[DB]**
The customiser receives **hydrated** fields (with `yearChoices`, `academicYears`, `currentYear`) and sends them back. The backend merges `...f.options` into storage. ABC's `classes` field options now permanently contain a 2026-27 snapshot. Hydration overwrites them on read, so the UI is fine today, but code that reads raw DB options (e.g. Students' `adminConfiguredYear = fields.find(f => f.options?.currentYear)`) can pick up **stale years**.
**Fix:** strip `yearChoices/academicYears/currentYear` before saving (frontend or backend).

### E-7. Stale options after a type change **[DB]**
`hasLab` (Toggle) stores `choices, adminOnly, isYearWise, minLength, maxLength`. The backend merge `{...found.options, ...f.options}` keeps old keys even when the frontend cleaned them. It's harmless now but confusing, and `adminOnly` on a toggle could lock behaviour later.

### E-8. Field-config GET has write side effects
`resolveInstModule` creates a **global** `Module` row and an InstitutionModule when the key isn't found. `ensureDefaultFields` upserts and soft-deletes rows. A GET with a typo'd moduleKey that matches a default key can create master modules from the staff portal.

## 🟡 Low / quality

| ID | Issue | Where |
|---|---|---|
| E-9 | `createdBy/updatedBy = req.user.id \|\| req.user.email`, but the staff JWT has neither, so it's always `null` **[DB]**. Use `req.user.staffId` | controller create/update/delete |
| E-10 | `limit` unbounded (`limit=all` returns everything) | `getSubjects` |
| E-11 | Every filter and search is implemented **twice** (Prisma + raw SQL) and they already differ: raw search covers `classFrom/classTo/array_to_string`, while Prisma search uses `classes hasSome` exact match and string_contains on custom keys | controller |
| E-12 | No zod validator on subject routes (other modules have one) | `subject.routes.js` |
| E-13 | Frontend `DEFAULT_SUBJECT_FIELDS` duplicates the backend defaults and has drifted (duplicate `order:6`; `isUnique` inside options) | `Subjects.tsx:123` |
| E-14 | Duplicate `order` values in the DB (4/4, 5/5, 7/7) **[DB]**, so ordering between them is arbitrary | field rows |
| E-15 | `subjectTypeId` mostly null **[DB]**; the string `subjectType` is the real link. Renaming a subject type doesn't cascade | create/update |
| E-16 | `updateSubject` **replaces** `customFields` entirely; a client sending a partial `customFields` wipes other custom values | `:515-523` |
| E-17 | `bulkUpdateModuleFields` runs N sequential queries with no transaction, so a mid-way failure leaves a half-saved form | `moduleField.controller.js:388` |
| E-18 | Re-adding a custom field with the same label as a deleted one upserts the old row, so **old hidden values reappear** (could be a feature or a surprise) | bulk upsert |
| E-19 | Frontend export button needs `report`; the backend only needs `read` (inconsistent gate) | page vs routes |
| E-20 | `classOptions` hard-coded fallback (PRE KG…Class 12) can show classes the school doesn't have | `Subjects.tsx:358` |
| E-21 | Page is ~1,270 lines in one component; `Switch` and `toCamelCase` are re-declared locally even though shared ones exist | `Subjects.tsx` |

## Verdict
The architecture (config rows, a generic renderer and validator, JSONB custom fields) is solid and worth copying. The weak spots are **authorization on field config (E-1)**, **the duplicated Prisma/SQL code paths (E-2, E-3, E-11)**, **the import path skipping the engine (E-4)**, and **options hygiene in the customiser (E-5 to E-7)**. When re-applying this to another module, copy the pattern but fix these points in the new module instead of copying them (see 07).
