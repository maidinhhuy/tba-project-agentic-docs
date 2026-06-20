# Story 7.1: Admin Project Detail — Tabs View/Manage + Markdown Description

**Epic:** 7 — UI Bug Fixes  
**Story ID:** 7.1  
**Status:** ready-for-dev  
**Type:** Bug Fix + UI Refactor (FE-only)

---

## User Story

As an admin,
I want to see complete project information (description, reference URL) in a dedicated "Thông tin" tab with Markdown rendering, and manage project status/milestones in a separate "Quản lý" tab,
So that viewing and managing are clearly separated and I have full context about the project.

---

## Root Cause Analysis (Đã Xác Nhận)

**BE đúng — FE thiếu + cấu trúc page chưa hợp lý.**

`GET /api/v1/admin/projects/{id}` đã trả về `description` và `reference`:

```java
// AdminProjectDetailResponse.java (modules/infrastructure)
public record AdminProjectDetailResponse(
    String projectId,
    Long customerId,
    String customerEmail,
    String name,
    String productType,
    String description,   // ✅ đã có
    String reference,     // ✅ đã có
    String status,
    ...
```

FE interface thiếu 2 field này và page không có cấu trúc tabs rõ ràng:

```typescript
// page.tsx HIỆN TẠI — thiếu description, reference; view/update mixed
interface AdminProjectDetail {
  projectId: string; name: string; customerEmail: string; status: string;
  allowedTransitions: string[]; revisionCount: number;
  milestones: {...}[]; statusHistory: {...}[]; version: number;
  // ❌ thiếu: description, reference, productType
}
```

---

## Acceptance Criteria

### Tab "Thông tin" (Info tab)

**AC1 — Tab structure:**
**Given** Admin load `/[locale]/admin/projects/{id}`,
**When** page render hoàn thành,
**Then** hiện 2 tabs: "Thông tin" (active mặc định) và "Quản lý".

**AC2 — Description render Markdown:**
**Given** project có `description` (không null/rỗng),
**When** Admin xem tab "Thông tin",
**Then** description được render bằng `react-markdown` + `remark-gfm` (GFM: bold, italic, links, lists, code blocks).

**AC3 — Reference URL:**
**Given** project có `reference` (không null/rỗng),
**When** Admin xem tab "Thông tin",
**Then** render `reference` dưới dạng external link với `target="_blank" rel="noopener noreferrer"`.

**AC4 — Ẩn section khi null/rỗng:**
**Given** `description` hoặc `reference` là null/rỗng,
**When** render tab "Thông tin",
**Then** section tương ứng không hiện (không có label rỗng).

**AC5 — Status History ở tab "Thông tin":**
**Given** project có status history,
**When** Admin xem tab "Thông tin",
**Then** status history vẫn hiển thị ở tab này (giữ nguyên logic hiện tại).

### Tab "Quản lý" (Manage tab)

**AC6 — UpdateStatusForm ở tab "Quản lý":**
**Given** Admin click tab "Quản lý",
**When** tab render,
**Then** hiện `UpdateStatusForm` (update status + progress note) và Milestone section — giữ nguyên toàn bộ logic hiện tại.

**AC7 — Không thay đổi logic update:**
**Given** Admin thực hiện update status hoặc milestone,
**When** action hoàn thành,
**Then** behavior giữ nguyên như trước (toast, revalidate, v.v.) — không có regression.

---

## Technical Specification

### Files cần thay đổi

```
tba-project-agentic-fe/
  src/
    app/[locale]/(admin)/admin/projects/[id]/
      page.tsx                        ← SỬA CHÍNH
    components/ui/
      tabs.tsx                        ← TẠO MỚI (shadcn add)
    messages/
      vi.json                         ← THÊM i18n keys
      en.json                         ← THÊM i18n keys
```

> **KHÔNG sửa:** `_actions.ts`, `UpdateStatusForm.tsx`

### Bước 0 — Thêm shadcn Tabs component

