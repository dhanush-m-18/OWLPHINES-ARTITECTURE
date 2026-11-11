# Project Structure

```
OWLPHIN/
├── AGENTS.md                         # mandatory rules (read first)
├── .agents/skills/owlphin-architecture/SKILL.md   # detailed rule book
├── backend/
├── Staff_ERP/
├── Admin_ERP/
└── owlphine-dhansuh/                 # ← these notes (untracked; add to .gitignore if you don't want to commit them)
```

## backend/ (~32k lines JS, ES modules, `"type": "module"`)

```
backend/
├── package.json            scripts: dev(nodemon) start prisma:* (test script points to a missing file)
├── prisma/
│   ├── schema/*.prisma     ONE MODEL PER FILE (+ base.prisma: generator + datasource, prismaSchemaFolder preview)
│   ├── migrations/         ⚠ only an old 'init' migration with OLD table names → team uses `prisma db push`
│   ├── seed.js             master data seeding
│   └── reset_db.js
└── src/
    ├── server.js           http server + socket init + DB connect with 5 retries (Neon cold start)
    ├── app.js              express app: helmet, cookies, cors, json, auth rate limit, /health, /api, 404, errorHandler
    ├── routes/index.js     mounts /admin and /staff (+ /staff/me alias)
    ├── config/
    │   ├── env.js          ALL env vars with defaults (single source)
    │   ├── database.js     PrismaClient singleton
    │   ├── socket.js       initSocket / emitToInstitution / emitToAll
    │   ├── r2.js           S3Client for Cloudflare R2
    │   └── defaultFields.config.js   default form fields per moduleKey
    ├── middlewares/        shared: rateLimiter (OTP send/verify), upload (multer image/attachment/logo)
    ├── services/           shared: otp.service, storage/r2.service, image/imageCompression.service
    ├── utils/
    │   ├── response.js     sendSuccess / sendError   ← always use
    │   ├── error.js        central errorHandler
    │   ├── jwt.js          generateAdminToken / generateStaffToken / generateManagementToken / verifyToken
    │   ├── date.js         parseFlexibleDate, formatDDMMYYYY, buildDateSearchConditions, getDateSearchVariants
    │   ├── fieldValidator.js  validateSingleField / validateModuleFields / validateModuleUniqueFields
    │   ├── filterOptions.js   populateDynamicFieldFilterOptions
    │   ├── feeStructureMatcher.js  match student → fee structure by criteria
    │   ├── zodHelpers.js   booleanToggle, stringOrNumber(Optional)
    │   ├── address.js, password.js
    ├── admin_side/
    │   ├── routes/         auth, categories, boards, institution-types, modules, academics, institutions,
    │   │                   institutions/:id/academic-config, attendance-config, staff, upload
    │   ├── controllers/    one per resource
    │   ├── middlewares/    auth (admin_token), admin (userType SYSTEM_ADMIN), validate
    │   ├── services/       auth.service, institution.service
    │   └── validators/     zod schemas
    └── staff_side/
        ├── routes/         28 routers (see API_REFERENCE.md)
        ├── controllers/    biggest: staff(2415) student(2414) feeReport(2363) feeStructure(1529) feePayment(1496)
        │                   attendance(1438) subject(1218) classBatch(1087)
        ├── middlewares/    auth, staff, permission(requirePermission), superadmin(⚠ no role check), validate
        ├── services/       permission, auth, institution, record, export(Excel/PDF), attendanceAccess
        └── validators/     zod schemas per resource
```

## Staff_ERP/ (~53k lines TS/TSX)

```
Staff_ERP/
├── package.json        dev/build/lint(oxlint); postinstall copies TinyMCE to public/tinymce
├── tsconfig.app.json   ⚠ strict:false, noImplicitAny:false
├── vite.config.ts      port 5174, host 0.0.0.0
└── src/
    ├── main.tsx        Redux Provider, blocks wheel-scroll on number inputs
    ├── App.tsx         Providers (Theme→Toast→Auth→Institution→Router), ProtectedLayout, ModuleRecordsWrapper
    ├── app/
    │   ├── createAppSlice.ts   buildCreateSlice + asyncThunkCreator
    │   ├── hooks.ts            useAppDispatch / useAppSelector (withTypes)
    │   └── store.ts            combineSlices + makeStore + setupListeners + ALL socket→invalidateTags wiring
    ├── store/
    │   ├── index.ts            re-export store (compat)
    │   └── apiSlice.ts         staffApi.util.resetApiState() (resets every slice) + hook re-exports (compat)
    ├── features/<name>/<name>ApiSlice.ts   one RTK Query createApi per feature (26 features)
    ├── config/
    │   ├── app.config.ts       API_BASE_URL, static defaults
    │   └── moduleRegistry.ts   moduleKey → page component
    ├── context/                AuthContext, InstitutionContext, ThemeContext, ToastContext
    ├── hooks/                  useDebounce, useModulePermission, useTableFilterSkeleton
    ├── services/               api.ts (fetch auth calls), socket.ts (io client + joinInstitutionRoom)
    ├── pages/                  one page per module: Student, Staff, Subjects, ClassBatch, Fee*, Attendance,
    │                           Timetable, AccessControl, Dashboard, Settings, Login, Reports/...
    ├── components/
    │   ├── ui/                 33 shared primitives (Button, Card, Drawer, Modal, ConfirmModal, SearchInput,
    │   │                       PaginationFooter, TableColumnFilterRow, HeaderButtons, ActionButton,
    │   │                       ExportButtons, ExportModal, Skeleton, DatePicker, TimePicker, FilledIcon …)
    │   ├── dynamic-fields/     DynamicFormField, DynamicTableCell, ModuleFormCustomizerDrawer, FieldOptionsEditor
    │   ├── common/             ImageUpload, MultiImageUpload, Cropper, RichTextEditor(TinyMCE)
    │   ├── layout/             PageLayout, Sidebar, Header
    │   └── staff/ student/ institution/ attendance/ timetable/ feeCollection/ …  (module specific)
    ├── utils/                  date.ts, export.ts, file.ts, string.ts
    ├── themes/                 blue/green/red.css
    └── types/index.ts
```

## Admin_ERP/

Same skeleton as Staff_ERP but smaller (10 pages). Features: `institutions`, `masterModules`, `boards`, `categories`, `institutionTypes`, `academicConfig`, `staff` (admin users), `upload`. Uses `adminApi.util.resetApiState()`.

## Where to put new code

| You're adding… | Put it in |
|---|---|
| New staff API resource | `staff_side/routes/<x>.routes.js` + `controllers/<x>.controller.js` + `validators/<x>.validator.js`, mount in `staff_side/routes/index.js` |
| Shared backend logic used by 2+ modules | `backend/src/utils/` |
| New DB table | `backend/prisma/schema/<model>.prisma` with `@@map("tbl_<name>")`, back-relation in `institution.prisma` |
| Default form fields for a module | `config/defaultFields.config.js` → `DEFAULT_MODULE_FIELDS` |
| Frontend API | `src/features/<x>/<x>ApiSlice.ts` → register in `app/store.ts` (reducer + middleware) **and** `store/apiSlice.ts` (resetApiState) |
| Socket refresh | `socket.on('<x>_updated', …invalidateTags)` in `app/store.ts` |
| New page | `src/pages/<X>.tsx` + `CUSTOM_MODULE_REGISTRY` entry |
| Reusable UI | `src/components/ui/` |
