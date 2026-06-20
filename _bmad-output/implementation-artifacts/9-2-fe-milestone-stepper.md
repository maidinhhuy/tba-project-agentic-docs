# Story 9.2: FE Customer — Horizontal Milestone Stepper

**Status:** ready-for-dev
**Story ID:** 9.2
**Epic:** 9 — Customer Portal UX Enhancement & Edit Project

---

## Story

As a customer,
I want to see my project's milestones displayed as a horizontal stepper with an expandable panel for the active step,
So that I can quickly grasp overall progress and take review actions without leaving the page.

---

## Acceptance Criteria

**AC-1 — Replace vertical list với horizontal stepper:**
**Given** Customer truy cập `/[locale]/projects/[id]`,
**When** trang load xong và project có milestones,
**Then** milestone list hiện tại (dạng rows trong sidebar 4/12) được thay thế bằng một horizontal stepper full-width.
**And** mỗi step gồm: circle indicator + tên milestone bên dưới circle + status label bên dưới tên.
**And** circle size: 48px cho DONE và PENDING; 56px cho ACTIVE.
**And** connector line giữa các circles: `bg-primary` cho đoạn đã DONE, `bg-border` cho đoạn còn pending.

**AC-2 — Circle styling:**
- **DONE**: 48px, `bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]`, hiển thị checkmark SVG icon.
- **ACTIVE**: 56px, `bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]`, hiển thị số thứ tự (position). Có `margin-top: -4px` để circle nhô lên khỏi connector line.
- **PENDING**: 48px, `bg-background border-2 border-border text-muted-foreground`, hiển thị số thứ tự.

**AC-3 — Panel expand tự động cho ACTIVE step:**
**Given** ACTIVE milestone có `reviewStatus = PENDING_REVIEW`,
**When** trang load xong,
**Then** panel expand mở tự động cho ACTIVE step.
**And** panel background: `bg-teal-50 border border-teal-100 rounded-lg`.
**And** panel có triangle arrow pointing up, fill `teal-50`, căn chỉnh trên active circle.

**AC-4 — Panel content:**
**Given** panel expand đang hiển thị,
**When** ACTIVE milestone có `reviewStatus = PENDING_REVIEW`,
**Then** panel chứa: `MilestoneDeliverable` (link + download + delivery note) và `MilestoneReviewForm` (action buttons).

**AC-5 — Accordion behavior:**
**Given** panel của ACTIVE step đang mở,
**When** Customer click vào circle của step khác (nếu step đó cho phép expand),
**Then** chỉ một panel mở tại một thời điểm — panel cũ tự động collapse.

**AC-6 — "Yêu cầu chỉnh sửa" expand textarea:**
**Given** Customer click "Yêu cầu chỉnh sửa" trong panel,
**When** button click,
**Then** textarea bắt buộc expand trong form (đã có trong `MilestoneReviewForm`).
**And** "Gửi yêu cầu" disabled đến khi textarea có nội dung.

**AC-7 — Submit revalidate:**
**Given** Customer submit review action (approve hoặc revision),
**When** Server Action thành công,
**Then** page revalidate, stepper cập nhật trạng thái mới, toast success hiển thị.

**AC-8 — Empty state:**
**Given** project chưa có milestone nào,
**When** render trang,
**Then** stepper section không render, thay bằng empty state: "Chưa có milestone nào."

**AC-9 — Skeleton loading:**
**Given** trang đang fetch dữ liệu,
**When** skeleton loading hiển thị,
**Then** stepper area hiển thị skeleton placeholder (3 circles + connectors) với `aria-busy="true"`.

**AC-10 — Locale-aware navigation:**
**All** `href` trong component phải dùng locale prefix. Server Component dùng `getLocale()`, Client Component dùng `useParams()`.

---

## ⚠️ Dev Notes — Đọc kỹ trước khi code

### Bug quan trọng: COMPLETED vs DONE

**CRITICAL — ĐÂY LÀ BUG HIỆN TẠI:**

`page.tsx` line 20 dùng:
```typescript
status: 'ACTIVE' | 'PENDING' | 'COMPLETED'
```

