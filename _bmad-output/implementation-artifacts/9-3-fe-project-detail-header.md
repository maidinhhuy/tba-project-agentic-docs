# Story 9.3: FE Customer — Project Detail Header Enhancement

**Status:** ready-for-dev
**Story ID:** 9.3
**Epic:** 9 — Customer Portal UX Enhancement & Edit Project

---

## Story

As a customer,
I want the project detail page header to show the revision counter alongside the status badge, a collapsible description with reference link, and an edit button,
So that I can quickly understand project status and access key actions without scrolling.

---

## Acceptance Criteria

**AC-1 — Revision counter chip trong header:**
**Given** Customer truy cập `/[locale]/projects/[id]`,
**When** trang load xong,
**Then** revision counter chip hiển thị trong project header area, nằm cạnh status badge (không phải bên dưới stepper như trước).
**And** chip chỉ hiển thị khi `project.status = AWAITING_REVIEW` hoặc `IN_REVISION`.
**And** chip text: "Còn {n} lần chỉnh sửa". Khi `revisionCount ≤ 1`: chữ đổi sang `text-destructive`, text đổi thành "Còn 1 lần chỉnh sửa cuối".
**And** chip ẩn hoàn toàn khi `status = DELIVERED` hoặc `CANCELLED`.

> **Note thực tế:** Revision counter chip ĐÃ được implement đúng vị trí trong header (lines 171–184 của page.tsx hiện tại, trong flex row cùng với StatusBadge). Story này KHÔNG cần move revision counter. Tuy nhiên cần:
> 1. Fix i18n cho chip text (hiện dùng `revisionRemaining` / `revisionsRemaining` — check xem text có khớp AC chưa)
> 2. Fix hardcoded warning "Bạn đã sử dụng hết lượt chỉnh sửa." (line 193) sang i18n key

**AC-2 — Collapsible description:**
**Given** project có description,
**When** render Project Detail,
**Then** description hiển thị dưới dạng `CollapsibleDescription` component (`src/components/ui/CollapsibleDescription.tsx`):
- Mặc định collapsed: 3 dòng visible via `-webkit-line-clamp: 3`
- Button toggle: text "Xem thêm" + chevron-down khi collapsed; text "Thu gọn" + chevron-up (xoay 180°) khi expanded
- Click toggle → expand full content; click lại → collapse
- Toggle state không cần persist (reset mỗi lần load trang — session only)

**AC-3 — Reference link:**
**Given** project có `reference` (non-null, non-empty string),
**When** render Project Detail,
**Then** reference field hiển thị bên dưới collapsible description, phân cách bằng top border.
**And** render như external link: icon `ExternalLink` (lucide) + text URL, `target="_blank"`, `rel="noopener noreferrer"`.
**And** khi `reference` là null hoặc empty string, reference field không render (ẩn hoàn toàn).

> **Note:** Reference đã hiển thị trong page.tsx hiện tại (lines 208–230). Story 9-3 chỉ cần đảm bảo reference vẫn render đúng sau khi wrap description bằng `CollapsibleDescription`. Không cần thay đổi reference implementation — giữ nguyên.

**AC-4 — Edit button trong header:**
**Given** Customer đang xem trang `/[locale]/projects/[id]`,
**When** render project header,
**Then** button "Chỉnh sửa" hiển thị trong header area (cùng flex row với StatusBadge và revision counter chip).
**And** button chỉ visible cho Customer — route `/[locale]/projects/[id]` là customer-only route (protected bởi middleware RBAC), nên button luôn render (không cần role check ở component level).
**And** click button → navigate đến `/${locale}/projects/${id}/edit` (locale-aware, dùng locale từ `params` vì page là Server Component).

**AC-5 — Accessibility cho collapsible description:**
**Given** Customer dùng screen reader,
**When** tương tác với collapsible description,
**Then** toggle button có `aria-expanded={isExpanded}`, content container có `aria-hidden={!isExpanded}` khi collapsed.

