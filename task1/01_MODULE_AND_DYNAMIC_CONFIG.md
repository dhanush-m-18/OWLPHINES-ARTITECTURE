# 1. The concept: modules and dynamic configuration

## 1.1 Three layers

| Layer | Table | Who owns it | For Subjects (real ABC data) |
|---|---|---|---|
| **System module** (global catalogue) | `tbl_system_modules` (`Module`) | OWLPHIN Admin (Admin_ERP) | `moduleKey='subjects'`, name `Subjects`, defaultPath `/subjects`, defaultIcon `FaBook` |
| **Institution module** (the school's copy) | `tbl_institution_modules` (`InstitutionModule`) | Each school (Settings → Modules) | ABC: name `Subjects`, path `/subjects`, icon `FaBook`, `isEnabled=true`, priority `6`, `show={form:true, report:true}` |
| **Module fields** (the form definition) | `tbl_institution_module_fields` (`InstitutionModuleField`) | Seeded from code defaults, then edited by the school in "Modify Form" | 10 rows for ABC (8 active, 2 soft-deleted custom fields) |

Then the actual data lives in the module's **own table**: `tbl_subjects`.

```
Module (subjects) 1 ──── N InstitutionModule (one per school) 1 ──── N InstitutionModuleField
                                         │
                                         ├── RoleModulePermission / StaffModulePermission  (who can read/add/edit/delete)
                                         └── (Subjects data itself is in tbl_subjects, keyed by institutionId)
```

### Why `moduleKey` matters
`moduleKey = 'subjects'` is the **stable glue** across the whole system:
- Backend permission: `requirePermission('subjects', 'read')`
- Default fields: `DEFAULT_MODULE_FIELDS['subjects']`
- Frontend page: `CUSTOM_MODULE_REGISTRY['subjects'] = SubjectsPage`
- Field API: `GET /staff/modules/subjects/fields`

The school can rename the module ("Subjects" to "Courses") or change its path or icon. `moduleKey` never changes.

## 1.2 How the URL `/subjects` becomes the Subjects page
1. Login or `/me` returns `modules[]`. Each entry has `moduleKey`, `path`, `icon`, `permissions`, built by `getStaffAccessibleModules()` (only modules with `read=true`).
2. The sidebar renders those modules (icon via `<FilledIcon name="FaBook"/>`).
3. `App.tsx` route `/:moduleParam` runs `ModuleRecordsWrapper`, which matches `subjects` against each module's id, key, path or name.
4. `getCustomModuleComponent('subjects')` returns `SubjectsPage`, rendered as `<SubjectsPage moduleItem={module} />`.
5. If the module isn't in the list (disabled, or no read permission), the user is redirected to `/dashboard`.

## 1.3 A field definition row, explained

| Column | Meaning | Example (`classes`) |
|---|---|---|
| `fieldKey` | Stable programmatic key. For system fields it's the **column name**. For custom fields it's the **JSONB key**. Never changes after creation. | `classes` |
| `label` | What the user sees. Freely editable. | `Class` |
| `fieldType` | Decides the input widget, the table cell renderer, validation and search behaviour | `MultiSelect` |
| `isSystemField` | `true` = real column on `tbl_subjects`; can't be deleted | `true` |
| `visible` | `false` = hidden from form, table and export (data kept) | `true` |
| `required` | Validation (frontend + backend) | `true` |
| `isUnique` | Backend checks no other active record has this value | `false` |
| `order` | Position in form and table | `5` |
| `options` (JSONB) | Type-specific settings: `choices`, `minLength/maxLength`, `trueLabel/falseLabel`, `showInListing`, `showInExport`, `isYearWise`, `adminOnly`, `yearChoices` | see below |
| `isDeleted` | Soft-deleted field (custom fields removed in Modify Form) | `false` |

Supported `fieldType`s (`Staff_ERP/src/components/dynamic-fields/types.ts`):
`Textbox, Dropdown, MultiSelect, Date, Toggle, Email, Contact, Number, Textarea, Character, Upload, Attachment`.

## 1.4 Where the defaults come from
`backend/src/config/defaultFields.config.js` has `defaultSubjectFields`:

| order | fieldKey | type | required | unique | notes |
|---|---|---|---|---|---|
| 1 | subjectCode | Textbox | ✔ | ✔ | min 2 / max 20; forced Textbox, visible, required, order 1 (primary key field) |
| 2 | name | Textbox | ✔ | | min 2 / max 100 |
| 3 | subjectType | Dropdown | ✔ | | choices come from `tbl_subject_types` at runtime |
| 4 | hasLab | Toggle | | | Yes/No |
| 5 | hasTheory | Toggle | | | Yes/No |
| 6 | isElective | Toggle | | | Yes/No |
| 7 | classes | MultiSelect | ✔ | | `isYearWise`, `adminOnly`: choices come from academic config |
| 8 | isActive | Toggle | | | Active/Inactive |

`ensureDefaultFields(instModuleId, 'subjects')` (in `moduleField.controller.js`) runs whenever fields are read:
- If a default system field is **missing or soft-deleted**, it's upserted back. The school can never lose a system field.
- If a stored system field is **no longer** in the defaults, it's soft-deleted (that's how a system field gets retired).
- Fast path: if nothing changed, it returns after 1 query.

