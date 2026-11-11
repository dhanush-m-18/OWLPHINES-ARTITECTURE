# 2. Validator and routes

## 2.1 `backend/src/staff_side/validators/event.validator.js` (new)

**Purpose:** a quick shape check before the controller runs (same idea as `student.validator.js`). The real field rules (required, length, type, dropdown choices) come from the DB field rows and are checked in the controller, because each school's form can be different.

```js
export const createEventSchema = {
  body: z.object({
    title: stringOrNumber.refine((v) => v.length >= 1, 'Event Title is required'),
    startDate: z.any().optional().nullable(),
    endDate: z.any().optional().nullable(),
    startTime: stringOrNumberOptional,
    endTime: stringOrNumberOptional,
    description: stringOrNumberOptional,
    isActive: booleanToggle(true),                     // missing → true
    customFields: z.record(z.any()).optional().nullable(),
  }).passthrough(),
};
```
- `stringOrNumber` / `stringOrNumberOptional` / `booleanToggle` are the shared helpers in `utils/zodHelpers.js`. They accept strings or numbers (and `'true'`, `1` and so on for booleans) and convert them.
- `.passthrough()` keeps unknown top-level keys. That's safe here because the controller keeps **only configured field keys** when building `data`.
- `updateEventSchema`: the same fields, all optional, and `booleanToggle()` has no default, so a partial update doesn't accidentally reset `isActive`.
- `bulkDeleteEventSchema`: `ids` must be a non-empty array of strings.

Invalid shape gives 400 `"Validation Error"` with messages, from the shared `validate.middleware.js`.

## 2.2 `backend/src/staff_side/routes/event.routes.js` (new)

```js
router.use(authMiddleware, staffMiddleware);   // must be logged in as staff

// Export routes (must be before /:id to avoid route collision)
router.get('/export/excel', requirePermission('events', 'read'), exportEventExcel);
router.get('/export/pdf',   requirePermission('events', 'read'), exportEventPdf);

router.get('/',             requirePermission('events', 'read'),   getEvents);
router.post('/bulk-delete', requirePermission('events', 'delete'), validateRequest(bulkDeleteEventSchema), bulkDeleteEvents);
router.post('/',            requirePermission('events', 'add'),    validateRequest(createEventSchema), createEvent);

router.get('/:id',    requirePermission('events', 'read'),   getEventById);
router.put('/:id',    requirePermission('events', 'edit'),   validateRequest(updateEventSchema), updateEvent);
router.delete('/:id', requirePermission('events', 'delete'), deleteEvent);
```
Middleware chain for every request:
1. `authMiddleware`: reads the `staff_token` cookie and sets `req.user = { staffId, institutionId, roleId, roleName }`.
2. `staffMiddleware`: only STAFF or MANAGEMENT tokens.
3. `requirePermission('events', action)`: the school must have the Events module enabled, and the person's role or override must allow the action. SUPERADMIN bypasses this.
4. `validateRequest(schema)`: the shape check above.
5. The controller.

**Why the order matters:** `/export/excel` and `/bulk-delete` are declared before `/:id`, otherwise Express would treat "export" as an id.

## 2.3 `backend/src/staff_side/routes/index.js` (2 lines added)
```js
import eventRoutes from './event.routes.js';
...
router.use('/events', eventRoutes);   // → /api/staff/events
```

## 2.4 Endpoint summary
| Method | URL | Permission | Does |
|---|---|---|---|
| GET | `/api/staff/events?page&limit&search&sortBy&sortOrder&from&to&<fieldKey>=…` | read | list |
| GET | `/api/staff/events/:id` | read | one event |
| POST | `/api/staff/events` | add | create |
| PUT | `/api/staff/events/:id` | edit | partial update (merge) |
| DELETE | `/api/staff/events/:id` | delete | soft delete |
| POST | `/api/staff/events/bulk-delete` `{ids:[…]}` | delete | soft delete many |
| GET | `/api/staff/events/export/excel?fields=…&<filters>` | read | Excel file |
| GET | `/api/staff/events/export/pdf?fields=…&<filters>` | read | PDF file |
| GET | `/api/staff/modules/events/fields` | (existing route) | the form definition, seeds defaults |
| PUT | `/api/staff/modules/events/fields/bulk` | (existing route) | Modify Form save |
