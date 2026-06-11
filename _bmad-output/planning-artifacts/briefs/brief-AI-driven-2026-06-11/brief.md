---
title: Product Brief - TBA-agentic
status: final
created: 2026-06-11
updated: 2026-06-11
---

# Product Brief: TBA-agentic

## Executive Summary

TBA-agentic là nền tảng prototype-as-a-service, giúp startup founder và doanh nghiệp biến ý tưởng sản phẩm thành prototype, demo, hoặc MVP chạy được — nhanh hơn và rẻ hơn so với con đường truyền thống. Khác với freelancer hay agency thông thường, TBA-agentic deliver source code có thể mở rộng: khách hàng không nhận một bản demo bỏ đi mà nhận một launchpad để tiếp tục phát triển sản phẩm. Toàn bộ pipeline vận hành bằng quy trình AI-native — AI coding agents kết hợp BMAD methodology — cho phép scale production mà không scale chi phí nhân sự tuyến tính.

Giai đoạn đầu vận hành theo mô hình agency có hỗ trợ AI. Lộ trình hướng đến tự động hóa toàn bộ từ input ý tưởng đến sản phẩm được deploy.

## The Problem

Startup founder và doanh nghiệp muốn validate ý tưởng sản phẩm đối mặt với bài toán nan giải: tự build thì thiếu nguồn lực kỹ thuật, thuê freelancer thì chậm và khó kiểm soát, thuê agency thì đắt và thường deliver thứ không thể tái sử dụng. Nhiều ý tưởng tốt chết trước khi được test với thị trường — hoặc tốn quá nhiều tiền và thời gian chỉ để biết idea không work.

Vấn đề cốt lõi không chỉ là chi phí mà là sự thiếu predictable: không rõ timeline, không rõ scope, không kiểm soát được chất lượng output.

## The Solution

TBA-agentic cung cấp một luồng có cấu trúc từ ý tưởng đến sản phẩm chạy được:

1. Khách hàng mô tả ý tưởng và tạo dự án trên platform
2. Team vận hành phân tích nghiệp vụ bằng BMAD, lập kế hoạch, tạo ticket nguyên tử
3. AI coding agents thực thi implement và deploy
4. Khách hàng nhận updates theo milestone, confirm tiến độ trong giới hạn revision đã thỏa thuận
5. Kết quả là web app hoặc mobile app được deploy, kèm source code extensible

## What Makes This Different

**Launchpad, không phải throwaway prototype.** Source code được build theo chuẩn extensible — khách hàng có thể tiếp tục phát triển từ đó sau khi nhận bàn giao, không bị lock-in vào TBA-agentic.

**AI-native production.** Toàn bộ pipeline từ phân tích nghiệp vụ đến coding đến deploy đều AI-assisted, tạo ra tốc độ và chi phí khác biệt rõ ràng so với quy trình thuần thủ công.

**Scope được kiểm soát từ đầu.** Revision limits và milestone structure đảm bảo cả khách hàng và team đều rõ ràng về kỳ vọng — không scope creep, không bất ngờ cuối dự án.

## Who This Serves

**Primary — Startup founder early-stage:** Người có ý tưởng sản phẩm nhưng chưa có team kỹ thuật hoặc không đủ ngân sách thuê dev full-time. Cần bản demo chạy được để pitch nhà đầu tư hoặc test với người dùng thực. Ưu tiên hàng đầu: tốc độ và predictability về chi phí, timeline.

**Secondary — Doanh nghiệp muốn PoC:** Team product hoặc innovation nội bộ muốn prove một concept trước khi đầu tư full build. Cần deliverable cụ thể, không cần production-grade ngay.

## Success Criteria

**Giai đoạn validate (hiện tại — chưa charge phí):**
- Hoàn thành ít nhất 3 dự án thực từ khách hàng bên ngoài
- Deliver đúng cam kết về timeline và scope
- Ít nhất 1 khách hàng tiếp tục phát triển sản phẩm từ source code được bàn giao

**Giai đoạn tiếp theo:**
- Platform tự phục vụ được luồng submit → tracking mà không cần admin xử lý thủ công phần communication
- Có thể nhận và xử lý đồng thời nhiều dự án song song

## Scope — MVP Platform

**Customer side:**
- Form tạo dự án: tên, mô tả ý tưởng, loại sản phẩm mong muốn (web / mobile)
- Danh sách dự án của khách hàng
- Chi tiết dự án: trạng thái hiện tại, milestones, updates từ team

**Admin side:**
- Dashboard danh sách tất cả dự án (cross-customer)
- Cập nhật trạng thái và milestone cho từng dự án
- Notification qua email khi có update mới

**Ngoài scope MVP:**
- Payment và billing
- Tự động hóa BMAD workflow
- Self-serve onboarding không cần can thiệp admin
- Revision tracking tự động
- Mobile app cho admin

## Vision

TBA-agentic trở thành infrastructure layer cho software ideation — bất kỳ ai có ý tưởng đều có thể biến nó thành sản phẩm thực trong vài tuần với chi phí predictable. Về dài hạn, platform tự động hóa toàn bộ pipeline: khách hàng submit ý tưởng, AI phân tích nghiệp vụ, agents implement và deploy, khách hàng nhận sản phẩm mà không cần human-in-the-loop cho dự án standard. TBA-agentic không chỉ là nơi build prototype — mà là điểm khởi đầu của mọi sản phẩm phần mềm.
