---
name: TBA-agentic
status: final
sources:
  - _bmad-output/planning-artifacts/briefs/brief-AI-driven-2026-06-11/brief.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/DESIGN.md
created: 2026-06-11
updated: 2026-06-20
---

# TBA-agentic — Experience Spine

## Foundation

Responsive web, desktop-primary. shadcn/ui on Next.js + Tailwind CSS. `DESIGN.md` is the visual identity reference; this spine is the experience. Two distinct role surfaces sharing the same codebase: **Customer Portal** and **Admin Dashboard**, separated by role-based routing.

Light mode only for MVP. Dark mode deferred to backlog.

## Information Architecture

### Customer Portal

| Surface | Route | Reached from | Purpose |
|---|---|---|---|
| Dashboard | `/` | App open / nav | Project cards with status at-a-glance |
| Submit Project | `/projects/new` | Dashboard CTA / nav | New project form |
| Project Detail | `/projects/:id` | Dashboard card | Milestone stepper, status, description, review actions, activity feed |
| Edit Project | `/projects/:id/edit` | Project Detail header "Chỉnh sửa" button | Edit project name, description, reference |

### Admin Dashboard

| Surface | Route | Reached from | Purpose |
|---|---|---|---|
| Project List | `/admin` | App open (admin role) | All projects across customers — table view |
| Project Management | `/admin/projects/:id` | Table row | Two-tab layout: Tab "Thông tin" (description + reference) · Tab "Quản lý" (status form, milestone rows, deliver modal, review history) |

Navigation: persistent top nav on both portals. Customer nav: logo + "Dự án của tôi" + "Tạo dự án" + user avatar. Admin nav: logo + "Tất cả dự án" + admin avatar. Role routing enforced server-side; accessing `/admin` as customer returns 403, not a redirect.

→ Key screen references: `mockups/customer-dashboard.html`, `mockups/submit-project.html`, `mockups/project-detail.html`, `mockups/admin-dashboard.html`. Spine wins on conflict.

→ Epic 8 UX decisions recorded in `.decision-log.md` (2026-06-20).

## Voice and Tone

Microcopy. Brand voice and aesthetic posture live in `DESIGN.md`.

| Context | Do | Don't |
|---|---|---|
| Status labels | "Đang phân tích" / "Đang xây dựng" | "Processing..." / "Status: 3" |
| Empty states | "Bạn chưa có dự án nào. Bắt đầu ngay." | "No projects found." |
| Revision counter | "Còn 3 lần chỉnh sửa" | "3 revisions left (max: 5)" |
| Error messages | "Không thể lưu. Thử lại." | "Error 500: Internal server error." |
| Success confirmations | "Dự án đã được gửi." | "Your submission was successful!" |
| Admin progress notes | Ngắn gọn, cụ thể: "Hoàn thiện auth module, đang build core API" | Vague: "Making progress" |

Language: Tiếng Việt cho toàn bộ UI. Technical terms (status labels) có thể giữ tiếng Anh nếu cần thiết cho dev clarity, nhưng microcopy và empty states luôn tiếng Việt.

## Component Patterns

Behavioral. Visual specs live in `DESIGN.md.Components`.

