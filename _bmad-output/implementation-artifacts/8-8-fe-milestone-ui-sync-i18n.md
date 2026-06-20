# Story 8.8: FE — Milestone Card UI Sync + Full i18n

**Status:** ready-for-dev
**Story ID:** 8.8
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As a developer,
I want all Epic 8 UI components to use next-intl translations and for the Admin milestone card to match the quality of the rest of the UI,
So that the app is fully localizable and the Admin experience is visually consistent.

---

## Root Cause Analysis

1. **Duplicate milestone sections**: `page.tsx` render standalone Milestone card (lines 88–107) PLUS `ProjectDetailTabs` cũng render "Manage" tab với milestone grid đơn giản. Standalone card chứa Epic 8 delivery/review UI nhưng nằm ngoài tab context — nhân đôi UI và không nhất quán.

2. **Hardcoded strings trong 6 components**: `DeliverMilestoneForm`, `ReviewStatusBadge`, `MilestoneReviewHistory`, `MilestoneDeliverable`, `MilestoneReviewForm`, và `ProjectDetailTabs` (inline locale check) đều bypass next-intl.

3. **Missing i18n keys**: `en.json` và `vi.json` chưa có keys cho milestone delivery/review vocabulary.

---

## Acceptance Criteria

### AC-1 — Xóa standalone Milestone section khỏi `page.tsx`

**Given** `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`,
**When** updated,
**Then** toàn bộ `<div>` với `h2 "Milestones"` (lines 88–107 hiện tại) bị **xóa** khỏi file.
**And** các imports `DeliverMilestoneForm`, `MilestoneReviewHistory`, `ReviewStatusBadge` bị xóa khỏi file (chúng sẽ được import trong `ProjectDetailTabs.tsx` thay thế).

### AC-2 — `ProjectDetailTabs.tsx` "Manage" tab thay thế simple grid bằng full delivery UI

**Given** "Manage" tab trong `ProjectDetailTabs.tsx`,
**When** updated,
**Then** 2-column grid đơn giản được **thay thế** bằng full milestone list:
- Mỗi milestone row: tên milestone (left) + `ReviewStatusBadge` (right), cùng hàng
- Nếu `reviewStatus === 'PENDING_DELIVERY'` hoặc `'REVISION_REQUESTED'`: render `DeliverMilestoneForm` bên dưới row
- `MilestoneReviewHistory` render bên dưới mỗi milestone nếu `reviews.length > 0`
- Visual style: card rows `border border-gray-100 rounded-lg p-3` matching standalone section đã xóa
**And** tab labels đổi từ inline locale check sang `t('tabs.info')` và `t('tabs.manage')`.

### AC-3 — i18n keys thêm vào `en.json` và `vi.json`

**Given** `src/messages/en.json` và `src/messages/vi.json`,
**When** updated,
**Then** có thêm top-level key `"milestone"` với toàn bộ nested structure (xem Tasks/Subtasks).
**And** `"admin.tabs"` keys được thêm vào cho tab labels.

### AC-4 — `ReviewStatusBadge.tsx` dùng i18n

**Given** `ReviewStatusBadge.tsx`,
**When** updated,
**Then** thêm `'use client'` directive.
**And** import `useTranslations` từ `'next-intl'`.
**And** labels không còn hardcoded trong `labels` object — dùng `t(status)` trong namespace `'milestone.reviewStatus'`.
**And** fallback: `t(status) ?? status` nếu key không tồn tại.

### AC-5 — `MilestoneReviewHistory.tsx` dùng i18n + locale-aware date

**Given** `MilestoneReviewHistory.tsx`,
**When** updated,
**Then** thêm `'use client'` directive.
**And** import `useTranslations`, `useLocale` từ `'next-intl'`.
**And** `'Lịch sử review'` → `t('title')` (namespace `'milestone.reviewHistory'`).
**And** `'Đã phê duyệt'` → `t('approved')`.
**And** `'Yêu cầu chỉnh sửa'` → `t('revisionRequested')`.
**And** date format thay `toLocaleString('vi-VN')` → `new Date(r.reviewedAt).toLocaleString(locale)` (locale-aware).

### AC-6 — `MilestoneDeliverable.tsx` dùng i18n

**Given** `MilestoneDeliverable.tsx`,
**When** updated,
**Then** import `useTranslations` từ `'next-intl'`.
**And** `'Xem deliverable →'` → `t('viewLink')` (namespace `'milestone.deliverable'`).
**And** `'Tải file: {fileName}'` → `t('downloadFile', { fileName: deliverableFileName || 'deliverable.zip' })`.

