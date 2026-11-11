# 7. Re-applying the Subjects pattern to another module: checklist

Replace `Thing` / `things` / `thing` with the new module. Follow the order. Each step names the Subjects file to copy from.

## A. Database
- [ ] `backend/prisma/schema/thing.prisma`: system columns + `customFields Json? @db.JsonB` + `institutionId` relation (Cascade) + `isActive`, `isDeleted`, `createdBy`, `updatedBy`, `createdAt`, `updatedAt` + `@@unique([institutionId, <code>])` + `@@index([institutionId])`, `@@index([institutionId, isDeleted])` + `@@map("tbl_things")`. *(copy `subject.prisma`)*
- [ ] Back-relation `things Thing[]` in `institution.prisma`.
- [ ] `npx prisma validate`, then `npx prisma db push`, then `npx prisma generate`.
- [ ] Decide which fields are **system** (need typing, sorting, joins) vs left to **custom** fields.

## B. Default field config
- [ ] `config/defaultFields.config.js`: add `defaultThingFields` (fieldKey = exact column name for system fields, order unique and sequential, the primary code field at order 1 with `isUnique:true`).
- [ ] Register it: `DEFAULT_MODULE_FIELDS['things'] = defaultThingFields`.
- [ ] If a field's choices come from master data, mark it `adminOnly`/`isYearWise` and extend `hydrateFieldsWithAcademicOptions`, **or** inject the choices on the frontend like `subjectType`.

## C. Module registration
- [ ] Admin_ERP: create the master module with `moduleKey='things'`, name, `defaultPath='/things'`, `defaultIcon` (a react-icons name). It gets provisioned per institution.
- [ ] Staff_ERP Access Control: give roles read/add/edit/delete/report on it.

## D. Backend
- [ ] `validators/thing.validator.js` (zod, with `booleanToggle`/`stringOrNumber`). *(Subjects lacks this, so add it: E-12)*
- [ ] `controllers/thing.controller.js` *(copy `subject.controller.js` structure)*:
  - [ ] Helpers: `resolveThingInstModule`, `getThingModuleFields` (with `ensureDefaultFields(…,'things')`), `getThingFieldValue`, `buildThingSearchConditions`, `buildThingFilterConditions`, `buildThingDynamicFilterOptions`, `getThingExportData`.
  - [ ] `getThings`: clamp `limit` (≤500), **whitelist `sortBy`** (system columns only; custom keys via SQL `customFields->>`) *(fixes E-2)*. If you need natural sort with raw SQL, make sure **every** filter, including any special param, exists in both paths *(E-3)*. Better still, generate both from one helper.
  - [ ] `createThing` / `updateThing`: destructure fields, `validateModuleFields` + `validateModuleUniqueFields`, duplicate-code check, revive soft-deleted rows, `createdBy/updatedBy: req.user.staffId` *(E-9)*. On update, **merge** customFields: `{...existing.customFields, ...incoming}` *(E-16)*.
  - [ ] `deleteThing` / `bulkDeleteThings`: soft delete, `institutionId` inside `where`.
  - [ ] `bulkImportThings`: run `validateModuleFields` per row, keep only `customFields` keys that exist in the module fields, report real inserted counts *(E-4)*.
  - [ ] `exportThingExcel` / `exportThingPdf` via `services/export.service.js`.
  - [ ] Every mutation: `emitToInstitution(institutionId, 'things_updated', {...ids only})`.
- [ ] `routes/thing.routes.js`: `router.use(authMiddleware, staffMiddleware)`, static routes before `/:id`, `requirePermission('things', action)` + `validateRequest(schema)` on each. *(copy `subject.routes.js`)*
- [ ] Mount it: `router.use('/things', thingRoutes)` in `staff_side/routes/index.js`.

## E. Frontend
- [ ] `features/things/thingsApiSlice.ts` *(copy `subjectsApiSlice.ts`)*, tag `Things`.
- [ ] Register in `app/store.ts`: `combineSlices(…)`, `.concat(thingsApiSlice.middleware)`, and `socket.on('things_updated', () => store.dispatch(thingsApiSlice.util.invalidateTags(['Things'])))`.
- [ ] Add to `staffApi.util.resetApiState` in `store/apiSlice.ts` (plus hook re-exports if you import from there).
- [ ] `pages/Things.tsx` *(copy `Subjects.tsx`)*:
  - [ ] Queries with `{ refetchOnMountOrArgChange: true }`: list + `useGetModuleFieldsQuery('things')` + any choice sources.
  - [ ] `visibleFields`, `listingFields`, `exportFieldOptions` from config.
  - [ ] Form state = typed system fields + `customFields: {}`. `getVal`/`setVal` route by `fieldKey in formData`.
  - [ ] `renderFormField`: augment choices, then `<DynamicFormField/>`.
  - [ ] `renderTableCell`: switch for system fields, `default` uses `<DynamicTableCell/>`.
  - [ ] `validateForm` (mirror the backend rules) + the **No-Change Toast Guard**.
  - [ ] Only make headers sortable for sortable fields *(E-2)*.
  - [ ] Layout = Unified Card Standard (header with ModifyFormButton/AddButton, toolbar, table with TableColumnFilterRow, PaginationFooter), plus Drawer, ConfirmModal ×2, ExportModal, ModuleFormCustomizerDrawer (`moduleKey='things'`).
  - [ ] Gate buttons with permissions (match the backend: export = `read` or `report`, decide one).
- [ ] `config/moduleRegistry.ts`: `things: ThingsPage`.
- [ ] Don't duplicate default fields in the frontend, or keep them identical to the backend *(E-13)*.

## F. Verify
- [ ] `npx tsc --noEmit -p tsconfig.app.json` in Staff_ERP (and Admin_ERP if touched).
- [ ] Manual: as SUPERADMIN, open `/things`: the default fields are auto-seeded (check `tbl_institution_module_fields`).
- [ ] Add a record, then edit with no change (expect "No changes found"), then edit, then delete, then re-add the same code (it should be revived).
- [ ] Modify Form: add a custom field, then save a record, and check `customFields` in Prisma Studio (http://localhost:5555).
- [ ] Search, column filters, sort on each column, pagination, Excel/PDF export.
- [ ] Log in as a **non-admin teacher** and confirm the buttons are hidden and direct API calls get 403.
- [ ] Open two browsers and confirm the real-time refresh works.

## G. Known shared issues you inherit (not module-specific, raise with seniors)
E-1 (`superadminMiddleware` role check), E-5 (custom unique forced off), E-6/E-7 (options hygiene in the customiser), E-8 (GET side effects). These live in shared code, so fixing them once fixes every module.
