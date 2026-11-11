# Code Templates (copy, rename `Thing`/`things`/`thing`, adapt)

These follow the house style exactly. Replace `Thing`, `things` and `moduleKey`.

## 1. Prisma model: `backend/prisma/schema/thing.prisma`
```prisma
model Thing {
  id            String      @id @default(uuid())
  institutionId String
  institution   Institution @relation(fields: [institutionId], references: [id], onDelete: Cascade)
  code          String
  name          String
  customFields  Json?       @db.JsonB
  isActive      Boolean     @default(true)
  isDeleted     Boolean     @default(false)
  createdBy     String?
  updatedBy     String?
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  @@unique([institutionId, code])
  @@index([institutionId])
  @@index([institutionId, isDeleted])
  @@map("tbl_things")
}
```
Then add `things Thing[]` to `institution.prisma`, and run `npx prisma validate && npx prisma db push && npx prisma generate`.

## 2. Validator: `staff_side/validators/thing.validator.js`
```js
import { z } from 'zod';
import { booleanToggle, stringOrNumber, stringOrNumberOptional } from '../../utils/zodHelpers.js';

export const createThingSchema = {
  body: z.object({
    code: stringOrNumber.refine((v) => v.length >= 1, 'Code is required'),
    name: stringOrNumber.refine((v) => v.length >= 2, 'Name is required (min 2 characters)'),
    isActive: booleanToggle(true),
    customFields: z.any().optional().nullable(),
  }).passthrough(),
};

export const updateThingSchema = {
  body: z.object({
    code: stringOrNumberOptional,
    name: stringOrNumberOptional,
    isActive: booleanToggle(),
    customFields: z.any().optional().nullable(),
  }).passthrough(),
};
```

## 3. Controller: `staff_side/controllers/thing.controller.js`
```js
import prisma from '../../config/database.js';
import { sendSuccess, sendError } from '../../utils/response.js';
import { emitToInstitution } from '../../config/socket.js';

const MAX_LIMIT = 500;

// ===========================================================================
// GET /things — Paginated list with search
// ===========================================================================
export const getThings = async (req, res, next) => {
  try {
    const { institutionId } = req.user;
    const { search, page, limit, tab } = req.query;

    const pageNum = Math.max(parseInt(page, 10) || 1, 1);
    const limitNum = Math.min(parseInt(limit, 10) || 10, MAX_LIMIT);

    const where = { institutionId, isDeleted: String(tab || '').toLowerCase() === 'trash' };
    if (search && typeof search === 'string' && search.trim()) {
      const q = search.trim();
      where.OR = [
        { name: { contains: q, mode: 'insensitive' } },
        { code: { contains: q, mode: 'insensitive' } },
      ];
    }

    const [items, total] = await Promise.all([
      prisma.thing.findMany({ where, orderBy: { createdAt: 'desc' }, skip: (pageNum - 1) * limitNum, take: limitNum }),
      prisma.thing.count({ where }),
    ]);

    const pagination = { page: pageNum, limit: limitNum, total, totalPages: Math.ceil(total / limitNum) || 1 };
    return sendSuccess(res, 200, 'Things retrieved successfully', items, { pagination, meta: { pagination } });
  } catch (error) {
    return next(error);
  }
};

// ===========================================================================
// POST /things — Create
// ===========================================================================
export const createThing = async (req, res, next) => {
  try {
    const { institutionId, staffId } = req.user;
    const { code, name, isActive, customFields } = req.body;   // destructure, never spread req.body

    const exists = await prisma.thing.findFirst({ where: { institutionId, code, isDeleted: false } });
    if (exists) return sendError(res, 400, `Code '${code}' is already in use in this institution.`);

    const created = await prisma.thing.create({
      data: { institutionId, code, name, isActive, customFields: customFields || null, createdBy: staffId, updatedBy: staffId },
    });

    emitToInstitution(institutionId, 'things_updated', { id: created.id });
    return sendSuccess(res, 201, 'Thing created successfully', created);
  } catch (error) {
    return next(error);
  }
};

// ===========================================================================
// PUT /things/:id — Update
// ===========================================================================
export const updateThing = async (req, res, next) => {
  try {
    const { institutionId, staffId } = req.user;
    const { id } = req.params;
    const { code, name, isActive, customFields } = req.body;

    const existing = await prisma.thing.findFirst({ where: { id, institutionId, isDeleted: false } });
    if (!existing) return sendError(res, 404, 'Thing not found');

    const data = { updatedBy: staffId };
    if (code !== undefined) data.code = code;
    if (name) data.name = name;
    if (isActive !== undefined) data.isActive = isActive;
    if (customFields !== undefined) data.customFields = customFields;

    const updated = await prisma.thing.update({ where: { id }, data });
    emitToInstitution(institutionId, 'things_updated', { id });
    return sendSuccess(res, 200, 'Thing updated successfully', updated);
  } catch (error) {
    return next(error);
  }
};

// ===========================================================================
// DELETE /things/:id — Soft delete
// ===========================================================================
export const deleteThing = async (req, res, next) => {
  try {
    const { institutionId, staffId } = req.user;
    const { id } = req.params;

    const existing = await prisma.thing.findFirst({ where: { id, institutionId, isDeleted: false } });
    if (!existing) return sendError(res, 404, 'Thing not found');

    await prisma.thing.update({ where: { id }, data: { isDeleted: true, isActive: false, updatedBy: staffId } });
    emitToInstitution(institutionId, 'things_updated', { id });
    return sendSuccess(res, 200, 'Thing deleted successfully');
  } catch (error) {
    return next(error);
  }
};
```