```bash
npx shadcn@latest add tabs
```

Tạo file `src/components/ui/tabs.tsx`. Dùng import:
```typescript
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
```

### Bước 1 — Cập nhật TypeScript interface

```typescript
interface AdminProjectDetail {
  projectId: string
  name: string
  customerEmail: string
  productType: string          // ← THÊM (đã có trong response)
  description: string | null  // ← THÊM
  reference: string | null    // ← THÊM
  status: string
  allowedTransitions: string[]
  revisionCount: number
  milestones: { milestoneId: string; name: string; status: string; position: number }[]
  statusHistory: {
    previousStatus: string | null
    newStatus: string
    changedBy: string
    reason: string | null
    changedAt: string
  }[]
  version: number
}
```

> **Lưu ý:** BE trả về field tên `reference` (không phải `referenceUrl`). FE phải dùng đúng tên này.

### Bước 2 — Cấu trúc page mới với Tabs

```tsx
'use client' // ← CẦN THÊM vì Tabs dùng client-side state

// Hoặc giữ page là Server Component và tách Tabs ra _components/ProjectDetailTabs.tsx (client component)
// → Recommended: tách ra component riêng để không mất server rendering cho data fetch
```

**Pattern khuyến nghị — tách client component:**

```tsx
// page.tsx (Server Component — giữ nguyên async fetch)
import { ProjectDetailTabs } from './_components/ProjectDetailTabs'

export default async function AdminProjectDetailPage(...) {
  const project = await apiFetch<AdminProjectDetail>(...)
  
  return (
    <div className="max-w-4xl mx-auto py-8 px-4 space-y-6">
      {/* Back Link — giữ nguyên */}
      
      {/* Project Header Card — giữ nguyên */}
      <div className="bg-white border border-gray-200/80 rounded-xl p-6 shadow-sm ...">
        <h1>{project.name}</h1>
        <p>{project.customerEmail}</p>
        <StatusBadge status={project.status} />
      </div>

      {/* Tabs — client component nhận toàn bộ project data */}
      <ProjectDetailTabs project={project} />
    </div>
  )
}
```

```tsx
// _components/ProjectDetailTabs.tsx (Client Component)
'use client'
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs'
import { UpdateStatusForm } from './UpdateStatusForm'

export function ProjectDetailTabs({ project }: { project: AdminProjectDetail }) {
  return (
    <Tabs defaultValue="info">
      <TabsList>
        <TabsTrigger value="info">Thông tin</TabsTrigger>
        <TabsTrigger value="manage">Quản lý</TabsTrigger>
      </TabsList>

      {/* Tab 1: Thông tin */}
      <TabsContent value="info" className="space-y-6 mt-4">
        {/* Description (Markdown) */}
        {project.description && (
          <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-3">
            <h2 className="text-lg font-semibold text-gray-900 pb-3 border-b border-gray-100">
              Mô tả dự án
            </h2>
            <div className="prose prose-sm prose-gray max-w-none">
              <ReactMarkdown remarkPlugins={[remarkGfm]}>
                {project.description}
              </ReactMarkdown>
            </div>
          </div>
        )}

        {/* Reference URL */}
        {project.reference && (
          <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-2">
            <p className="text-xs font-medium text-gray-500 uppercase tracking-wide">
              Tài liệu tham khảo
            </p>
            <a
              href={project.reference}
              target="_blank"
              rel="noopener noreferrer"
              className="text-sm text-teal-600 hover:text-teal-700 hover:underline break-all transition-colors"
            >
              {project.reference}
            </a>
          </div>
        )}

        {/* Status History — di chuyển từ page.tsx vào đây */}
        {project.statusHistory.length > 0 && (
          <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-4">
            <h2 className="...">Lịch sử trạng thái</h2>
            {/* giữ nguyên JSX history hiện tại */}
          </div>
        )}
      </TabsContent>

      {/* Tab 2: Quản lý */}
      <TabsContent value="manage" className="space-y-6 mt-4">
        {/* UpdateStatusForm — giữ nguyên props */}
        <UpdateStatusForm
          projectId={project.projectId}
          currentStatus={project.status}
          allowedTransitions={project.allowedTransitions}
          revisionCount={project.revisionCount}
          version={project.version}
        />

        {/* Milestones — di chuyển từ page.tsx vào đây */}
        {project.milestones && project.milestones.length > 0 && (
          <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-4">
            {/* giữ nguyên JSX milestones hiện tại */}
          </div>
        )}
      </TabsContent>
    </Tabs>
  )
}
```

