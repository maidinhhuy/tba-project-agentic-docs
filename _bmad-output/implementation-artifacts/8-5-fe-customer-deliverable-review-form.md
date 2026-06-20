# Story 8.5: FE Customer — View Deliverable + Review Form

**Status:** ready-for-dev
**Story ID:** 8.5
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As a customer,
I want to view deliverables for each milestone and submit an approval or revision request with a comment,
So that I can give feedback and control the quality of work delivered to me.

---

## Acceptance Criteria

**AC-1** — Milestone hiển thị deliverable khi đã được giao:
**Given** customer xem `/[locale]/projects/[id]`,
**When** milestone có `reviewStatus = PENDING_REVIEW`,
**Then** hiển thị:
- Link "Xem deliverable" (nếu có `deliverableUrl`)
- Nút "Tải file" dẫn đến presigned URL (nếu có `deliverableFileKey`)
- `deliveryNote` nếu có

**AC-2** — Form review hiển thị đúng:
**Given** milestone có `reviewStatus = PENDING_REVIEW`,
**Then** hiển thị form:
- Radio/button chọn: "Phê duyệt" hoặc "Yêu cầu chỉnh sửa"
- Textarea comment (bắt buộc khi chọn "Yêu cầu chỉnh sửa", ẩn khi "Phê duyệt")
- Nút submit

**AC-3** — Submit phê duyệt thành công:
**Given** customer chọn "Phê duyệt" và click submit,
**When** Server Action thành công,
**Then** toast success "Đã phê duyệt milestone", trang reload.

**AC-4** — Submit revision thành công (còn lượt):
**Given** customer chọn "Yêu cầu chỉnh sửa", điền comment, click submit,
**When** Server Action thành công và `warningRevisionExhausted = false`,
**Then** toast success "Đã gửi yêu cầu chỉnh sửa", trang reload.

**AC-5** — Submit revision hết lượt (warning):
**Given** customer submit revision khi `revisionCount = 0`,
**When** Server Action thành công với `warningRevisionExhausted = true`,
**Then** toast warning "Đã gửi yêu cầu chỉnh sửa. Lưu ý: bạn đã hết số lần chỉnh sửa, lần này sẽ được xem xét đặc biệt."

**AC-6** — Comment required validation:
**Given** customer chọn "Yêu cầu chỉnh sửa" và để trống comment,
**Then** hiển thị lỗi inline "Vui lòng nhập nội dung yêu cầu chỉnh sửa."

**AC-7** — Milestone đã APPROVED hiển thị badge, không có form:
**Given** milestone có `reviewStatus = APPROVED`,
**Then** hiển thị badge "Đã phê duyệt" (xanh lá), không có form review.

**AC-8** — revisionCount hiển thị cảnh báo khi = 0:
**Given** `project.revisionCount = 0`,
**Then** hiển thị banner/badge cảnh báo nhỏ: "Bạn đã sử dụng hết lượt chỉnh sửa."

**AC-9** — Download presigned URL:
**Given** customer click "Tải file",
**When** Server Action gọi `GET /api/customer/milestones/{id}/download-url`,
**Then** mở URL presigned trong tab mới.

**AC-10** — Locale-correct routing:
**Given** locale bất kỳ,
**Then** tất cả paths và `useParams()` / `getLocale()` đúng — không hardcode locale.

---

## Tasks / Subtasks

### BE — Customer Project Detail API Update

- [ ] **Task 1: Cập nhật GET /api/customer/projects/{id} response**
  - Tìm controller customer get project detail
  - Thêm vào `MilestoneResponse`:
    - `deliverableUrl: String | null`
    - `deliverableFileName: String | null`
    - `deliverableFileKey: String | null` (dùng để check xem có file không — không trả về raw S3 key)
    - `deliveryNote: String | null`
    - `reviewStatus: String` (PENDING_DELIVERY | PENDING_REVIEW | APPROVED | REVISION_REQUESTED)
  - **Không** trả về `deliverableFileKey` về FE trực tiếp — chỉ trả về `hasFile: boolean` hoặc dùng endpoint presigned riêng
  - Thêm `revisionCount` vào project response (nếu chưa có)

