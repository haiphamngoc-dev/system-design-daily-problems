---
name: book-quality
description: >-
  Tiêu chuẩn chất lượng cho sách mdBook: định dạng Markdown, cấu trúc tiêu đề,
  quản lý hình ảnh, ngôn ngữ viết, và kiểm tra liên kết. Áp dụng khi viết hoặc
  chỉnh sửa bất kỳ file markdown nào trong thư mục src/**, chỉnh sửa SUMMARY.md,
  hoặc khi đề cập đến formatting, chính tả, hình ảnh hay build sách.
---

# Tiêu chuẩn Chất lượng Sách mdBook

## Quy tắc Bắt buộc

- **LUÔN chạy `mdbook build`** sau khi thay đổi bất kỳ nội dung nào để đảm bảo sách biên dịch không lỗi:

  ```bash
  mdbook build
  ```

- **Mọi liên kết nội bộ** (cross-reference) phải hoạt động chính xác. Tránh sử dụng đường dẫn tuyệt đối (như `https://...`); hãy dùng đường dẫn tương đối (như `./chapter_1.md`).
- **Ngôn ngữ nội dung**: Toàn bộ nội dung sách, bài viết phải viết bằng **tiếng Việt** chuẩn xác, văn phong chuyên nghiệp, dễ hiểu.
- **Thuật ngữ kỹ thuật**: Giữ nguyên tiếng Anh đối với các thuật ngữ chuyên ngành phổ biến (ví dụ: Cache, Load Balancer, Sharding, Database, API Gateway) hoặc mở ngoặc giải nghĩa ở lần xuất hiện đầu tiên.

---

## Tiêu chuẩn & Định dạng Markdown

### 1. Cấu trúc Tiêu đề (Heading Hierarchy)

- **Mỗi file markdown chỉ được phép có duy nhất một thẻ `#` (H1)** đại diện cho tiêu đề của chương/bài viết đó.
- Sử dụng các cấp tiêu đề tiếp theo (`##`, `###`, `####`) một cách hợp lý và phân cấp rõ ràng. Không nhảy cấp tiêu đề (ví dụ: từ `#` nhảy thẳng xuống `###`).

### 2. Định dạng Mã nguồn (Code Blocks)

- Luôn chỉ định ngôn ngữ cho các code block để bật tính năng syntax highlighting:

  ````markdown
  ```json
  {
    "id": 1,
    "name": "caching"
  }
  ```
  ````

- Đối với mã nguồn mẫu, sử dụng đúng ngôn ngữ tương ứng (`java`, `python`, `go`, `sql`, `yaml`, `json`...).

### 3. Sơ đồ & Minh họa (Mermaid / PlantUML)

- Ưu tiên sử dụng sơ đồ text-based (như **Mermaid**) để reviewer dễ dàng theo dõi và cập nhật trực tiếp trong file markdown.
- Đặt code block Mermaid như sau:

  ````markdown
  ```mermaid
  sequenceDiagram
      Client->>Gateway: API Request
      Gateway->>Service: Forward Request
  ```
  ````

---

## Quản lý Hình ảnh & Tài nguyên

- Tất cả tệp hình ảnh tĩnh (PNG, JPG, SVG) phải được lưu trữ trong thư mục tài nguyên chung (ví dụ: `/src/images/` hoặc `/src/assets/`). Không đặt ảnh rải rác ngoài thư mục quy định.
- Khi chèn ảnh vào Markdown, luôn sử dụng cú pháp chuẩn kèm theo văn bản thay thế (alt-text) mô tả sơ đồ:

  ```markdown
  ![Sơ đồ kiến trúc API Gateway](./images/api-gateway-architecture.png)
  ```

- Nén hình ảnh trước khi commit để tối ưu hóa dung lượng repository.

---

## Checklist trước khi Commit

- [ ] Chạy `mdbook build` thành công, không có cảnh báo (warnings) hay lỗi (errors).
- [ ] Không có liên kết nào bị hỏng (broken link).
- [ ] Bài viết có cấu trúc phân cấp tiêu đề rõ ràng (chỉ duy nhất một H1).
- [ ] Mọi code block đều được chỉ định ngôn ngữ syntax highlighting.
- [ ] Các sơ đồ Mermaid được viết đúng cú pháp và kết xuất chính xác.
- [ ] Hình ảnh được đặt đúng vị trí quy định và hiển thị tốt.
- [ ] Kiểm tra lỗi chính tả tiếng Việt và tính nhất quán của các thuật ngữ kỹ thuật.