Backend Java enum (`MilestoneStatus.java`) thực tế là:
```java
public enum MilestoneStatus { PENDING, ACTIVE, DONE }
```

**Backend serialize thành `"DONE"`, không phải `"COMPLETED"`. Toàn bộ logic `m.status === 'COMPLETED'` trong trang hiện tại không bao giờ match.**

Khi tạo `MilestoneStepper`, phải dùng `'DONE'` — và **cần sửa luôn interface trong `page.tsx`** từ `'COMPLETED'` → `'DONE'`.

### Files hiện tại cần đọc và hiểu

#### 1. `page.tsx` — file chính sẽ bị thay đổi nhiều

**Path:** `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`

**Cấu trúc hiện tại:**
- `ProjectDetailContent` là Server Component, fetch data từ `/api/v1/customer/projects/${id}`
- `Milestone` interface có `status: 'ACTIVE' | 'PENDING' | 'COMPLETED'` — **BUG, sửa thành `'DONE'`**
- Layout hiện tại: 2-column grid (`md:col-span-8` content + `md:col-span-4` milestone sidebar)
- Milestone sidebar ở lines 264–317: `<ol>` vertical list với `MilestoneIcon`

**Thay đổi cần làm trong `page.tsx`:**
1. Sửa `Milestone.status` type từ `'COMPLETED'` → `'DONE'`
2. Xóa toàn bộ `MilestoneIcon` function (không còn dùng)
3. Xóa import `MilestoneDeliverable` và `MilestoneReviewForm` (chuyển vào `MilestoneStepper`)
4. Thêm import `MilestoneStepper`
5. Thêm stepper section trước grid (full-width)
6. Xóa `md:col-span-4` milestone sidebar section
7. Đổi grid từ `grid-cols-12` thành `space-y-6` hoặc giữ grid nhưng `md:col-span-12` cho content

**Layout mới:**
```tsx
<div className="max-w-5xl mx-auto space-y-6">
  {/* Header */}
  ...
  
  {/* Revision warning */}
  ...
  
  {/* ← NEW: Horizontal Milestone Stepper (full-width) */}
  <MilestoneStepper milestones={sortedMilestones} projectId={project.projectId} />
  
  {/* Content: full-width now (remove 8/12 + 4/12 grid) */}
  <Card>{/* description */}</Card>
  {activityEntries.length > 0 && <Card>{/* activity */}</Card>}
</div>
```

#### 2. `_actions.ts` — Server Actions cần reuse

**Path:** `src/app/[locale]/(dashboard)/projects/[id]/_actions.ts`

Hai actions đã tồn tại — **ĐỪNG tạo lại:**
- `reviewMilestone(milestoneId, action, comment)` → POST review, trả `{ success, warningRevisionExhausted }`
- `getMilestoneDownloadUrl(milestoneId)` → GET presigned URL

#### 3. `MilestoneReviewForm.tsx` — Reuse, không sửa

**Path:** `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx`

- `'use client'` component
- Props: `{ milestoneId: number }`
- Dùng `useTranslations('milestone.reviewForm')`
- Import từ `'../actions'` — giữ nguyên import path khi dùng trong MilestoneStepper (cần import qua đường dẫn tương đối đúng)

#### 4. `MilestoneDeliverable.tsx` — Reuse, không sửa

**Path:** `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx`

- `'use client'` component
- Props: `{ milestoneId, deliverableUrl, deliverableFileName, hasDeliverableFile, deliveryNote }`
- Dùng `useTranslations('milestone.deliverable')`

### Component mới: MilestoneStepper

**Path:** `src/components/customer/MilestoneStepper.tsx`

**Loại:** Client Component (`'use client'`) — cần `useState` cho accordion.

**Props interface:**
```typescript
interface Milestone {
  milestoneId: number
  name: string
  status: 'ACTIVE' | 'PENDING' | 'DONE'
  position: number
  reviewStatus: 'PENDING_DELIVERY' | 'PENDING_REVIEW' | 'APPROVED' | 'REVISION_REQUESTED'
  deliverableUrl: string | null
  deliverableFileName: string | null
  hasDeliverableFile: boolean
  deliveryNote: string | null
}

interface Props {
  milestones: Milestone[]
  projectId: string
}
```