### FE — Types

- [ ] **Task 2: Cập nhật TypeScript interfaces**
  - File: `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`
  - Cập nhật `Milestone` interface:
  ```typescript
  interface Milestone {
    milestoneId: number;
    name: string;
    status: 'ACTIVE' | 'PENDING' | 'COMPLETED';
    position: number;
    reviewStatus: 'PENDING_DELIVERY' | 'PENDING_REVIEW' | 'APPROVED' | 'REVISION_REQUESTED';
    deliverableUrl: string | null;
    deliverableFileName: string | null;
    hasDeliverableFile: boolean;
    deliveryNote: string | null;
  }

  // Đảm bảo ProjectDetail có revisionCount
  interface ProjectDetail {
    // ... existing fields
    revisionCount: number;
  }
  ```

### FE — Server Actions

- [ ] **Task 3: Tạo reviewMilestone Server Action**
  - File: `src/app/[locale]/(dashboard)/projects/[id]/_actions.ts` (tạo mới nếu chưa có)
  ```typescript
  'use server';

  import { revalidatePath } from 'next/cache';
  import { getAuthSession } from '@/lib/auth'; // adjust import path

  export async function reviewMilestone(
    milestoneId: number,
    action: 'APPROVED' | 'REVISION_REQUESTED',
    comment: string | null
  ): Promise<{ success: boolean; message?: string; warningRevisionExhausted?: boolean }> {
    const { token } = await getAuthSession();

    const response = await fetch(
      `${process.env.API_BASE_URL}/api/customer/milestones/${milestoneId}/review`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ action, comment }),
      }
    );

    if (!response.ok) {
      const error = await response.json().catch(() => ({ message: 'Lỗi không xác định' }));
      return { success: false, message: error.message };
    }

    const data = await response.json();
    revalidatePath('/[locale]/projects/[id]', 'page');
    return { success: true, warningRevisionExhausted: data.warningRevisionExhausted };
  }

  export async function getMilestoneDownloadUrl(milestoneId: number): Promise<string | null> {
    const { token } = await getAuthSession();

    const response = await fetch(
      `${process.env.API_BASE_URL}/api/customer/milestones/${milestoneId}/download-url`,
      { headers: { Authorization: `Bearer ${token}` } }
    );

    if (!response.ok) return null;
    const data = await response.json();
    return data.url;
  }
  ```

### FE — Components

- [ ] **Task 4: Tạo MilestoneDeliverable component**
  - File: `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneDeliverable.tsx`
  ```typescript
  'use client';

  import { getMilestoneDownloadUrl } from '../_actions';

  interface Props {
    milestoneId: number;
    deliverableUrl: string | null;
    deliverableFileName: string | null;
    hasDeliverableFile: boolean;
    deliveryNote: string | null;
  }

  export function MilestoneDeliverable({
    milestoneId, deliverableUrl, deliverableFileName, hasDeliverableFile, deliveryNote
  }: Props) {
    const handleDownload = async () => {
      const url = await getMilestoneDownloadUrl(milestoneId);
      if (url) window.open(url, '_blank');
    };

    return (
      <div className="mt-2 space-y-1">
        {deliverableUrl && (
          <a
            href={deliverableUrl}
            target="_blank"
            rel="noopener noreferrer"
            className="inline-flex items-center gap-1 text-sm text-blue-600 hover:underline"
          >
            Xem deliverable →
          </a>
        )}
        {hasDeliverableFile && (
          <button
            onClick={handleDownload}
            className="block text-sm text-blue-600 hover:underline"
          >
            Tải file: {deliverableFileName}
          </button>
        )}
        {deliveryNote && (
          <p className="text-sm text-gray-500 italic">"{deliveryNote}"</p>
        )}
      </div>
    );
  }
  ```

