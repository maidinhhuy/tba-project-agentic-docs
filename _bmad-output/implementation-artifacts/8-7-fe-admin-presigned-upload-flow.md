# Story 8.7: FE Admin — Presigned Upload Flow Refactor

**Status:** ready-for-dev
**Story ID:** 8.7
**Epic:** 8 — Milestone Delivery & Customer Review
**Depends on:** Story 8.6 (BE presigned endpoint deployed) + Story 8.8 (i18n keys phải tồn tại)

---

## Story

As an admin,
I want to upload deliverable files directly from my browser to S3 using a presigned URL,
So that the upload is reliable regardless of Next.js server action multipart limitations.

---

## Acceptance Criteria

**AC-1** — Server action `requestUploadUrl` mới:
**Given** `_actions.ts` cho admin project detail,
**When** updated,
**Then** export thêm server action:
```typescript
export async function requestUploadUrl(
  milestoneId: number,
  fileName: string,
  contentType: string
): Promise<{ uploadUrl: string; s3Key: string } | { error: string }>
```
Gọi `POST /api/admin/milestones/{milestoneId}/request-upload-url` với JSON body `{ fileName, contentType }`, forward cookies cho auth.

**AC-2** — Server action `deliverMilestone` refactored:
**Given** `deliverMilestone` hiện tại trong `_actions.ts`,
**When** updated,
**Then** signature đổi thành:
```typescript
export async function deliverMilestone(
  milestoneId: number,
  payload: { deliverableUrl: string | null; deliveryNote: string | null; s3Key: string | null; fileName: string | null }
): Promise<{ success: boolean; message?: string }>
```
Gửi `POST /api/admin/milestones/{milestoneId}/deliver` với `Content-Type: application/json`.
**And** không còn multipart, không còn Buffer construction, không còn URLSearchParams fallback.
**And** tất cả `console.log` debug statements bị xóa.

**AC-3** — Upload flow khi có file:
**Given** `DeliverMilestoneForm.tsx` với file được chọn,
**When** Admin click submit,
**Then** flow thực thi theo thứ tự:
1. State hiển thị `t('milestone.deliver.uploading')` (e.g. "Đang tải file...")
2. Gọi `requestUploadUrl(milestoneId, file.name, file.type || 'application/octet-stream')` → `{ uploadUrl, s3Key }`
3. Thực thi `fetch(uploadUrl, { method: 'PUT', body: file, headers: { 'Content-Type': file.type || 'application/octet-stream' } })` — browser PUT trực tiếp lên S3 (không có Authorization header — presigned URL tự chứa auth)
4. Nếu S3 PUT fail → toast error `t('milestone.deliver.uploadFailed')`, form vẫn mở
5. State hiển thị `t('milestone.deliver.submitting')` 
6. Gọi `deliverMilestone(milestoneId, { deliverableUrl, deliveryNote, s3Key, fileName: file.name })`

**AC-4** — Submit flow không có file (URL only):
**Given** Form chỉ có URL, không có file,
**When** Admin click submit,
**Then** bỏ qua upload step, gọi ngay `deliverMilestone(milestoneId, { deliverableUrl, deliveryNote, s3Key: null, fileName: null })`.

**AC-5** — File info display:
**Given** Admin chọn file trong input,
**When** file được chọn,
**Then** hiển thị tên file + formatted size (e.g. `"design-v2.fig — 2.4 MB"`) bên dưới file input.
**And** có nút "×" để deselect file (reset về trạng thái chưa chọn file).

**AC-6** — i18n:
**Given** `DeliverMilestoneForm.tsx`,
**When** updated,
**Then** tất cả user-facing strings dùng `useTranslations('milestone.deliver')` — không còn hardcoded text.

---

## Tasks / Subtasks

### FE — Server Actions