## 4. Routes: `staff_side/routes/thing.routes.js` (+ mount in `routes/index.js`: `router.use('/things', thingRoutes);`)
```js
import { Router } from 'express';
import { getThings, createThing, updateThing, deleteThing } from '../controllers/thing.controller.js';
import { authMiddleware } from '../middlewares/auth.middleware.js';
import { staffMiddleware } from '../middlewares/staff.middleware.js';
import { requirePermission } from '../middlewares/permission.middleware.js';
import { validateRequest } from '../middlewares/validate.middleware.js';
import { createThingSchema, updateThingSchema } from '../validators/thing.validator.js';

const router = Router();

router.use(authMiddleware, staffMiddleware);

router.get('/', requirePermission('things', 'read'), getThings);
router.post('/', requirePermission('things', 'add'), validateRequest(createThingSchema), createThing);
router.put('/:id', requirePermission('things', 'edit'), validateRequest(updateThingSchema), updateThing);
router.delete('/:id', requirePermission('things', 'delete'), deleteThing);

export default router;
```

## 5. Safe raw SQL snippet
```js
import { Prisma } from '@prisma/client';
// table name verified from @@map in thing.prisma → tbl_things
const dir = sortOrder === 'desc' ? Prisma.sql`DESC` : Prisma.sql`ASC`;
const search = q ? Prisma.sql`AND LOWER(t.name) LIKE ${`%${q.toLowerCase()}%`}` : Prisma.empty;
const rows = await prisma.$queryRaw`
  SELECT t.id FROM tbl_things t
  WHERE t."institutionId" = ${institutionId} AND t."isDeleted" = false
  ${search}
  ORDER BY NULLIF(regexp_replace(t.code, '\\D', '', 'g'), '')::bigint ${dir}, t.code ${dir}
  LIMIT ${take} OFFSET ${skip}
`;
```

## 6. Frontend API slice: `Staff_ERP/src/features/things/thingsApiSlice.ts`
```ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import { API_BASE_URL } from '../../config/app.config';

export const thingsApiSlice = createApi({
  reducerPath: 'thingsApi',
  baseQuery: fetchBaseQuery({ baseUrl: API_BASE_URL, credentials: 'include' }),
  tagTypes: ['Things'],
  endpoints: (builder) => ({
    getThings: builder.query({
      query: (params?: { page?: number; limit?: number; search?: string; tab?: string }) => ({ url: '/staff/things', params }),
      providesTags: ['Things'],
    }),
    createThing: builder.mutation({
      query: (body) => ({ url: '/staff/things', method: 'POST', body }),
      invalidatesTags: ['Things'],
    }),
    updateThing: builder.mutation({
      query: ({ id, ...body }) => ({ url: `/staff/things/${id}`, method: 'PUT', body }),
      invalidatesTags: ['Things'],
    }),
    deleteThing: builder.mutation({
      query: (id: string) => ({ url: `/staff/things/${id}`, method: 'DELETE' }),
      invalidatesTags: ['Things'],
    }),
  }),
});

export const { useGetThingsQuery, useCreateThingMutation, useUpdateThingMutation, useDeleteThingMutation } = thingsApiSlice;
```
Register it in `app/store.ts` (in `combineSlices(...)` and `.concat(thingsApiSlice.middleware)`), add
`socket.on('things_updated', () => store.dispatch(thingsApiSlice.util.invalidateTags(['Things'])));`
and add `dispatch(thingsApiSlice.util.resetApiState());` in `store/apiSlice.ts`.