**Logic auto-expand:**
```typescript
const activePendingReview = milestones.find(
  m => m.status === 'ACTIVE' && m.reviewStatus === 'PENDING_REVIEW'
)
const [openMilestoneId, setOpenMilestoneId] = useState<number | null>(
  activePendingReview?.milestoneId ?? null
)
```

**Render logic (pseudocode):**
```
if milestones.length === 0 → empty state

stepper row (flex, items-end, relative):
  for each milestone:
    flex-col, items-center, flex-1, relative
    [connector left half] [circle] [connector right half]
    below: name text
    below: status label

expand panel (nếu openMilestoneId !== null):
  relative, with triangle arrow pointing to active circle position
  MilestoneDeliverable + MilestoneReviewForm
```

### Visual Implementation Guide

#### Circle sizes & classes

```typescript
function getCircleClasses(status: 'DONE' | 'ACTIVE' | 'PENDING') {
  const base = 'flex items-center justify-center rounded-full font-semibold transition-all'
  if (status === 'DONE') return `${base} w-12 h-12 bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]`
  if (status === 'ACTIVE') return `${base} w-14 h-14 bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)] -mt-1`
  return `${base} w-12 h-12 bg-background border-2 border-border text-muted-foreground`
}
```

#### Connector line (between circles)

Connector phải fill teal nếu step bên trái là DONE, còn lại bg-border. Render:
- Left half of connector: bên trái circle (fill theo status của step trước đó)
- Right half: bên phải circle (fill theo status của step này nếu DONE, else bg-border)

Approach đơn giản: absolute positioned divs hoặc dùng `flex-1 h-0.5` nằm ở vị trí center height.

```tsx
{/* Connector: half-left của step i connects to step i-1 */}
{i > 0 && (
  <div className={`flex-1 h-0.5 ${milestones[i-1].status === 'DONE' ? 'bg-primary' : 'bg-border'}`} />
)}
{/* Circle */}
<div className={getCircleClasses(milestone.status)}>...</div>
{/* Connector: half-right */}
{i < milestones.length - 1 && (
  <div className={`flex-1 h-0.5 ${milestone.status === 'DONE' ? 'bg-primary' : 'bg-border'}`} />
)}
```

**Connector alignment issue:** Circle có size khác nhau (48px vs 56px), nên connector phải center-align với circle. Dùng `items-center` trên parent flex row của mỗi step.

#### Triangle arrow

Dùng CSS border trick (Tailwind arbitrary values):
```tsx
{/* Arrow pointing up, teal-50 fill, aligned to active step */}
<div
  className="absolute top-0 w-0 h-0"
  style={{
    left: `calc(${activeIndex} / ${milestones.length - 1} * 100%)`,
    transform: 'translate(-50%, 0)',
    borderLeft: '10px solid transparent',
    borderRight: '10px solid transparent',
    borderBottom: '10px solid rgb(240 253 250)', // teal-50
  }}
/>
```

#### DONE circle checkmark SVG

```tsx
<svg className="h-5 w-5" viewBox="0 0 20 20" fill="none">
  <path
    d="M4 10l4 4 8-8"
    stroke="currentColor"
    strokeWidth="2.5"
    strokeLinecap="round"
    strokeLinejoin="round"
  />
</svg>
```

### Translation keys cần thêm

**Thêm vào `vi.json` và `en.json`** trong namespace `milestone.stepper`:

```json
// vi.json
"milestone": {
  // ... existing keys ...
  "stepper": {
    "statusDone": "Hoàn thành",
    "statusActive": "Đang thực hiện",
    "statusPending": "Chờ xử lý",
    "emptyState": "Chưa có milestone nào."
  }
}

// en.json
"milestone": {
  // ... existing keys ...
  "stepper": {
    "statusDone": "Completed",
    "statusActive": "In progress",
    "statusPending": "Pending",
    "emptyState": "No milestones yet."
  }
}
```

### Skeleton cho stepper

