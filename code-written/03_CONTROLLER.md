# 3. `backend/src/staff_side/controllers/event.controller.js`, section by section

The structure follows `subject.controller.js` (the senior's pattern): helpers first (resolver, field fetcher, value extractor, search/filter builder, export), then one handler per route. Every handler is `try { … } catch (e) { return next(e) }` and responds with `sendSuccess` / `sendError`.

---

## Section A: constants and small helpers
```js
const EVENT_MODULE_KEY = 'events';
const MAX_PAGE_LIMIT = 500;      // normal pages are clamped
const MAX_ALL_LIMIT = 1000;      // limit=all (calendar month) is capped too
const RESERVED_QUERY_KEYS = new Set(['page','limit','search','sortBy','sortOrder','order','from','to','fields','ids', ...]);
```
- `RESERVED_QUERY_KEYS`: query params that are **not** column filters. Every other param whose name is a configured `fieldKey` becomes a filter (`?venue=hall`).
- `parseMultiQueryValues('a,b')` gives `['a','b']`, so multi-value filters work (same helper as Subjects).
- `normalizeTime(val)`: accepts `"09:00"`, `"9:00"`, `"17:30:00"` and `"09:00 AM"`, returns `"HH:mm"`; `''` for empty; **`null` if invalid** (e.g. `25:00`), which becomes a 400.
- `formatTime12h("14:30")` gives `"02:30 PM"`, used only in exports.
- `normalizeDate(val)` uses the shared `parseFlexibleDate`, then `formatYYYYMMDD`. Accepts `DD-MM-YYYY` or `YYYY-MM-DD`, returns `"YYYY-MM-DD"`, or `null` if invalid.
- `toBoolean(val)`: `true`, `'true'`, `'1'`, `'yes'`, `'active'` give `true`.

## Section B: step 1, Resolver
```js
const resolveEventInstModule = (institutionId) => prisma.institutionModule.findFirst({
  where: { institutionId, isDeleted: false, module: { moduleKey: 'events' } }, select: { id: true } });
```
Finds **this school's** Events module. Its `id` is the `institutionModuleId` stored on every event row and on every field row.

## Section C: step 2, Field fetcher
```js
const getEventModuleFields = async (institutionId) => {
  const instMod = await resolveEventInstModule(institutionId);
  ... findMany(tbl_institution_module_fields WHERE institutionModuleId AND isDeleted=false ORDER BY order)
  if (fields.length === 0) { await ensureDefaultFields(instMod.id, 'events'); …reload }
  return { instModId: instMod.id, fields };
};
```
Same as `getSubjectModuleFields`: loads the form definition and **auto-seeds** the 7 defaults the first time.

## Section D: API shape adapter (Events-specific)
```js
const toEventResponse = (record, fields) => { … }
```
The DB stores everything flat in `data`. The frontend dynamic-field pattern expects **system keys at the top level and custom keys under `customFields`** (exactly like a Subject row). This helper splits them using `isSystemField`:
```
data {title, startDate, …, venue}   →   { id, title, startDate, …, customFields: { venue } }
```
**Why:** the Events page can reuse the same `fieldKey in formData` rule, `DynamicFormField` and `DynamicTableCell` with no special cases.

## Section E: building `data` from the request
```js
const buildEventData = (body, fields) => { … return { changes, errors }; }
```
For **each configured field** (and only those):
- System field: read from `body[fieldKey]`. Custom field: read from `body.customFields[fieldKey]`.
- Not sent: skipped (that's what makes partial updates work).
- Normalised by type: Date → `YYYY-MM-DD`, Time → `HH:mm` (invalid → error), Toggle → boolean, text trimmed, arrays cleaned.

**Unknown keys are dropped.** In the test, `customFields: {junkKey: "…"}` was sent and the stored event had `customFields: {}`. This avoids the Subjects problem of junk keys being saved.

`toValidationPayload(data, fields)` reshapes flat data into `{ ...systemKeys, customFields }`, the shape the shared `validateModuleFields` expects.

## Section F: rules the generic validator can't express
```js
const validateEventSchedule = ({ startDate, endDate, startTime, endTime }) => {
  if (endDate < startDate) return 'End Date cannot be earlier than Start Date.';
  if (startDate === endDate && startTime && endTime && endTime <= startTime)
    return 'End Time must be later than Start Time for a single-day event.';
};
```
Text comparison works because the formats are `YYYY-MM-DD` and `HH:mm`.

## Section G: unique fields inside JSON
```js
const validateEventUniqueFields = async ({ institutionId, instModId, fields, data, excludeId }) => {
  … prisma.moduleRecord.findFirst({ where: { institutionId, institutionModuleId: instModId, isDeleted: false,
        id: { not: excludeId }, data: { path: [fieldKey], equals: value } } })
};
```
The shared `validateModuleUniqueFields` looks at real columns or `customFields`, which don't exist on `tbl_module_records`, so this version checks the key inside `data`. No default event field is unique; this is for fields a school marks unique later.

## Section H: step 3, Value extractor (exports)
```js
const getEventFieldValue = (event, field) => …
```
Toggle gives its true/false label (Active/Inactive), Date gives `DD-MM-YYYY` (project standard), Time gives `hh:mm AM`, arrays are joined with commas, custom keys are read from `customFields`.

## Section I: step 4, Search, filter, sort and date-range builder (raw SQL)
```js
const buildEventQuerySql = ({ institutionId, instModId, fields, query }) => { … return { whereSql, orderSql }; }
```
Builds **parameterised** SQL fragments with `Prisma.sql` / `Prisma.join`. Every value, and even the JSON key, is a bound parameter (`r."data"->>${key}`), so there's no injection.

| Part | SQL produced (simplified) |
|---|---|
| Tenant + module + not deleted | `r."institutionId" = $1 AND r."institutionModuleId" = $2 AND r."isDeleted" = false` |
| Selected ids (export) | `r.id IN ($…)` |
| Calendar range `from`/`to` | `COALESCE(endDate, startDate) >= from AND startDate <= to`, which finds events that **overlap** the range, including multi-day events that start earlier |
| Search | `EXISTS (SELECT 1 FROM jsonb_each_text(r."data") kv WHERE kv.key IN (<visible keys>) AND kv.value ILIKE '%q%' …)`. It searches **values only** (not key names) of visible fields, plus date variants (`20-11-2026` matches `2026-11-20`) and time (`9:00 am` matches `09:00`). "active"/"inactive" match the status |
| Column filters | by field type: Toggle `= 'true'/'false'` (isActive defaults to true when missing), Date by variants, Time exact, Dropdown case-insensitive equal, MultiSelect JSON array contains (`@>`), text `ILIKE` |
| Sort | only a **configured fieldKey** is accepted (anything else falls back to `startDate`); `ORDER BY r."data"->>key ASC/DESC NULLS LAST`; sorting by startDate also sorts by startTime; final tie-break `createdAt DESC` |

**Why a whitelist on sort:** Subjects crashes when sorting by a custom key (its `orderBy` goes straight to Prisma). Here any field, including custom ones, can be sorted safely through SQL.

## Section J: handlers
### `getEvents` (GET /)
1. `getEventModuleFields` (404 if the school has no Events module).
2. Page and limit are clamped (≤500; `limit=all` capped at 1000).
3. Builds `whereSql`/`orderSql`, then runs **COUNT** and **SELECT** in parallel on `tbl_module_records`.
4. Maps rows with `toEventResponse`. `filterOptions` come from the shared `populateDynamicFieldFilterOptions` (e.g. Active/Inactive).
5. Responds `{ data, pagination, filterOptions, meta: { pagination, filterOptions } }`, the same shape as Subjects.

### `getEventById` (GET /:id)
`findFirst({ id, institutionId, institutionModuleId, isDeleted: false })`. A wrong school or deleted event gives 404.

### `createEvent` (POST /)
```
0 module exists? fields configured? (guard added after the test bug: no fields → clear 500, never an empty row)
1 buildEventData → only configured keys, normalised (bad date/time → 400)
2 data = { isActive: true, ...changes }
3 validateModuleFields (required / length / type / choices, from the DB field rows) → 400
4 validateEventSchedule (date/time order) → 400
5 validateEventUniqueFields → 400
6 prisma.moduleRecord.create({ institutionId, institutionModuleId, data, createdBy: staffId, updatedBy: staffId })
7 emitToInstitution(institutionId, 'events_updated') → 201
```
`createdBy` uses `req.user.staffId`, which really is in the token (Subjects uses `req.user.id`, which is always empty).

### `updateEvent` (PUT /:id)
Finds the event (tenant + module + not deleted), builds `changes`, validates **only the sent keys** (`isUpdate: true`), then
```js
const merged = { ...existing.data, ...changes };   // MERGE, never replace
```
Keys not sent, including values of hidden custom fields, are kept. (Subjects replaces the whole `customFields` object, which can erase data.) Then the schedule is re-checked on the merged data, followed by the unique check excluding itself, update, and emit.

### `deleteEvent` (DELETE /:id) and `bulkDeleteEvents` (POST /bulk-delete)
**Soft delete**: `isDeleted = true` (`tbl_module_records` has no `isActive` column). Bulk uses `updateMany` with `institutionId` + `institutionModuleId` inside `where`, so IDs from another school or module are simply ignored.

## Section K: step 5, Export pipeline
`getEventExportData(institutionId, selectedKeys, query)` picks the columns (requested `fields=` keys, otherwise visible fields with `showInExport`), reuses **the same** `buildEventQuerySql` (so the export matches the list's search, filters and selection), formats values with `getEventFieldValue`, and returns `{ headers, rows }`.
- `exportEventExcel` uses the shared `generateGenericExcel` (institution header and branding).
- `exportEventPdf` uses the shared `generateGenericBulkPdf` (streams the PDF).

Unlike Subjects, the list and export share **one** query builder, so they can't drift apart.

---

## Differences from Subjects (deliberate improvements)
| Subjects | Events |
|---|---|
| Separate Prisma path and raw-SQL path for filters | One SQL builder used by list **and** export |
| Sorting by a custom field crashes | Any configured field sorts safely |
| `classFilter` ignored on the default sort | No special params outside the builder |
| `customFields` replaced on update | Merged on update |
| Unknown keys saved into `customFields` | Unknown keys dropped |
| `createdBy = req.user.id` (always empty) | `createdBy = req.user.staffId` |
| No zod validator | zod validator on create, update and bulk delete |
| `limit` unbounded | Clamped (500 / 1000) |