| Component | Surface | Behavioral rules |
|---|---|---|
| `project-card` | Customer Dashboard | Click anywhere → Project Detail. Non-clickable elements: none — whole card is the target. Status badge rendered inline with project name. Last update timestamp shown below milestone subtitle. |
| `status-badge` | Dashboard cards, Project Detail, Admin table | Read-only on customer surfaces. Admin can update via dropdown on Project Management surface. Transition animates via CSS `transition-colors`. |
| Submit form | Submit Project | Single-page form: project name (text input), description (textarea, min 50 chars), type (Select: Web / Mobile). Submit button disabled until all fields valid. No multi-step wizard at MVP. |
| `milestone-stepper` | Customer Project Detail | Full-width horizontal stepper spanning the project card. Each step: circle indicator (48px done/pending; 56px active) + milestone name + status label below. Connector line between circles fills teal for completed segments. Active step expands a panel directly below the stepper showing: deliverable link, delivery note, and two action buttons ("Chấp nhận" / "Yêu cầu chỉnh sửa"). Panel collapses when step is not active. No interaction on done or pending steps. Visual spec: `{components.milestone-stepper}` in DESIGN.md. |
| `collapsible-description` | Customer Project Detail, Admin Tab "Thông tin" | Description defaults to **collapsed** — 3 lines visible via `-webkit-line-clamp`. "Xem thêm" toggle expands to full content; text changes to "Thu gọn" when expanded. Arrow icon rotates 180° on expand. Reference field (URL) rendered as a clickable external link with icon, separated by a top border, always visible below the description toggle. Reference only renders when value is non-null and non-empty. |
| `revision-counter` | Customer Project Detail header | Inline chip in the project header area alongside the status badge. Visible only when status is IN_REVISION or AWAITING_REVIEW. Switches to destructive color + text "Còn 1 lần chỉnh sửa cuối" when count ≤ 1. Hidden when DELIVERED or CANCELLED. |
| `admin-milestone-row` | Admin Tab "Quản lý" | Each milestone renders as a row with: position indicator, name, and a review-status badge. "Giao milestone" button (primary) appears **only** when all three conditions are met: project status = `IN_DEVELOPMENT`, milestone status = `ACTIVE`, milestone `review_status` = `PENDING_DELIVERY`. Button is hidden (not disabled) in all other states. Review-status badge mapping: `PENDING_DELIVERY` → no badge (button shown instead); `PENDING_REVIEW` → "Đang chờ review" (yellow); `APPROVED` → "Đã chấp nhận" (green); `REVISION_REQUESTED` → "Cần chỉnh sửa" (orange). |
| `delivery-modal` | Admin Tab "Quản lý" | Opens when Admin clicks "Giao milestone". shadcn `Dialog`. Header: eyebrow "Giao milestone" + milestone name as title. Body: (1) **Deliverable group** — URL input and file dropzone in one bordered container, separated by an "Hoặc" divider; group label shows "Bắt buộc có ít nhất 1". Submit disabled until URL or file is provided. (2) **Ghi chú** — optional textarea below the group, with hint "Nội dung này sẽ hiển thị trực tiếp cho khách hàng." Footer: left-side informational note "Khách hàng sẽ nhận email thông báo ngay." + Cancel + "Giao ngay →". File dropzone accepts PDF/ZIP/Figma/video; max 50 MB; shows file name after selection. Visual spec: `{components.delivery-modal}` in DESIGN.md. |
| Update form | Admin Tab "Quản lý" | Status dropdown (valid transitions only from current state, not all 8 states) + Progress note textarea (optional). Save button triggers email notification to customer. |
| Admin project table | Admin Dashboard | Columns: customer name, project name, status badge, last updated, actions. Sortable by status and last updated. Filter by status (multi-select). Clicking row → Admin Project Management. |

## State Patterns

### Project Status States (8 fixed states)

State machine — valid transitions:
```
SUBMITTED → ANALYZING → IN_DEVELOPMENT ──(Admin delivers milestone)──→ AWAITING_REVIEW
                                ↑                                              |
                                └──(Customer approves, non-final milestone)───┘
                                                                               |
                                                              ┌────────────────┘
                                                              ↓
                                               IN_REVISION ←──────── (Customer requests revision)
                                                              │
                                               (Admin acts)  ↓
                                                       AWAITING_REVIEW
                                                              │
                                               (Customer approves final milestone)
                                                              ↓
                                                         DELIVERED

Any state (except DELIVERED) → CANCELLED
```

Trigger ownership:
- Admin triggers: `SUBMITTED→ANALYZING`, `ANALYZING→IN_DEVELOPMENT`, `AWAITING_REVIEW→IN_REVISION` (manual override), `*→CANCELLED`
- System triggers automatically: `IN_DEVELOPMENT→AWAITING_REVIEW` when Admin delivers a milestone via the delivery modal
- Customer triggers: `AWAITING_REVIEW→IN_DEVELOPMENT` (approve, non-final milestone), `AWAITING_REVIEW→DELIVERED` (approve, final milestone), `AWAITING_REVIEW→IN_REVISION` (request revision)

`AWAITING_REVIEW→IN_REVISION` cycle: revision_count decrements on each customer-triggered revision request. Admin can force-override at count = 0 via the status dropdown with confirmation.

