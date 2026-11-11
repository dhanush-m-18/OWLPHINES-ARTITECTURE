# 5. Frontend: what already exists and what is needed

## 5.1 Already implemented and reusable as-is (no changes needed)
| Piece | File | Used for |
|---|---|---|
| Field API | `features/fields/fieldsApiSlice.ts` → `useGetModuleFieldsQuery('events')` | loads the 7 event fields (+ custom) |
| Form renderer | `components/dynamic-fields/DynamicFormField.tsx` | already supports `Textbox`, `Date` (DatePicker), **`Time` (TimePicker, emits `HH:mm`)**, `Textarea`, `Toggle`, `Dropdown`, `Upload`… |
| Cell renderer | `components/dynamic-fields/DynamicTableCell.tsx` | custom field cells |
| Modify Form | `components/dynamic-fields/ModuleFormCustomizerDrawer.tsx` with `moduleKey="events"` | add, rename, hide or require fields, works through the existing `/modules/events/fields/bulk` |
| UI kit | `Drawer`, `ConfirmModal`, `SearchInput`, `FilterButton`, `ModifyFormButton`, `AddButton`, `TableColumnFilterRow`, `PaginationFooter`, `ExportModal`, `ImportExportDropdown`, `EditActionButton`, `DeleteActionButton`, skeletons, `EmptyState` | same layout as Subjects |
| Hooks | `useModulePermission('events')`, `useDebounce`, `useTableFilterSkeleton` | permissions, search debounce, skeletons |
| Routing | `config/moduleRegistry.ts` → `events: CalendarEventsPage` | `/events` already opens the Events page |

**So no new dynamic-field component is needed.** The dynamic-field system is complete and generic; Events just has to plug into it.

## 5.2 What is NOT done yet (current Events page)
`pages/CalendarEvents.tsx` today:
- reads and writes **`localStorage`** through `features/calendarEvents/calendarEventsSlice.ts`, so it doesn't call the new API;
- has **hard-coded columns and form inputs** (`EVENT_COLUMNS`, `EventFormData`), not driven by field rows;
- has **no Modify Form button**;
- exports CSV and print-PDF in the browser instead of through the backend.

`pages/Calendar.tsx` also reads the same localStorage events, and it imports `formatTimeWithAmPm`, `formatTimingSpan` and `parseTimeTo24H` from `CalendarEvents.tsx`.

## 5.3 What has to be added for Events (frontend)
| # | Work | Type |
|---|---|---|
| 1 | `features/events/eventsApiSlice.ts` (RTK Query: getEvents, getEventById, create, update, delete, bulkDelete) | **new file** |
| 2 | Register the slice in `app/store.ts` (`combineSlices` + `.concat(middleware)`) and add `socket.on('events_updated')` → invalidate | **add lines** to an existing file |
| 3 | Add `eventsApiSlice.util.resetApiState()` to `staffApi.util.resetApiState` in `store/apiSlice.ts` (logout cache reset, AGENTS rule) | **add 1 line** |
| 4 | The Events page itself, rebuilt on the Subjects pattern: field rows → table columns, form, filters and export; `formData` = system keys + `customFields`; Modify Form; server-side search, filter, sort and pagination; backend Excel/PDF export | either **edit `CalendarEvents.tsx`** or **new page file** (see 5.4) |

## 5.4 Decision needed: how to apply step 4 without disturbing existing code
| Option | What changes | Pros / cons |
|---|---|---|
| **A. New page file** `pages/EventsDynamic.tsx` and change **one line** in `moduleRegistry.ts` (`events: EventsDynamicPage`) | `CalendarEvents.tsx` and `Calendar.tsx` stay untouched; easy to roll back by reverting one line | Two Events pages exist in the code until the old one is removed |
| **B. Rewrite `CalendarEvents.tsx`** in place, keeping its exported time helpers so `Calendar.tsx` still compiles | One Events page only | Changes an existing file heavily |

Out of scope unless approved: `Calendar.tsx` (month view) would keep showing the old localStorage events until it's switched to `GET /staff/events?from=…&to=…&limit=all`, which the backend already supports.
