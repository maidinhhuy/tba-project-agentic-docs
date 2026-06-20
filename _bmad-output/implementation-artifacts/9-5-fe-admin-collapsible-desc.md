# Story 9.5: FE Admin — Collapsible Description trong Tab "Thông tin"

**Status:** ready-for-dev
**Story ID:** 9.5
**Epic:** 9 — Project Detail UX Polish

---

## Story

As an admin,
I want the project description in the "Thông tin" tab to be collapsible with a reference link displayed below,
So that the admin management page is visually consistent and long descriptions don't crowd the workspace.

---

## Acceptance Criteria

**AC-1 — CollapsibleDescription trong Tab "Thông tin":**
Given Admin truy cập `/[locale]/admin/projects/[id]` và chọn Tab "Thông tin",
When tab render,
Then description hiển thị dùng `CollapsibleDescription` component (`src/components/ui/CollapsibleDescription.tsx` — từ Story 9.3, không tạo lại).
And behavior giống hệt customer side: collapsed 3 dòng, toggle "Xem thêm"/"Thu gọn", arrow xoay 180°.

**AC-2 — Reference link bên dưới collapsible description:**
Given project có `reference` (non-null, non-empty),
When Tab "Thông tin" render,
Then reference hiển thị bên dưới collapsible description trong cùng một card, cách nhau bởi `border-t`: icon `ExternalLink` + URL/label, `target="_blank"`, `rel="noopener noreferrer"`.
And khi `reference` null hoặc empty, reference field không render.

**AC-3 — Long description không chiếm hết viewport:**
Given description rất dài (> 3 dòng),
When Admin mở Tab "Thông tin",
Then chỉ 3 dòng đầu visible, không chiếm hết viewport, Admin có thể ngay lập tức thấy và thao tác với các phần khác của tab.

**AC-4 — Accessibility:**
Given Admin dùng screen reader,
When tương tác với collapsible description trong admin tab,
Then toggle button có `aria-expanded={isExpanded}`, accessibility behavior giống customer side (đã được xử lý bên trong `CollapsibleDescription`).

---

## ⚠️ Dev Notes — Đọc kỹ trước khi code

### Dependency: Story 9.3 phải hoàn thành trước

Story này **depends on Story 9.3**. Component `CollapsibleDescription` được tạo ở 9.3:
- Path: `src/components/ui/CollapsibleDescription.tsx`
- **Không implement lại** — chỉ import và dùng.

### File cần sửa (duy nhất)

**`src/app/[locale]/(admin)/admin/projects/[id]/_components/ProjectDetailTabs.tsx`**

Không cần sửa `page.tsx`, layouts, hay message files.

### Thay đổi cụ thể trong ProjectDetailTabs.tsx

**1. Thêm import `CollapsibleDescription`:**
```tsx
import { CollapsibleDescription } from '@/components/ui/CollapsibleDescription'
```

**2. Xoá imports không còn dùng (nếu không dùng ở nơi khác trong file):**
```tsx
// Xoá nếu không còn dùng:
import ReactMarkdown from 'react-markdown'
import remarkGfm from 'remark-gfm'
```
Kiểm tra toàn bộ file trước khi xoá — nếu `ReactMarkdown` còn dùng ở chỗ khác thì giữ lại.

**3. Merge Description Card + Reference Card thành 1 card:**

Thay thế 2 card riêng biệt (Description Card và Reference Card) bằng 1 card hợp nhất:

```tsx
{(project.description || project.reference) && (
  <div className="bg-white border border-gray-200/80 rounded-xl p-5 shadow-sm space-y-4">
    <h2 className="text-lg font-semibold text-gray-900 pb-3 border-b border-gray-100">
      {tProject('detail.description')}
    </h2>
    {project.description && (
      <CollapsibleDescription content={project.description} />
    )}
    {project.reference && (
      <div className="pt-4 border-t border-gray-100">
        <a
          href={project.reference}
          target="_blank"
          rel="noopener noreferrer"
          className="inline-flex items-center gap-1.5 text-sm text-teal-600 hover:text-teal-700 break-all transition-colors"
        >
          <svg className="h-4 w-4 shrink-0" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14" />
          </svg>
          {project.reference}
        </a>
      </div>
    )}
  </div>
)}
```

### i18n — Không cần thêm translation key mới

- `CollapsibleDescription` tự xử lý "Xem thêm"/"Thu gọn" bên trong component (đã handled ở Story 9.3).
- `tProject('detail.description')` và `tProject('detail.reference')` đã tồn tại trong `vi.json` → `projects.detail`.
- **Không cần sửa message files.**

### Lưu ý về icon ExternalLink

SVG inline trong pattern trên tương đương với `lucide-react`'s `ExternalLink`. Nếu project đã import `lucide-react`, có thể dùng:
```tsx
import { ExternalLink } from 'lucide-react'
// ...
<ExternalLink className="h-4 w-4 shrink-0" />
```
Kiểm tra các import hiện có trong file để quyết định dùng SVG inline hay lucide.

---

## Tasks / Subtasks

### Task 1: Cập nhật ProjectDetailTabs.tsx
**File:** `src/app/[locale]/(admin)/admin/projects/[id]/_components/ProjectDetailTabs.tsx`

- [ ] Thêm import `CollapsibleDescription` từ `@/components/ui/CollapsibleDescription`
- [ ] Kiểm tra `ReactMarkdown` và `remarkGfm` còn dùng ở chỗ nào khác trong file không; nếu không thì xoá cả import lẫn usage
- [ ] Xoá Description Card cũ (có `ReactMarkdown`) và Reference Card cũ (card riêng biệt)
- [ ] Thêm merged card mới: `(project.description || project.reference)` → hiển thị `CollapsibleDescription` + reference link bên dưới `border-t`
- [ ] Đảm bảo reference link có `target="_blank"`, `rel="noopener noreferrer"`, icon ExternalLink, class `break-all`
- [ ] Đảm bảo khi `project.reference` null/empty thì reference section không render

### Task 2: Kiểm tra visual & behavior
- [ ] Chạy dev server, truy cập `/[locale]/admin/projects/[id]` → Tab "Thông tin"
- [ ] Verify description collapse/expand hoạt động đúng (3 dòng, toggle button, arrow)
- [ ] Verify reference link hiển thị đúng bên dưới description khi có data
- [ ] Verify reference không render khi project không có reference
- [ ] Verify không có lỗi TypeScript hay ESLint

---

## Verification Checklist

- [ ] `CollapsibleDescription` được import từ `@/components/ui/CollapsibleDescription` (không tạo lại)
- [ ] Description Card và Reference Card được merge thành 1 card duy nhất
- [ ] `ReactMarkdown`/`remarkGfm` imports được xoá nếu không còn dùng (không để dead import)
- [ ] Reference link có đủ: `target="_blank"`, `rel="noopener noreferrer"`, ExternalLink icon
- [ ] Khi `reference` là null/empty, reference section không render
- [ ] Không có thay đổi nào ở `page.tsx`, layouts, hay message files
- [ ] Story 9.3 đã hoàn thành và `CollapsibleDescription.tsx` tồn tại trước khi dev story này

---

## Files Touched Summary

| Action | File |
|--------|------|
| MODIFY | `src/app/[locale]/(admin)/admin/projects/[id]/_components/ProjectDetailTabs.tsx` |
