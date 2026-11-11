# Code Conventions (how the senior devs write it)

The source of truth is `AGENTS.md` + `.agents/skills/owlphin-architecture/SKILL.md`. This file is the practical, observed version.

## 0. Core principle
**Zero divergence plus reusability-first.** Modules that look alike (Students, Staff, Subjects, Class & Batch, Bank Accounts, Fee *) must be built *identically*. If logic is used in 2+ places, extract it to `utils/`, `components/ui/` or `hooks/`.

---

## 1. Backend (JavaScript, ESM)

### Style
- ES modules (`import … from '../../x.js'`, **always with the `.js` extension**).
- 2-space indent, single quotes, semicolons, trailing commas in multi-line objects.
- `export const fn = async (req, res, next) => { … }`, so arrow functions everywhere. No classes.
- Section separators in big controllers:
  ```js
  // ---------------------------------------------------------------------------
  // Helper: Resolve active InstitutionModule for 'student' moduleKey
  // ---------------------------------------------------------------------------
  ```
  and `// ===…===` banners with `// GET /students/:id — description` above each handler.
- Numbered step comments inside long handlers: `// 1. Verify Student`, `// 2. Validate…`.
- Log prefixes in brackets: `console.error('[R2 Service] …')`, `'[sendOtp] …'`.

### Controller skeleton (every handler)
```js
export const getThings = async (req, res, next) => {
  try {
    const { institutionId } = req.user;              // tenant ALWAYS from token
    // …prisma work scoped by institutionId + isDeleted:false
    return sendSuccess(res, 200, 'Things retrieved successfully', data, { pagination, meta });
  } catch (error) {
    return next(error);                               // central errorHandler formats it
  }
};
```
- Expected failures: `return sendError(res, 4xx, 'Human message')`.
- Services throw **plain objects**: `throw { statusCode: 403, message: '…' }`. The errorHandler reads `statusCode` and `message`.
- Messages are full English sentences: `'Student not found'`, `'Student created successfully'`.

### Response shape (never deviate)
```json
{ "success": true, "message": "…", "data": …, "pagination": {…}, "meta": {…} }
{ "success": false, "message": "…", "errors": ["…"] }
```
List endpoints return `pagination` **and** duplicate it inside `meta` (with `filterOptions` and `tabCounts`) for frontend compatibility.

### Multi-tenancy and soft delete (mandatory)
```js
where: { id, institutionId, isDeleted: false }       // read/update target lookup
data:  { isDeleted: true, isActive: false }          // delete
data:  { isDeleted: false, isActive: true }          // restore
```
- Fetch-then-mutate: `findFirst({ where:{ id, institutionId, isDeleted:false } })`, then 404 if missing, then `update({ where:{ id } })`. The ownership check happens in the `findFirst`.
- Related rows that must change together go in `prisma.$transaction([...])` or `prisma.$transaction(async (tx) => …)`.

### Route file pattern
```js
const router = Router();
router.use(authMiddleware, staffMiddleware);
// static paths (export, bulk-*) BEFORE '/:id'
router.get('/export/excel', requirePermission('students', 'read'), exportStudentExcel);
router.get('/', requirePermission('students', 'read'), getStudentList);
router.post('/', requirePermission('students', 'add'), validateRequest(createStudentSchema), createStudent);
router.put('/:id', requirePermission('students', 'edit'), validateRequest(updateStudentSchema), updateStudent);
router.delete('/:id', requirePermission('students', 'delete'), deleteStudent);
router.post('/:id/restore', requirePermission('students', 'delete'), restoreStudent);
export default router;
```
Actions: `read | add | edit | delete | approve | report`. Restore uses `delete` permission. Exports use `read`.

### Validators (zod)
```js
export const createXSchema = {
  body: z.object({
    name: stringOrNumber.refine((v) => v.length >= 2, 'Name is required (min 2 characters)'),
    isActive: booleanToggle(true),       // accepts true/'true'/1/'1'
    customFields: z.any().optional().nullable(),
  }).passthrough(),                      // ⚠ house style passes unknown keys through
};
```
Use `utils/zodHelpers.js` (`booleanToggle`, `stringOrNumber`, `stringOrNumberOptional`) because the frontend and Excel imports send mixed types.

