# 7. Switch from `tbl_module_records` to `tbl_events` (04-11-2026)

**Why:** the lead asked for Events to have its own table, like Subjects (`tbl_subjects`), instead of the shared generic `tbl_module_records`.

**Result:** events are now stored in `tbl_events`. Fixed fields are real columns and custom fields go in `customFields`. Files 01 and 03 describe the earlier `tbl_module_records` version; where they differ, this file is current.

## 7.1 What changed
| File / place | Change | Why |
|---|---|---|
| `backend/prisma/schema/event.prisma` | **new** `Event` model → `@@map("tbl_events")` | dedicated table; `startDate`/`endDate` are real `DATE` columns, times `"HH:mm"` text, custom fields in `customFields` JSONB; indexes on school + isDeleted / startDate / endDate, GIN on customFields |
| `backend/prisma/schema/institution.prisma` | **+1 line** `events Event[]` | Prisma needs the relation on both sides (no DB column) |
| Shared DB | `CREATE TABLE "tbl_events"` + indexes + FK, **only** | see 7.2 |
| `npx prisma generate` | client regenerated → `prisma.event` | needed by the controller |
| `backend/src/staff_side/controllers/event.controller.js` | storage swapped from `prisma.moduleRecord` to `prisma.event` | see 7.3 |
| Data | 2 events copied from `tbl_module_records` with the **same ids** | users see the same events |

**Not changed:** `event.routes.js`, `event.validator.js`, `defaultEventFields`, the field rows in `tbl_institution_module_fields` (including Venue), and **all frontend files**. The API response shape is identical.

## 7.2 How the table was created safely
The usual `npx prisma db push` makes the DB match the branch's schema files. The shared DB contains teammates' tables that are **not** in `dev` (`tbl_token`, `tbl_institution_fee_installment_plans`, `tbl_institution_fee_installment_items`, and `tbl_student_info.lastOtp` / `loginSessions`), so `db push` would have **dropped** them.

What was done instead:
1. `prisma migrate diff` between the committed schema and the schema with the new model gave a SQL script containing **only** `CREATE TABLE "tbl_events"`, 5 indexes and the FK to `tbl_institutions`.
2. `prisma db execute --file` ran just that script.
3. Verified afterwards: `tbl_token` (8 rows), fee installment plans (1) and items (2), and the two student columns all still exist.

⚠️ For the team: until those teammates' models are in `dev`, **nobody** should run `prisma db push` from `dev` on the shared DB.

## 7.3 Controller changes
| Part | Before (`tbl_module_records`) | Now (`tbl_events`) |
|---|---|---|
| Save | everything into `data` JSON | system fields → columns (`toDbDate` for dates), custom fields → `customFields` (`splitEventValues`) |
| Read | split `data` into top-level + customFields | columns returned directly; `DATE` formatted back to `"YYYY-MM-DD"` (`toEventResponse`) |
| Query | `data->>key` for everything | system keys use columns (`EVENT_COLUMN_SQL`, e.g. `to_char(e."startDate",'YYYY-MM-DD')`), custom keys `e."customFields"->>key`; date range on real `DATE` columns |
| Sort | `data->>key` | startDate/endDate on real columns; others on the column or the JSON key (whitelisted) |
| Unique check | custom helper on `data` | shared `validateModuleUniqueFields(prisma.event, …)`, same as Subjects |
| Update | merge into `data` | columns updated only if sent; `customFields` **merged** with existing |
| Required columns | n/a | title and startDate must exist (columns are NOT NULL); endDate defaults to startDate |
| Delete | `isDeleted = true` | `isDeleted = true` **and** `isActive = false` (AGENTS.md rule) |

## 7.4 Data copy
A one-time script (kept outside the repo) read the events of every school's `events` module from `tbl_module_records` and upserted them into `tbl_events` with the same id, dates and audit fields:
| id | title | result |
|---|---|---|
| `c9042032…` | sport (Venue = Main Ground) | copied, active |
| `5d522e50…` | dhanush Test Event | copied, stays soft-deleted |
| `1e644d44…` | (blank, registry-bug row) | skipped: no title or date, already soft-deleted |

The old rows in `tbl_module_records` were **not changed**. They're no longer read by anything.

## 7.5 Tests after the switch (all passed)
| Check | Result |
|---|---|
| List shows the copied "sport" event with Venue | ✅ |
| Create with `25-11-2026` and `10:00 AM` | ✅ stored as DATE `2026-11-25`, `10:00`; junk key dropped |
| Update `customFields.venue` only | ✅ merged; other values unchanged |
| Filter `?venue=main`, search by date `25-11-2026`, date range, sort by Venue, `isActive=false` | ✅ |
| End date before start date / end time before start time | ✅ 400 with clear messages |
| Excel / PDF export | ✅ 7 KB / 9 KB |
| Raw row in `tbl_events` | ✅ real columns + `customFields: {"venue":"Main Hall"}` |
| Delete | ✅ `isDeleted = true`, `isActive = false`; gone from the list |

The test event "dhanush API Test" (`5d4f7c49…`) is soft-deleted in `tbl_events`.