Thêm vào `ProjectDetailSkeleton` function trong `page.tsx` — thay skeleton hiện tại của milestones sidebar bằng:

```tsx
{/* Milestone stepper skeleton */}
<div className="flex items-center gap-0 w-full" aria-busy="true">
  {[0, 1, 2].map((i) => (
    <div key={i} className="flex flex-1 items-center">
      <Skeleton className="w-12 h-12 rounded-full shrink-0" />
      {i < 2 && <div className="flex-1 h-0.5 bg-border" />}
    </div>
  ))}
</div>
```

### Import paths quan trọng

Khi `MilestoneStepper.tsx` import `MilestoneReviewForm` và `MilestoneDeliverable`:
```typescript
// Từ src/components/customer/MilestoneStepper.tsx → _components/
import { MilestoneReviewForm } from '@/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm'
import { MilestoneDeliverable } from '@/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable'
```

Hoặc có thể move components này vào `src/components/customer/` nếu thấy import path quá dài.

### Locale-aware (AC-10)

`MilestoneStepper.tsx` là Client Component. Nếu cần locale (hiện tại story này không cần redirect), dùng:
```typescript
import { useParams } from 'next/navigation'
const { locale } = useParams<{ locale: string }>()
```

---

## Tasks / Subtasks

### Task 1: Sửa bug `'COMPLETED'` → `'DONE'` trong page.tsx

**File:** `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`

- [ ] Sửa `Milestone` interface line 20: `status: 'ACTIVE' | 'PENDING' | 'COMPLETED'` → `status: 'ACTIVE' | 'PENDING' | 'DONE'`
- [ ] Xóa function `MilestoneIcon` (lines 51–71) — không còn dùng
- [ ] Xóa import `MilestoneDeliverable` và `MilestoneReviewForm` (sẽ move vào MilestoneStepper)
- [ ] Giữ nguyên `reviewMilestone` và `getMilestoneDownloadUrl` actions trong `_actions.ts`

---

### Task 2: Tạo `MilestoneStepper.tsx`

**File mới:** `src/components/customer/MilestoneStepper.tsx`

