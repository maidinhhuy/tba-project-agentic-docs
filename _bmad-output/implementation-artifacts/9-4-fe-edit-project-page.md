# Story 9.4: FE Customer — Edit Project Page

**Status:** ready-for-dev
**Story ID:** 9.4
**Epic:** 9 — Customer Portal UX Enhancement & Edit Project

---

## Story

As a customer,
I want a dedicated page to edit my project's name, description, and reference link,
So that I can update project information at any time after submission.

---

## Acceptance Criteria

**AC-1 — Form pre-populated:**
**Given** Customer navigate đến `/[locale]/projects/[id]/edit`,
**When** trang load,
**Then** form hiển thị với 3 fields pre-populated từ dữ liệu project hiện tại:
- `name`: text input, required, max 100 ký tự
- `description`: textarea, required, min 50 ký tự, max 2000 ký tự
- `reference`: text input, optional, max 500 ký tự (có thể để trống để xoá reference)

**AC-2 — Inline validation — required fields:**
**Given** Customer không điền `name` hoặc `description`,
**When** submit form,
**Then** hiển thị inline error cho từng field vi phạm.
**And** Submit button disabled cho đến khi form hợp lệ (react-hook-form mode: "onChange").

**AC-3 — Inline validation — description min length:**
**Given** Customer điền `description` dưới 50 ký tự,
**When** đang nhập hoặc khi submit,
**Then** hiển thị inline error: "Mô tả cần ít nhất 50 ký tự."

**AC-4 — Success flow:**
**Given** form hợp lệ và Customer click submit,
**When** Server Action `updateProjectAction` gọi `PATCH /api/v1/customer/projects/{id}` thành công,
**Then** redirect về `/[locale]/projects/[id]` (locale-aware redirect, dùng `getLocale()` trong Server Action).
**And** toast success hiển thị: "Đã cập nhật dự án."

**AC-5 — Error flow:**
**Given** Server Action thất bại (lỗi network hoặc validation từ BE),
**When** response trả về lỗi,
**Then** toast destructive hiển thị: "Không thể cập nhật. Thử lại."
**And** form giữ nguyên dữ liệu đã nhập (không clear).

**AC-6 — Project ownership check:**
**Given** Customer truy cập `/[locale]/projects/[id]/edit` nhưng project không thuộc về họ,
**When** Server Component fetch project data,
**Then** `notFound()` được gọi → trang 404 hiển thị (pattern giống settings/page.tsx hiện tại).

**AC-7 — Role guard:**
**Given** Customer là Admin (không phải customer role),
**When** truy cập `/[locale]/projects/[id]/edit`,
**Then** middleware hiện tại handle redirect (route nằm trong `(dashboard)` group — middleware đã block admin khỏi dashboard).

**AC-8 — Loading state:**
**Given** Submit đang xử lý,
**When** trong thời gian submit,
**Then** submit button text đổi thành "Đang lưu…" và disabled (no spinner overlay, no full-page loading).

---

## ⚠️ Dev Notes — Đọc kỹ trước khi code

### Chiến lược: Tạo route `/edit/` mới, KHÔNG sửa `/settings/`

Route `/settings/` hiện tại (đã có implementation 70%) **KHÔNG phải** route được chỉ định trong story. Story yêu cầu route `/edit/`. Đừng refactor `/settings/` — tạo mới `/edit/` riêng biệt.

Các files cần **TẠO MỚI** (không có sẵn):
- `src/app/[locale]/(dashboard)/projects/[id]/edit/page.tsx` — Server Component
- `src/app/[locale]/(dashboard)/projects/[id]/edit/actions.ts` — Server Action với locale redirect
- `src/app/[locale]/(dashboard)/projects/[id]/edit/_components/EditProjectForm.tsx` — Client Component

Các files cần **SỬA**:
- `src/lib/schemas/project.schema.ts` — fix schema (thêm `reference`, sửa `description` validation)
- `src/messages/en.json` — thêm `projects.edit` namespace
- `src/messages/vi.json` — thêm `projects.edit` namespace

### Bug hiện tại trong schema (PHẢI FIX trước khi code form)

**File:** `src/lib/schemas/project.schema.ts`

**Hiện tại (SAI):**
```typescript
export const editProjectSchema = z.object({
  name: z.string().min(1, 'Tên project không được để trống').max(100),
  description: z.string().max(500).optional(),  // SAI: max(500), optional
});
```