### Tailwind Prose (Markdown styling)

`prose` class từ `@tailwindcss/typography` cần được cài nếu chưa có:

```bash
# Kiểm tra trước
grep "typography" tba-project-agentic-fe/package.json
```

Nếu chưa có:
```bash
npm install @tailwindcss/typography
# Thêm vào tailwind.config.ts: plugins: [require('@tailwindcss/typography')]
```

Nếu không muốn thêm plugin, dùng custom CSS thay `prose`:
```tsx
<div className="text-sm text-gray-700 leading-relaxed [&_strong]:font-semibold [&_a]:text-teal-600 [&_ul]:list-disc [&_ul]:pl-4 [&_ol]:list-decimal [&_ol]:pl-4">
  <ReactMarkdown remarkPlugins={[remarkGfm]}>{project.description}</ReactMarkdown>
</div>
```

### i18n (nếu dùng `getTranslations`)

Nếu chuyển sang client component, cần truyền translated strings qua props từ Server Component (client component không dùng được `getTranslations`).

**Option A (đơn giản):** Hardcode tiếng Việt trong `ProjectDetailTabs.tsx` vì trang admin thường chỉ có 1 ngôn ngữ.

**Option B (đúng i18n):** Fetch translations ở `page.tsx` và truyền xuống:
```tsx
// page.tsx
const t = await getTranslations('admin')
// ...
<ProjectDetailTabs project={project} labels={{ info: t('tabs.info'), manage: t('tabs.manage'), ... }} />
```

---

## Dev Checklist

- [ ] Chạy `npx shadcn@latest add tabs` để tạo `src/components/ui/tabs.tsx`
- [ ] Kiểm tra `@tailwindcss/typography` — cài nếu cần, hoặc dùng custom CSS approach
- [ ] Cập nhật TypeScript interface với `description`, `reference`, `productType`
- [ ] Tạo `_components/ProjectDetailTabs.tsx` (Client Component)
- [ ] Tab "Thông tin": description (ReactMarkdown + remark-gfm), reference URL, status history
- [ ] Tab "Quản lý": UpdateStatusForm + milestones (move từ page.tsx)
- [ ] `page.tsx` refactor: giữ async fetch, render Header Card + `<ProjectDetailTabs />`
- [ ] Kiểm tra: project có description Markdown → render đúng (bold, list, link)
- [ ] Kiểm tra: project không có description/reference → section ẩn
- [ ] Kiểm tra: update status vẫn hoạt động đúng ở tab "Quản lý"
- [ ] Kiểm tra: milestone update vẫn hoạt động đúng
- [ ] Không có TypeScript errors

---

## Phạm Vi Tuyệt Đối Không Thay Đổi

- Không sửa BE
- Không sửa `_actions.ts`
- Không sửa logic bên trong `UpdateStatusForm.tsx`
- Không thay đổi routing hay auth logic
- Behavior của update status + milestone giữ nguyên hoàn toàn

---

## Story Notes

**Effort:** ~1.5–2 giờ  
**Risk:** Thấp — không đụng logic, chỉ refactor cấu trúc UI + thêm render  
**Dependency:** shadcn Tabs chưa có → phải `shadcn add tabs` trước  
**Possible blocker:** `@tailwindcss/typography` — cần quyết định dùng `prose` hay custom CSS  