```tsx
'use client'

import { useState } from 'react'
import { useTranslations } from 'next-intl'
import { MilestoneDeliverable } from '@/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable'
import { MilestoneReviewForm } from '@/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm'

interface Milestone {
  milestoneId: number
  name: string
  status: 'ACTIVE' | 'PENDING' | 'DONE'
  position: number
  reviewStatus: 'PENDING_DELIVERY' | 'PENDING_REVIEW' | 'APPROVED' | 'REVISION_REQUESTED'
  deliverableUrl: string | null
  deliverableFileName: string | null
  hasDeliverableFile: boolean
  deliveryNote: string | null
}

interface Props {
  milestones: Milestone[]
  projectId: string
}

function CheckIcon() {
  return (
    <svg className="h-5 w-5" viewBox="0 0 20 20" fill="none">
      <path d="M4 10l4 4 8-8" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  )
}

function getCircleClasses(status: 'DONE' | 'ACTIVE' | 'PENDING') {
  const base = 'flex items-center justify-center rounded-full font-semibold transition-all duration-200 shrink-0'
  if (status === 'DONE') return `${base} w-12 h-12 bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)]`
  if (status === 'ACTIVE') return `${base} w-14 h-14 bg-primary text-white shadow-[0_4px_16px_rgba(13,148,136,.18)] -mt-1`
  return `${base} w-12 h-12 bg-background border-2 border-border text-muted-foreground`
}

export function MilestoneStepper({ milestones, projectId }: Props) {
  const t = useTranslations('milestone.stepper')

  const sorted = [...milestones].sort((a, b) => a.position - b.position)
  const activePendingReview = sorted.find(
    m => m.status === 'ACTIVE' && m.reviewStatus === 'PENDING_REVIEW'
  )
  const [openMilestoneId, setOpenMilestoneId] = useState<number | null>(
    activePendingReview?.milestoneId ?? null
  )

  if (sorted.length === 0) {
    return (
      <p className="text-sm text-muted-foreground py-4">{t('emptyState')}</p>
    )
  }

  const openMilestone = sorted.find(m => m.milestoneId === openMilestoneId)
  const openIndex = sorted.findIndex(m => m.milestoneId === openMilestoneId)

  const handleCircleClick = (m: Milestone) => {
    if (m.status !== 'ACTIVE') return
    setOpenMilestoneId(prev => prev === m.milestoneId ? null : m.milestoneId)
  }

  const statusLabel = (status: 'DONE' | 'ACTIVE' | 'PENDING') => {
    if (status === 'DONE') return t('statusDone')
    if (status === 'ACTIVE') return t('statusActive')
    return t('statusPending')
  }

  return (
    <div className="space-y-0">
      {/* Stepper row */}
      <div className="flex items-center w-full px-4 py-6">
        {sorted.map((m, i) => (
          <div key={m.milestoneId} className="flex flex-1 flex-col items-center">
            {/* Circle + connectors row */}
            <div className="flex items-center w-full">
              {/* Left connector */}
              {i > 0 && (
                <div className={`flex-1 h-0.5 ${sorted[i - 1].status === 'DONE' ? 'bg-primary' : 'bg-border'}`} />
              )}

              {/* Circle */}
              <button
                type="button"
                onClick={() => handleCircleClick(m)}
                disabled={m.status !== 'ACTIVE'}
                aria-label={m.name}
                className={`${getCircleClasses(m.status)} ${m.status === 'ACTIVE' ? 'cursor-pointer' : 'cursor-default'}`}
              >
                {m.status === 'DONE' ? <CheckIcon /> : <span className="text-sm">{m.position}</span>}
              </button>

              {/* Right connector */}
              {i < sorted.length - 1 && (
                <div className={`flex-1 h-0.5 ${m.status === 'DONE' ? 'bg-primary' : 'bg-border'}`} />
              )}
            </div>

            {/* Label below circle */}
            <div className="mt-2 text-center max-w-[100px]">
              <p className={`text-xs font-medium truncate ${m.status === 'ACTIVE' ? 'text-foreground' : 'text-muted-foreground'}`}>
                {m.name}
              </p>
              <p className="text-[11px] text-muted-foreground mt-0.5">
                {statusLabel(m.status)}
              </p>
            </div>
          </div>
        ))}
      </div>

      {/* Expand panel */}
      {openMilestone && openMilestone.reviewStatus === 'PENDING_REVIEW' && (
        <div className="relative">
          {/* Triangle arrow */}
          {openIndex >= 0 && sorted.length > 1 && (
            <div
              className="absolute -top-2.5 w-0 h-0"
              style={{
                left: `calc(${openIndex} / ${sorted.length - 1} * (100% - 56px) + 28px)`,
                transform: 'translateX(-50%)',
                borderLeft: '10px solid transparent',
                borderRight: '10px solid transparent',
                borderBottom: '10px solid rgb(240 253 250)',
              }}
            />
          )}
          <div className="bg-teal-50 border border-teal-100 rounded-lg p-4 space-y-4">
            <MilestoneDeliverable
              milestoneId={openMilestone.milestoneId}
              deliverableUrl={openMilestone.deliverableUrl}
              deliverableFileName={openMilestone.deliverableFileName}
              hasDeliverableFile={openMilestone.hasDeliverableFile}
              deliveryNote={openMilestone.deliveryNote}
            />
            <MilestoneReviewForm milestoneId={openMilestone.milestoneId} />
          </div>
        </div>
      )}
    </div>
  )
}
```

- [ ] Tạo file `src/components/customer/MilestoneStepper.tsx` với nội dung trên
- [ ] Tạo thư mục `src/components/customer/` nếu chưa có

---

### Task 3: Cập nhật page.tsx — layout + import

**File:** `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`

**Thay đổi:**

1. **Sửa Milestone interface:**
   ```typescript
   status: 'ACTIVE' | 'PENDING' | 'DONE'  // was: 'COMPLETED'
   ```