### AC-7 — `MilestoneReviewForm.tsx` dùng i18n

**Given** `MilestoneReviewForm.tsx`,
**When** updated,
**Then** import `useTranslations` từ `'next-intl'`.
**And** tất cả hardcoded Vietnamese strings trong component được thay bằng `t(key)` với namespace `'milestone.reviewForm'`.

---

## Tasks / Subtasks

### Task 1: Thêm i18n keys vào `en.json`

File: `src/messages/en.json`

Thêm key `"milestone"` ở top-level (sau `"admin"` block), và thêm `"tabs"` vào `"admin"` object:

```json
"admin": {
  ...existing keys...,
  "tabs": {
    "info": "Information",
    "manage": "Management"
  }
},
"milestone": {
  "reviewStatus": {
    "PENDING_DELIVERY": "Awaiting delivery",
    "PENDING_REVIEW": "Awaiting review",
    "APPROVED": "Approved",
    "REVISION_REQUESTED": "Revision requested"
  },
  "deliver": {
    "button": "Deliver milestone",
    "title": "Deliver milestone",
    "linkLabel": "Deliverable link",
    "linkPlaceholder": "https://...",
    "fileLabel": "Upload file",
    "noteLabel": "Notes",
    "notePlaceholder": "Notes for customer (optional)",
    "validation": "Please provide a URL or file.",
    "uploading": "Uploading file...",
    "submitting": "Delivering...",
    "submit": "Deliver",
    "cancel": "Cancel",
    "uploadFailed": "File upload failed. Please try again.",
    "success": "Milestone delivered and customer notified."
  },
  "reviewHistory": {
    "title": "Review history",
    "approved": "Approved",
    "revisionRequested": "Revision requested"
  },
  "deliverable": {
    "viewLink": "View result",
    "downloadFile": "Download: {fileName}"
  },
  "reviewForm": {
    "title": "Review this milestone:",
    "approve": "Approve",
    "requestRevision": "Request revision",
    "commentLabel": "Reason & revision details",
    "commentPlaceholder": "Describe revision details in detail...",
    "commentRequired": "Please provide revision details.",
    "submit": "Confirm",
    "submitting": "Submitting...",
    "approvedSuccess": "Milestone approved.",
    "revisionSuccess": "Revision request sent.",
    "revisionExhausted": "Revision request sent. Note: you have used all revisions — this will be reviewed as a special case.",
    "revisionWarning1": "You have 1 revision remaining. Please describe your requirements fully.",
    "noRevisionsWarning": "You have used all revisions. This request will be reviewed as a special case — please describe in detail."
  }
}
```

### Task 2: Thêm i18n keys vào `vi.json`

File: `src/messages/vi.json`

Thêm key `"milestone"` và `"admin.tabs"` tương tự như en.json với Vietnamese values:

```json
"admin": {
  ...existing keys...,
  "tabs": {
    "info": "Thông tin",
    "manage": "Quản lý"
  }
},
"milestone": {
  "reviewStatus": {
    "PENDING_DELIVERY": "Chờ giao",
    "PENDING_REVIEW": "Chờ review",
    "APPROVED": "Đã duyệt",
    "REVISION_REQUESTED": "Yêu cầu sửa"
  },
  "deliver": {
    "button": "Giao milestone",
    "title": "Giao milestone",
    "linkLabel": "Link kết quả",
    "linkPlaceholder": "https://...",
    "fileLabel": "Tải lên file",
    "noteLabel": "Ghi chú",
    "notePlaceholder": "Ghi chú cho khách hàng (tùy chọn)",
    "validation": "Vui lòng cung cấp URL hoặc file deliverable.",
    "uploading": "Đang tải file...",
    "submitting": "Đang giao...",
    "submit": "Giao ngay",
    "cancel": "Hủy",
    "uploadFailed": "Tải file thất bại. Vui lòng thử lại.",
    "success": "Đã giao milestone và thông báo cho khách hàng."
  },
  "reviewHistory": {
    "title": "Lịch sử review",
    "approved": "Đã phê duyệt",
    "revisionRequested": "Yêu cầu chỉnh sửa"
  },
  "deliverable": {
    "viewLink": "Xem kết quả",
    "downloadFile": "Tải file: {fileName}"
  },
  "reviewForm": {
    "title": "Xem xét milestone này:",
    "approve": "Phê duyệt",
    "requestRevision": "Yêu cầu chỉnh sửa",
    "commentLabel": "Lý do & Nội dung yêu cầu chỉnh sửa",
    "commentPlaceholder": "Mô tả chi tiết yêu cầu chỉnh sửa...",
    "commentRequired": "Vui lòng nhập nội dung yêu cầu chỉnh sửa.",
    "submit": "Xác nhận",
    "submitting": "Đang gửi...",
    "approvedSuccess": "Đã phê duyệt milestone.",
    "revisionSuccess": "Đã gửi yêu cầu chỉnh sửa.",
    "revisionExhausted": "Đã gửi yêu cầu chỉnh sửa. Lưu ý: bạn đã hết số lần chỉnh sửa, lần này sẽ được xem xét đặc biệt.",
    "revisionWarning1": "Bạn còn 1 lần chỉnh sửa. Hãy mô tả đầy đủ yêu cầu.",
    "noRevisionsWarning": "Bạn đã dùng hết số lần chỉnh sửa. Lần này sẽ được xem xét đặc biệt — hãy mô tả thật chi tiết."
  }
}
```

