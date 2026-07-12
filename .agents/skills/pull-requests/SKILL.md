---
name: pull-requests
description: >-
  Tạo và review pull request theo đúng tiêu chuẩn dự án trên GitHub bao gồm định dạng
  tiêu đề theo Conventional Commits, template mô tả, và checklist review.
  Áp dụng khi tạo PR, viết mô tả PR, hoặc review pull request.
---

# Hướng dẫn Pull Request

## Trước khi Tạo PR

Trước khi commit và push code, **bắt buộc** thực hiện:

1. **Kiểm tra build sách local**:
   Chạy lệnh sau để đảm bảo sách build thành công mà không có lỗi markdown hay link hỏng:

   ```bash
   mdbook build
   ```

2. **Chạy các bài kiểm thử tự động (nếu có)**:

   ```bash
   mdbook test
   ```

---

## Tạo PR

### Template Mô tả PR (Mặc định)

```markdown
# Mô tả thay đổi

Tóm tắt nội dung chương/bài viết mới hoặc các sửa đổi chính.

# Hướng dẫn kiểm thử & Đọc thử

Mô tả cách build sách hoặc xem trước các thay đổi.

---

refs #123
```

### Tiêu đề PR

**BẮT BUỘC:** Tiêu đề PR phải tuân theo chuẩn **Conventional Commits** và viết bằng **tiếng Anh**.

```text
# Hợp lệ
feat(caching): add chapter on distributed caching
fix(gateway): resolve broken markdown links in api gateway chapter
refactor(summary): reorganize summary ordering for book structure
docs: update readme with build instructions

# Không hợp lệ
feat(caching): Thêm chương về caching        # Không dùng tiếng Việt
Add caching chapter                          # Thiếu type và scope
feat: add caching chapter                    # Thiếu scope
```

### Mô tả PR

**Mô tả PR viết bằng tiếng Việt** để reviewer dễ nắm bắt nội dung thay đổi. Sử dụng template phù hợp.

**Hướng dẫn viết mô tả PR:**

- **Giữ ngắn gọn** — Tránh mô tả quá dài dòng.
- **Tập trung vào thay đổi tổng quan** — Tránh liệt kê từng dòng hay từng từ sửa đổi.
- **Súc tích và đi thẳng vào vấn đề** — Chỉ mô tả "tại sao" và "cái gì" ở mức tổng quan.

---

## Các Template PR trên GitHub

Dự án hỗ trợ template PR tại `.github/pull_request_template.md`. Chọn template phù hợp khi tạo PR trên GitHub.

---

## Quy trình Tạo PR Description

Khi được yêu cầu tạo PR description để merge từ nhánh nguồn vào nhánh đích, AI thực hiện theo các bước sau:

### Bước 1: Phân tích Diff

1. Xác định nhánh nguồn và nhánh đích.
2. Chạy `git log origin/main..<nhánh-nguồn> --oneline` để xem danh sách commit.
3. Chạy `git diff origin/main..<nhánh-nguồn> --stat` để xem tổng quan file thay đổi.
4. Đọc diff chi tiết của các file markdown quan trọng nếu cần.

### Bước 2: Tạo Tiêu đề PR

- Theo chuẩn **Conventional Commits**, viết bằng **tiếng Anh**.
- Xác định `type` từ nội dung thay đổi (`feat`, `fix`, `refactor`...).
- Xác định `scope` từ chương/nội dung chính bị ảnh hưởng.
- Viết mô tả ngắn gọn (dưới 72 ký tự).

### Bước 3: Viết Mô tả PR

- Viết bằng **tiếng Việt**.
- Tóm tắt nội dung chính và nguyên nhân cần thay đổi.
- Liệt kê các chương được thêm mới/sửa đổi.
- Thêm hướng dẫn build/xem trước sách ở local.
- Thêm reference tới issue/ticket nếu có.

### Bước 4: Gắn Nhãn

- Thêm nhãn **"AI-Made"** nếu PR được tạo bởi AI.

### Ví dụ Output

AI sẽ trả về kết quả theo format sau:

```text
Tiêu đề PR:
feat(caching): add distributed caching strategies chapter

Mô tả PR:
```

```markdown
# Mô tả thay đổi

Thêm chương mới chi tiết về chiến dịch Caching trong thiết kế hệ thống.

**Nội dung chính:**

- Phân tích các mô hình Caching (Cache-aside, Write-through, Write-behind).
- Cách xử lý Cache Invalidation và Cache Penetration/Stampede.
- So sánh Redis và Memcached.

# Hướng dẫn kiểm thử & Đọc thử

1. Chạy lệnh: `mdbook serve` tại thư mục gốc.
2. Truy cập `http://localhost:3000` trên trình duyệt để đọc thử chương Caching.
3. Xác minh liên kết tại `SUMMARY.md` hoạt động chính xác.

---

refs #123
```

---

## Quy trình Review

### Với vai trò Tác giả PR

- **Phản hồi mọi comment** — Giải quyết từng phản hồi của reviewer.
- **Cập nhật nội dung theo phản hồi** — Thực hiện các thay đổi được yêu cầu.
- **Giữ PR cập nhật** — Thường xuyên rebase trên `main` để tránh conflict.

## Checklist Review

- [ ] Nội dung chương viết đúng cấu trúc chuẩn của dự án.
- [ ] Định dạng markdown chuẩn xác, không bị vỡ giao diện.
- [ ] Chạy `mdbook build` thành công, không bị lỗi liên kết hỏng.
- [ ] Hình ảnh minh họa được lưu trữ đúng thư mục tài nguyên và hiển thị tốt.
- [ ] Tiêu đề PR đúng chuẩn Conventional Commits và bằng tiếng Anh.
- [ ] Các sơ đồ Mermaid hoặc PlantUML hiển thị chính xác.
