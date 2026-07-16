# Nhật ký Thay đổi

Mọi thay đổi lớn đối với dự án này sẽ được ghi nhận tại đây.

## [Chưa phát hành]

### Added

- Thêm Bài toán 02: Tiêu diệt vấn đề truy vấn N+1 vào mục lục `SUMMARY.md`.
- Thêm Bài toán 03: Giới hạn tần suất yêu cầu không bị bùng nổ ở ranh giới vào mục lục `SUMMARY.md`.
- Thêm Bài toán 04: Ngăn chặn trùng lặp giao dịch thanh toán vào mục lục `SUMMARY.md`.
- Thêm Bài toán 05: Lựa chọn chiến lược phân mảnh cơ sở dữ liệu vào mục lục `SUMMARY.md`.
- Thêm Bài toán 06: Khóa phân tán an toàn cho tác vụ định kỳ vào mục lục `SUMMARY.md`.
- Thêm Bài toán 07: Thứ tự sự kiện trong Message Queue vào mục lục `SUMMARY.md`.

### Changed

- Tái cấu trúc Bài toán 02: Tiêu diệt vấn đề truy vấn N+1 theo chuẩn 6 phần, bổ sung sơ đồ Mermaid và ví dụ mã nguồn (Prisma, Hibernate).
- Tối ưu hóa độ tương phản của các sơ đồ Mermaid bằng cách loại bỏ các khối màu nền (rect background).
- Tái cấu trúc Bài toán 03: Giới hạn tần suất yêu cầu không bị bùng nổ ở ranh giới theo chuẩn 6 phần, bổ sung phân tích thuật toán Lazy Refill và ví dụ mã nguồn (Redis Lua Script, Bucket4j).
- Kích hoạt tính năng hỗ trợ công thức toán học MathJax trong `book.toml`.
- Tái cấu trúc Bài toán 04: Ngăn chặn trùng lặp giao dịch thanh toán theo chuẩn 6 phần, bổ sung sơ đồ Mermaid và ví dụ mã nguồn (TypeScript Express middleware, Java Spring aspect).
- Tái cấu trúc Bài toán 05: Lựa chọn chiến lược phân mảnh cơ sở dữ liệu theo chuẩn 6 phần, bổ sung sơ đồ Mermaid và ví dụ mã nguồn (TypeScript Express, Java Spring routing datasource).
- Tái cấu trúc Bài toán 06: Khóa phân tán an toàn cho tác vụ định kỳ theo chuẩn 6 phần, bổ sung sơ đồ Mermaid và ví dụ mã nguồn (TypeScript Node.js, Java Spring Boot).
- Tái cấu trúc Bài toán 07: Thứ tự sự kiện trong Message Queue theo chuẩn 6 phần, bổ sung sơ đồ Mermaid và ví dụ mã nguồn (TypeScript SQS client, Java Spring Cloud AWS).

### Fixed

- Sửa lỗi cú pháp Mermaid (thiếu dấu ngoặc kép cho nhãn subgraph chứa ký tự đặc biệt) trong Bài toán 03.
