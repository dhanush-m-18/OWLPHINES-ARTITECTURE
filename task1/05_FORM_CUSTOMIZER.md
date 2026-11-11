# 5. "Modify Form": how the school changes the Subjects form

Component: `Staff_ERP/src/components/dynamic-fields/ModuleFormCustomizerDrawer.tsx` (shared by every module).
Backend: `PUT /api/staff/modules/subjects/fields/bulk` → `bulkUpdateModuleFields` (`moduleField.controller.js`).

## 5.1 Opening the drawer
```tsx
<ModuleFormCustomizerDrawer
  moduleId={moduleItem?.id || moduleItem?.moduleKey || 'subjects'}   // → resolves to 'subjects'
  moduleKey="subjects"
  initialFields={rawFields}            // the HYDRATED fields from GET /modules/subjects/fields
  onSaveSuccess={() => refetchFields()} />
```
On open it deep-clones `initialFields`, sorts them (`sortModuleFields`: primary key field first, then `order`, uploads first on ties), and keeps an `initialSnapshot` for the no-change guard. It also loads roles, departments, shifts, staff, subject types and bank accounts to show "dynamic source" hints for relation dropdowns.

## 5.2 What the user can do per field
| Action | Effect in state |
|---|---|
| Rename label | `label` changes. For a **new** custom field the `fieldKey` is re-derived as `toCamelCase(label)` ("Credit Hours" becomes `creditHours`). Existing keys never change. |
| Change type | `fieldType` changes and options irrelevant to the new type are stripped (choices only for choice types, min/max only for text types, true/false labels only for Toggle, allowLogin only for Contact) |
| Toggle visible / required / unique | flags |
| Show in listing / export | `options.showInListing` / `options.showInExport` |
| Edit choices | `options.choices[]` (Add / edit / remove choice). Locked for `adminOnly` fields |
| Min/max length | `options.minLength/maxLength` |
| Reorder | `order` |
| Add custom field | new row `{fieldKey:'', label:'New Custom Field', fieldType:'Textbox', isSystemField:false, visible:true, …}` |
| Remove field | removed from the array (only custom fields are actually deleted server-side) |

## 5.3 Save (`handleSave`)
1. **No-change guard:** `JSON.stringify(editingFields) === JSON.stringify(initialSnapshot)` shows "No changes found".
2. Every label must be non-empty.
3. Build the payload per field: `{ id?, fieldKey, label, fieldType, order, visible, required, isSystemField, isUnique, allowLogin?, options }`, with options cleaned by type.
   - ⚠ `isUnique = (isInstitutionModule || !f.isSystemField) ? false : …`, so **custom fields are always sent as non-unique**.
4. `PUT /staff/modules/subjects/fields/bulk { fields:[…] }`, then a toast, close, and `refetchFields()`.

## 5.4 Backend `bulkUpdateModuleFields`
```
guard: authMiddleware → staffMiddleware → superadminMiddleware (⚠ no role check)
instMod = resolveInstModule(instId, 'subjects')
existing = non-deleted fields of instMod

A. Removed custom fields: any existing NON-system field whose id/fieldKey is not in the payload
   → update { isDeleted:true, visible:false }       (system fields can never be removed this way)

B. For each incoming field (sequential, no transaction):
   if f.id matches an existing row → UPDATE (never changes fieldKey):
       isPrimarySystemKey (system && (order 1 || key ends with 'Key' || key=='subjectCode'))
          → force fieldType='Textbox', visible=true, required=true, order=1
       options = { ...stored options, ...incoming options, choices cleaned, showInListing, showInExport }
       adminOnly field → choices forced back to stored choices (school can't overwrite admin choices)
       other fields: order forced to ≥ 2
   else (new) → fieldKey = toCamelCase(label), made unique with suffix 2,3…; UPSERT by (institutionModuleId, fieldKey)
       (upsert means re-adding a label of a previously deleted field REVIVES that old row and its old data)

C. Ensure the order-1 system field stays at 1
D. Reload, hydrate, emit 'module_fields_updated', return the new list
```

## 5.5 After saving: how everything else reacts
- `module_fields_updated` socket event: `store.ts` invalidates `ModuleFields` (and EnrollmentForms), so every open Subjects page refetches the config and the table, form and export update instantly.
- No DB migration happens. A new custom field simply starts appearing as a key in `customFields` the next time a subject is saved.
- Hiding a field (`visible=false`) removes it from the form, table, export **and backend validation** (the validator skips invisible fields), but stored values stay.

## 5.6 Other field-config endpoints (exist but the Subjects page uses only GET + bulk)
`POST /seed`, `POST /` (single create, key from label, 400 if the key exists), `PUT /:fieldId` (system+required fields stay required), `DELETE /:fieldId` (403 for system fields, soft delete otherwise), `PUT /reorder`.

## 5.7 Lifecycle summary of one custom field
```
Modify Form: add "Credit Hours" (Number)
  → INSERT tbl_institution_module_fields {fieldKey:'creditHours', fieldType:'Number', isSystemField:false, order:9}
Add Subject: type 4
  → formData.customFields.creditHours = '4'      (because 'creditHours' is not a key of formData)
  → POST … customFields:{creditHours:'4'}
  → validateSingleField(Number) passes → tbl_subjects.customFields = {"creditHours":"4"}
Table: column "Credit Hours" → DynamicTableCell → "4"
Search "4": customFields JSONB path string_contains / ::text ILIKE
Filter: customFields->>'creditHours' LIKE '%4%'
Export: getSubjectFieldValue default branch → customFields.creditHours
Modify Form: remove field → isDeleted=true (values stay in JSONB, just hidden)
```
