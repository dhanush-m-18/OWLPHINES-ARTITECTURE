# 2. Frontend flow: `Staff_ERP/src/pages/Subjects.tsx` (~1,270 lines)

## 2.1 Files involved
| File | Role |
|---|---|
| `pages/Subjects.tsx` | The whole page: state, table, drawer form, validation, export, delete |
| `features/subjects/subjectsApiSlice.ts` | RTK Query: `getSubjects`, `getSubjectById`, `create`, `update`, `delete`, `bulkDelete`, `bulkImport` (tag `Subjects`) |
| `features/fields/fieldsApiSlice.ts` | `getModuleFields(moduleId)`, `bulkUpdateModuleFields` (tag `ModuleFields`) |
| `features/academicConfig/…` | `getAcademicConfig` returns current year and class list |
| `features/subjectTypes/…` | `getSubjectTypes` returns the subjectType dropdown choices |
| `components/dynamic-fields/DynamicFormField.tsx` | Renders one input by `fieldType` |
| `components/dynamic-fields/DynamicTableCell.tsx` | Renders one table cell by `fieldType` (custom fields) |
| `components/dynamic-fields/ModuleFormCustomizerDrawer.tsx` | The "Modify Form" editor |
| `components/ui/*` | `SearchInput`, `FilterButton`, `TableColumnFilterRow`, `PaginationFooter`, `Drawer`, `ConfirmModal`, `ExportModal`, `EditActionButton`, `DeleteActionButton`, `TableFilterBodySkeleton`, `EmptyState` |
| `hooks/useModulePermission`, `useDebounce`, `useTableFilterSkeleton` | Permissions, 300 ms search debounce, skeleton logic |
| `app/store.ts` | `socket.on('subjects_updated')` invalidates the `Subjects` tag |

## 2.2 On mount: 4 parallel requests (all `refetchOnMountOrArgChange: true`)
```ts
useGetSubjectsQuery({ search, page, limit, sortBy, order, ...columnFilters })  // GET /staff/subjects?...
useGetModuleFieldsQuery('subjects')                                             // GET /staff/modules/subjects/fields
useGetAcademicConfigQuery()                                                     // GET /staff/academic-config
useGetSubjectTypesQuery()                                                       // GET /staff/subject-types
```
Permissions: `moduleItem.permissions` (from login) or the `useModulePermission('subjects')` fallback, giving `read/add/edit/delete/report`.

## 2.3 Turning config into UI
```ts
rawFields      = fieldsResponse.data  (fallback: DEFAULT_SUBJECT_FIELDS)
visibleFields  = rawFields.filter(visible !== false).sort(subjectCode first, then order)
listingFields  = visibleFields.filter(options.showInListing !== false)   → TABLE COLUMNS
exportFields   = visibleFields.filter(options.showInExport  !== false)   → EXPORT MODAL CHECKBOXES
form           = visibleFields.map(renderFormField)                      → DRAWER INPUTS
```
**No column or input is hard-coded.** Add a field in Modify Form and it appears in all four places.

Dropdown choices are resolved in this order:
- `classOptions`: academic-config classes, then `classes` field `options.choices`, then a hard-coded fallback list (PRE KG … Class 12).
- `subjectTypeOptions`: names from `tbl_subject_types`.
- `filterOptions` (column filter dropdowns): backend `filterOptions`, merged with the above, plus fixed Yes/No and Active/Inactive lists. They're cached in a `useRef` so the dropdowns don't flicker while refetching.

## 2.4 The table
- Header: the select-all checkbox, then one `<th>` per `listingField` (click calls `handleSort(fieldKey)`, which toggles asc/desc), then Actions.
- `TableColumnFilterRow` is shown when `FilterButton` is toggled on. It writes `columnFilters[fieldKey] = value`, which goes into the query params.
- Rows: `renderTableCell(subject, field)` has a **switch** for system fields (subjectCode, name, subjectType, hasLab, hasTheory, isElective, isActive, classes) and `default:` uses `<DynamicTableCell field value={subject.customFields[key]} />`. Toggle shows Yes/No, Date shows DD-MM-YYYY, Upload shows a thumbnail, Attachment shows a "📎 n Files" badge, and so on.
- Clicking a row opens the Edit drawer (if `edit`). The action buttons use `stopPropagation`.
- Loading shows `TableSkeleton` (first load) or `TableFilterBodySkeleton` (refetch while filtering). Empty data shows `EmptyState` with "Add Subject" or "Clear filters".
- Footer: `PaginationFooter` (page and page size), which resets page to 1 on size change.