### Task 3: Cập nhật `ReviewStatusBadge.tsx`

File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/ReviewStatusBadge.tsx`

```typescript
'use client';

import { useTranslations } from 'next-intl';

const colors: Record<string, string> = {
  PENDING_DELIVERY: 'bg-gray-100 text-gray-600',
  PENDING_REVIEW: 'bg-blue-100 text-blue-700',
  APPROVED: 'bg-green-100 text-green-700',
  REVISION_REQUESTED: 'bg-orange-100 text-orange-700',
};

export function ReviewStatusBadge({ status }: { status: string }) {
  const t = useTranslations('milestone.reviewStatus');
  return (
    <span className={`px-2 py-0.5 rounded-full text-xs font-medium ${colors[status] ?? 'bg-gray-100 text-gray-500'}`}>
      {t(status as any) ?? status}
    </span>
  );
}
```

**Chú ý**: `useTranslations` yêu cầu `'use client'` — component này không có directive đó hiện tại.

### Task 4: Cập nhật `MilestoneReviewHistory.tsx`

File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/MilestoneReviewHistory.tsx`

```typescript
'use client';

import { useTranslations, useLocale } from 'next-intl';

interface MilestoneReview {
  action: 'APPROVED' | 'REVISION_REQUESTED';
  comment: string | null;
  reviewedAt: string;
}

export function MilestoneReviewHistory({ reviews }: { reviews: MilestoneReview[] }) {
  const t = useTranslations('milestone.reviewHistory');
  const locale = useLocale();

  if (!reviews.length) return null;
  return (
    <div className="mt-3 space-y-2">
      <p className="text-xs font-semibold text-gray-500 uppercase tracking-wide">{t('title')}</p>
      {reviews.map((r, i) => (
        <div key={i} className="flex items-start gap-2 text-sm">
          <span className={`shrink-0 mt-0.5 px-2 py-0.5 rounded-full text-xs font-medium ${
            r.action === 'APPROVED' ? 'bg-green-100 text-green-700' : 'bg-orange-100 text-orange-700'
          }`}>
            {r.action === 'APPROVED' ? t('approved') : t('revisionRequested')}
          </span>
          <div>
            {r.comment && <p className="text-gray-600">{r.comment}</p>}
            <p className="text-xs text-gray-400">{new Date(r.reviewedAt).toLocaleString(locale)}</p>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Task 5: Cập nhật `MilestoneDeliverable.tsx`

File: `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx`

- Thêm: `import { useTranslations } from 'next-intl'`
- Thêm: `const t = useTranslations('milestone.deliverable')` trong component
- `'Xem deliverable →'` → `t('viewLink')`
- `'Tải file: {deliverableFileName || "deliverable.zip"}'` → `t('downloadFile', { fileName: deliverableFileName || 'deliverable.zip' })`

Giữ nguyên: tất cả className, icons (ExternalLink, FileDown, MessageSquare), toàn bộ logic `getMilestoneDownloadUrl`.

### Task 6: Cập nhật `MilestoneReviewForm.tsx`

File: `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx`

- Thêm: `import { useTranslations } from 'next-intl'` (đầu file, sau các imports hiện có)
- Thêm: `const t = useTranslations('milestone.reviewForm')` đầu function body

Mapping strings cần thay (component đã có `'use client'`):
| Hardcoded (vi) | i18n key |
|---|---|
| `'Xem xét milestone này:'` | `t('title')` |
| `'Phê duyệt'` | `t('approve')` |
| `'Yêu cầu chỉnh sửa'` | `t('requestRevision')` |
| `'Lý do & Nội dung yêu cầu chỉnh sửa'` (label) | `t('commentLabel')` |
| `'Mô tả chi tiết yêu cầu chỉnh sửa...'` (placeholder) | `t('commentPlaceholder')` |
| `'Vui lòng nhập nội dung yêu cầu chỉnh sửa.'` (error) | `t('commentRequired')` |
| `'Đang gửi...'` | `t('submitting')` |
| `'Xác nhận'` | `t('submit')` |
| Toast: `'Đã gửi yêu cầu chỉnh sửa. Lưu ý: ...'` | `t('revisionExhausted')` |
| Toast: `'Đã phê duyệt milestone.'` | `t('approvedSuccess')` |
| Toast: `'Đã gửi yêu cầu chỉnh sửa.'` | `t('revisionSuccess')` |

Giữ nguyên: tất cả className, icons (CheckCircle2, AlertCircle, Send), logic `reviewMilestone`, state management.

### Task 7: Cập nhật `ProjectDetailTabs.tsx`

File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/ProjectDetailTabs.tsx`