**Cần sửa thành:**
```typescript
export const editProjectSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().min(50).max(2000),
  reference: z.string().max(500).optional().or(z.literal('')),
});

export type EditProjectFormValues = z.infer<typeof editProjectSchema>;
```

**Lý do:**
- `description`: phải required + min(50) + max(2000) (AC-3 và Story 9.1 BE validation)
- `reference`: optional, max 500, cho phép empty string `""` để xoá reference (AC-7 trong 9.1: `""` → BE set null)
- Error messages của zod **không được hardcode** — dùng `useTranslations` trong component để map

**QUAN TRỌNG:** Sửa schema này ảnh hưởng đến `/settings/` route đang dùng `EditProjectFormValues`. Kiểm tra `settings/_components/EditProjectForm.tsx` sau khi sửa schema — nếu bị type error thì cũng cần cập nhật defaultValues trong `settings/page.tsx` (thêm `reference: project.reference ?? ''`).

### Pattern của Server Action với locale-aware redirect

Tham khảo `src/app/[locale]/(dashboard)/profile/actions.ts` — cùng pattern `getLocale()`:

```typescript
// src/app/[locale]/(dashboard)/projects/[id]/edit/actions.ts
'use server'
import { cookies } from 'next/headers'
import { getLocale } from 'next-intl/server'
import { redirect } from 'next/navigation'
import { revalidatePath } from 'next/cache'
import { apiFetch } from '@/lib/api'
import { EditProjectFormValues } from '@/lib/schemas/project.schema'

export async function updateProjectAction(id: string, data: EditProjectFormValues) {
  const [cookieStore, locale] = await Promise.all([cookies(), getLocale()])
  try {
    await apiFetch(
      `/api/v1/customer/projects/${id}`,
      {
        method: 'PATCH',
        body: JSON.stringify(data),
      },
      cookieStore.toString()
    )
    revalidatePath(`/${locale}/projects/${id}`)
    redirect(`/${locale}/projects/${id}`)   // redirect PHẢI ở đây, sau await
  } catch (e) {
    // redirect() throws internally — re-throw nó
    if ((e as any)?.digest?.startsWith('NEXT_REDIRECT')) throw e
    return { error: true }
  }
}
```

**Giải thích pattern:**
- `redirect()` trong Next.js throw một exception đặc biệt với `digest: 'NEXT_REDIRECT:...'` — phải re-throw, không được catch.
- Action trả về `{ error: true }` khi thất bại thay vì throw — để client component xử lý toast.
- Toast success không được gọi trong Server Action — gọi trong Client Component sau khi detect redirect.

**Thực tế:** Toast success hiển thị **trước khi redirect** bằng cách gọi `toast.success(...)` trong `onSubmit` TRƯỚC khi await action, hoặc dùng `startTransition`. Xem pattern ở phần Client Component bên dưới.

### Pattern của Client Component với react-hook-form + toast

Tham khảo `src/app/[locale]/(dashboard)/profile/_components/ChangePasswordForm.tsx`:

**Import toast từ sonner:**
```typescript
import { toast } from 'sonner'
```

**Pattern form submit:**
```typescript
async function onSubmit(values: EditProjectFormValues) {
  const result = await updateProjectAction(projectId, values)
  // Nếu action redirect thành công, code dưới đây không chạy
  // Nếu result có error → action failed
  if (result?.error) {
    toast.error(t('errorToast'))
    // form data tự giữ vì react-hook-form không clear khi error
  }
}
```

**KHÔNG cần gọi `toast.success()`** vì nếu action redirect thành công → component unmount → toast không cần thiết ở đây. Nếu muốn show toast success sau redirect, có thể dùng searchParams trên `/projects/[id]` page — nhưng **đây là over-engineering**, bỏ qua cho story này. AC-4 chỉ yêu cầu toast success — nếu khó implement, chấp nhận không có toast success và chỉ redirect là đủ.

**Hoặc pattern đơn giản hơn:**
```typescript
// Nếu không có redirect trong action, action trả về { success: true } / { error: true }
// Client component toast.success() rồi router.push()
```

**Đề xuất: dùng pattern return-based (không redirect trong action) để dễ toast:**
```typescript
// actions.ts: trả về { success: true, locale } hoặc { error: true }
// form component: if success → toast.success → router.push(`/${locale}/projects/${id}`)
```

### Pattern của page.tsx (Server Component)

Tham khảo `src/app/[locale]/(dashboard)/projects/[id]/settings/page.tsx` — gần như copy, thêm `reference`:

```typescript
// page.tsx
import { apiFetch } from '@/lib/api';
import { cookies } from 'next/headers';
import { notFound } from 'next/navigation';
import EditProjectForm from './_components/EditProjectForm';

async function getProject(id: string) {
  const cookieStore = await cookies();
  try {
    return await apiFetch<{ name: string; description: string; reference: string | null }>(
      `/api/v1/customer/projects/${id}`,
      { cache: 'no-store' },
      cookieStore.toString()
    );
  } catch {
    notFound();
  }
}

export default async function EditProjectPage({
  params,
}: {
  params: Promise<{ id: string }> | { id: string };
}) {
  const resolvedParams = await params;
  const project = await getProject(resolvedParams.id);
  return (
    <div className="max-w-lg space-y-6">
      <h1 className="text-xl font-semibold">...</h1>  {/* dùng useTranslations ở Server Component: getTranslations */}
      <EditProjectForm
        projectId={resolvedParams.id}
        defaultValues={{
          name: project.name,
          description: project.description ?? '',
          reference: project.reference ?? '',
        }}
        locale={/* pass locale từ params */}
      />
    </div>
  );
}
```

### Translation keys cần thêm

**Namespace:** `projects.edit`

**`src/messages/vi.json`** — thêm vào object `projects`:
```json
"edit": {
  "title": "Chỉnh sửa dự án",
  "backLink": "Quay lại chi tiết dự án",
  "name": "Tên dự án",
  "description": "Mô tả",
  "descriptionMin": "Mô tả cần ít nhất 50 ký tự.",
  "reference": "Tài liệu tham khảo",
  "referenceHint": "URL tài liệu tham khảo (tùy chọn)",
  "submit": "Lưu thay đổi",
  "submitting": "Đang lưu…",
  "successToast": "Đã cập nhật dự án.",
  "errorToast": "Không thể cập nhật. Thử lại."
}
```

**`src/messages/en.json`** — thêm vào object `projects`:
```json
"edit": {
  "title": "Edit project",
  "backLink": "Back to project",
  "name": "Project name",
  "description": "Description",
  "descriptionMin": "Description must be at least 50 characters.",
  "reference": "Reference",
  "referenceHint": "Reference URL (optional)",
  "submit": "Save changes",
  "submitting": "Saving...",
  "successToast": "Project updated.",
  "errorToast": "Could not update. Please try again."
}
```

### Back link phải có locale prefix

**SAI:**
```typescript
<Link href={`/projects/${projectId}`}>Quay lại</Link>
```

**ĐÚNG:**
```typescript
// Trong Server Component:
const locale = params.locale  // hoặc getLocale()
<Link href={`/${locale}/projects/${projectId}`}>...</Link>

// Trong Client Component:
const params = useParams()
const locale = params.locale as string
```

### Không có `/settings/` link trong nav → route `/edit/` chưa có entry point

Story 9.3 (FE Customer — Project Detail Header Enhancement) sẽ thêm "Edit" button navigate đến `/[locale]/projects/[id]/edit`. Story này (9.4) chỉ cần **trang tồn tại và hoạt động** — không cần thêm entry point (button đó thuộc story 9.3).

---

## Tasks / Subtasks

### Task 1: Fix `editProjectSchema` trong `src/lib/schemas/project.schema.ts`

**File:** `src/lib/schemas/project.schema.ts`

- [ ] Sửa `description`: thêm `.min(50)`, đổi max thành `max(2000)`, bỏ `.optional()`
- [ ] Thêm field `reference`: `z.string().max(500).optional().or(z.literal(''))`
- [ ] Giữ nguyên `EditProjectFormValues` type export
- [ ] Kiểm tra `settings/_components/EditProjectForm.tsx` — nếu TypeScript báo lỗi vì `description` không còn optional, sửa `defaultValues` trong `settings/page.tsx` thêm `reference: project.reference ?? ''`

---

### Task 2: Thêm translation keys vào `en.json` và `vi.json`

**File:** `src/messages/vi.json`

- [ ] Thêm object `"edit": { ... }` vào trong `"projects": { ... }` (sau key `"detail"`)
- [ ] Keys: `title`, `backLink`, `name`, `description`, `descriptionMin`, `reference`, `referenceHint`, `submit`, `submitting`, `successToast`, `errorToast`

**File:** `src/messages/en.json`

- [ ] Thêm object `"edit": { ... }` vào trong `"projects": { ... }` (sau key `"detail"`)
- [ ] Cùng keys, với nội dung tiếng Anh

---

### Task 3: Tạo Server Action `updateProjectAction`