### Milestone Review States (per milestone)

Each milestone has an independent `review_status` that drives the `admin-milestone-row` and `milestone-stepper` components:

| review_status | Meaning | Admin sees | Customer sees |
|---|---|---|---|
| `PENDING_DELIVERY` | Not yet delivered | "Giao milestone" button | Step pending (grayed) |
| `PENDING_REVIEW` | Delivered, awaiting customer action | "Đang chờ review" badge | Stepper expands with deliverable + action buttons |
| `APPROVED` | Customer accepted | "Đã chấp nhận" badge | Step marked done (checkmark) |
| `REVISION_REQUESTED` | Customer requested changes | "Cần chỉnh sửa" badge | Step reverts to active with revision note visible |

### Surface Loading States

| Surface | State | Treatment |
|---|---|---|
| Customer Dashboard | Loading | shadcn `Skeleton` — 3 card-shaped skeletons matching card height |
| Customer Dashboard | Empty (no projects) | Illustration-free empty state: heading "Bạn chưa có dự án nào", subtext, single primary CTA "Tạo dự án đầu tiên" |
| Project Detail | Loading | Skeleton rows for milestone list and update feed |
| Admin Dashboard | Loading | shadcn `Skeleton` table rows (5-6 rows) |
| Admin Dashboard | Empty | "Chưa có dự án nào được gửi." — no CTA needed |
| Submit form | Submitting | Button text → "Đang gửi…", disabled state, no spinner overlay |
| Admin save | Saving | Button text → "Đang lưu…", disabled. Toast on success: "Đã cập nhật và thông báo cho khách hàng." |

### Error States

| Scenario | Treatment |
|---|---|
| Form submit fails | shadcn Toast (destructive): "Không thể gửi dự án. Thử lại." Form data retained. |
| Admin save fails | Toast (destructive): "Không thể lưu. Thử lại." Form state retained. |
| Unauthorized access to `/admin` | 403 page: "Bạn không có quyền truy cập trang này." Link back to dashboard. |
| Project not found | 404 page: "Dự án không tồn tại." Link back to dashboard. |

## Interaction Primitives

**Mouse-primary** for MVP. Keyboard support follows shadcn/Radix defaults (Tab, Enter, Escape) without custom keyboard shortcuts.

- **Project card:** entire card surface is clickable (no nested click targets). Cursor `pointer`.
- **Status badge on admin:** clicking badge opens shadcn `Select` dropdown inline — no separate edit mode.
- **Forms:** submit on Enter when focus is on last field. Tab order matches visual reading order top-to-bottom.
- **Toast notifications:** auto-dismiss after 4 seconds. No action required.
- **Banned at MVP:** drag interactions, bulk-select, inline editing on customer surfaces, keyboard shortcuts beyond shadcn defaults.

## Accessibility Floor