**Import thêm**:
```typescript
import { DeliverMilestoneForm } from './DeliverMilestoneForm'
import { MilestoneReviewHistory } from './MilestoneReviewHistory'
import { ReviewStatusBadge } from './ReviewStatusBadge'
import { Milestone } from '../page'
```

**Xóa** inline locale check:
```typescript
// XÓA:
const isVi = locale === 'vi'
const tabInfoLabel = isVi ? 'Thông tin' : 'Information'
const tabManageLabel = isVi ? 'Quản lý' : 'Management'

// XÓA useLocale() import nếu không còn dùng ở đâu khác
```

**Thay tab labels**:
```tsx
<TabsTrigger value="info">{t('tabs.info')}</TabsTrigger>
<TabsTrigger value="manage">{t('tabs.manage')}</TabsTrigger>
```

**Thay toàn bộ "Milestones Card" trong "manage" tab** (từ `{/* Milestones Card */}` đến closing `</div>`):
```tsx
{/* Milestones Card */}
{project.milestones && project.milestones.length > 0 && (
  <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-4">
    <h2 className="text-lg font-semibold text-gray-900 pb-3 border-b border-gray-100">
      {t('detail.milestones')}
    </h2>
    <div className="space-y-3">
      {[...project.milestones]
        .sort((a, b) => a.position - b.position)
        .map((m) => (
          <div key={m.milestoneId} className="p-3 border border-gray-100 rounded-lg">
            <div className="flex items-center justify-between">
              <span className="font-medium text-sm">{m.name}</span>
              <ReviewStatusBadge status={m.reviewStatus} />
            </div>
            {(m.reviewStatus === 'PENDING_DELIVERY' || m.reviewStatus === 'REVISION_REQUESTED') && (
              <DeliverMilestoneForm milestoneId={m.milestoneId} milestoneName={m.name} />
            )}
            <MilestoneReviewHistory reviews={m.reviews ?? []} />
          </div>
        ))}
    </div>
  </div>
)}
```

**Giữ nguyên**: `UpdateStatusForm`, Info tab content (description, reference, statusHistory), tất cả className hiện có.

### Task 8: Cập nhật `page.tsx` — Xóa standalone Milestone section

File: `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`

**Xóa** hoàn toàn block (hiện tại lines 88–107):
```tsx
{/* Milestones Card */}
<div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-4">
  <h2 className="text-lg font-semibold text-gray-900 pb-3 border-b border-gray-100">
    Milestones
  </h2>
  <div className="space-y-3">
    {milestones.map((m) => (
      ...
    ))}
  </div>
</div>
```

**Xóa** các imports không còn dùng trong file này:
```typescript
// XÓA (vì đã chuyển sang ProjectDetailTabs):
import { DeliverMilestoneForm } from './_components/DeliverMilestoneForm'
import { MilestoneReviewHistory } from './_components/MilestoneReviewHistory'
import { ReviewStatusBadge } from './_components/ReviewStatusBadge'
```

**Xóa** biến `milestones` nếu chỉ dùng trong block đã xóa:
```typescript
// XÓA nếu chỉ dùng trong milestone card:
const milestones = [...(project.milestones || [])].sort((a, b) => a.position - b.position)
```

