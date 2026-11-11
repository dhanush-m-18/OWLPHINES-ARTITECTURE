# 1. Storage and default fields

## 1.1 Where an event is stored: `tbl_module_records` (already exists)
```prisma
model ModuleRecord {
  id                  String  @id @default(uuid())
  institutionId       String              // which school   ← from req.user (login token)
  institutionModuleId String              // which module   ← this school's 'events' module
  data                Json    @db.JsonB   // ALL values of the event
  createdBy           String?             // req.user.staffId
  updatedBy           String?
  isDeleted           Boolean @default(false)   // soft delete
  createdAt, updatedAt
  @@map("tbl_module_records")
}
```
One event equals one row. Real example produced by the test:
```json
data = {
  "title": "Claude Test Event",
  "startDate": "2026-11-20", "endDate": "2026-11-21",
  "startTime": "09:00",      "endTime": "13:00",
  "description": "api test", "isActive": true
}
```
A custom field added later in Modify Form (e.g. Venue) simply becomes another key: `"venue": "Main Ground"`.

**Why no new table:** the senior chose to reuse the generic table. `institutionModuleId` separates Events from other generic modules, and `institutionId` separates schools.

**Formats chosen inside `data`:**
| Type | Stored as | Reason |
|---|---|---|
| Date | `"YYYY-MM-DD"` | sorts and compares correctly as text (`'2026-11-20' < '2026-11-21'`), so range queries need no casts |
| Time | `"HH:mm"` 24h | sorts correctly as text; it's also what the shared `TimePicker` component emits; the UI shows AM/PM |
| Toggle | `true` / `false` | real JSON booleans |
| Text | trimmed string | |

## 1.2 Default fields: `backend/src/config/defaultFields.config.js`

### Section added: `defaultEventFields`
```js
// Events are stored in tbl_module_records (data JSONB) — these fieldKeys are the keys inside `data`.
export const defaultEventFields = [
  { fieldKey: 'title',       label: 'Event Title', fieldType: 'Textbox',  order: 1, isSystemField: true, required: true,
    options: { minLength: 2, maxLength: 150, showInListing: true, showInExport: true } },
  { fieldKey: 'startDate',   label: 'Start Date',  fieldType: 'Date',     order: 2, isSystemField: true, required: true, ... },
  { fieldKey: 'endDate',     label: 'End Date',    fieldType: 'Date',     order: 3, isSystemField: true, required: true, ... },
  { fieldKey: 'startTime',   label: 'Start Time',  fieldType: 'Time',     order: 4, isSystemField: true, required: false, ... },
  { fieldKey: 'endTime',     label: 'End Time',    fieldType: 'Time',     order: 5, isSystemField: true, required: false, ... },
  { fieldKey: 'description', label: 'Description', fieldType: 'Textarea', order: 6, isSystemField: true,
    options: { maxLength: 1000, showInListing: false, showInExport: true } },
  { fieldKey: 'isActive',    label: 'Status',      fieldType: 'Toggle',   order: 7, isSystemField: true,
    options: { trueLabel: 'Active', falseLabel: 'Inactive', defaultValue: true, ... } },
];
```
What each property does:
- `fieldKey`: the key inside `data` (and in the API). It never changes.
- `isSystemField: true`: the school can't delete it (only hide it). The API returns it at the top level, not under `customFields`.
- `order: 1` on `title`: Modify Form treats the order-1 system field as the primary field (always visible, required and first).
- `required`: checked by `validateModuleFields` in the backend and by the form in the frontend.
- `options.showInListing` / `showInExport`: whether the field becomes a table column or an export column. Description is hidden from the table because it's long.
- `fieldType: 'Time'`: rendered by the shared `TimePicker` (already supported by `DynamicFormField`).

### Line added to the registry
```js
export const DEFAULT_MODULE_FIELDS = {
  ...
  timetable: defaultTimetableFields,
  events: defaultEventFields,   // ← added
};
```
**Why it's needed:** `ensureDefaultFields(instModuleId, 'events')` looks up `DEFAULT_MODULE_FIELDS['events']`. Without this line, no field rows are created, and the form, table and validation have nothing to work with. (During testing, the first event was saved with an empty `data` for exactly this reason. That row was soft-deleted afterwards, and a guard was added to the controller; see 03.)

### What happens automatically
The first time ABC's Events fields are loaded, 7 rows are inserted into `tbl_institution_module_fields` (already done for ABC during testing):
```
1 title Textbox sys req
2 startDate Date sys req
3 endDate Date sys req
4 startTime Time sys
5 endTime Time sys
6 description Textarea sys
7 isActive Toggle sys
```
Other schools get theirs the first time they open Events. Custom fields the school adds later are extra rows with `isSystemField: false`.
