# 6. Frontend code: section by section

Decision taken: **Option A**. A new page file in the same design as the old Events page; the old `CalendarEvents.tsx` and `Calendar.tsx` are **untouched** (the Calendar will be done later). `/events` now opens the new page.

## 6.1 Files
| File | New / changed | Change |
|---|---|---|
| `Staff_ERP/src/features/events/eventsApiSlice.ts` | **new** | RTK Query API for `/staff/events` |
| `Staff_ERP/src/pages/EventsDynamic.tsx` | **new** | the Events page (dynamic fields + backend) |
| `Staff_ERP/src/app/store.ts` | **added lines** | import, `combineSlices(..., eventsApiSlice)`, `.concat(eventsApiSlice.middleware)`, `socket.on('events_updated')` |
| `Staff_ERP/src/store/apiSlice.ts` | **added lines** | import + `dispatch(eventsApiSlice.util.resetApiState())` in the logout reset |
| `Staff_ERP/src/config/moduleRegistry.ts` | **1 import added, 1 line changed** | `events: CalendarEventsPage` → `events: EventsDynamicPage` |

To roll back the page: change that one registry line back.

## 6.2 `eventsApiSlice.ts`
Same structure as `subjectsApiSlice.ts`:
```ts
createApi({ reducerPath: 'eventsApi', baseQuery: fetchBaseQuery({ baseUrl: API_BASE_URL, credentials: 'include' }),
  tagTypes: ['Events'], endpoints: { getEvents, getEventById, createEvent, updateEvent, deleteEvent, bulkDeleteEvents } })
```
- `credentials: 'include'` sends the `staff_token` cookie.
- `getEvents` `providesTags: ['Events']`; every mutation `invalidatesTags: ['Events']`, so the list refetches after a save or delete.

## 6.3 Store registration (AGENTS.md rules)
```ts
// app/store.ts
combineSlices(..., calendarEventsSlice, eventsApiSlice)
.concat(timetableApiSlice.middleware).concat(eventsApiSlice.middleware)
socket.on('events_updated', () => store.dispatch(eventsApiSlice.util.invalidateTags(['Events'])));
// store/apiSlice.ts  (logout / user switch)
dispatch(eventsApiSlice.util.resetApiState());
```
- **Socket:** the backend emits `events_updated` to the school's room, so every open Events screen in that school refetches.
- **Reset:** clears cached events on logout so the next user never sees the previous school's data.

## 6.4 `EventsDynamic.tsx`, section by section

### A. Types and defaults
- `EventFormData`: fixed slots for the 7 system keys plus `customFields: {}` (same idea as `SubjectFormData`). Dates are `YYYY-MM-DD`; times are `HH:mm` (what `TimePicker` emits and what the backend stores).
- `buildEmptyForm()`: today's date and 09:00–17:00, as the old page did.
- `DEFAULT_EVENT_FIELDS`: fallback only, used if the API returns no fields; identical to the backend `defaultEventFields`.
- `FULL_WIDTH_TYPES`: Textbox, Textarea, uploads and MultiSelect take the full drawer width; dates and times pair two per row, which keeps the old drawer layout.

### B. State
Search (debounced 300 ms), page, limit, sortBy (`startDate`), sortOrder (`asc`), column filters, selection, drawer and edit mode, `formData` + `initialFormData` (for the no-change guard), errors, Modify Form open, export modal, delete target. All the same as Subjects.

### C. Data loading
```ts
useGetEventsQuery({ search, page, limit, sortBy, sortOrder, ...columnFilters }, { refetchOnMountOrArgChange: true })
useGetModuleFieldsQuery('events', { refetchOnMountOrArgChange: true })
```
Search, filters, sorting and pagination now happen **on the server** (the old page did them in the browser on localStorage data).

### D. Config to UI
```ts
rawFields     = fields from API (fallback DEFAULT_EVENT_FIELDS)
visibleFields = visible, sorted by order        → drawer inputs
listingFields = visibleFields with showInListing → table columns
tableColumns  = listingFields, with startTime+endTime merged into one "Timing" column (old design)
exportFieldOptions = visibleFields with showInExport → export modal checkboxes
```
A custom field added in Modify Form appears automatically in the form, the table, the filters and the export.

