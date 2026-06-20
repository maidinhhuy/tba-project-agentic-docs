# Story 8.4: FE Admin — Deliver Form + Review History

**Status:** ready-for-dev
**Story ID:** 8.4
**Epic:** 8 — Milestone Delivery & Customer Review

---

## Story

As an admin,
I want to deliver a milestone by filling a form (URL + optional file upload) and view the review history left by the customer,
So that I can track delivery status and customer feedback in one place.

---

## Acceptance Criteria

**AC-1** — Nút "Giao milestone" hiện trên Admin Project Detail:
**Given** admin xem trang `/[locale]/admin/projects/[id]`,
**When** có milestone nào đó có `reviewStatus = PENDING_DELIVERY` hoặc `REVISION_REQUESTED`,
**Then** milestone đó có nút "Giao milestone" bên cạnh.

**AC-2** — Form delivery hoạt động:
**Given** admin click "Giao milestone",
**When** form mở (dialog/inline),
**Then** form có:
- Input URL (optional, type=url, label "Link deliverable")
- File upload (optional, max 50MB, label "Tải lên file")
- Textarea delivery note (optional)
- Nút "Giao" + nút "Hủy"
- Validation: ít nhất 1 trong URL hoặc file phải được cung cấp

**AC-3** — Submit delivery thành công:
**Given** admin điền form và click "Giao",
**When** Server Action thành công,
**Then** toast success "Đã giao milestone thành công", trang reload để hiển thị trạng thái mới.

**AC-4** — Submit delivery thất bại:
**Given** server trả lỗi,
**Then** toast error với message từ server, form vẫn mở.

**AC-5** — Review history visible:
**Given** milestone có `reviewStatus = APPROVED | REVISION_REQUESTED`,
**When** admin xem trang,
**Then** hiển thị timeline review history bên dưới milestone đó:
- Mỗi review entry: action badge (APPROVED = xanh, REVISION_REQUESTED = cam), comment (nếu có), thời gian

**AC-6** — Locale-correct routing:
**Given** locale hiện tại là "vi" hoặc "en",
**Then** tất cả paths sử dụng `[locale]` prefix — không hardcode locale.

---

## Tasks / Subtasks

### BE — Admin Project Detail API Update

- [ ] **Task 1: Cập nhật API response cho milestone**
  - Tìm controller admin get project detail (xem route `/api/admin/projects/{id}`)
  - Thêm vào `MilestoneResponse` (hoặc tương đương):
    - `deliverableUrl: String | null`
    - `deliverableFileName: String | null`
    - `deliverableFileKey: String | null`
    - `deliveryNote: String | null`
    - `deliveredAt: String | null`
    - `reviewStatus: String` (PENDING_DELIVERY | PENDING_REVIEW | APPROVED | REVISION_REQUESTED)
    - `reviews: List<MilestoneReviewResponse>` (action, comment, reviewedAt)
  - Cập nhật jOOQ query trong repository để JOIN `milestone_reviews` và map vào response

### FE — Types

- [ ] **Task 2: Cập nhật TypeScript interfaces**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`
  - Cập nhật `Milestone` interface:
  ```typescript
  interface MilestoneReview {
    action: 'APPROVED' | 'REVISION_REQUESTED';
    comment: string | null;
    reviewedAt: string;
  }

  interface Milestone {
    milestoneId: number;
    name: string;
    status: string;
    position: number;
    reviewStatus: 'PENDING_DELIVERY' | 'PENDING_REVIEW' | 'APPROVED' | 'REVISION_REQUESTED';
    deliverableUrl: string | null;
    deliverableFileName: string | null;
    deliveryNote: string | null;
    deliveredAt: string | null;
    reviews: MilestoneReview[];
  }
  ```

### FE — Server Action

- [ ] **Task 3: Tạo deliverMilestone Server Action**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts`
  - Thêm:
  ```typescript
  'use server';

  export async function deliverMilestone(milestoneId: number, formData: FormData) {
    const { token } = await getAuthSession();

    const response = await fetch(
      `${process.env.BACKEND_URL}/api/admin/milestones/${milestoneId}/deliver`,
      {
        method: 'POST',
        headers: { Authorization: `Bearer ${token}` },
        body: formData,
      }
    );

    if (!response.ok) {
      const error = await response.json().catch(() => ({ message: 'Lỗi không xác định' }));
      return { success: false, message: error.message };
    }

    revalidatePath('/[locale]/admin/projects/[id]', 'page');
    return { success: true };
  }
  ```
  - Import `revalidatePath` từ `next/cache`, `getAuthSession` từ auth helper hiện tại

