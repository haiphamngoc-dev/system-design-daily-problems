---
name: commits
description: >-
  Tạo commit message theo đúng định dạng Conventional Commits. Áp dụng khi
  tạo commit, viết commit message, hoặc khi người dùng yêu cầu commit
  các thay đổi.
---

# Hướng dẫn Commit Message

**BẮT BUỘC: Commit message phải luôn viết bằng tiếng Anh.** Bao gồm subject, body, và footer.

Tuân thủ định dạng **Conventional Commits**:

```text
<loại>(<phạm-vi>): <mô-tả>

<nội-dung-chi-tiết>

<footer>
```

## Các Loại Commit

- `feat` — Thêm chương mới, bài viết mới hoặc nội dung kiến thức mới
- `fix` — Sửa lỗi chính tả, sửa link hỏng, sửa logic thiết kế bị sai
- `refactor` — Tái cấu trúc cấu trúc thư mục, tối ưu hóa cách viết hoặc sắp xếp lại các phần trong chương
- `test` — Thêm hoặc cập nhật test (ví dụ: markdown link checker, build scripts)
- `docs` — Thay đổi tài liệu phụ (như README, CONTRIBUTING, hoặc tài liệu cấu hình ngoài book.toml)
- `chore` — Công việc bảo trì (nâng cấp mdbook, cấu hình workflow github)
- `style` — Thay đổi định dạng markdown, khoảng trắng, canh lề (không ảnh hưởng nội dung)
- `perf` — Cải thiện tốc độ build sách hoặc tối ưu dung lượng hình ảnh

## Ví dụ

```bash
feat(cache): add chapter detailing distributed caching strategies

Implements write-through, write-behind, and cache-aside patterns.

refs #123

---

fix(links): resolve broken cross-references in system-apis chapter

Ensure all internal chapter links point to correct relative paths.

Fixes #456

---

test(workflow): add github action for markdown link checker

refactor(summary): reorganize chapters order in SUMMARY.md

docs: update contribution guidelines in README

chore: upgrade mdbook version to 0.4.37
```

## Thực hành Tốt

### Commit Tốt

- Dòng mô tả rõ ràng, mang tính mô tả (giới hạn 50-72 ký tự)
- Thay đổi nguyên tử (mỗi commit chỉ chứa một thay đổi logic)
- Tham chiếu issue/ticket trong phần body
- Giải thích **tại sao**, không chỉ **cái gì**
- **Giữ ngắn gọn** — Không liệt kê từng file thay đổi trong body

```bash
feat(load-balancer): add chapter on load balancing algorithms

Explains Round Robin, Least Connections, and IP Hash algorithms
with their pros and cons.

refs #123
```

### Commit Không Tốt

```bash
# Quá mơ hồ
fix stuff
WIP
update

# Quá rộng
add chapter 1, fix links, update config
```

## Tham chiếu Issue

- **GitHub**: `refs #XXX` hoặc `closes #XXX` hoặc `resolves #XXX` hoặc `#XXX`
- Dùng `#` để tự động liên kết trong commit message và pull request.
