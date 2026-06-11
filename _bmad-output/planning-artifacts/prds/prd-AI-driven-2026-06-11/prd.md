---
title: PRD - TBA-agentic Platform
status: final
created: 2026-06-11
updated: 2026-06-11
---

# PRD: TBA-agentic Platform

## 0. Document Purpose

PRD này dành cho PM, Tech Lead, và các downstream workflows (architecture, epics/stories). Nó xác định requirements cho **phần platform quản lý dự án** của TBA-agentic — bao gồm customer portal và admin dashboard. Pipeline AI coding (AI coding agents + BMAD workflow) là infrastructure đã có sẵn và nằm ngoài scope tài liệu này.

UX design đã hoàn thiện tại `_bmad-output/planning-artifacts/ux-designs/ux-AI-driven-2026-06-11/` (DESIGN.md + EXPERIENCE.md + mockups). PRD này builds on UX spec, không duplicate.

Vocabulary trong PRD dùng đúng các term trong §3 Glossary. FRs được đánh số globally (FR-1 … FR-N) để downstream artifacts có stable references.

---

## 1. Vision

TBA-agentic là nền tảng prototype-as-a-service giúp startup founder và doanh nghiệp biến ý tưởng sản phẩm thành prototype, demo, hoặc MVP chạy được — nhanh hơn và rẻ hơn so với thuê freelancer hoặc agency truyền thống. Khác biệt cốt lõi: khách hàng không nhận một bản demo bỏ đi mà nhận một **launchpad** — source code extensible có thể tiếp tục phát triển.

Platform này là mặt tiền vận hành của dịch vụ. Nó cho phép khách hàng gửi ý tưởng, theo dõi tiến độ minh bạch theo milestone, và nhận sản phẩm — trong khi admin có đủ công cụ để quản lý toàn bộ pipeline dự án mà không cần trao đổi ngoài hệ thống.

Giai đoạn v1 vận hành theo mô hình agency: admin tạo tài khoản thủ công, phân tích yêu cầu thủ công bằng BMAD, và cập nhật trạng thái dự án qua dashboard. Lộ trình hướng đến tự động hoá.

---

## 2. Target User

### 2.1 Jobs To Be Done

**Customer:**
- Biến ý tưởng sản phẩm thành bản chạy được mà không cần team kỹ thuật riêng
- Theo dõi tiến độ dự án bất cứ lúc nào mà không cần hỏi trực tiếp
- Biết rõ mình còn bao nhiêu lần có thể yêu cầu chỉnh sửa
- Nhận sản phẩm đủ tốt để pitch nhà đầu tư hoặc test với người dùng thực

**Admin (Huy):**
- Nhìn thấy tất cả dự án đang chạy trong một nơi
- Cập nhật tiến độ và thông báo cho khách hàng mà không cần email thủ công
- Kiểm soát revision scope theo giới hạn đã thoả thuận

### 2.2 Non-Users (v1)

- Người dùng tự đăng ký — v1 chỉ hỗ trợ tài khoản do admin tạo
- Nhóm/team với nhiều người cùng quản lý một dự án — v1 là 1 customer = 1 tài khoản
- Người dùng không có tiếng Việt là ngôn ngữ chính — v1 UI toàn tiếng Việt

### 2.3 Key User Journeys

**UJ-1. Minh gửi dự án đầu tiên.**
- **Persona + context:** Minh, startup founder fintech, cần demo để pitch nhà đầu tư tuần sau. Đã có tài khoản do admin tạo.
- **Entry state:** Đã đăng nhập, đang ở Customer Dashboard, chưa có dự án nào.
- **Path:** (1) Thấy empty state với CTA "Tạo dự án đầu tiên". (2) Mở Submit Form, điền tên "FinTrack MVP", chọn Web App, nhập mô tả ý tưởng. (3) Nhấn "Gửi dự án".
- **Climax:** Redirect về Dashboard, card "FinTrack MVP" xuất hiện với badge SUBMITTED. Minh biết dự án đã được tiếp nhận mà không cần email xác nhận riêng.
- **Resolution:** Minh đóng tab, biết mình sẽ nhận được update trong 24 giờ.
- **Edge case:** Submit lỗi mạng — form giữ nguyên dữ liệu, Toast thông báo lỗi, Minh thử lại.