### The 5-step listing pipeline (Students / Staff / Subjects / ClassBatch / Fees)
Every operational controller has these helpers, in this order:
1. **Resolver**: `resolve<X>InstModule(institutionId)` finds the InstitutionModule by `moduleKey`.
2. **Field fetcher**: `get<X>ModuleFields(institutionId, visibleOnly)` loads `InstitutionModuleField` rows and auto-seeds defaults with `ensureDefaultFields`.
3. **Value extractor**: `get<X>FieldValue(record, fieldKey, fieldType)` turns a system column or `customFields[key]` into a display string (dates as `DD-MM-YYYY`, toggles as Yes/No).
4. **Search builder**: `build<X>SearchConditions(activeFields, q)` returns a Prisma `OR[]` built dynamically from configured fields (dates via `buildDateSearchConditions`).
5. **Export pipeline**: `get<X>ExportData()` builds the row data, then `generateGenericExcel` / `generateGenericBulkPdf` from `services/export.service.js`.

Plus `build<X>FilterConditions(activeFields, req.query)` for the column filter row, and a `filterOptions` builder for dropdown values.

### Raw SQL rules
- Only `prisma.$queryRaw` tagged templates (parameterised). Compose fragments with `Prisma.sql`, `Prisma.empty` and `Prisma.join`.
- `Prisma.raw()` only with **hard-coded whitelisted** identifiers (e.g. `colName = sortBy === 'staffId' ? 'staffId' : 'staffKey'`).
- **Verify the table name in the `@@map("…")` of the `.prisma` file** (`tbl_staff`, `tbl_student_info`, `tbl_roles`, `tbl_department`, `tbl_shift`, `tbl_class_batches`, `tbl_module_records`, …).
- Natural numeric sort: `ORDER BY NULLIF(regexp_replace(col,'\D','','g'),'')::bigint`.

### Dates
- Incoming: `parseFlexibleDate(val)` (handles `DD-MM-YYYY`, `YYYY-MM-DD`, `/` variants, ISO, and ignores bare `1995` and `05-1995`).
- Output in exports: `formatDDMMYYYY`.
- Search: `buildDateSearchConditions(key, q, isSystemField)` and `getDateSearchVariants(q)`.
- Stored: system columns are `DateTime` (UTC midnight). Custom JSONB dates are `'YYYY-MM-DD'` strings.

### Real-time
After every successful mutation: `emitToInstitution(institutionId, '<entity>_updated', { minimal payload })`. Event names are `snake_case`, usually `<entity>_updated` or `<entity>_created/_deleted/_approved`.

### Naming
| Thing | Convention | Example |
|---|---|---|
| Files | `camelCase.<layer>.js` | `feeStructure.controller.js` |
| Handlers | verb+Entity | `getStudentList`, `bulkImportStudents`, `exportStudentPdf` |
| Prisma models | PascalCase singular | `StudentInfo`, `FeeHead` |
| Tables | `tbl_snake_case` (singular/plural inconsistent, so check!) | `tbl_staff`, `tbl_roles` |
| moduleKey | lowercase, `_` for groups | `students`, `classandbatch`, `fees_heads` |
| URLs | kebab-case plural | `/class-batches`, `/fees/bank-accounts` |

---

## 2. Frontend (TypeScript/TSX, React 19)

### Style
- Function components, named exports for pages (`export const StudentPage = …`) and default export for `App`.
- TS is **loose** (`strict:false`). `any` is common. Prefer typing new code anyway.
- Tailwind utility classes only, plus theme classes (`bg-surface`, `border-surface`, `text-main`, `text-muted`, `btn-primary`, `text-theme-primary`).