### FE — Components

- [ ] **Task 4: Tạo DeliverMilestoneForm component**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/DeliverMilestoneForm.tsx`
  - `'use client'` component
  - Props: `milestoneId: number`, `milestoneName: string`, `onSuccess: () => void`
  - State: `isOpen: boolean`, `isPending: boolean`
  - Form fields: `deliverableUrl` (text input), `file` (file input, accept="*"), `deliveryNote` (textarea)
  - Client-side validation: ít nhất URL hoặc file
  - Submit: `startTransition(() => deliverMilestone(milestoneId, formData))` → toast
  - Render: trigger button "Giao milestone" + dialog/modal khi `isOpen = true`
  - Dùng pattern card styling hiện tại: `bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm`
  - Toast pattern: theo cách các component khác trong project đang dùng (tìm ví dụ trong `_components/`)

  ```typescript
  'use client';

  import { useTransition, useState } from 'react';
  import { deliverMilestone } from '../_actions';

  interface Props {
    milestoneId: number;
    milestoneName: string;
  }

  export function DeliverMilestoneForm({ milestoneId, milestoneName }: Props) {
    const [isOpen, setIsOpen] = useState(false);
    const [isPending, startTransition] = useTransition();
    const [error, setError] = useState<string | null>(null);

    const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
      e.preventDefault();
      const formData = new FormData(e.currentTarget);
      const url = formData.get('deliverableUrl') as string;
      const file = formData.get('file') as File;
      if (!url && !file?.size) {
        setError('Vui lòng cung cấp URL hoặc file deliverable.');
        return;
      }
      setError(null);
      startTransition(async () => {
        const result = await deliverMilestone(milestoneId, formData);
        if (result.success) {
          setIsOpen(false);
          // show toast success
        } else {
          setError(result.message ?? 'Đã xảy ra lỗi');
        }
      });
    };

    if (!isOpen) {
      return (
        <button
          onClick={() => setIsOpen(true)}
          className="text-sm px-3 py-1 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
        >
          Giao milestone
        </button>
      );
    }

    return (
      <div className="mt-3 p-4 border border-blue-100 rounded-xl bg-blue-50">
        <form onSubmit={handleSubmit} className="space-y-3">
          <div>
            <label className="text-sm font-medium text-gray-700">Link deliverable</label>
            <input
              type="url"
              name="deliverableUrl"
              className="mt-1 w-full border border-gray-300 rounded-lg px-3 py-2 text-sm"
              placeholder="https://..."
            />
          </div>
          <div>
            <label className="text-sm font-medium text-gray-700">Tải lên file</label>
            <input type="file" name="file" className="mt-1 text-sm" />
          </div>
          <div>
            <label className="text-sm font-medium text-gray-700">Ghi chú</label>
            <textarea
              name="deliveryNote"
              rows={3}
              className="mt-1 w-full border border-gray-300 rounded-lg px-3 py-2 text-sm"
            />
          </div>
          {error && <p className="text-sm text-red-600">{error}</p>}
          <div className="flex gap-2 justify-end">
            <button
              type="button"
              onClick={() => setIsOpen(false)}
              className="px-4 py-2 text-sm border rounded-lg"
            >
              Hủy
            </button>
            <button
              type="submit"
              disabled={isPending}
              className="px-4 py-2 text-sm bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
            >
              {isPending ? 'Đang giao...' : 'Giao milestone'}
            </button>
          </div>
        </form>
      </div>
    );
  }
  ```

- [ ] **Task 5: Tạo MilestoneReviewHistory component**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/MilestoneReviewHistory.tsx`
  ```typescript
  interface MilestoneReview {
    action: 'APPROVED' | 'REVISION_REQUESTED';
    comment: string | null;
    reviewedAt: string;
  }

  interface Props {
    reviews: MilestoneReview[];
  }

  export function MilestoneReviewHistory({ reviews }: Props) {
    if (!reviews.length) return null;
    return (
      <div className="mt-3 space-y-2">
        <p className="text-xs font-semibold text-gray-500 uppercase tracking-wide">Lịch sử review</p>
        {reviews.map((r, i) => (
          <div key={i} className="flex items-start gap-2 text-sm">
            <span className={`shrink-0 mt-0.5 px-2 py-0.5 rounded-full text-xs font-medium ${
              r.action === 'APPROVED'
                ? 'bg-green-100 text-green-700'
                : 'bg-orange-100 text-orange-700'
            }`}>
              {r.action === 'APPROVED' ? 'Đã phê duyệt' : 'Yêu cầu chỉnh sửa'}
            </span>
            <div>
              {r.comment && <p className="text-gray-600">{r.comment}</p>}
              <p className="text-xs text-gray-400">{new Date(r.reviewedAt).toLocaleString('vi-VN')}</p>
            </div>
          </div>
        ))}
      </div>
    );
  }
  ```