**UJ-2. Minh check tiến độ trước buổi pitch.**
- **Persona + context:** Minh, sáng thứ Tư, 3 ngày sau khi gửi dự án.
- **Entry state:** Đã đăng nhập, đang ở Customer Dashboard.
- **Path:** (1) Thấy card "FinTrack MVP" với badge IN_DEVELOPMENT. (2) Click vào card. (3) Đọc milestone list (1 done, 1 active, 1 pending), revision counter "Còn 3 lần chỉnh sửa", update mới nhất từ admin.
- **Climax:** Minh đọc xong trong 20 giây. Không có gì cần làm. Anh thấy tiến độ cụ thể và không lo lắng.
- **Resolution:** Đóng tab, quay lại chuẩn bị pitch.

**UJ-3. Admin cập nhật milestone sau khi AI agents hoàn thành sprint.**
- **Persona + context:** Huy, sau khi AI agents hoàn thành backend sprint.
- **Entry state:** Đã đăng nhập vào Admin Dashboard, đang xem table dự án.
- **Path:** (1) Tìm "FinTrack MVP" trong table. (2) Click vào row, vào Project Management. (3) Chuyển status từ IN_DEVELOPMENT sang AWAITING_REVIEW. (4) Điền progress note, nhấn "Lưu và thông báo".
- **Climax:** Toast xác nhận. Admin table cập nhật ngay lập tức. Email tự động gửi cho Minh.
- **Resolution:** Huy chuyển sang dự án tiếp theo trong queue.

---

## 3. Glossary

- **Project (Dự án)** — Một yêu cầu tạo prototype được Customer gửi, được theo dõi qua lifecycle từ SUBMITTED đến DELIVERED. Một Customer có thể có nhiều Projects.
- **Customer** — Người dùng platform, gửi Projects và theo dõi tiến độ. V1: tài khoản do Admin tạo thủ công.
- **Admin** — Người vận hành platform (Huy), quản lý tất cả Projects, tạo tài khoản Customer, cập nhật Status và Milestone.
- **Status** — Một trong 8 trạng thái cố định xác định Project đang ở giai đoạn nào trong delivery pipeline. Chỉ Admin mới thay đổi Status.
- **Status State Machine** — Luồng chuyển đổi hợp lệ giữa các Status: `SUBMITTED → ANALYZING → IN_DEVELOPMENT → AWAITING_REVIEW ⇄ IN_REVISION → FINALIZING → DELIVERED`. Bất kỳ state nào (trừ DELIVERED) có thể chuyển sang CANCELLED.
- **Milestone** — Một giai đoạn công việc được đặt tên trong Project, tạo từ Milestone Template bởi Admin. Customer chỉ xem, không tương tác.
- **Milestone Template** — Tập hợp tên milestones mặc định áp dụng cho Projects mới. [ASSUMPTION: template mặc định gồm 3 milestones: "UI/UX & Prototype", "Backend Development", "Frontend Integration & Deploy".]
- **Revision** — Một vòng chỉnh sửa, được tính khi Admin chuyển Status từ AWAITING_REVIEW sang IN_REVISION. Mỗi Project có tối đa 3 Revisions.
- **Revision Counter** — Số lần Revision còn lại, hiển thị công khai trên Project Detail. Bắt đầu ở 3, giảm 1 mỗi khi vào IN_REVISION.
- **Progress Note** — Văn bản do Admin viết khi cập nhật Status, hiển thị cho Customer như một update trong activity feed của Project.
- **Activity Feed** — Danh sách các Progress Notes theo thứ tự thời gian trên Project Detail.

---

## 4. Features

### 4.1 Account Management

**Description:** V1 không có self-signup. Admin tạo tài khoản Customer thủ công (email + mật khẩu tạm). Customer đăng nhập bằng email/password. Mỗi Customer chỉ thấy Projects của mình; Admin thấy tất cả Projects. Không có account management UI cho Customer ở v1 (đổi mật khẩu, profile) — [ASSUMPTION: nằm ngoài MVP scope].

**Functional Requirements:**

#### FR-1: Admin tạo tài khoản Customer

Admin có thể tạo tài khoản Customer với email và mật khẩu tạm thời từ Admin Dashboard.

**Consequences:**
- Tài khoản được tạo với role `customer`; không thể truy cập `/admin/*`
- Email xác nhận gửi đến Customer với thông tin đăng nhập [ASSUMPTION: gửi qua email system, không cần Customer tự kích hoạt ở v1]

#### FR-2: Customer đăng nhập