- [ ] **Task 5: Tạo MilestoneReviewForm component**
  - File: `src/app/[locale]/(dashboard)/projects/[id]/_components/MilestoneReviewForm.tsx`
  ```typescript
  'use client';

  import { useState, useTransition } from 'react';
  import { reviewMilestone } from '../_actions';

  interface Props {
    milestoneId: number;
  }

  export function MilestoneReviewForm({ milestoneId }: Props) {
    const [action, setAction] = useState<'APPROVED' | 'REVISION_REQUESTED' | null>(null);
    const [comment, setComment] = useState('');
    const [error, setError] = useState<string | null>(null);
    const [isPending, startTransition] = useTransition();

    const handleSubmit = () => {
      if (!action) return;
      if (action === 'REVISION_REQUESTED' && !comment.trim()) {
        setError('Vui lòng nhập nội dung yêu cầu chỉnh sửa.');
        return;
      }
      setError(null);
      startTransition(async () => {
        const result = await reviewMilestone(
          milestoneId,
          action,
          action === 'REVISION_REQUESTED' ? comment : null
        );
        if (!result.success) {
          setError(result.message ?? 'Đã xảy ra lỗi');
          return;
        }
        if (result.warningRevisionExhausted) {
          // show warning toast
          alert('Đã gửi yêu cầu chỉnh sửa. Lưu ý: bạn đã hết số lần chỉnh sửa, lần này sẽ được xem xét đặc biệt.');
        } else if (action === 'APPROVED') {
          // show success toast
          alert('Đã phê duyệt milestone.');
        } else {
          alert('Đã gửi yêu cầu chỉnh sửa.');
        }
      });
    };

    return (
      <div className="mt-3 p-4 border border-gray-100 rounded-xl bg-gray-50">
        <p className="text-sm font-medium text-gray-700 mb-3">Xem xét milestone này:</p>
        <div className="flex gap-3 mb-3">
          <button
            onClick={() => setAction('APPROVED')}
            className={`px-4 py-2 rounded-lg text-sm font-medium border transition-colors ${
              action === 'APPROVED'
                ? 'bg-green-600 text-white border-green-600'
                : 'border-gray-300 text-gray-600 hover:border-green-400'
            }`}
          >
            Phê duyệt
          </button>
          <button
            onClick={() => setAction('REVISION_REQUESTED')}
            className={`px-4 py-2 rounded-lg text-sm font-medium border transition-colors ${
              action === 'REVISION_REQUESTED'
                ? 'bg-orange-500 text-white border-orange-500'
                : 'border-gray-300 text-gray-600 hover:border-orange-400'
            }`}
          >
            Yêu cầu chỉnh sửa
          </button>
        </div>

        {action === 'REVISION_REQUESTED' && (
          <textarea
            value={comment}
            onChange={(e) => setComment(e.target.value)}
            rows={4}
            placeholder="Mô tả chi tiết yêu cầu chỉnh sửa..."
            className="w-full border border-gray-300 rounded-lg px-3 py-2 text-sm resize-none focus:outline-none focus:ring-2 focus:ring-orange-300"
          />
        )}

        {error && <p className="text-sm text-red-600 mt-2">{error}</p>}

        {action && (
          <button
            onClick={handleSubmit}
            disabled={isPending}
            className="mt-3 w-full px-4 py-2 bg-gray-900 text-white rounded-lg text-sm font-medium hover:bg-gray-800 disabled:opacity-50"
          >
            {isPending ? 'Đang gửi...' : 'Xác nhận'}
          </button>
        )}
      </div>
    );
  }
  ```

### FE — Customer Project Detail Page Update

