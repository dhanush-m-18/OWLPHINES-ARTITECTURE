# 4. Database: what is stored where (real data, institution ABC)

## 4.1 Tables touched by the Subjects page
| Table | Model | Read/Write by Subjects |
|---|---|---|
| `tbl_system_modules` | Module | read (moduleKey `subjects`) |
| `tbl_institution_modules` | InstitutionModule | read (resolve module, permissions) |
| `tbl_role_module_permissions`, `tbl_staff_module_permissions` | permissions | read (`requirePermission`) |
| `tbl_institution_module_fields` | InstitutionModuleField | read (form config); write via Modify Form |
| `tbl_institution_academic_config` | InstitutionAcademicConfig | read (classes per year) |
| `tbl_subject_types` | SubjectType | read (dropdown + `subjectTypeId` link) |
| `tbl_class_batches` | ClassBatch | read (class filter options) |
| **`tbl_subjects`** | **Subject** | **read/write: the data** |

## 4.2 `tbl_subjects` (`prisma/schema/subject.prisma`)
```prisma
model Subject {
  id            String  @id @default(uuid())
  institutionId String                       // tenant (from JWT)
  subjectCode   String                       // system field, UPPERCASE, natural-sorted
  name          String
  subjectType   String                       // the NAME (source of truth for display/filter)
  subjectTypeId String?                      // optional FK → tbl_subject_types (often null)
  hasLab        Boolean @default(false)
  hasTheory     Boolean @default(false)
  isElective    Boolean @default(false)
  classFrom     String?                      // legacy: = classes[0]
  classTo       String?                      // legacy: = classes[last]
  classes       String[] @default([])        // Postgres text[]: real class list
  customFields  Json?   @db.JsonB            // ← ALL school-defined fields
  isActive      Boolean @default(true)
  isDeleted     Boolean @default(false)      // soft delete
  createdBy, updatedBy String?               // ⚠ always null in practice
  createdAt, updatedAt
  @@unique([institutionId, subjectCode])      // per school; also covers deleted rows → create revives them
  @@index([institutionId]) @@index([institutionId, isDeleted]) @@index([institutionId, subjectType])
  @@map("tbl_subjects")
}
```
Relations: `StaffSubjectAssignment`, `AttendanceSession`, `TimetableSlot` point to `Subject.id`. That's why subjects are soft-deleted, not removed.

## 4.3 Real field config rows for ABC (`tbl_institution_module_fields`, institutionModuleId `23edb4be…`)
| fieldKey | type | system | visible | required | unique | order | deleted | options keys stored |
|---|---|---|---|---|---|---|---|---|
| subjectCode | Textbox | ✔ | ✔ | ✔ | ✔ | 1 | | maxLength, minLength, showInExport, showInListing |
| name | Textbox | ✔ | ✔ | ✔ | | 2 | | maxLength, minLength, showIn* |
| subjectType | Dropdown | ✔ | ✔ | ✔ | | 3 | | choices, showIn* |
| hasLab | Toggle | ✔ | ✔ | | | 4 | | **choices, adminOnly, maxLength, minLength, isYearWise**, trueLabel, falseLabel, showIn* (junk from type changes) |
| isElective | Toggle | ✔ | ✔ | | | 4 | | trueLabel, falseLabel, showIn* |
| classes | MultiSelect | ✔ | ✔ | ✔ | | 5 | | choices, adminOnly, isYearWise, **currentYear, yearChoices, academicYears** (runtime data persisted) |
| hasTheory | Toggle | ✔ | ✔ | | | 5 | | trueLabel, falseLabel, showIn* |
| lab | Toggle | | ✗ | | | 7 | ✔ | (deleted custom field) |
| isActive | Toggle | ✔ | ✔ | | | 7 | | trueLabel, falseLabel, showIn* |
| newCustomField | Textbox | | ✗ | | | 8 | ✔ | (deleted custom field) |

Notes: duplicate `order` values (4/4, 5/5, 7/7), so display order between them is decided by the sort tie-breaker.

Stored `classes.options` (shows the persisted snapshot):
```json
{"choices":[],"adminOnly":true,"isYearWise":true,"currentYear":"2026-2027",
 "yearChoices":{"2024-2025":["LKG","UKG","I"],"2025-2026":["LKG","UKG","I"],
                "2026-2027":["LKG","UKG","I","II"],"2027-2028":["LKG","UKG","I","II"]},
 "showInExport":true,"academicYears":["2024-2025","2025-2026","2026-2027","2027-2028"],"showInListing":true}
```

## 4.4 Real data row (`tbl_subjects`, ABC: 6 active, 0 deleted)
```json
{ "subjectCode": "ENGLISH", "name": "ENGLISH", "subjectType": "LANGUAGE", "subjectTypeId": null,
  "hasLab": false, "hasTheory": false, "isElective": true,
  "classFrom": "LKG", "classTo": "UKG", "classes": ["LKG","UKG"],
  "customFields": { "lab": true, "newCustomField": "" },
  "isActive": true, "isDeleted": false, "createdBy": null, "updatedBy": null }
```
What this shows:
- System fields land in **their own typed columns**.
- Custom fields land in **`customFields` JSONB**. `lab` and `newCustomField` are values from fields that were later deleted. The data is kept but no longer displayed.
- `subjectTypeId` is null even though type `LANGUAGE` exists, so the string `subjectType` is what the app relies on.
- `classFrom` and `classTo` are derived duplicates of `classes`, kept for backward compatibility.

## 4.5 Supporting master data for ABC
- Subject types: SCIENCE, SOCIAL STUDIES, COMMERCE, EXTRACURRICULAR, VOCATIONAL, OTHER, CORE, LANGUAGE, MATHEMATICS.
- Academic config: 2024-25 `[LKG,UKG,I]`, 2025-26 `[LKG,UKG,I]`, **2026-27 (current)** `[LKG,UKG,I,II]`, 2027-28 `[LKG,UKG,I,II]`.

## 4.6 How the DB answers each UI action
| UI action | SQL / Prisma (simplified) |
|---|---|
| Open page | `SELECT … FROM tbl_subjects WHERE "institutionId"=$1 AND "isDeleted"=false ORDER BY natural(subjectCode) DESC LIMIT 10` + `COUNT(*)` |
| Search "eng" | `… AND (LOWER(subjectCode) LIKE '%eng%' OR LOWER(name) LIKE … OR array_to_string(classes) LIKE … OR customFields::text ILIKE …)` |
| Filter Class = I | `… AND ('I' = ANY(s."classes") OR classFrom='I' OR classTo='I')` |
| Filter custom Dropdown | `LOWER(s."customFields"->>'key') = LOWER($v)` |
| Create | `INSERT INTO tbl_subjects (...)` (or `UPDATE` reviving a soft-deleted code) |
| Unique check | `SELECT id FROM tbl_subjects WHERE institutionId AND isDeleted=false AND subjectCode ILIKE $v AND id <> $self` |
| Delete | `UPDATE tbl_subjects SET "isDeleted"=true, "isActive"=false WHERE id=$1` |
| Modify Form save | `UPDATE/UPSERT tbl_institution_module_fields …` (one statement per field) |

## 4.7 Indexing
Covered: `[institutionId]`, `[institutionId, isDeleted]`, `[institutionId, subjectType]`, unique `[institutionId, subjectCode]`.
Not covered: `customFields` (no GIN index on subjects, unlike staff/students), `classes` array (no GIN), and the `LIKE '%…%'` searches. All fine at ~6–1,000 subjects per school.