**Giữ nguyên**: Back link, Project header card, `<ProjectDetailTabs project={project} />`.

---

## Dev Notes

### next-intl — Namespace Chứa Dấu Chấm

`useTranslations('milestone.reviewStatus')` là hợp lệ với next-intl — namespace `milestone.reviewStatus` tương ứng với object lồng nhau trong JSON.

**KHÔNG** đặt dấu `.` trong key string bên trong `t(key)`:
```typescript
// SAI:
const t = useTranslations('milestone')
t('reviewStatus.APPROVED') // Không hoạt động

// ĐÚNG:
const t = useTranslations('milestone.reviewStatus')
t('APPROVED') // Hoạt động
```

### `useTranslations` Chỉ Dùng Trong `'use client'` Components

Nếu component là Server Component (không có `'use client'`): dùng `getTranslations()` từ `'next-intl/server'`. 

Tất cả 6 components trong story này đều cần `'use client'` vì:
- Gọi hooks (`useState`, `useTransition`)
- Hoặc được render trong client tree (child của client component)

### `ReviewStatusBadge.tsx` — Từ Server Sang Client

Hiện tại `ReviewStatusBadge.tsx` không có `'use client'`. Sau khi thêm `useTranslations`, component trở thành Client Component. Điều này OK vì nó chỉ render một badge dựa trên prop — không có state phức tạp.

### `MilestoneReviewHistory.tsx` — Tương Tự

Hiện tại không có `'use client'`. Thêm `'use client'` khi thêm `useTranslations`.

### Thứ Tự Thực Hiện

Thứ tự khuyến nghị để tránh broken imports trong khi dev:

1. **Task 1 + 2**: Thêm i18n keys vào `en.json` + `vi.json` trước — tránh runtime missing translation errors
2. **Task 3**: `ReviewStatusBadge.tsx` (nhỏ nhất, không có dependency)
3. **Task 4**: `MilestoneReviewHistory.tsx`
4. **Task 5**: `MilestoneDeliverable.tsx`
5. **Task 6**: `MilestoneReviewForm.tsx`
6. **Task 7**: `ProjectDetailTabs.tsx` (import components từ Task 3+4)
7. **Task 8**: `page.tsx` (xóa imports + block sau khi Task 7 hoàn thành)

### `admin.detail.milestones` Key Đã Có

Key `admin.detail.milestones` đã tồn tại trong cả `en.json` ("Project milestones") và `vi.json`. Dùng `t('detail.milestones')` (namespace `'admin'`) trong `ProjectDetailTabs.tsx` — giữ nguyên, không đổi.

### Files Cần Sửa

| File | Loại | Thay đổi |
|------|------|----------|
| `src/messages/en.json` | UPDATE | Thêm `milestone.*` + `admin.tabs.*` |
| `src/messages/vi.json` | UPDATE | Thêm `milestone.*` + `admin.tabs.*` |
| `src/app/[locale]/(admin)/admin/projects/[id]/_components/ReviewStatusBadge.tsx` | UPDATE | `'use client'` + `useTranslations('milestone.reviewStatus')` |
| `src/app/[locale]/(admin)/admin/projects/[id]/_components/MilestoneReviewHistory.tsx` | UPDATE | `'use client'` + `useTranslations('milestone.reviewHistory')` + locale date |
| `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx` | UPDATE | `useTranslations('milestone.deliverable')` |
| `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx` | UPDATE | `useTranslations('milestone.reviewForm')` |
| `src/app/[locale]/(admin)/admin/projects/[id]/_components/ProjectDetailTabs.tsx` | UPDATE | Tab labels i18n + full milestone list in manage tab |
| `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx` | UPDATE | Xóa standalone milestone section + unused imports |

---

## References

- `page.tsx` lines 88–107 — standalone milestone section cần xóa
- `ProjectDetailTabs.tsx` lines 22–24 — inline locale check cần thay bằng i18n
- `ProjectDetailTabs.tsx` lines 113–136 — simple milestone grid cần thay bằng full delivery UI
- `ReviewStatusBadge.tsx` — `labels` object hardcoded cần xóa
- `MilestoneReviewHistory.tsx` line 22 — hardcoded `'vi-VN'` cần đổi sang `locale`
- `en.json` + `vi.json` — thêm `milestone.*` keys (31 keys mỗi file)
- Epic 8.8 spec: `epics.md` lines 1376-1508