### FE — Admin Project Detail Page Update

- [ ] **Task 6: Tích hợp components vào trang Admin Project Detail**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`
  - Tại phần render milestones, thêm:
    - `DeliverMilestoneForm` khi `milestone.reviewStatus === 'PENDING_DELIVERY' || milestone.reviewStatus === 'REVISION_REQUESTED'`
    - `MilestoneReviewHistory` khi `milestone.reviews.length > 0`
    - Badge hiển thị `reviewStatus` cho mỗi milestone (PENDING_DELIVERY=xám, PENDING_REVIEW=xanh dương, APPROVED=xanh lá, REVISION_REQUESTED=cam)
  - Ví dụ addition trong milestone list:
  ```tsx
  {milestones.map((m) => (
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
  ```

- [ ] **Task 7: Tạo ReviewStatusBadge component (inline hoặc file riêng)**
  ```typescript
  const labels = {
    PENDING_DELIVERY: 'Chờ giao',
    PENDING_REVIEW: 'Chờ review',
    APPROVED: 'Đã duyệt',
    REVISION_REQUESTED: 'Yêu cầu sửa',
  };
  const colors = {
    PENDING_DELIVERY: 'bg-gray-100 text-gray-600',
    PENDING_REVIEW: 'bg-blue-100 text-blue-700',
    APPROVED: 'bg-green-100 text-green-700',
    REVISION_REQUESTED: 'bg-orange-100 text-orange-700',
  };
  ```

---

## Dev Notes

### File Pattern Tham Khảo

Admin project detail:
- Page: `src/app/[locale]/(admin)/admin/projects/[id]/page.tsx`
- Actions: `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts`
- Components: `src/app/[locale]/(admin)/admin/projects/[id]/_components/`

Cần đọc `_actions.ts` hiện tại để biết cách `getAuthSession()` và `apiFetch` đang được gọi. Giữ đúng pattern để tránh breaking changes.

### `AdminProjectDetail` Interface Hiện Tại

```typescript
interface AdminProjectDetail {
  projectId: string;
  name: string;
  customerEmail: string;
  status: string;
  allowedTransitions: string[];
  revisionCount: number;
  milestones: Milestone[];
  statusHistory: StatusHistory[];
  version: number;
}
```

Cần extend `Milestone` type để add delivery/review fields — không sửa các field hiện có.

### Server Action — multipart/form-data

Server Action trực tiếp nhận `FormData` → forward nguyên đến Spring Boot `/api/admin/milestones/{id}/deliver` với `Content-Type: multipart/form-data`.

Spring Boot nhận `@RequestParam MultipartFile file` và `@RequestParam String deliverableUrl`.

**Không** parse FormData trong Server Action — chỉ pass thẳng vào `fetch()` body.

### No Dialog Library

Project chưa có dialog library (Radix, shadcn). Dùng inline expand (không dùng `<dialog>`) — đơn giản hơn và không cần thêm dep.

### Toast Pattern

Kiểm tra cách các form khác (`UpdateStatusForm` hay `_components/`) hiện đang show toast. Có thể dùng `react-hot-toast` hoặc custom. Không import library mới nếu project chưa có.
