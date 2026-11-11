# 4. Backend test results (04-11-2026)

Run against the live backend at `http://192.168.29.95:5001` (nodemon), logged in as ABC SUPERADMIN (6361249205, test OTP), on the shared Neon database.

| # | Request | Expected | Result |
|---|---|---|---|
| 1 | `GET /staff/modules/events/fields` | 7 default fields seeded | ✅ title, startDate, endDate, startTime, endTime, description, isActive |
| 2 | `POST /staff/events` with `startTime:"09:00 AM"` and `customFields:{junkKey}` | 201; time normalised; junk dropped | ✅ `startTime:"09:00"`, `customFields:{}`, `createdBy` = staff id |
| 3 | `GET /staff/events?page=1&limit=10` | list with the event | ✅ |
| 4 | `GET ?search=claude` | found | ✅ |
| 5 | `GET ?from=2026-11-01&to=2026-11-30&limit=all` | found (overlaps November) | ✅ |
| 6 | `GET ?from=2026-12-01&to=2026-12-31` | empty | ✅ |
| 7 | `GET ?isActive=false` | empty (the event is active) | ✅ |
| 8 | `GET ?sortBy=title&sortOrder=desc` | 200 | ✅ |
| 9 | `PUT /staff/events/:id {description}` | only description changes | ✅ other keys kept (merge) |
| 10 | `POST` with endDate before startDate | 400 | ✅ "End Date cannot be earlier than Start Date." |
| 11 | `POST` with `startTime:"25:00"` | 400 | ✅ "'Start Time' must be a valid time (e.g. 09:30 AM)." |
| 12 | `POST` with title only | 400 | ✅ "'Start Date' is required." (from the DB field rules) |
| 13 | `GET /export/excel` | xlsx | ✅ HTTP 200, 7 KB |
| 14 | `GET /export/pdf` | pdf | ✅ HTTP 200, 9 KB |
| 15 | `POST /bulk-delete` with both test ids | soft delete 2 | ✅ "Successfully deleted 2 event(s)" |
| 16 | `GET /staff/events` after delete | empty | ✅ |
| 17 | `GET /staff/events/:id` of a deleted event | 404 | ✅ "Event not found" |

## Issue found during testing (fixed)
The first create happened **before** `events: defaultEventFields` was in the registry (that edit initially failed because the file had changed upstream). With no field rows, the event was saved with `data = {"isActive": true}` and no title.
- Fix 1: added the registry line.
- Fix 2: `createEvent` now refuses to save when no field config exists (500, "Event form fields are not configured").
- That blank row (`1e644d44-…`) and the test event (`5d522e50-…`) were **soft-deleted** in step 15. They're still in `tbl_module_records` with `isDeleted = true`.

## Data left in the shared DB
- 7 field rows in `tbl_institution_module_fields` for ABC's Events module (intended; created automatically on first use).
- 2 soft-deleted test rows in `tbl_module_records` (hidden everywhere).
