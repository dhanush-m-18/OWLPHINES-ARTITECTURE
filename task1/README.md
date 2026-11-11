# Task 1: How the Subjects module works, end to end

Page analysed: `http://192.168.29.95:5174/subjects` (Staff_ERP)
Analysed 04-11-2026 against the code on `main` (ba5c5bf) and the live staging DB (institution **ABC**).
Purpose: understand this pattern fully before re-applying it to another module.

## Read in this order
| # | File | What you learn |
|---|---|---|
| 1 | [01_MODULE_AND_DYNAMIC_CONFIG.md](01_MODULE_AND_DYNAMIC_CONFIG.md) | The **concept**: what a "module" is, and how a form is defined by DB rows instead of code |
| 2 | [02_FRONTEND_FLOW.md](02_FRONTEND_FLOW.md) | How the page loads, builds the table and form from config, collects input, validates, and sends it |
| 3 | [03_BACKEND_FLOW.md](03_BACKEND_FLOW.md) | Route, middleware, controller: list, search, filter, sort, create, update, delete, import, export |
| 4 | [04_DATABASE_STORAGE.md](04_DATABASE_STORAGE.md) | Exactly what lands in which table and column, with **real rows** from the DB |
| 5 | [05_FORM_CUSTOMIZER.md](05_FORM_CUSTOMIZER.md) | The "Modify Form" button: how a staff member changes the form and how that's saved |
| 6 | [06_EVALUATION.md](06_EVALUATION.md) | Bugs, risks and quality issues found (verified) |
| 7 | [07_REPLICATION_GUIDE.md](07_REPLICATION_GUIDE.md) | Checklist for building the same feature in another module |

## The whole thing in one picture

```
            ┌───────────────────────── CONFIG (what the form looks like) ─────────────────────────┐
            │ tbl_system_modules (moduleKey='subjects')                                           │
            │    └─ tbl_institution_modules (ABC's copy: name, path, icon, enabled)               │
            │          └─ tbl_institution_module_fields  (one row per form field:                 │
            │                fieldKey, label, fieldType, visible, required, isUnique, order,      │
            │                isSystemField, options{choices, minLength, showInListing...})        │
            └──────────────────────────────────────────────────────────────────────────────────────┘
                     │ GET /api/staff/modules/subjects/fields      (+ academic-config, subject-types)
                     ▼
  Subjects.tsx ── visibleFields ──► table columns (listingFields) + drawer form (DynamicFormField per field)
       │                                         │
       │ GET /api/staff/subjects?page&search&... │ POST/PUT /api/staff/subjects  { subjectCode, name, subjectType,
       ▼                                         ▼                  hasLab, hasTheory, isElective, classes[], isActive,
  subject.routes.js → auth → staff → requirePermission('subjects', read|add|edit|delete)   customFields:{...} }
       ▼
  subject.controller.js
     getSubjectModuleFields()  ── reads the CONFIG rows (auto-seeds defaults if none)
     validateModuleFields()    ── validates the payload against the CONFIG (type, required, length, choices)
     validateModuleUniqueFields() ── isUnique fields checked in the DB
     build*Search/FilterConditions() ── search and filter built from the CONFIG
       ▼
  ┌──────────────────────────── DATA (the actual subjects) ────────────────────────────┐
  │ tbl_subjects: fixed columns for system fields (subjectCode, name, subjectType,     │
  │               hasLab, hasTheory, isElective, classes[], isActive …)                │
  │               + customFields JSONB for every field a school adds itself            │
  └────────────────────────────────────────────────────────────────────────────────────┘
       ▼ emitToInstitution('subjects_updated') → every open Subjects page refetches
```

## Key ideas in 7 lines
1. A **module** is a feature (Subjects). The global definition lives in `tbl_system_modules`, and each school has its own copy in `tbl_institution_modules`.
2. The **form is data, not code**: each field is a row in `tbl_institution_module_fields`. Change a row and the form, table, search, filters and export all change.
3. **System fields** (`isSystemField=true`) map to real columns of `tbl_subjects`. **Custom fields** are stored inside the `customFields` JSONB column.
4. Defaults come from `backend/src/config/defaultFields.config.js` (`defaultSubjectFields`) and are **auto-seeded** the first time the module is opened.
5. The frontend never hard-codes columns. It loops over `visibleFields` / `listingFields` and renders `DynamicFormField` / `DynamicTableCell` by `fieldType`.
6. The backend validates the same config again (`validateModuleFields`), so frontend validation is only for UX.
7. Classes are **not** typed by the school. They come from the per-year academic config set by Admin (`adminOnly`, `isYearWise`).

## Top findings (details in 06)
- 🔴 Any logged-in staff member can change the Subjects **form layout** (`superadminMiddleware` doesn't check the role), even though the UI only shows the button to users with `edit`.
- 🟠 Clicking a **custom-field column header to sort** sends `sortBy=<customKey>` to Prisma `orderBy`, which errors.
- 🟠 The `classFilter` param is ignored on the default sort path (subjectCode).
- 🟠 Bulk import skips all dynamic validation and dumps every unknown Excel column into `customFields`.
- 🟠 Custom fields can **never** be made unique: the customiser forces `isUnique=false` for non-system fields.
- 🟡 Hydrated runtime data (`yearChoices`, `academicYears`, `currentYear`) gets saved back into field options. This is confirmed in the DB.
- 🟡 `createdBy` / `updatedBy` are always `null` (they use `req.user.id/email`, which the staff token doesn't have).
