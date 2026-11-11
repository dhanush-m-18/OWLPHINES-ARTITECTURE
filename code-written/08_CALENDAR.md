# 8. Calendar: institution events from `tbl_events`, view-only (04-11-2026)

## Requirements (from the lead / task)
1. Show only **Active** events.
2. Staff with **Calendar** access must not get a 403 even without Events access.
3. **View-only**: tap a day → side panel lists that day's events; tap an event → all its fields, including custom fields (Venue…).
4. **Remove the localStorage** events code.

## How "only my institution" works
The calendar calls `GET /api/staff/events?from=<first grid day>&to=<last grid day>&limit=all&isActive=true`. The backend filters by the `institutionId` in the login token, so each staff member only ever sees their own school's events. The calendar never sends a school id.

## Files
| File | Change |
|---|---|
| `Staff_ERP/src/pages/Calendar.tsx` | **rewritten as view-only**: data from `useGetEventsQuery` (visible month, Active only), field rows from `useGetModuleFieldsQuery('events')`; day panel lists events; tapping one shows every visible field (system fields formatted, custom fields through `DynamicTableCell`); "Back to day" and "Close"; no add, edit or delete |
| `Staff_ERP/src/utils/time.ts` | **new**: `formatTimeWithAmPm`, `formatTimingSpan` (moved out of the old `CalendarEvents.tsx`, shared by Calendar and Events) |
| `Staff_ERP/src/pages/EventsDynamic.tsx` | imports the time helpers from `utils/time` |
| `Staff_ERP/src/config/moduleRegistry.ts` | keys `event` and `calendarevents` now also open `EventsDynamicPage`; old page imports removed |
| `Staff_ERP/src/app/store.ts` | `calendarEventsSlice` removed from `combineSlices` |
| `Staff_ERP/src/store/apiSlice.ts` | `calendarEventsSlice` re-exports removed |
| `Staff_ERP/src/types/index.ts` | unused `CalendarEvent` type removed |
| **deleted** `Staff_ERP/src/features/calendarEvents/calendarEventsSlice.ts` | localStorage events store |
| **deleted** `Staff_ERP/src/pages/CalendarEvents.tsx` | old localStorage Events page |
| **deleted** `Staff_ERP/src/pages/Events.tsx` | wrapper of the old page |
| `backend/src/staff_side/routes/event.routes.js` | `GET /` and `GET /:id` use `requireEventsOrCalendarRead` |

`AuthContext.tsx` still removes the old `owlphin_calendar_events` key on login/logout. That only cleans leftover data from browsers that used the old version, so it was left as is.

## Why a local guard instead of `requirePermission(['events', 'calendar'], 'read')`
The shared `requirePermission` loops over the keys and **stops at the first enabled module**, even if that module denies the action. With Events enabled but no Events read, the staff member would get a 403 although Calendar read is allowed. The local guard checks both and allows the request if **either** grants `read`. SUPERADMIN and MANAGEMENT bypass it, as before. Add, edit, delete and export still require the `events` permission.

## Checks
| Check | Result |
|---|---|
| `tsc --noEmit` Staff_ERP / Admin_ERP | ✅ 0 errors |
| oxlint on all changed frontend files | ✅ clean |
| Backend syntax | ✅ |
| No remaining references to the deleted files | ✅ |
| SUPERADMIN: `GET /staff/events?from=2026-11-01&to=2026-12-05&limit=all&isActive=true` | ✅ ABC's active events (e.g. "inguration", Venue MAIN HALL) |
| Teacher (role staff1): same query | ✅ 200 with the same events |
| Vite compiles `Calendar.tsx` | ✅ |