- [ ] **Task 1: Thêm `requestUploadUrl` server action**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts`
  ```typescript
  export async function requestUploadUrl(
    milestoneId: number,
    fileName: string,
    contentType: string
  ): Promise<{ uploadUrl: string; s3Key: string } | { error: string }> {
    const cookieStore = await cookies()
    const backendUrl = process.env.API_BASE_URL || 'http://localhost:8080'
    const response = await fetch(
      `${backendUrl}/api/admin/milestones/${milestoneId}/request-upload-url`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Cookie: cookieStore.toString(),
        },
        body: JSON.stringify({ fileName, contentType }),
      }
    )
    if (!response.ok) {
      return { error: 'Failed to get upload URL' }
    }
    return response.json()
  }
  ```

- [ ] **Task 2: Refactor `deliverMilestone` server action**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts`
  - **Xóa hoàn toàn** implementation hiện tại (multipart buffer building, URLSearchParams fallback)
  - **Xóa** tất cả `console.log` statements
  - **Viết lại**:
  ```typescript
  export async function deliverMilestone(
    milestoneId: number,
    payload: {
      deliverableUrl: string | null
      deliveryNote: string | null
      s3Key: string | null
      fileName: string | null
    }
  ): Promise<{ success: boolean; message?: string }> {
    const cookieStore = await cookies()
    const backendUrl = process.env.API_BASE_URL || 'http://localhost:8080'

    const response = await fetch(
      `${backendUrl}/api/admin/milestones/${milestoneId}/deliver`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Cookie: cookieStore.toString(),
        },
        body: JSON.stringify({
          deliverableUrl: payload.deliverableUrl || null,
          deliverableS3Key: payload.s3Key || null,
          deliverableFileName: payload.fileName || null,
          deliveryNote: payload.deliveryNote || null,
        }),
      }
    )

    if (!response.ok) {
      const errorBody = await response.text()
      let message = 'Lỗi không xác định'
      try { message = JSON.parse(errorBody).message ?? message } catch {}
      return { success: false, message }
    }

    revalidatePath('/[locale]/admin/projects/[id]', 'page')
    return { success: true }
  }
  ```

### FE — DeliverMilestoneForm Component

- [ ] **Task 3: Refactor `DeliverMilestoneForm.tsx`**
  - File: `src/app/[locale]/(admin)/admin/projects/[id]/_components/DeliverMilestoneForm.tsx`
  - **Import** thêm:
    ```typescript
    import { useTranslations } from 'next-intl'
    import { requestUploadUrl } from '../_actions'
    ```
  - **State mới**: `selectedFile: File | null` (để track file đã chọn + hiển thị info)
  - **State mới**: `uploadStep: 'idle' | 'uploading' | 'delivering'` (thay thế `isPending` boolean đơn giản)
  - **`handleSubmit`** refactored:
    ```typescript
    const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
      e.preventDefault()
      const form = e.currentTarget
      const deliverableUrl = (form.elements.namedItem('deliverableUrl') as HTMLInputElement).value
      const deliveryNote = (form.elements.namedItem('deliveryNote') as HTMLTextAreaElement).value
      
      if (!deliverableUrl && !selectedFile) {
        setError(t('validation'))
        return
      }
      setError(null)

      let s3Key: string | null = null
      let fileName: string | null = null

      if (selectedFile) {
        setUploadStep('uploading')
        const contentType = selectedFile.type || 'application/octet-stream'
        const urlResult = await requestUploadUrl(milestoneId, selectedFile.name, contentType)
        if ('error' in urlResult) {
          setError(t('uploadFailed'))
          toast.error(t('uploadFailed'))
          setUploadStep('idle')
          return
        }
        const s3Response = await fetch(urlResult.uploadUrl, {
          method: 'PUT',
          body: selectedFile,
          headers: { 'Content-Type': contentType },
        })
        if (!s3Response.ok) {
          setError(t('uploadFailed'))
          toast.error(t('uploadFailed'))
          setUploadStep('idle')
          return
        }
        s3Key = urlResult.s3Key
        fileName = selectedFile.name
      }

      setUploadStep('delivering')
      startTransition(async () => {
        const result = await deliverMilestone(milestoneId, {
          deliverableUrl: deliverableUrl || null,
          deliveryNote: deliveryNote || null,
          s3Key,
          fileName,
        })
        if (result.success) {
          toast.success(t('success'))
          setIsOpen(false)
          setSelectedFile(null)
        } else {
          setError(result.message ?? 'Đã xảy ra lỗi')
          toast.error(result.message ?? 'Đã xảy ra lỗi')
        }
        setUploadStep('idle')
      })
    }
    ```
  - **Submit button text**: `uploadStep === 'uploading' ? t('uploading') : uploadStep === 'delivering' ? t('submitting') : t('submit')`
  - **`disabled`** condition: `uploadStep !== 'idle'`
  - **File input section** — thêm display khi file được chọn:
    ```tsx
    <div>
      <label className="text-sm font-medium text-gray-700 block">{t('fileLabel')}</label>
      <input
        type="file"
        name="file"
        className="mt-1 text-sm block w-full file:mr-4 file:py-1 file:px-3 file:rounded-md file:border-0 file:text-xs file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
        onChange={(e) => setSelectedFile(e.target.files?.[0] ?? null)}
      />
      {selectedFile && (
        <div className="mt-1 flex items-center gap-2 text-xs text-gray-600">
          <span>{selectedFile.name} — {formatFileSize(selectedFile.size)}</span>
          <button
            type="button"
            onClick={() => setSelectedFile(null)}
            className="text-gray-400 hover:text-red-500 font-bold"
          >
            ×
          </button>
        </div>
      )}
    </div>
    ```
  - **Helper `formatFileSize`** (inline trong file):
    ```typescript
    function formatFileSize(bytes: number): string {
      if (bytes < 1024) return `${bytes} B`
      if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
      return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
    }
    ```
  - **i18n**: thêm `const t = useTranslations('milestone.deliver')` và thay tất cả hardcoded strings:
    - `'Vui lòng cung cấp URL hoặc file deliverable.'` → `t('validation')`
    - `'Đã giao milestone thành công'` → `t('success')`
    - `'Giao milestone'` (button) → `t('button')`
    - `'Link deliverable'` → `t('linkLabel')`
    - `'https://...'` placeholder → `t('linkPlaceholder')`
    - `'Tải lên file'` → `t('fileLabel')`
    - `'Ghi chú'` → `t('noteLabel')`
    - `'Hủy'` → `t('cancel')`
    - `'Đang giao...'` → `t('submitting')`
    - `'Giao milestone'` (submit) → `t('submit')`

