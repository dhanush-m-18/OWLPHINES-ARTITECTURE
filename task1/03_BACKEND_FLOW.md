# 3. Backend flow: `backend/src/staff_side/`

## 3.1 Routes (`routes/subject.routes.js`, mounted at `/api/staff/subjects`)
```
router.use(authMiddleware, staffMiddleware)
GET    /export/excel     requirePermission('subjects','read')   exportSubjectExcel
GET    /export/pdf       requirePermission('subjects','read')   exportSubjectPdf
GET    /                 requirePermission('subjects','read')   getSubjects
POST   /bulk-import      requirePermission('subjects','add')    bulkImportSubjects
POST   /bulk-delete      requirePermission('subjects','delete') bulkDeleteSubjects
POST   /                 requirePermission('subjects','add')    createSubject
GET    /:id              requirePermission('subjects','read')   getSubjectById
PUT    /:id              requirePermission('subjects','edit')   updateSubject
DELETE /:id              requirePermission('subjects','delete') deleteSubject
```
- Static paths (`/export/*`, `/bulk-*`) are declared **before** `/:id`, otherwise Express would treat "export" as an id.
- ⚠ There's **no zod `validateRequest`** here (Students and Staff have one). All validation happens inside the controller via the dynamic validator.

Field config routes (`routes/moduleField.routes.js`, mounted at `/api/staff/modules/:moduleId/fields`):
```
GET  /            getModuleFields        (login only)
POST /seed        superadminMiddleware   seedModuleFields
PUT  /bulk        superadminMiddleware   bulkUpdateModuleFields   ← used by Modify Form
POST /            superadminMiddleware   createModuleField
PUT  /reorder     superadminMiddleware   reorderModuleFields
PUT  /:fieldId    superadminMiddleware   updateModuleField
DELETE /:fieldId  superadminMiddleware   deleteModuleField
```

## 3.2 Middleware chain for `GET /api/staff/subjects`
1. `authMiddleware` reads the `staff_token` cookie, `jwt.verify`, and sets `req.user = {staffId, institutionId, roleId, roleName, userType}`.
2. `staffMiddleware` requires userType STAFF or MANAGEMENT.
3. `requirePermission('subjects','read')` finds the InstitutionModule with `moduleKey='subjects'` for this institution (it must be enabled), then checks the staff override, then the role permission. SUPERADMIN bypasses this.
4. Controller.

## 3.3 Controller helpers (the "5-step pipeline")
| Step | Helper | What it does |
|---|---|---|
| 1 Resolver | `resolveSubjectInstModule(instId)` | Finds the InstitutionModule for `subject`/`subjects` |
| 2 Field fetcher | `getSubjectModuleFields(instId, onlyVisible)` | Loads `tbl_institution_module_fields`; if empty, runs `ensureDefaultFields(…,'subjects')` and reloads |
| 3 Value extractor | `getSubjectFieldValue(subject, key, type)` | Turns a record into a display string for export (Yes/No, Active/Inactive, joined classes, DD-MM-YYYY, `customFields[key]`) |
| 4 Search builder | `buildSubjectSearchConditions(fields, q)` | Builds a Prisma `OR[]` from config: system text columns use `contains`, `classes` uses `hasSome`, toggles respond to yes/no/active words, custom Toggle/Date/text use JSONB `path` queries |
| 5 Export pipeline | `getSubjectExportData()`, then `generateGenericExcel` / `generateGenericBulkPdf` | Builds headers and rows from config (selected or visible fields) |
| + | `buildSubjectFilterConditions(fields, query)` | Column filters (Prisma). Every query param that isn't reserved is treated as a fieldKey filter; multi-values are comma-separated |
| + | `buildSubjectSqlFilterConditions(fields, query)` | The **same filters as raw SQL** fragments for the natural-sort query |
| + | `buildSubjectDynamicFilterOptions()` | Dropdown values for filters: distinct ClassBatch classNames, subject types, plus `populateDynamicFieldFilterOptions` (Toggle labels, Dropdown choices) |

## 3.4 `getSubjects` (list)
```
params: search, page=1, limit=10 ('all' = no paging), sortBy='subjectCode', order='desc', classFilter, <any fieldKey>=value
where = { institutionId, isDeleted:false } + AND filterConditions + OR searchConditions (+ classFilter)

if sortBy === 'subjectCode':        ← DEFAULT path
   raw SQL on tbl_subjects:
     WHERE institutionId AND isDeleted=false + filterSql + searchCondition
     ORDER BY NULLIF(regexp_replace(subjectCode,'\D','','g'),'')::bigint DESC NULLS LAST, subjectCode, createdAt
     LIMIT/OFFSET                      (natural sort: 2 < 10 < 100)
     search here = LIKE on code/name/type/classFrom/classTo/array_to_string(classes) + customFields::text ILIKE (date variants)
else:
   prisma.subject.findMany({ where, orderBy:[{[sortBy]:order},{createdAt:'desc'}], skip, take }) + count

filterOptions = buildSubjectDynamicFilterOptions()
→ sendSuccess(200, 'Subjects retrieved successfully', subjects, { pagination, filterOptions, meta:{pagination, filterOptions} })
```
Why two paths: Prisma `orderBy` can't do natural numeric sorting, so the default code sort uses raw SQL. That forces every filter to be written twice (Prisma and SQL), which is where the bugs in 06 come from.