## 2.5 Collecting form data: the key part
The form state has a **fixed shape** for system fields and one bag for custom fields:
```ts
interface SubjectFormData {
  subjectCode: string; name: string; subjectType: string;
  hasLab: boolean; hasTheory: boolean; isElective: boolean;
  classes: string[]; classFrom?: string; classTo?: string;
  isActive: boolean;
  customFields: Record<string, any>;   // ← every non-system field goes here
}
```
`renderFormField(field)` connects each config row to that state:
```ts
getVal(): if fieldKey is isActive/status → formData.isActive
          else if fieldKey in formData   → formData[fieldKey]          (system field)
          else                           → formData.customFields[fieldKey] (custom field)

setVal(v): subjectCode → uppercased
           isActive/status → formData.isActive = Boolean(v)
           fieldKey in formData → formData[fieldKey] = v
           else → formData.customFields[fieldKey] = v
```
**The routing rule is `fieldKey in formData`.** That's how a value ends up in a column or in JSONB. It's the frontend half of the system/custom storage rule.

Before rendering, the field is "augmented":
- `classes`/`class`: forced to `MultiSelect`, with choices = field choices or `classOptions`.
- `subjectType` with no stored choices: choices = `subjectTypeOptions` (plus the current value if it's no longer in the list).

Then `<DynamicFormField field value onChange error activeAcademicYear />` picks the widget:

| fieldType | Widget | Input rules |
|---|---|---|
| Textbox/Text/URL | text input | min/max length counter |
| Character | text | letters and spaces only |
| Number/Currency | numeric | digits only |
| Contact/Phone | numeric, maxLength 10 | `^[6-9]\d{9}$` |
| Email | email | |
| Textarea | textarea | length |
| Date | `<DatePicker/>` | emits `YYYY-MM-DD` |
| Dropdown | `SearchableSelect` | choices = `yearChoices[activeYear]` or `choices` |
| MultiSelect | `MultiSearchableSelect` | same, array value |
| Toggle | switch | trueLabel/falseLabel |
| Upload | `ImageUpload` (crop, R2, WebP) | stores the URL |
| Attachment | `MultiImageUpload` | stores an array of URLs |

## 2.6 Submitting (`handleSubmit`)
1. `validateForm()`: subjectCode and name are required, then for every visible field: required check (arrays joined), then type checks (contact, email, number, character, text length). Errors go into `formErrors[fieldKey]` and show inline (`<form noValidate>`, no browser tooltips).
2. **No-Change Guard:** on edit, `JSON.stringify(formData) === JSON.stringify(initialFormData)` shows `toast.info('No changes found')` and closes.
3. Payload: `{ ...formData, isActive: Boolean, customFields: cleaned }` (strips stray `isActive/status/hasTheory` keys from customFields).
4. `createSubject(payload)` (POST) or `updateSubject({id, ...payload})` (PUT), then a toast and the drawer closes.
5. On error: shows the server message, and if the message mentions a field's label or key, it marks that field red.
6. The mutation invalidates the `Subjects` tag, so the list refetches. Other users refetch through the socket event `subjects_updated`.

Example payload actually sent:
```json
{ "subjectCode": "PHY101", "name": "Physics", "subjectType": "SCIENCE",
  "hasLab": true, "hasTheory": true, "isElective": false,
  "classes": ["I", "II"], "classFrom": "", "classTo": "",
  "isActive": true,
  "customFields": { "creditHours": "4" } }
```

## 2.7 Edit drawer prefill (`handleOpenEditDrawer`)
Normalises `classes` (array, comma string, or classFrom/classTo legacy columns), copies system columns, copies `customFields` (minus legacy keys), then takes a deep-copy snapshot into `initialFormData`.

## 2.8 Delete / bulk delete / export
- Delete: `ConfirmModal`, then `DELETE /staff/subjects/:id` (soft delete).
- Bulk: select rows, "Choose Action (n)", Delete Selected, `POST /bulk-delete {ids}`.
- Export (`report` permission): `ExportModal` (pick columns and filters), then the browser opens `GET /staff/subjects/export/excel|pdf?fields=a,b&sortBy&sortOrder&search&ids&filters`. The cookie authenticates the download.

## 2.9 Frontend permission gating (UX only)
| Button | Needs |
|---|---|
| Add Subject | `add` |
| Modify Form | `edit` (the backend actually only requires login, see 06) |
| Row click / Edit | `edit` |
| Delete | `delete` |
| Export | `report` (the backend requires only `read`) |