---

## ⚠️ Dev Notes — Đọc kỹ trước khi code

### Trạng thái hiện tại của page.tsx (CRITICAL — đọc trước khi code)

File: `src/app/[locale]/(dashboard)/projects/[id]/page.tsx` (330 lines hiện tại)

**Sau story 9-2 (MilestoneStepper), page.tsx sẽ đã được thay đổi:**
- Đã sửa `Milestone.status: 'COMPLETED' → 'DONE'`
- Đã xóa `MilestoneIcon` function (lines 51–71)
- Đã xóa imports `MilestoneDeliverable`, `MilestoneReviewForm`
- Đã thêm `MilestoneStepper` component
- Layout: không còn 2-column grid (8+4) — content section đã full-width

**QUAN TRỌNG:** Story 9-3 CÓ THỂ implement độc lập (không cần 9-2 hoàn thành trước) vì:
- `CollapsibleDescription` nằm trong Card description — không phụ thuộc layout stepper
- Edit button nằm trong header — không phụ thuộc layout stepper
- Nếu implement 9-3 trước 9-2: file page.tsx vẫn còn grid layout cũ (8+4) — OK, chỉ cần thêm đúng chỗ

### Revision counter — Đã đúng vị trí, cần check text

**Hiện tại (lines 169–185 page.tsx):**
```tsx
<div className="flex flex-row sm:flex-col items-center sm:items-end gap-3 shrink-0">
  <StatusBadge status={project.status} className="text-sm px-3 py-1" />
  {showRevisionCounter && (
    <span ... >
      {project.revisionCount <= 1
        ? t('revisionRemaining')      // vi.json: "Còn lại 1 lần chỉnh sửa"
        : t('revisionsRemaining', { count: project.revisionCount })}  // "Còn lại {count} lần chỉnh sửa"
    </span>
  )}
</div>
```

**AC yêu cầu text:** "Còn {n} lần chỉnh sửa" / "Còn 1 lần chỉnh sửa cuối"

**Hiện tại vi.json có:**
- `revisionRemaining`: "Còn lại 1 lần chỉnh sửa"
- `revisionsRemaining`: "Còn lại {count} lần chỉnh sửa"

**Cần update translation keys** để khớp đúng AC text:
- `revisionRemaining` → "Còn 1 lần chỉnh sửa cuối" (khi count ≤ 1)
- `revisionsRemaining` → "Còn {count} lần chỉnh sửa" (khi count > 1)

### Hardcoded string cần i18n (line 193)

**Hiện tại (line 188–195 page.tsx):**
```tsx
{project.revisionCount === 0 && (
  <div className="flex items-center gap-2 p-3.5 bg-orange-50 ...">
    ...
    <span>Bạn đã sử dụng hết lượt chỉnh sửa.</span>  {/* HARDCODED — cần i18n */}
  </div>
)}
```

**Fix:** Thêm key `projects.detail.revisionExhausted` vào vi.json và en.json, dùng `t('revisionExhausted')`.

### Edit button — Server Component approach

Page `page.tsx` là Server Component. `params` đã được destructure từ `ProjectDetailContent({ id, locale })`. Construct href trực tiếp:

```tsx
<Link
  href={`/${locale}/projects/${id}/edit`}
  className="inline-flex items-center gap-1.5 px-3 py-1.5 text-sm font-medium rounded-lg border border-teal-600 text-teal-600 hover:bg-teal-50 dark:border-teal-400 dark:text-teal-400 dark:hover:bg-teal-950/20 transition-colors"
>
  <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
  </svg>
  {t('editProject')}
</Link>
```

**Lý do không cần role check:** Route `/[locale]/projects/[id]` được bảo vệ bởi middleware RBAC — chỉ Customer mới vào được. Admin navigate vào đây bình thường sẽ bị redirect. Không cần thêm conditional render.

### Vị trí Edit button trong header flex row