Customer có thể đăng nhập bằng email và mật khẩu.

**Consequences:**
- Đăng nhập thành công → redirect đến Customer Dashboard
- Sai credentials → thông báo lỗi, form giữ nguyên email
- Sau 5 lần sai liên tiếp → tài khoản bị khoá tạm thời 15 phút [ASSUMPTION]

#### FR-3: Phân quyền theo role

Hệ thống ngăn Customer truy cập `/admin/*` và ngăn Admin truy cập Customer Dashboard.

**Consequences:**
- Customer truy cập `/admin/*` → HTTP 403, redirect về Customer Dashboard
- Mỗi Customer chỉ thấy Projects của chính mình trong mọi API response

**Out of Scope:** Đổi mật khẩu, quên mật khẩu, 2FA, SSO — tất cả deferred v2.

---

### 4.2 Project Submission

**Description:** Customer tạo Project mới qua Submit Form. Form gồm: tên dự án, loại sản phẩm (Web / Mobile), mô tả ý tưởng (tối thiểu 50 ký tự), và trường tham khảo tuỳ chọn. Sau khi submit thành công, Project được tạo với Status SUBMITTED và Milestones từ Milestone Template mặc định. Realizes UJ-1.

**Functional Requirements:**

#### FR-4: Customer gửi Project mới

Customer có thể tạo Project mới bằng cách điền form và submit.

**Consequences:**
- Project được tạo với Status `SUBMITTED`
- Milestones được tạo từ Milestone Template mặc định (3 milestones)
- Customer redirect về Dashboard; Project card xuất hiện ngay
- Admin nhận email thông báo có Project mới [xem FR-17]

#### FR-5: Validation form submission

Hệ thống validate dữ liệu trước khi tạo Project.

**Consequences:**
- Tên dự án: bắt buộc, tối đa 100 ký tự
- Loại sản phẩm: bắt buộc, chỉ chấp nhận `web` hoặc `mobile`
- Mô tả: bắt buộc, tối thiểu 50 ký tự, tối đa 2000 ký tự
- Trường tham khảo: tuỳ chọn, tối đa 500 ký tự
- Submit button disabled cho đến khi tất cả required fields hợp lệ
- Lỗi validation hiển thị inline bên dưới từng field

---

### 4.3 Project Tracking (Customer Portal)

**Description:** Customer theo dõi tất cả Projects của mình qua Dashboard (card grid) và Project Detail (status đầy đủ, milestones, activity feed, revision counter). Toàn bộ là read-only đối với Customer — không có interaction nào trên data. Realizes UJ-2.

**Functional Requirements:**

#### FR-6: Customer Dashboard — danh sách Projects

Customer có thể xem tất cả Projects của mình dưới dạng card grid.

**Consequences:**
- Mỗi card hiển thị: tên Project, Status badge (icon + màu + label), milestone đang active, timestamp cập nhật gần nhất
- Cards sắp xếp mặc định: mới nhất trước (theo `updated_at`)
- Empty state khi chưa có Project: heading + CTA "Tạo dự án đầu tiên"
- Skeleton loader khi đang fetch data

#### FR-7: Project Detail — status và milestones

Customer có thể xem chi tiết Project bao gồm Status hiện tại, danh sách Milestones, và Revision Counter.

**Consequences:**
- Status badge hiển thị ở header cùng tên Project
- Milestone list: ordered, mỗi item có tên + trạng thái (done / active / pending). Done items có strikethrough. Active item có left border màu primary teal
- Revision Counter hiển thị khi Project đang ở state AWAITING_REVIEW hoặc IN_REVISION. Format: "Còn X lần chỉnh sửa". Chuyển sang màu destructive khi X ≤ 1
- Revision Counter ẩn khi Project ở SUBMITTED, ANALYZING, IN_DEVELOPMENT, FINALIZING, DELIVERED, CANCELLED

#### FR-8: Project Detail — activity feed

Customer có thể xem lịch sử Progress Notes theo thứ tự thời gian ngược.

**Consequences:**
- Mỗi entry hiển thị: timestamp, nội dung Progress Note
- Entries sắp xếp mới nhất trước
- Nếu chưa có Progress Note nào: không hiển thị section này (không empty state)

---

### 4.4 Admin Project Management

**Description:** Admin quản lý tất cả Projects qua Admin Dashboard (table) và Project Management screen (cập nhật Status, Milestone, Progress Note). Status transition tuân theo State Machine. Mỗi lần Admin chuyển sang IN_REVISION, Revision Counter của Project giảm 1. Realizes UJ-3.