### E. Form value routing (the key rule)
```ts
getVal: fieldKey in formData ? formData[fieldKey] : formData.customFields[fieldKey]
setVal: fieldKey in formData ? formData[fieldKey] = v : formData.customFields[fieldKey] = v
```
Plus the old page's convenience: changing **Start Date** also moves an unchanged End Date (single-day default).
Each input is `<DynamicFormField field value onChange error />`, so the widget is chosen by `fieldType` (Date → DatePicker, Time → TimePicker, Textarea, Toggle, Dropdown, Upload…).

### F. Validation and save
- `validateForm()` loops over the visible fields (required, length, email, contact, number, character), the same checks the backend does, then the schedule rules: end date ≥ start date, and end time > start time on a single day.
- **No-Change Toast Guard**: `JSON.stringify(formData) === JSON.stringify(initialFormData)` shows "No changes found".
- `createEvent(payload)` / `updateEvent({ id, ...payload })`; server error messages are shown in a toast and placed under the matching field when the message names it.

### G. Table
- Header: select-all, one sortable column per `tableColumns` entry (sorting works for any field, including custom ones, because the backend whitelists keys), Actions.
- `TableColumnFilterRow` with backend `filterOptions` (e.g. Active/Inactive).
- Cells: `renderTableCell`. Title as text; dates via `formatDate` (DD-MM-YYYY); Timing via `formatTimingSpan` (09:00 AM – 05:00 PM, reused from the old page's exports); single times via `formatTimeWithAmPm`; description truncated; status Active/Inactive; **custom fields → `DynamicTableCell`**.
- Skeletons while loading, `EmptyState` when there are no results, and `PaginationFooter` driven by the server `pagination.total`.

### H. Header and toolbar
- Header: `FilterButton`, **`ModifyFormButton`** (only if `permissions.edit`) and `AddButton` (only if `permissions.add`).
- Toolbar: `SearchInput`, `ImportExportDropdown` (only if `permissions.report`, same as Subjects), and "Choose Action (n)" → Delete Selected (only if `permissions.delete`).

### I. Drawers and modals
- **Add/Edit drawer**: same title, subtitle, icon, width (`max-w-lg`) and footer buttons as the old page; inputs are now generated from the field rows. The button shows "Saving..." while the request runs.
- **ModuleFormCustomizerDrawer** with `moduleKey="events"`: the shared Modify Form, which saves to the existing `/modules/events/fields/bulk`.
- **ExportModal**: pick columns and filters, then open `/staff/events/export/excel|pdf?...` (the server builds the file with the school's header).
- **ConfirmModal** for single and bulk delete. The message no longer says "permanently removed", because the server soft-deletes.

## 6.5 Old page vs new page
| | Old `CalendarEvents.tsx` | New `EventsDynamic.tsx` |
|---|---|---|
| Data | browser localStorage, per user | backend → `tbl_module_records`, per school, shared |
| Fields | hard-coded 6 inputs and 5 columns | from `tbl_institution_module_fields` (7 system + any custom) |
| Modify Form | none | yes |
| Search, filter, sort, paging | in the browser | on the server |
| Export | browser CSV and print window | server Excel/PDF with school branding |
| Permissions | none (everyone sees all buttons) | add/edit/delete/report respected |
| Real-time | none | `events_updated` socket |
| Delete | removed from localStorage | soft delete |

## 6.6 Checks run
| Check | Result |
|---|---|
| `npx tsc --noEmit -p tsconfig.app.json` (Staff_ERP) | ✅ 0 errors |
| `npx tsc --noEmit -p tsconfig.app.json` (Admin_ERP) | ✅ 0 errors |
| `oxlint` on all changed frontend files | ✅ no warnings |
| Vite dev server compiles `EventsDynamic.tsx`, `eventsApiSlice.ts`, `store.ts`, `moduleRegistry.ts` | ✅ HTTP 200 |
| Clicking through the page in a browser | ⏳ not done by me; see the checklist in README |