Header flex row hiện tại (lines 151–186 page.tsx):
```tsx
<div className="flex flex-col sm:flex-row sm:items-start justify-between gap-4 pb-4 border-b">
  <div>  {/* left: back link + title + productType */}
    ...
  </div>
  <div className="flex flex-row sm:flex-col items-center sm:items-end gap-3 shrink-0">
    {/* right: StatusBadge + revision chip */}
    <StatusBadge ... />
    {showRevisionCounter && <span>...</span>}
  </div>
</div>
```

**Thêm Edit button** vào right div, TRƯỚC StatusBadge (hoặc sau — tùy UI preference, đặt cuối cùng hợp lý hơn):
```tsx
<div className="flex flex-row sm:flex-col items-center sm:items-end gap-3 shrink-0">
  <StatusBadge status={project.status} className="text-sm px-3 py-1" />
  {showRevisionCounter && <span>...</span>}
  <Link href={`/${locale}/projects/${id}/edit`} ...>
    {t('editProject')}
  </Link>
</div>
```

### CollapsibleDescription component

**Path:** `src/components/ui/CollapsibleDescription.tsx` (NEW — chưa tồn tại)

**Loại:** `'use client'` component (cần `useState`).

**Implementation:**
```tsx
'use client'

import { useState } from 'react'
import { useTranslations } from 'next-intl'

interface CollapsibleDescriptionProps {
  content: string
  className?: string
}

export function CollapsibleDescription({ content, className }: CollapsibleDescriptionProps) {
  const [isExpanded, setIsExpanded] = useState(false)
  const t = useTranslations('projects.detail')

  return (
    <div className={className}>
      <div
        aria-hidden={!isExpanded}
        style={
          !isExpanded
            ? {
                display: '-webkit-box',
                WebkitLineClamp: 3,
                WebkitBoxOrient: 'vertical',
                overflow: 'hidden',
              }
            : undefined
        }
        className="text-sm text-gray-700 dark:text-gray-300 leading-relaxed whitespace-pre-wrap"
      >
        {content}
      </div>
      <button
        type="button"
        aria-expanded={isExpanded}
        onClick={() => setIsExpanded(prev => !prev)}
        className="mt-2 inline-flex items-center gap-1 text-sm font-medium text-teal-600 hover:text-teal-700 dark:text-teal-400 dark:hover:text-teal-300 transition-colors"
      >
        {isExpanded ? t('showLess') : t('showMore')}
        <svg
          className={`h-4 w-4 transition-transform duration-200 ${isExpanded ? 'rotate-180' : ''}`}
          fill="none"
          viewBox="0 0 24 24"
          stroke="currentColor"
        >
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 9l-7 7-7-7" />
        </svg>
      </button>
    </div>
  )
}
```

**Note:** Dùng `useTranslations('projects.detail')` để lấy `showMore` / `showLess` keys. Component này đặt trong `src/components/ui/` để đồng bộ với các UI component khác, và để story 9-5 (Admin) có thể reuse dễ dàng.

### Wrap description trong page.tsx

**Hiện tại (lines 199–232 page.tsx) — Description Card:**
```tsx
<Card>
  <CardHeader className="pb-3">
    <h2>...</h2>
  </CardHeader>
  <CardContent className="space-y-4">
    <MarkdownPreview content={project.description} />   {/* ← REPLACE bằng CollapsibleDescription */}

    {project.reference && (
      <div className="mt-4 pt-4 border-t ...">
        ...
      </div>
    )}
  </CardContent>
</Card>
```

**Sau khi sửa:**
```tsx
<Card>
  <CardHeader className="pb-3">
    <h2>...</h2>
  </CardHeader>
  <CardContent className="space-y-4">
    <CollapsibleDescription content={project.description} />  {/* NEW */}

    {project.reference && (
      <div className="mt-4 pt-4 border-t ...">
        ...  {/* giữ nguyên */}
      </div>
    )}
  </CardContent>
</Card>
```