2. **Thêm import MilestoneStepper, xóa cũ:**
   ```typescript
   // Xóa:
   import { MilestoneDeliverable } from './_components/MilestoneDeliverable'
   import { MilestoneReviewForm } from './_components/MilestoneReviewForm'
   
   // Thêm:
   import { MilestoneStepper } from '@/components/customer/MilestoneStepper'
   ```

3. **Xóa `MilestoneIcon` function** (lines 51–71 trong file gốc).

4. **Cập nhật `ProjectDetailSkeleton`:** Xóa skeleton milestones sidebar 4-col, thêm stepper skeleton:
   ```tsx
   {/* Milestone stepper skeleton */}
   <div className="flex items-center w-full px-4 py-6" aria-busy="true">
     {[0, 1, 2].map((i) => (
       <div key={i} className="flex flex-1 items-center">
         <Skeleton className="w-12 h-12 rounded-full shrink-0" />
         {i < 2 && <div className="flex-1 h-0.5 bg-border mx-2" />}
       </div>
     ))}
   </div>
   ```

5. **Cập nhật `ProjectDetailContent` layout:**
   
   Xóa toàn bộ `md:col-span-4` milestone sidebar section (lines 264–317).
   
   Thêm `<MilestoneStepper>` trước content area:
   ```tsx
   {/* Horizontal milestone stepper */}
   <MilestoneStepper milestones={sortedMilestones} projectId={project.projectId} />
   
   {/* Content (full width now) */}
   <div className="space-y-6">
     <Card>
       {/* description */}
     </Card>
     {activityEntries.length > 0 && (
       <Card>
         {/* activity */}
       </Card>
     )}
   </div>
   ```

- [ ] Sửa Milestone.status type
- [ ] Xóa MilestoneIcon function
- [ ] Sửa imports
- [ ] Cập nhật skeleton
- [ ] Cập nhật layout: xóa 4-col sidebar, thêm MilestoneStepper, chuyển content sang full-width

---

### Task 4: Thêm translation keys

**File:** `src/messages/vi.json`

Thêm vào `milestone` object:
```json
"stepper": {
  "statusDone": "Hoàn thành",
  "statusActive": "Đang thực hiện",
  "statusPending": "Chờ xử lý",
  "emptyState": "Chưa có milestone nào."
}
```

**File:** `src/messages/en.json`

Thêm vào `milestone` object:
```json
"stepper": {
  "statusDone": "Completed",
  "statusActive": "In progress",
  "statusPending": "Pending",
  "emptyState": "No milestones yet."
}
```

- [ ] Thêm vào vi.json
- [ ] Thêm vào en.json

---

## Verification Checklist

- [ ] `npx tsc --noEmit` không lỗi
- [ ] Trang `/[locale]/projects/[id]` load được với horizontal stepper
- [ ] 3 circles hiển thị đúng: DONE (teal + checkmark), ACTIVE (56px teal), PENDING (white border)
- [ ] Connectors: teal cho đoạn DONE, border cho pending
- [ ] ACTIVE milestone có PENDING_REVIEW → panel tự động mở khi load
- [ ] Panel có MilestoneDeliverable content + MilestoneReviewForm buttons
- [ ] Click "Chấp nhận" → approve hoạt động, toast success, page revalidate
- [ ] Click "Yêu cầu chỉnh sửa" → textarea expand, button disabled khi rỗng
- [ ] Empty project (0 milestones) → "Chưa có milestone nào."
- [ ] Skeleton loading đúng format (3 circles + connectors)
- [ ] Kiểm tra locale: `/vi/projects/[id]` và `/en/projects/[id]` đều hoạt động
- [ ] Xóa milestone sidebar cũ (không còn `md:col-span-4`)
- [ ] Bug fix: `m.status === 'COMPLETED'` không còn trong codebase

---

## Files Touched Summary

| Action | File |
|--------|------|
| NEW | `src/components/customer/MilestoneStepper.tsx` |
| MODIFY | `src/app/[locale]/(dashboard)/projects/[id]/page.tsx` |
| MODIFY | `src/messages/vi.json` |
| MODIFY | `src/messages/en.json` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_actions.ts` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx` |
| NO CHANGE | `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx` |
