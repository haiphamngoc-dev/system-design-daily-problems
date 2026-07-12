---
name: branches
description: >-
  Tạo và đặt tên nhánh git theo đúng quy ước dự án. Áp dụng khi tạo nhánh,
  checkout nhánh mới, hoặc khi đề cập đến quy ước đặt tên nhánh.
---

# Quy ước Đặt tên Nhánh

Sử dụng lowercase kebab-case với tiền tố loại nhánh. Mã issue/ticket là **tùy chọn** (optional).

```bash
# Mẫu: <loại>/<mã-issue>/<mô-tả-ngắn>   (có mã issue)
# Mẫu: <loại>/<mô-tả-ngắn>              (không có mã issue)

# Ví dụ có mã issue
feature/123/add-chapter-caching
bugfix/789/fix-broken-links
docs/333/update-introduction
test/444/add-mdbook-tests

# Ví dụ không có mã issue
feature/add-rate-limiting-chapter
bugfix/fix-image-paths
docs/update-readme

# Nhánh đặc biệt
release/v1.0.0
```

## Các Loại Nhánh

- `feature/` — Viết chương mới, thêm bài viết mới hoặc tái cấu trúc nội dung.
- `bugfix/` — Sửa lỗi chính tả, sửa link hỏng, sửa thông tin không chính xác.
- `docs/` — Thay đổi các tài liệu bổ trợ hoặc cấu hình của sách (như book.toml, README).
- `test/` — Thay đổi liên quan đến test cấu trúc hoặc chạy link checker.
- `release/` — Nhánh chuẩn bị phát hành phiên bản sách mới.

## Tham chiếu Issue

- Sử dụng mã số (ID) của issue nếu có hệ thống quản lý công việc (GitHub Issues, Jira, Redmine...).
- Chỉ dùng `#` trong commit message và pull request, **không dùng ký tự `#` trong tên nhánh**.