## 3.5 `createSubject`
```
1  Destructure fixed fields from req.body (never spread req.body into data)
2  Manual required checks: subjectCode, name, subjectType  → 400
3  Normalise classes: array | "a,b" string | req.body.class | [classFrom, classTo]
4  isActive: top-level wins, else customFields.isActive, else true; strip isActive/status from customFields
5  subjectCode → trim + UPPERCASE; name → trim
6  moduleFields = getSubjectModuleFields(instId, false)
   validateModuleFields(fields, payload, null, {isUpdate:false})  → type/required/length/choices (first error → 400)
   validateModuleUniqueFields(prisma.subject, instId, fields, payload) → isUnique system cols / JSONB keys
7  Explicit duplicate check: same code, not deleted → 400 "already in use"
8  If a SOFT-DELETED subject with the same code exists → REVIVE it (update + isDeleted:false)
   (needed because @@unique([institutionId, subjectCode]) also covers deleted rows)
9  Resolve subjectTypeId by name (uppercase) in tbl_subject_types (optional link)
10 create → tbl_subjects { institutionId, subjectCode, name, subjectType, subjectTypeId, hasLab, hasTheory,
            isElective, classes, classFrom=classes[0], classTo=classes[last], customFields, isActive, createdBy }
11 emitToInstitution(instId, 'subjects_updated') → 201
```

## 3.6 `updateSubject`
Fetch `{id, institutionId, isDeleted:false}` (404 if missing), then `validateModuleFields(…, {isUpdate:true})` (only fields present in the body are validated), then the unique check excluding itself. Then build `updateData` **only for keys present** (partial update): a code conflict check if the code changed, re-resolve type, re-normalise classes, set classFrom/classTo, strip `isActive` from customFields. `customFields` is **replaced as a whole object** (not merged), which is why the frontend always sends the full `customFields`.

## 3.7 Delete, bulk delete, getById
- `deleteSubject`: find, then `update { isDeleted:true, isActive:false }` (soft), then emit.
- `bulkDeleteSubjects`: `updateMany({ id in ids, institutionId, isDeleted:false })`. The tenant filter inside `where` makes foreign ids harmless.
- `getSubjectById`: `findFirst({id, institutionId, isDeleted:false})`.

## 3.8 `bulkImportSubjects` (`POST /bulk-import {rows:[…]}`)
Accepts either camelCase keys or Excel header names (`'Subject Code'`, `'Lab Included'`…). It checks required code and name, and duplicates against the DB and within the file. **Every unrecognised column goes into `customFields`**. Then `createMany({skipDuplicates:true})` and it returns `{importedCount, errorCount, errors}`. (No dynamic validation; see 06.)

## 3.9 Export
`GET /export/excel?fields=subjectCode,name,classes&sortBy&sortOrder&search&ids&<filters>` builds `getSubjectExportData` (fields = requested keys or visible fields; same filter and search builders; optional `ids`), then `generateGenericExcel` (ExcelJS, institution header) or `generateGenericBulkPdf` (PDFKit, streamed). Optional S.No column (`includeSlNo`).

## 3.10 Validation engine (`utils/fieldValidator.js`)
`validateModuleFields` loops over **visible** fields. The value comes from `payload[fieldKey]` (system field, or any top-level key) or from `payload.customFields[fieldKey]`. Then `validateSingleField`:

| type | rule |
|---|---|
| required (any) | empty means an error (auto keys like studentKey/staffKey are skipped) |
| contact/phone | exactly 10 digits, digits only |
| number | numeric regex + min/max **digit count** |
| character | letters and spaces + length |
| textbox/text/textarea | length |
| email | regex |
| date | `parseFlexibleDate` |
| dropdown/select | value must be in `yearChoices[year]` or `choices` (case-insensitive), **only if choices are configured** |
| multiselect, toggle, upload… | no type rule (only required) |

`validateModuleUniqueFields`: for each visible `isUnique` field, `findFirst` on the model, scoped by `institutionId` and `isDeleted:false`, excluding the current id. System column: `equals insensitive`. Custom: JSONB `path equals` (string or number).

## 3.11 Field-config endpoints (`moduleField.controller.js`)
- `resolveInstModule(instId, 'subjects')` tries id, moduleId and moduleKey variants (plural/singular, `_`/`and`). ⚠ If not found, it **creates** the master Module and InstitutionModule on the fly (even from a GET).
- `getModuleFields`: resolve, then `ensureDefaultFields`, then load non-deleted fields ordered, then `hydrateFieldsWithAcademicOptions` (inject classes and years). Details in 05.