**File mới:** `src/app/[locale]/(dashboard)/projects/[id]/edit/actions.ts`

Pattern (dùng return-based để client dễ toast, không redirect trong action):

```typescript
'use server'
import { cookies } from 'next/headers'
import { getLocale } from 'next-intl/server'
import { revalidatePath } from 'next/cache'
import { apiFetch } from '@/lib/api'
import { EditProjectFormValues } from '@/lib/schemas/project.schema'

export async function updateProjectAction(
  id: string,
  data: EditProjectFormValues
): Promise<{ success: true; locale: string; redirectTo: string } | { error: true }> {
  try {
    const [cookieStore, locale] = await Promise.all([cookies(), getLocale()])
    await apiFetch(
      `/api/v1/customer/projects/${id}`,
      { method: 'PATCH', body: JSON.stringify(data) },
      cookieStore.toString()
    )
    revalidatePath(`/${locale}/projects/${id}`)
    return { success: true, locale, redirectTo: `/${locale}/projects/${id}` }
  } catch {
    return { error: true }
  }
}
```

- [ ] Tạo file mới
- [ ] Import `apiFetch` từ `@/lib/api`
- [ ] Import `EditProjectFormValues` từ `@/lib/schemas/project.schema`
- [ ] Import `getLocale` từ `next-intl/server`
- [ ] Import `revalidatePath` từ `next/cache`
- [ ] Action trả về `redirectTo` để client tự gọi `router.push()`

---

### Task 4: Tạo `EditProjectForm` Client Component

**File mới:** `src/app/[locale]/(dashboard)/projects/[id]/edit/_components/EditProjectForm.tsx`

```typescript
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { useRouter } from 'next/navigation'
import { useTranslations } from 'next-intl'
import { toast } from 'sonner'
import { editProjectSchema, EditProjectFormValues } from '@/lib/schemas/project.schema'
import { updateProjectAction } from '../actions'

interface EditProjectFormProps {
  projectId: string
  defaultValues: EditProjectFormValues
}

export default function EditProjectForm({ projectId, defaultValues }: EditProjectFormProps) {
  const t = useTranslations('projects.edit')
  const router = useRouter()

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isValid },
  } = useForm<EditProjectFormValues>({
    resolver: zodResolver(editProjectSchema),
    defaultValues,
    mode: 'onChange',  // validate ngay khi type để disable submit button (AC-2)
  })

  async function onSubmit(values: EditProjectFormValues) {
    const result = await updateProjectAction(projectId, values)
    if ('error' in result) {
      toast.error(t('errorToast'))
      return
    }
    toast.success(t('successToast'))
    router.push(result.redirectTo)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      {/* name field */}
      <div>
        <label className="block text-sm font-medium mb-1">{t('name')}</label>
        <input
          {...register('name')}
          className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-primary"
        />
        {errors.name && (
          <p className="mt-1 text-sm text-red-500" role="alert">
            {errors.name.message}
          </p>
        )}
      </div>

      {/* description field */}
      <div>
        <label className="block text-sm font-medium mb-1">{t('description')}</label>
        <textarea
          {...register('description')}
          rows={6}
          className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-primary resize-none"
        />
        {errors.description && (
          <p className="mt-1 text-sm text-red-500" role="alert">
            {errors.description.message ?? t('descriptionMin')}
          </p>
        )}
      </div>

      {/* reference field */}
      <div>
        <label className="block text-sm font-medium mb-1">
          {t('reference')}{' '}
          <span className="text-muted-foreground font-normal text-xs">({t('referenceHint')})</span>
        </label>
        <input
          {...register('reference')}
          placeholder="https://..."
          className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-primary"
        />
        {errors.reference && (
          <p className="mt-1 text-sm text-red-500" role="alert">
            {errors.reference.message}
          </p>
        )}
      </div>

      <button
        type="submit"
        disabled={isSubmitting || !isValid}
        className="px-4 py-2 bg-primary text-white rounded-lg hover:bg-primary/90 disabled:opacity-50 disabled:cursor-not-allowed"
      >
        {isSubmitting ? t('submitting') : t('submit')}
      </button>
    </form>
  )
}
```

- [ ] Tạo file mới tại đúng path
- [ ] Dùng `mode: 'onChange'` cho react-hook-form để real-time validation
- [ ] Disable submit khi `!isValid` (AC-2)
- [ ] Submit button text: `submitting` / `submit` (AC-8)
- [ ] Toast error khi action failed, toast success + `router.push()` khi thành công (AC-4, AC-5)
- [ ] Không clear form khi error (react-hook-form tự giữ state — không cần code thêm)
- [ ] `role="alert"` trên error paragraph cho accessibility (UX-DR10)