---

## Dev Notes

### S3 PUT từ Browser — Không Có Auth Header

Presigned URL đã chứa AWS signature trong query params. **KHÔNG** thêm `Authorization` header khi fetch presigned URL — sẽ gây lỗi "SignatureDoesNotMatch".

```typescript
// ĐÚNG:
fetch(uploadUrl, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } })

// SAI — không add Authorization:
fetch(uploadUrl, { method: 'PUT', body: file, headers: { 'Authorization': '...', 'Content-Type': file.type } })
```

### `useTransition` + async flow

`startTransition` wrapper không hỗ trợ `async/await` tốt cho multi-step flow (upload → deliver). Pattern khuyến nghị:

1. Handle S3 upload **bên ngoài** `startTransition` (trước khi transition bắt đầu)
2. Chỉ wrap `deliverMilestone()` server action trong `startTransition`

Xem `handleSubmit` trong Task 3 — S3 upload xảy ra trước `startTransition`.

### `file.type` có thể rỗng

Một số browser/OS không set `file.type` cho format files lạ (`.fig`, `.sketch`, `.psd`). Luôn fallback:
```typescript
const contentType = selectedFile.type || 'application/octet-stream'
```

### Locale-aware server action path

`revalidatePath('/[locale]/admin/projects/[id]', 'page')` — giữ nguyên format này để revalidate đúng cho tất cả locale variants.

### `_actions.ts` — Import Đầy Đủ

File hiện có: `apiFetch`, `cookies`, `revalidatePath` — check xem có dùng `apiFetch` cho `requestUploadUrl` không. **Nếu** `apiFetch` không handle POST với JSON body tốt (chỉ dùng cookies forwarding), dùng thẳng `fetch()` như example trong Task 1.

Xem file hiện tại: `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts`:
- `updateProjectStatusAction` dùng `apiFetch` (wrap qua `@/lib/api`)
- `deliverMilestone` cũ dùng raw `fetch` trực tiếp
- Tiếp tục pattern raw `fetch` cho cả `requestUploadUrl` và `deliverMilestone` mới

### Files Cần Sửa

| File | Loại | Thay đổi |
|------|------|----------|
| `src/app/[locale]/(admin)/admin/projects/[id]/_actions.ts` | UPDATE | Thêm `requestUploadUrl`, refactor `deliverMilestone` sang JSON, xóa debug logs |
| `src/app/[locale]/(admin)/admin/projects/[id]/_components/DeliverMilestoneForm.tsx` | UPDATE | Upload flow, file display, i18n |

---

## References

- `_actions.ts` current state — đọc để thấy `deliverMilestone` cũ cần xóa (lines 25-91)
- `DeliverMilestoneForm.tsx` current state — đọc để giữ CSS classes, thêm selected file display
- i18n keys: Story 8.8 (`milestone.deliver.*`) — keys phải tồn tại trong `en.json`/`vi.json` trước hoặc cùng lúc story này
- Epic 8.7 spec: `epics.md` lines 1314-1373