**QUAN TRỌNG:** Xóa `<MarkdownPreview content={project.description} />` và thay bằng `<CollapsibleDescription content={project.description} />`. Nếu `MarkdownPreview` không còn dùng ở đây (nhưng vẫn dùng trong Activity Feed line 249), **đừng xóa import `MarkdownPreview`**.

### Translation keys cần thêm/cập nhật

**vi.json — trong `projects.detail` object:**
```json
"editProject": "Chỉnh sửa dự án",
"showMore": "Xem thêm",
"showLess": "Thu gọn",
"revisionExhausted": "Bạn đã sử dụng hết lượt chỉnh sửa.",
"revisionRemaining": "Còn 1 lần chỉnh sửa cuối",
"revisionsRemaining": "Còn {count} lần chỉnh sửa"
```

**en.json — trong `projects.detail` object:**
```json
"editProject": "Edit Project",
"showMore": "Show more",
"showLess": "Show less",
"revisionExhausted": "You have used all revision attempts.",
"revisionRemaining": "1 revision remaining (last)",
"revisionsRemaining": "{count} revisions remaining"
```

**Lưu ý:** `revisionRemaining` và `revisionsRemaining` đã tồn tại — **cập nhật giá trị** chứ không tạo mới. Đảm bảo không làm mất format `{count}` placeholder.

### Kiểm tra import MarkdownPreview

`MarkdownPreview` vẫn được dùng trong Activity Feed (line 249):
```tsx
<MarkdownPreview content={entry.reason ?? ''} className="text-sm" />
```

Khi xóa `<MarkdownPreview content={project.description} />` trong Description Card, **giữ import `MarkdownPreview`** ở đầu file vì nó vẫn dùng ở activity section.

---

## Tasks / Subtasks

### Task 1: Tạo component `CollapsibleDescription.tsx`

**File:** `src/components/ui/CollapsibleDescription.tsx` (NEW)

- [ ] Tạo file với implementation đúng như dev notes (có `'use client'`, `useState`, `useTranslations`)
- [ ] Thêm `aria-expanded` trên button, `aria-hidden` trên content div
- [ ] SVG chevron xoay 180° khi expanded (via `rotate-180` Tailwind class + `transition-transform`)
- [ ] Dùng `useTranslations('projects.detail')` cho keys `showMore` / `showLess`

---

### Task 2: Thêm translation keys vào vi.json và en.json

**File:** `src/messages/vi.json`

Trong `projects.detail` object, thêm mới và cập nhật:
```json
"editProject": "Chỉnh sửa dự án",
"showMore": "Xem thêm",
"showLess": "Thu gọn",
"revisionExhausted": "Bạn đã sử dụng hết lượt chỉnh sửa.",
"revisionRemaining": "Còn 1 lần chỉnh sửa cuối",
"revisionsRemaining": "Còn {count} lần chỉnh sửa"
```

**File:** `src/messages/en.json`

Trong `projects.detail` object, thêm mới và cập nhật:
```json
"editProject": "Edit Project",
"showMore": "Show more",
"showLess": "Show less",
"revisionExhausted": "You have used all revision attempts.",
"revisionRemaining": "1 revision remaining (last)",
"revisionsRemaining": "{count} revisions remaining"
```

- [ ] Cập nhật vi.json (update `revisionRemaining`, `revisionsRemaining`; thêm 4 keys mới)
- [ ] Cập nhật en.json (update `revisionRemaining`, `revisionsRemaining`; thêm 4 keys mới)

---

### Task 3: Cập nhật `page.tsx` — Edit button

**File:** `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`

**Thêm import `CollapsibleDescription`:**
```tsx
import { CollapsibleDescription } from '@/components/ui/CollapsibleDescription'
```

