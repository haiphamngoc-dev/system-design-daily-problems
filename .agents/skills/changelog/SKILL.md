---
name: changelog
description: >-
  Cập nhật file CHANGELOG.md theo chuẩn Keep a Changelog và Semantic Versioning.
  Áp dụng khi thêm chương mới, sửa lỗi bài viết, thay đổi cấu hình sách,
  hoặc khi người dùng yêu cầu cập nhật changelog, nhật ký thay đổi,
  hoặc ghi nhận thay đổi phiên bản.
---

# Hướng dẫn Cập nhật Changelog

## Định dạng

File `CHANGELOG.md` tuân thủ chuẩn [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) và [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

**BẮT BUỘC: Nội dung changelog viết bằng tiếng Việt.**

---

## Cấu trúc File

```markdown
# Nhật ký Thay đổi

## [Chưa phát hành]

### Added

- Thêm chương mới về thiết kế hệ thống Caching bằng tiếng Việt.

### Fixed

- Sửa lỗi chính tả và các liên kết bị hỏng trong chương API Gateway.

---

## [1.0.0] - 2026-07-12

### Added

- Khởi tạo dự án sách System Design Daily Problems.
```

---

## Các Loại Thay đổi

Sử dụng đúng category theo chuẩn Keep a Changelog. **Giữ nguyên tên category bằng tiếng Anh**:

| Category     | Ý nghĩa                                             |
| ------------ | --------------------------------------------------- |
| `Added`      | Chương/nội dung mới được thêm vào                   |
| `Changed`    | Thay đổi, cập nhật cấu trúc bài viết hiện tại       |
| `Deprecated` | Nội dung kiến thức lỗi thời sắp bị loại bỏ          |
| `Removed`    | Nội dung đã bị xóa                                  |
| `Fixed`      | Sửa lỗi chính tả, liên kết, hoặc logic thiết kế sai |
| `Security`   | Cập nhật liên quan đến bảo mật (nếu có)             |

---

## Quy tắc Viết

### 1. Viết vào phần `[Chưa phát hành]`

Mọi thay đổi mới luôn được ghi vào phần `[Chưa phát hành]` ở đầu file. Khi release phiên bản mới, phần này sẽ được chuyển thành phiên bản cụ thể.

### 2. Mỗi entry là một dòng bullet

- Bắt đầu bằng dấu `-` (dash)
- Mô tả ngắn gọn, rõ ràng bằng **tiếng Việt**
- Nếu liên quan đến issue, thêm reference ở cuối (VD: `(refs #123)`)

### 3. Sắp xếp theo category

Liệt kê các category theo thứ tự: `Added` → `Changed` → `Deprecated` → `Removed` → `Fixed` → `Security`. Chỉ liệt kê category nào có thay đổi.

### 4. Phiên bản mới

Khi release phiên bản:

```markdown
## [1.1.0] - 2026-08-01

### Added

- Thêm chương thiết kế hệ thống Chat thời gian thực. (refs #123)
- Thêm hình ảnh minh họa cho chương API Gateway. (refs #125)

### Fixed

- Sửa lỗi đường dẫn ảnh bị lỗi hiển thị ở chương 1. (refs #456)
```

---

## Checklist

- [ ] Thay đổi được ghi vào phần `[Chưa phát hành]`.
- [ ] Sử dụng đúng category (`Added`, `Changed`, `Fixed`...).
- [ ] Mô tả bằng tiếng Việt, ngắn gọn và rõ ràng.
- [ ] Có reference tới issue/ticket nếu có liên quan.
- [ ] Không ghi các thay đổi nhỏ không đáng chú ý (VD: fix typo, format code).
