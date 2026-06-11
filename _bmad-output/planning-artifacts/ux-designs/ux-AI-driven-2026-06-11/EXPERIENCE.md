---
name: TBA-agentic
status: final
sources:
  - _bmad-output/planning-artifacts/briefs/brief-AI-driven-2026-06-11/brief.md
  - _bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/DESIGN.md
created: 2026-06-11
updated: 2026-06-11
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
| Project Detail | `/projects/:id` | Dashboard card | Full status, milestones, updates, revision counter |

### Admin Dashboard

| Surface | Route | Reached from | Purpose |
|---|---|---|---|
| Project List | `/admin` | App open (admin role) | All projects across customers — table view |
| Project Management | `/admin/projects/:id` | Table row | Update status, milestone, progress note |

Navigation: persistent top nav on both portals. Customer nav: logo + "Dự án của tôi" + "Tạo dự án" + user avatar. Admin nav: logo + "Tất cả dự án" + admin avatar. Role routing enforced server-side; accessing `/admin` as customer returns 403, not a redirect.

→ Key screen references: `mockups/customer-dashboard.html`, `mockups/submit-project.html`, `mockups/project-detail.html`, `mockups/admin-dashboard.html`. Spine wins on conflict.

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
| Milestone list | Project Detail | Read-only ordered list. Completed milestones show checkmark + strikethrough label. Current milestone highlighted with `{colors.primary}` left border. Future milestones shown in `text-muted-foreground`. No interaction — display only. |
| `revision-counter` | Project Detail | Inline chip below milestone list. Always visible when project is in a revisable state (IN_REVISION, AWAITING_REVIEW). Hidden when DELIVERED or CANCELLED. Switches to destructive color at ≤ 1 remaining. |
| Update form | Admin Project Management | Status dropdown (all 8 states, current pre-selected) + Progress note textarea (optional, shown to customer as latest update). Save button triggers email notification to customer. |
| Admin project table | Admin Dashboard | Columns: customer name, project name, status badge, last updated, actions. Sortable by status and last updated. Filter by status (multi-select). Clicking row → Admin Project Management. |

## State Patterns

### Project Status States (8 fixed states)

State machine — valid transitions:
```
SUBMITTED → ANALYZING → IN_DEVELOPMENT → AWAITING_REVIEW ⇄ IN_REVISION
                                                               ↓
                                                          FINALIZING → DELIVERED

Any state (except DELIVERED) → CANCELLED
```

Admin triggers all forward transitions. `AWAITING_REVIEW ↔ IN_REVISION` cycle can repeat up to revision limit; counter decrements on each `IN_REVISION` entry.

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

### Flow 3 — Admin update milestone (Huy, sau khi AI agents hoàn thành sprint)

1. Huy mở `/admin`. Table hiện 4 dự án. "FinTrack MVP" của Minh đang `IN_DEVELOPMENT`, last updated 5h trước.
2. Huy click vào row → Admin Project Management.
3. Huy đổi status dropdown từ `IN_DEVELOPMENT` sang `AWAITING_REVIEW`.
4. Điền progress note: "Hoàn thiện core API và auth. Deploy lên staging tại [link]. Vui lòng review và feedback."
5. Click "Lưu và thông báo".
6. **Climax:** Toast: "Đã cập nhật và thông báo cho khách hàng." Admin table cập nhật ngay lập tức. Minh nhận email thông báo trong vài giây — mở ra thấy link staging và note từ Huy. Revision counter của Minh vẫn "Còn 3 lần" — chưa dùng lần nào.