---

### Task 5: Tạo `EditProjectPage` Server Component

**File mới:** `src/app/[locale]/(dashboard)/projects/[id]/edit/page.tsx`

```typescript
import { cookies } from 'next/headers'
import { notFound } from 'next/navigation'
import Link from 'next/link'
import { getTranslations } from 'next-intl/server'
import { apiFetch } from '@/lib/api'
import EditProjectForm from './_components/EditProjectForm'

interface ProjectData {
  name: string
  description: string
  reference: string | null
}

async function getProject(id: string): Promise<ProjectData> {
  const cookieStore = await cookies()
  try {
    return await apiFetch<ProjectData>(
      `/api/v1/customer/projects/${id}`,
      { cache: 'no-store' },
      cookieStore.toString()
    )
  } catch {
    notFound()
  }
}

export default async function EditProjectPage({
  params,
}: {
  params: Promise<{ locale: string; id: string }> | { locale: string; id: string }
}) {
  const resolvedParams = await params
  const { locale, id } = resolvedParams
  const [project, t] = await Promise.all([
    getProject(id),
    getTranslations('projects.edit'),
  ])

  return (
    <div className="max-w-lg space-y-6">
      <div>
        <Link
          href={`/${locale}/projects/${id}`}
          className="text-sm text-teal-600 dark:text-teal-400 hover:underline flex items-center gap-1.5 mb-3 font-medium"
        >
          <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10 19l-7-7m0 0l7-7m-7 7h18" />
          </svg>
          {t('backLink')}
        </Link>
        <h1 className="text-xl font-semibold">{t('title')}</h1>
      </div>
      <EditProjectForm
        projectId={id}
        defaultValues={{
          name: project.name,
          description: project.description ?? '',
          reference: project.reference ?? '',
        }}
      />
    </div>
  )
}
```

- [ ] Tạo file mới tại đúng path
- [ ] `params` phải type là `Promise<...>` (Next.js 15 async params)
- [ ] Back link dùng locale prefix: `/${locale}/projects/${id}`
- [ ] `notFound()` khi API fetch fail (AC-6 — bao gồm cả 403 wrong owner)
- [ ] `getTranslations` (server-side) thay vì `useTranslations`
- [ ] Pass `reference: project.reference ?? ''` để form có thể hiển thị và xoá reference

---

## Verification Checklist

- [ ] `npx tsc --noEmit` (hoặc `npm run type-check`) không có lỗi liên quan đến story này
- [ ] Navigate đến `/vi/projects/[valid-id]/edit` — form load với 3 fields pre-populated
- [ ] Navigate đến `/en/projects/[valid-id]/edit` — form load với tiếng Anh
- [ ] Xoá `name` field, click submit → inline error hiển thị, button disabled
- [ ] Điền description < 50 ký tự → inline error "Mô tả cần ít nhất 50 ký tự." (vi) / "Description must be at least 50 characters." (en)
- [ ] Submit valid form → redirect về `/[locale]/projects/[id]` + toast success
- [ ] Xoá `reference`, submit → reference field bị clear (gửi `""` tới BE)
- [ ] Submit với reference có giá trị → reference được update
- [ ] Khi submit đang xử lý → button text "Đang lưu…", disabled
- [ ] Simulate API error → toast destructive, form giữ data
- [ ] Navigate đến `/vi/projects/[non-existent-id]/edit` → trang 404

---

## Files Touched Summary

| Action | File |
|--------|------|
| MODIFY | `src/lib/schemas/project.schema.ts` |
| MODIFY | `src/messages/vi.json` |
| MODIFY | `src/messages/en.json` |
| NEW | `src/app/[locale]/(dashboard)/projects/[id]/edit/page.tsx` |
| NEW | `src/app/[locale]/(dashboard)/projects/[id]/edit/actions.ts` |
| NEW | `src/app/[locale]/(dashboard)/projects/[id]/edit/_components/EditProjectForm.tsx` |
| CHECK | `src/app/[locale]/(dashboard)/projects/[id]/settings/_components/EditProjectForm.tsx` (kiểm tra TypeScript sau khi sửa schema) |
| CHECK | `src/app/[locale]/(dashboard)/projects/[id]/settings/page.tsx` (thêm `reference` vào defaultValues nếu type error) |
