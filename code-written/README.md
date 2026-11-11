# Events module: code written (backend)

> **Update 04-11-2026:** per the lead, events now have their own table, **`tbl_events`**, instead of `tbl_module_records`. See [07_SWITCH_TO_TBL_EVENTS.md](07_SWITCH_TO_TBL_EVENTS.md). Files 01 and 03 describe the earlier storage; 07 is current where they differ.
>
> Extra files for the switch: `backend/prisma/schema/event.prisma` (new) and `backend/prisma/schema/institution.prisma` (+1 line `events Event[]`).

Task: make `http://192.168.29.95:5174/events` use the dynamic-field system, the same way Subjects does, **without a new table** (senior's decision). Events are stored in the existing `tbl_module_records`.

Written 04-11-2026 on branch `dhanush-m`. Nothing is committed yet.

## Files

| File | New / changed | What it is |
|---|---|---|
| `backend/src/config/defaultFields.config.js` | **added lines only** | `defaultEventFields` (7 system fields) + `events: defaultEventFields` in the registry |
| `backend/src/staff_side/validators/event.validator.js` | **new** | zod shape checks for create, update and bulk delete |
| `backend/src/staff_side/controllers/event.controller.js` | **new** | list/search/filter/sort/date-range, get, create, update, delete, bulk delete, Excel/PDF export |
| `backend/src/staff_side/routes/event.routes.js` | **new** | `/api/staff/events` routes with `requirePermission('events', …)` |
| `backend/src/staff_side/routes/index.js` | **added 2 lines only** | `import eventRoutes` + `router.use('/events', eventRoutes)` |

No existing function or logic was modified. No schema change and no migration.

### Frontend
| File | New / changed | What it is |
|---|---|---|
| `Staff_ERP/src/features/events/eventsApiSlice.ts` | **new** | RTK Query for `/staff/events` |
| `Staff_ERP/src/pages/EventsDynamic.tsx` | **new** | Events page: dynamic fields, backend data, Modify Form; same design as the old page |
| `Staff_ERP/src/app/store.ts` | **added lines** | slice, middleware, `events_updated` socket |
| `Staff_ERP/src/store/apiSlice.ts` | **added lines** | logout cache reset |
| `Staff_ERP/src/config/moduleRegistry.ts` | **1 line switched** | `events: EventsDynamicPage` |

Untouched: `CalendarEvents.tsx`, `Calendar.tsx`, `calendarEventsSlice.ts` (Calendar is a later task).

### Browser test checklist (http://192.168.29.95:5174/events, log in with 6361249205 / 123456)
1. The page opens with the 7-field table (Timing shown as one column) and an empty state.
2. **Add Event**: title, dates, times and description, then Save. The row appears.
3. Edit without changing anything: "No changes found".
4. Set End Date before Start Date: inline error.
5. **Modify Form**: add "Venue" (Textbox) and save. A Venue input appears in the drawer and a Venue column in the table.
6. Edit the event and fill Venue. Check it in Prisma Studio (`ModuleRecord`, `data.venue`).
7. Search, column filters, sorting (including Venue), pagination.
8. Export Excel/PDF.
9. Delete one, then bulk-delete. The rows disappear (soft delete).
10. Open the page in a second browser: changes appear without a refresh (socket).

## Read in this order
1. [01_STORAGE_AND_DEFAULT_FIELDS.md](01_STORAGE_AND_DEFAULT_FIELDS.md): where an event is stored, and the 7 default fields
2. [02_VALIDATOR_AND_ROUTES.md](02_VALIDATOR_AND_ROUTES.md): the request entry points
3. [03_CONTROLLER.md](03_CONTROLLER.md): every section of `event.controller.js` explained
4. [04_TEST_RESULTS.md](04_TEST_RESULTS.md): what was tested against the real API and the results
5. [05_FRONTEND_STATUS.md](05_FRONTEND_STATUS.md): what already exists in the frontend and what needed to change
6. [06_FRONTEND_CODE.md](06_FRONTEND_CODE.md): every section of the new frontend code explained

## One-paragraph summary
An event is **one row** in `tbl_module_records`. `institutionId` comes from the login token, `institutionModuleId` is the school's Events module, and **all values** (system keys like `title`, `startDate`, plus any custom keys like `venue`) are key–value pairs in the `data` JSONB column. The form definition comes from `tbl_institution_module_fields`, which is seeded automatically from `defaultEventFields`. The API returns events in the **same shape as Subjects**: system keys at the top level and custom keys under `customFields`. That way the frontend can reuse the existing dynamic-field components without changes.