## 7. Page skeleton: `Staff_ERP/src/pages/Things.tsx` (copy the real body from `ClassBatch.tsx`)
```tsx
import React, { useState } from 'react';
import { FaCubes } from 'react-icons/fa6';
import { useGetThingsQuery, useUpdateThingMutation } from '../features/things/thingsApiSlice';
import { useModulePermission } from '../hooks/useModulePermission';
import { useDebounce } from '../hooks/useDebounce';
import { SearchInput } from '../components/ui/SearchInput';
import { AddButton, ModifyFormButton } from '../components/ui/HeaderButtons';
import { PaginationFooter } from '../components/ui/PaginationFooter';
import { TableSkeleton } from '../components/ui/Skeleton';
import { useToast } from '../context/ToastContext';

export const ThingsPage = ({ moduleItem }: { moduleItem?: any }) => {
  const { add, edit, delete: canDelete } = useModulePermission('things');
  const toast = useToast();
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState('');
  const debounced = useDebounce(search, 400);
  const { data, isLoading, isFetching } = useGetThingsQuery(
    { page, limit: 10, search: debounced },
    { refetchOnMountOrArgChange: true }
  );
  const [updateThing] = useUpdateThingMutation();
  const [formData, setFormData] = useState<any>({});
  const [initialFormData, setInitialFormData] = useState<any>(null);
  const [editing, setEditing] = useState<any>(null);
  const [isDrawerOpen, setIsDrawerOpen] = useState(false);

  const openEdit = (row: any) => {
    const snapshot = { code: row.code, name: row.name, isActive: row.isActive };
    setEditing(row); setFormData(snapshot); setInitialFormData(JSON.parse(JSON.stringify(snapshot))); setIsDrawerOpen(true);
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (editing && initialFormData && JSON.stringify(formData) === JSON.stringify(initialFormData)) {
      toast.info('No changes found'); setIsDrawerOpen(false); setEditing(null); return;
    }
    try {
      const res = await updateThing({ id: editing.id, ...formData }).unwrap();
      toast.success(res?.message || 'Thing updated successfully');
      setIsDrawerOpen(false); setEditing(null);
    } catch (err: any) {
      toast.error(err?.data?.message || 'Failed to save');
    }
  };

  return (
    <div className="h-full flex flex-col min-h-0">
      <div className="bg-surface border border-surface rounded-2xl shadow-xs flex-1 flex flex-col min-h-0 overflow-hidden">
        <div className="px-4 sm:px-5 py-2.5 sm:py-2 border-b border-surface flex flex-col sm:flex-row sm:items-center justify-between gap-2.5 sm:gap-4 shrink-0">
          <h1 className="text-lg sm:text-xl font-bold text-main flex items-center gap-2"><FaCubes className="w-4 h-4" /> Things</h1>
          <div className="flex items-center gap-2">
            <ModifyFormButton onClick={() => {}} />
            {add && <AddButton label="Add Thing" onClick={() => {}} />}
          </div>
        </div>
        <div className="px-4 sm:px-5 py-2.5 border-b border-surface flex flex-col xl:flex-row xl:items-center justify-between gap-3 shrink-0">
          <SearchInput value={search} onChange={(v: string) => { setSearch(v); setPage(1); }} placeholder="Search things..." />
          {/* FilterButton, ExportExcelButton, ExportPdfButton, Choose Action (n) */}
        </div>
        <div className="flex-1 overflow-auto min-h-0 relative">
          {isLoading || isFetching ? <TableSkeleton /> : (
            <table className="w-full text-sm">{/* sticky thead + rows + EditActionButton/DeleteActionButton */}</table>
          )}
        </div>
        <div className="px-4 sm:px-5 py-2.5 border-t border-surface">
          <PaginationFooter page={page} totalItems={data?.pagination?.total || 0} onPageChange={setPage} />
        </div>
      </div>
      {/* <Drawer open={isDrawerOpen}> <form noValidate onSubmit={handleSubmit}> … </form> </Drawer> + <ConfirmModal/> */}
    </div>
  );
};
```
Then register `things: ThingsPage` in `config/moduleRegistry.ts`, and check the exact prop names of `SearchInput` / `PaginationFooter` in `components/ui` before use.

## 8. Fix snippets for review items (ready when approved)

**S-2 superadmin role check**
```js
const isSuper = staff.role?.name === 'SUPERADMIN' || staff.role?.roleKey?.toLowerCase() === 'superadmin';
if (!isSuper || staff.institutionId !== req.user.institutionId) {
  return sendError(res, 403, 'Access denied. Superadmin authorization required.');
}
```
**S-1 tenant-safe delete**
```js
const key = targetUrl.replace(`${R2_CONFIG.publicUrl}/`, '').split('?')[0];
const [root, , instId] = key.split('/');
if (root !== 'uploads' || instId !== req.user.institutionId) return sendError(res, 403, 'You cannot delete this file');
await deleteObjectFromR2(key);
```
**B-3 permission message**: replace `${targetKey}` with `${targetKeys[0]}`.
**B-2 table name**: `FROM module_records` becomes `FROM tbl_module_records` (3 places).
**S-6 OTP RNG**: `crypto.randomInt(100000, 1000000).toString()`.
**S-7 secret guard** (`config/env.js`): `if (!process.env.JWT_SECRET) throw new Error('JWT_SECRET is required');`