**Note:** the frontend also has a copy, `DEFAULT_SUBJECT_FIELDS` in `Subjects.tsx`. It's only used if the API returns no fields. It has already drifted from the backend copy (e.g. two fields with `order: 6`).

## 1.5 Runtime "hydration": dynamic choices
`GET /staff/modules/subjects/fields` runs `hydrateFieldsWithAcademicOptions()` before returning. For the `classes` field it injects:
```json
{ "isYearWise": true, "adminOnly": true,
  "currentYear": "2026-2027",
  "academicYears": ["2024-2025","2025-2026","2026-2027","2027-2028"],
  "yearChoices": { "2026-2027": ["LKG","UKG","I","II"], "2025-2026": ["LKG","UKG","I"], ... },
  "choices": ["LKG","UKG","I","II"] }
```
These come from `tbl_institution_academic_config` (set by Admin per year). That's why the school can't type its own class names: `adminOnly` locks them.
`subjectType` choices are added on the frontend from `GET /staff/subject-types` (ABC: SCIENCE, LANGUAGE, MATHEMATICS, CORE, …).

## 1.6 System field vs custom field: storage rule
| | System field | Custom field |
|---|---|---|
| Defined in | `defaultFields.config.js` | Created by school in Modify Form |
| `isSystemField` | `true` | `false` |
| Stored in | Its own column in `tbl_subjects` (`subjectCode`, `name`, …) | `tbl_subjects.customFields` JSONB under `fieldKey` |
| Can delete? | No (403); can only hide via `visible=false` | Yes (soft delete) |
| Search/filter | Hand-written per column in the controller | Generic JSONB `path`/`string_contains` |
| fieldKey | Fixed | `toCamelCase(label)`, e.g. "Credit Hours" becomes `creditHours` |

Example from the DB: subject `ENGLISH` has `customFields = {"lab": true, "newCustomField": ""}`. Both keys belong to custom fields that were later deleted. **Deleting a field hides it but never removes the stored values.**

## 1.7 Why it was designed this way
- **Every school wants a different form.** JSONB custom fields mean no migration and no deploy when a school adds "Credit Hours".
- **One engine, many modules.** The same field-definition table, validator, renderer and customiser serve Students, Staff, Subjects, Class & Batch, Fees… (rule: "Zero divergence").
- **System columns stay typed** (indexed, sortable, joinable, e.g. `classes` is used by the timetable and fee logic), while flexible data goes to JSONB.
- **Admin-controlled master data** (classes per year) keeps data consistent across modules (students, subjects, fees and timetable all use the same class names).