- [ ] **Task 6: Tích hợp vào Customer Project Detail page**
  - File: `src/app/[locale]/(dashboard)/projects/[id]/page.tsx`
  - Thêm `revisionCount` warning banner nếu `project.revisionCount === 0`:
  ```tsx
  {project.revisionCount === 0 && (
    <div className="mb-4 p-3 bg-orange-50 border border-orange-200 rounded-xl text-sm text-orange-700">
      Bạn đã sử dụng hết lượt chỉnh sửa.
    </div>
  )}
  ```
  - Trong danh sách milestones, thêm:
  ```tsx
  {m.reviewStatus === 'PENDING_REVIEW' && (
    <>
      <MilestoneDeliverable
        milestoneId={m.milestoneId}
        deliverableUrl={m.deliverableUrl}
        deliverableFileName={m.deliverableFileName}
        hasDeliverableFile={m.hasDeliverableFile}
        deliveryNote={m.deliveryNote}
      />
      <MilestoneReviewForm milestoneId={m.milestoneId} />
    </>
  )}
  {m.reviewStatus === 'APPROVED' && (
    <span className="inline-block mt-1 px-2 py-0.5 rounded-full text-xs bg-green-100 text-green-700">
      Đã phê duyệt
    </span>
  )}
  {m.reviewStatus === 'REVISION_REQUESTED' && (
    <span className="inline-block mt-1 px-2 py-0.5 rounded-full text-xs bg-orange-100 text-orange-700">
      Đang chờ chỉnh sửa
    </span>
  )}
  ```
  - Import `MilestoneDeliverable` và `MilestoneReviewForm` từ `_components/`
  - Tạo folder `_components/` nếu chưa có trong dashboard route

- [ ] **Task 7: Kiểm tra visual trên UI**
  - Start dev server: `npm run dev`
  - Login với customer account
  - Kiểm tra project detail: milestone states, deliverable links, form review
  - Test với milestone ở mỗi `reviewStatus` state
  - Kiểm tra `revisionCount = 0` warning banner

---

## Dev Notes

### Customer Project Detail — Interface Hiện Tại

```typescript
interface ProjectDetail {
  projectId: string;
  name: string;
  productType: string;
  description: string;
  reference: string;
  status: string;
  revisionCount: number;
  milestones: Milestone[];
  statusHistory: StatusHistory[];
  createdAt: string;
  updatedAt: string;
  version: number;
}

interface Milestone {
  milestoneId: number;
  name: string;
  status: 'ACTIVE' | 'PENDING' | 'COMPLETED';
  position: number;
}
```

Extend `Milestone` — không xóa field cũ.

### File Path: `_components/`

Dashboard route hiện chưa có `_components/` folder. Cần tạo:
```
src/app/[locale]/(dashboard)/projects/[id]/
├── page.tsx        ← update
├── _actions.ts     ← tạo mới
└── _components/
    ├── MilestoneDeliverable.tsx   ← tạo mới
    └── MilestoneReviewForm.tsx    ← tạo mới
```

### Toast vs Alert

Code trên dùng `alert()` làm placeholder. Kiểm tra xem project có toast library nào (react-hot-toast, sonner, v.v.) trong `package.json`. Nếu có, replace `alert()` bằng toast call đúng pattern. Nếu không có — giữ `alert()` hoặc dùng pattern inline message.

### Download File — Server Action vs Client Fetch

`getMilestoneDownloadUrl` là Server Action gọi BE để lấy presigned URL. Sau đó dùng `window.open(url, '_blank')` để download từ S3 trực tiếp. Cách này tránh routing S3 traffic qua server.

### `hasDeliverableFile: boolean` vs fileKey

FE không cần biết S3 key. BE chỉ trả về `hasDeliverableFile: true/false`. Khi cần download, FE gọi Server Action → BE generate presigned URL → FE nhận URL → open in new tab.

### Locale

- Dùng `useParams()` nếu cần locale trong client component
- Dùng `getLocale()` trong server components
- Không hardcode `/vi/` hay `/en/` — luôn dùng `locale` param
- `revalidatePath('/[locale]/projects/[id]', 'page')` — Next.js sẽ invalidate tất cả locale variants

### FE-Only Test — Dev Server

Vì không có Playwright/E2E, kiểm tra bằng dev server. Cần BE đang chạy (hoặc mock BE) để test flow đầy đủ. Nếu BE chưa ready (story 8.3 chưa done), kiểm tra UI rendering với mock data trực tiếp trong component.