**Functional Requirements:**

#### FR-9: Admin Dashboard — danh sách tất cả Projects

Admin có thể xem tất cả Projects của tất cả Customers trong một table.

**Consequences:**
- Columns: Customer name, Project name, Status badge, milestone đang active, timestamp cập nhật gần nhất, action link "Cập nhật"
- Table sortable theo: Status, updated_at (mặc định)
- Filter theo Status (multi-select, tất cả states)
- Skeleton loader khi đang fetch

#### FR-10: Admin cập nhật Status Project

Admin có thể thay đổi Status của một Project theo State Machine.

**Consequences:**
- Dropdown chỉ hiển thị các transitions hợp lệ từ Status hiện tại (theo State Machine)
- Chuyển sang IN_REVISION: Revision Counter giảm 1
- Chuyển sang IN_REVISION khi Revision Counter đã là 0: hệ thống cảnh báo Admin, cho phép override với xác nhận [ASSUMPTION: Admin có thể override nếu cần]
- Status change được lưu kèm timestamp và actor (admin)

#### FR-11: Admin thêm Progress Note

Admin có thể thêm Progress Note khi cập nhật Status.

**Consequences:**
- Progress Note là optional khi cập nhật Status
- Nội dung Progress Note xuất hiện trong Activity Feed của Customer
- Progress Note được lưu kèm timestamp của Status change

#### FR-12: Admin quản lý Milestones

Admin có thể xem và cập nhật trạng thái Milestones của một Project.

**Consequences:**
- Milestones được tạo tự động từ Milestone Template khi Project được submit (FR-4)
- Admin có thể đánh dấu milestone là done / active / pending
- Admin có thể chỉnh sửa tên milestone (tùy chỉnh theo dự án cụ thể) [ASSUMPTION]
- Thay đổi milestone không tự động trigger email — chỉ Status change mới trigger [xem FR-16]

#### FR-13: Admin tạo tài khoản Customer

*(Xem FR-1 — đặt ở đây để rõ context Admin workflow.)*

Admin có thể tạo tài khoản Customer mới từ Admin Dashboard.

**Consequences:**
- Form: email, tên hiển thị, mật khẩu tạm thời
- Tài khoản được tạo ngay lập tức với role `customer`

---

### 4.5 Email Notifications

**Description:** Hệ thống gửi email tự động ở hai sự kiện: khi Customer submit Project mới (admin được thông báo) và khi Admin cập nhật Status (Customer được thông báo). Email là kênh thông tin chính ở v1 — không có in-platform notification.

**Functional Requirements:**

#### FR-14: Thông báo Admin khi có Project mới

Khi Customer submit Project thành công, hệ thống gửi email thông báo cho Admin.

**Consequences:**
- Email gửi đến địa chỉ Admin được cấu hình trong system settings
- Nội dung: tên Project, tên Customer, loại sản phẩm, link trực tiếp đến Admin Project Management

#### FR-15: Thông báo Customer khi Admin cập nhật Status

Khi Admin lưu cập nhật Status của Project, hệ thống gửi email thông báo cho Customer.

**Consequences:**
- Email gửi đến email của Customer sở hữu Project
- Nội dung: tên Project, Status mới, Progress Note (nếu có), link đến Project Detail
- Email gửi đi sau khi Status change được lưu thành công; không gửi nếu save thất bại

**Feature-specific NFRs:**
- Email delivery: sử dụng transactional email service (SendGrid / Resend / tương đương) [ASSUMPTION]; không dùng SMTP trực tiếp
- Delivery timeout: nếu email service không phản hồi trong 5 giây, log lỗi và retry async — không block UI

---

## 5. Non-Goals (Explicit)

Phần này nêu các quyết định kiến trúc **có chủ ý** — không phải thiếu sót hay backlog. §6.2 liệt kê các tính năng bị defer vì timeline; phần này liệt kê những thứ **sẽ không build** vì lý do product.

- **Không có self-signup:** Platform là dịch vụ có kiểm soát — Admin vetting từng Customer là một phần của quy trình, không phải friction cần loại bỏ.
- **Không có customer interaction với milestones:** Milestone là cam kết của team, không phải negotiation — Customer theo dõi, không can thiệp.
- **Không có revision request UI cho Customer:** Admin chủ động chuyển IN_REVISION sau khi đánh giá feedback — không để Customer trigger revision tùy ý (bảo vệ revision limit có ý nghĩa).
- **Không có tự động hoá BMAD workflow:** Phân tích nghiệp vụ cần human judgment; platform là công cụ vận hành, không thay thế quy trình sáng tạo.