Behavioral. Visual contrast lives in `DESIGN.md` (inherits shadcn's WCAG AA-compliant defaults; teal primary verified at AA against white: contrast ratio 4.6:1 ✓).

- WCAG 2.2 AA across all surfaces.
- `status-badge` always renders icon + color + text label — never color alone.
- `revision-counter` color change at ≤ 1 remaining is supplemented by text change: "Còn 1 lần chỉnh sửa cuối" (not color only).
- All form inputs have visible labels (not placeholder-only). Error states use `aria-describedby` linking input to error message.
- Tab order matches reading order. Escape closes all open dialogs/sheets.
- Skeleton loaders use `aria-busy="true"` on the container during loading.

## Responsive & Platform

| Breakpoint | Customer Portal | Admin Dashboard |
|---|---|---|
| `≥ lg` (1024px+) | 2-column card grid, top nav | Full table with all columns, top nav |
| `md` (768–1023px) | 1-column card grid | Table scrolls horizontally; "actions" column pinned right |
| `< md` (sm) | 1-column card grid, condensed nav | Table collapses to card list; sort/filter in Sheet |

Admin dashboard is desktop-optimized. Mobile admin is functional but not primary — no dedicated mobile admin UX at MVP.

## Key Flows

### Flow 1 — Gửi dự án đầu tiên (Minh, founder fintech, 11 giờ tối, deadline pitch tuần sau)

1. Minh mở TBA-agentic lần đầu tiên. Dashboard load — empty state: "Bạn chưa có dự án nào. Bắt đầu ngay." CTA nổi bật.
2. Minh click "Tạo dự án đầu tiên". Form mở tại `/projects/new`.
3. Minh điền: tên "FinTrack MVP", mô tả ý tưởng app tracking chi tiêu có AI categorization, chọn "Web".
4. Submit. Button → "Đang gửi…". 1.2 giây sau: redirect về Dashboard.
5. **Climax:** Dashboard hiện 1 card mới — "FinTrack MVP" với badge `SUBMITTED` màu xanh dương nhạt. Toast: "Dự án đã được gửi. Chúng tôi sẽ liên hệ trong 24 giờ." Minh đóng laptop — biết mình đã làm xong việc của mình.

Failure: submit lỗi → Toast destructive, form data retained, Minh không mất gì đã điền.

---

### Flow 2 — Check tiến độ sau 3 ngày (Minh, sáng thứ Tư trước pitch)

1. Minh mở Dashboard. Card "FinTrack MVP" hiện: badge `IN_DEVELOPMENT` màu teal, milestone "Backend API + Auth". Timestamp: "Cập nhật 5 giờ trước."
2. Minh click vào card → Project Detail.
3. Màn hình hiện: status badge to hơn, milestone list với 3 items (1 ✓ done, 1 active, 1 pending), latest update từ admin: "Đã hoàn thiện authentication, đang build expense tracking API."
4. Minh thấy revision counter: "Còn 3 lần chỉnh sửa".
5. **Climax:** Minh đọc xong trong 20 giây. Không có gì cần làm. Không lo lắng — anh thấy tiến độ cụ thể, biết milestone tiếp theo, biết còn bao nhiêu revision. Đóng tab, quay lại chuẩn bị pitch.

---

### Flow 3 — Admin giao milestone (Huy, sau khi AI agents hoàn thành Backend Development sprint)

1. Huy mở `/admin`. Table hiện "FinTrack MVP" đang `IN_DEVELOPMENT`.
2. Huy click row → Admin Project Management → Tab "Quản lý".
3. Milestone row "Backend Development" hiện nút "Giao milestone" (ACTIVE + PENDING_DELIVERY). Hai milestone kia không có nút.
4. Huy click "Giao milestone" → `delivery-modal` mở.
5. Huy điền link staging vào URL field. Skip file. Thêm ghi chú: "Đã hoàn thiện auth và expense API. Vui lòng test thử."
6. Click "Giao ngay →".
7. **Climax:** Modal đóng. Badge trên milestone row chuyển sang "Đang chờ review" (yellow). Project status badge trên header chuyển tự động sang `AWAITING_REVIEW`. Toast: "Đã giao milestone và thông báo cho khách hàng." Minh nhận email ngay.

Failure path: thiếu cả URL lẫn file → "Giao ngay" disabled. Upload file > 50 MB → toast error trong modal, form giữ nguyên.

---

### Flow 4 — Customer review milestone (Minh, nhận email thông báo có kết quả mới)

1. Minh nhận email "Kết quả milestone đã sẵn sàng: Backend Development". Click link → `/projects/fintrack-id`.
2. Project Detail load. Stepper ngang hiển thị: Step 1 (✓ done), Step 2 active (56px circle, "Đang chờ review").
3. Panel expand bên dưới stepper hiện: link staging, ghi chú của Huy, 2 nút "Chấp nhận" / "Yêu cầu chỉnh sửa".
4. Minh click link staging, test thử vài tính năng. Thấy lỗi ở màn hình báo cáo tháng.
5. Minh click "Yêu cầu chỉnh sửa" → dialog/form nhỏ yêu cầu nhập comment (bắt buộc).
6. Minh nhập: "Biểu đồ tháng hiển thị sai — cột tháng 6 bị âm. Vui lòng kiểm tra lại."
7. **Climax:** Submit. Panel trong stepper chuyển trạng thái — "Yêu cầu chỉnh sửa đã gửi." Revision counter trong header giảm: "Còn 2 lần chỉnh sửa". Huy nhận email với đầy đủ comment. Minh đóng tab — biết feedback đã đến đúng người.