### Redux / RTK Query
```ts
export const thingsApiSlice = createApi({
  reducerPath: 'thingsApi',
  baseQuery: fetchBaseQuery({ baseUrl: API_BASE_URL, credentials: 'include' }),
  tagTypes: ['Things'],
  endpoints: (builder) => ({
    getThings: builder.query({ query: (params) => ({ url: '/staff/things', params }), providesTags: ['Things'] }),
    createThing: builder.mutation({ query: (body) => ({ url: '/staff/things', method: 'POST', body }), invalidatesTags: ['Things'] }),
  }),
});
export const { useGetThingsQuery, useCreateThingMutation } = thingsApiSlice;
```
- Use it as `useGetThingsQuery(params, { refetchOnMountOrArgChange: true })`. **Always pass this.**
- Register the slice in **3 places**: `combineSlices(...)`, `.concat(x.middleware)` in `app/store.ts`, and `staffApi.util.resetApiState` in `store/apiSlice.ts`.
- Use `useAppDispatch` / `useAppSelector`, never the raw hooks.

### Page layout (Unified Card Standard): copy `ClassBatch.tsx` / `Subjects.tsx`
```
<div className="h-full flex flex-col min-h-0">
  <div className="bg-surface border border-surface rounded-2xl shadow-xs flex-1 flex flex-col min-h-0 overflow-hidden">
    header:  px-4 sm:px-5 py-2.5 sm:py-2 border-b border-surface …  title | <ModifyFormButton/> <AddButton/>
    toolbar: px-4 sm:px-5 py-2.5 border-b border-surface …           <SearchInput/> | <FilterButton/> <ExportExcelButton/> <ExportPdfButton/> Choose Action (n)
    table:   flex-1 overflow-auto min-h-0 relative                   sticky thead, sort arrows, <TableColumnFilterRow/>, <EditActionButton/> <DeleteActionButton/>
    footer:  <PaginationFooter/> px-4 sm:px-5 py-2.5 border-t border-surface
```
Loading shows `TableSkeleton` / `PageSkeleton` / `TableFilterBodySkeleton`, never a blank area or spinner text. Empty data shows `<EmptyState/>`.

### Icons
- Module tabs, headers and cards use **solid** icons from `react-icons/fa6`, `md`, `ri`, `pi` or `bi`.
- Action and utility buttons (edit, delete, close, search, chevrons) use `lucide-react` line icons.
- Sidebar uses `<FilledIcon name={module.icon}/>` (the icon name comes from the DB).

### Forms
- `<form noValidate>` with inline errors via `<FieldError/>` (no browser tooltips).
- Field rules: phone `^[6-9]\d{9}$` max 10 digits numeric; Character `[A-Za-z\s]`; Number digits only; Date via `<DatePicker/>` emitting `YYYY-MM-DD`.
- **No-Change Toast Guard:**
  ```ts
  if (editing && initialFormData && JSON.stringify(formData) === JSON.stringify(initialFormData)) {
    toast.info('No changes found'); setIsDrawerOpen(false); setEditing(null); return;
  }
  ```
- Delete always goes through `<ConfirmModal/>`. Exports go through `<ExportModal/>` (column picker).
- Toasts: `toast.success(res?.message || '…')`, `toast.error(err?.data?.message || '…')`.

### Permissions in UI
`const { read, add, edit, delete: canDelete } = useModulePermission('students');` gates buttons. This is cosmetic, because the backend enforces.

### Dates in UI
Display with `formatDate()` from `utils/date.ts` (DD-MM-YYYY). Inputs use `YYYY-MM-DD`.

---

## 3. Before commit (from AGENTS.md)
```bash
cd Staff_ERP && npx tsc --noEmit -p tsconfig.app.json
cd Admin_ERP && npx tsc --noEmit -p tsconfig.app.json
```
Never commit `.env`. Make sure the backend runs on `localhost:5001`.

## 4. Quality observations on existing code (be aware, don't copy blindly)
- Very large controllers (2,000+ lines) with similar helpers repeated across modules. The rule says extract to utils, but in practice they're duplicated per module.
- The `validate.middleware.js` and `auth.middleware.js` files are duplicated between admin_side and staff_side.
- `.passthrough()` zod schemas plus destructuring in the controller: only destructured fields get saved, so it's OK, but don't spread `req.body` into Prisma `data`.
- `req.user.id` and `req.user.name` are used for `createdBy`/`updatedBy`, but the staff JWT only has `staffId`, so audit columns end up `null`/`undefined`. Use `req.user.staffId`.
- No automated tests, and TS strict is off.