---

## 6. MVP Scope

### 6.1 In Scope

- Đăng nhập bằng email/password (Customer + Admin)
- Admin tạo tài khoản Customer thủ công
- Customer submit Project (form: tên, loại, mô tả, tham khảo)
- Customer Dashboard: card grid với status + milestone đang active
- Customer Project Detail: status, milestone list (read-only), activity feed, revision counter
- Admin Dashboard: project table với sort + filter theo status
- Admin Project Management: update status, progress note, milestone management
- 8 fixed status states theo State Machine
- Revision Counter: tự động giảm khi IN_REVISION, giới hạn 3 lần
- Email notification: admin khi có project mới, customer khi status thay đổi
- Milestone Template mặc định (3 milestones) áp dụng cho mọi project mới

### 6.2 Out of Scope cho MVP

- Self-signup Customer
- Đổi mật khẩu / quên mật khẩu / 2FA
- Payment & billing
- In-platform notifications (push/bell icon)
- Dark mode
- Mobile app cho Admin
- Tự động hoá BMAD workflow
- Multi-admin support
- Customer revision request UI
- File/attachment upload
- Project archiving UI (ngoài CANCELLED/DELIVERED)
- Analytics dashboard

---

## 7. Success Metrics

**Primary**

- **SM-1:** Hoàn thành ít nhất 3 Projects thực từ Customer bên ngoài, deliver đúng cam kết scope và timeline. Validates FR-4 → FR-15 (end-to-end pipeline).
- **SM-2:** Customer mở Project Detail ít nhất 1 lần sau mỗi lần Admin cập nhật Status (email click-through). Validates FR-7, FR-15.

**Secondary**

- **SM-3:** Ít nhất 1 Customer tiếp tục phát triển sản phẩm từ source code được bàn giao (tín hiệu: customer liên hệ lại hoặc confirm). Validates overall value proposition.
- **SM-4:** Thời gian Admin thực hiện một Status update (bao gồm progress note) < 5 phút. Validates FR-10, FR-11.

**Counter-metrics (không tối ưu)**

- **SM-C1:** Không tối ưu SM-4 bằng cách bỏ progress note — note chất lượng là yếu tố tạo trust với Customer, quan trọng hơn tốc độ Admin.

---

## 8. Resolved Design Decisions

Các câu hỏi thiết kế đã được quyết định trong quá trình PRD — ghi lại để tham chiếu:

| Câu hỏi | Quyết định | Lý do |
|---|---|---|
| Milestone Template: Web vs Mobile có khác nhau? | Cùng 1 template cho v1 | Admin tùy chỉnh tên milestone theo từng dự án (FR-12) |
| Revision Counter = 0, Admin override → Customer nhận email đặc biệt? | Không — email thông báo status thay đổi bình thường | Tránh phức tạp hoá v1 |
| Admin audit trail Status transitions? | Deferred v2 | V1 đủ với `updated_at` + Progress Notes |
| Email: HTML template hay plain-text? | Plain-text đủ cho v1 | Tốc độ triển khai; HTML deferred v2 |

---

## 9. Assumptions Index

| ID | Section | Nội dung assumption | Cần confirm |
|---|---|---|---|
| A1 | §4.1 FR-1 | Email xác nhận tài khoản gửi ngay khi tạo, không cần Customer kích hoạt | - |
| A2 | §4.1 FR-2 | Tài khoản bị khoá 15 phút sau 5 lần đăng nhập sai liên tiếp | - |
| A3 | §3 Glossary | Milestone Template mặc định: "UI/UX & Prototype", "Backend Development", "Frontend Integration & Deploy" | Confirm với Huy |
| A4 | §4.1 | V1 chỉ có một Admin account; không có admin management UI | - |
| A5 | §4.4 FR-12 | Admin có thể chỉnh sửa tên milestone sau khi Project được tạo | - |
| A6 | §4.4 FR-10 | Admin có thể override revision limit với xác nhận khi Counter = 0 | - |
| A7 | §4.5 | Transactional email service: SendGrid / Resend / tương đương — quyết định ở architecture phase | Architecture |