**Thêm Edit button** vào header right div (sau revision counter chip, trước hoặc sau — đặt sau cùng):
```tsx
<div className="flex flex-row sm:flex-col items-center sm:items-end gap-3 shrink-0">
  <StatusBadge status={project.status} className="text-sm px-3 py-1" />
  {showRevisionCounter && (
    <span ...>{...}</span>
  )}
  <Link
    href={`/${locale}/projects/${id}/edit`}
    className="inline-flex items-center gap-1.5 px-3 py-1.5 text-sm font-medium rounded-lg border border-teal-600 text-teal-600 hover:bg-teal-50 dark:border-teal-400 dark:text-teal-400 dark:hover:bg-teal-950/20 transition-colors"
  >
    <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
      <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
    </svg>
    {t('editProject')}
  </Link>
</div>
```

- [ ] Thêm import `CollapsibleDescription`
- [ ] Thêm Edit button Link trong header flex row (right div)
- [ ] Verify `locale` và `id` đã available trong `ProjectDetailContent` props — đúng, cả hai có từ function signature

---

### Task 4: Cập nhật `page.tsx` — Wrap description + fix i18n

**File:** `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`

**4a — Thay `MarkdownPreview` bằng `CollapsibleDescription` trong description Card:**
```tsx
{/* Trước: */}
<MarkdownPreview content={project.description} />

{/* Sau: */}
<CollapsibleDescription content={project.description} />
```

**4b — Fix hardcoded string revisionExhausted (line 193):**
```tsx
{/* Trước: */}
<span>Bạn đã sử dụng hết lượt chỉnh sửa.</span>

{/* Sau: */}
<span>{t('revisionExhausted')}</span>
```

- [ ] Replace `<MarkdownPreview content={project.description} />` với `<CollapsibleDescription content={project.description} />`
- [ ] Đảm bảo import `MarkdownPreview` vẫn còn (vì vẫn dùng trong activity feed)
- [ ] Fix hardcoded "Bạn đã sử dụng hết lượt chỉnh sửa." → `{t('revisionExhausted')}`
- [ ] Verify revision counter chip text khớp i18n keys mới (không cần sửa logic, chỉ cần vi.json đúng)

---

## Verification Checklist

- [ ] `npx tsc --noEmit` không lỗi
- [ ] Trang `/vi/projects/[id]` load được, không crash
- [ ] Trang `/en/projects/[id]` load được, không crash
- [ ] Description Card hiển thị `CollapsibleDescription` — mặc định collapsed (3 dòng)
- [ ] Click "Xem thêm" → mở full content, button đổi sang "Thu gọn" + chevron xoay
- [ ] Click "Thu gọn" → collapse về 3 dòng, button đổi về "Xem thêm"
- [ ] `aria-expanded` và `aria-hidden` được set đúng trên các elements
- [ ] Reference link vẫn hiển thị đúng dưới description (nếu project có reference)
- [ ] Reference ẩn hoàn toàn nếu null/empty (không thay đổi behavior cũ)
- [ ] Edit button hiển thị trong header, cạnh StatusBadge
- [ ] Click Edit button → navigate đến `/${locale}/projects/${id}/edit` (404 expected vì story 9-4 chưa làm)
- [ ] Revision counter chip vẫn hiển thị đúng khi status = AWAITING_REVIEW hoặc IN_REVISION
- [ ] Text revision counter khớp: "Còn {n} lần chỉnh sửa" / "Còn 1 lần chỉnh sửa cuối"
- [ ] Warning orange block (revisionCount === 0): không còn hardcoded, dùng i18n key
- [ ] `MarkdownPreview` vẫn dùng trong Activity Feed (không bị xóa)
- [ ] `CollapsibleDescription` export có thể reuse từ `@/components/ui/CollapsibleDescription`

---

## Files Touched Summary

| Action | File |
|--------|------|
| NEW | `src/components/ui/CollapsibleDescription.tsx` |
| MODIFY | `src/app/[locale]/(dashboard)/projects/[id]/page.tsx` |
| MODIFY | `src/messages/vi.json` |
| MODIFY | `src/messages/en.json` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_actions.ts` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx` |
| NO CHANGE | `src/components/MarkdownPreview.tsx` |
