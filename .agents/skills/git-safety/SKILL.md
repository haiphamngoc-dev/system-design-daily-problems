---
name: git-safety
description: >-
  Quy tắc an toàn bắt buộc cho các nhánh được bảo vệ: không bao giờ commit,
  push, hay force-push trực tiếp vào main; không viết lại lịch sử phá hủy
  mà không có sự đồng ý rõ ràng từ người dùng. Áp dụng trước bất kỳ thao tác git
  nào liên quan đến nhánh được bảo vệ, trước force-push, reset --hard,
  viết lại lịch sử, hoặc xóa nhánh, hoặc khi người dùng hỏi về merge, push,
  hay nhánh main.
---

# Quy tắc An toàn Git cho AI Agent

## QUAN TRỌNG: Quy tắc Nhánh được Bảo vệ

**AI agent KHÔNG BAO GIỜ được commit hoặc push trực tiếp vào nhánh được bảo vệ trong bất kỳ trường hợp nào.**

### Các Nhánh được Bảo vệ

- `main` — Nhánh chính chứa nội dung sách đã ổn định và sẵn sàng xuất bản/deploy.

Đây là quy tắc không thể thương lượng, áp dụng cho mọi tình huống:

### Các Hành vi Bị Cấm

- **Commit trực tiếp** — Không chạy `git commit` khi đang ở nhánh `main`.
- **Push trực tiếp** — Không chạy `git push origin main` hoặc `git push` khi đang ở nhánh `main`.
- **Force push** — Không chạy `git push --force` hoặc `git push -f` nhắm vào nhánh `main`.
- **Merge vào nhánh bảo vệ trên local** — Không chạy `git merge <nhánh>` khi đang ở nhánh `main`.
- **Rebase nhánh bảo vệ** — Không chạy `git rebase` khi đang ở nhánh `main`.
- **Reset nhánh bảo vệ** — Không chạy `git reset` khi đang ở nhánh `main`.

### Quy trình Bắt buộc

1. **Luôn tạo feature branch** trước khi thực hiện bất kỳ thay đổi nào.
2. **Kiểm tra nhánh hiện tại** trước mọi thao tác git bằng `git branch --show-current`.
3. **Tạo Pull Request** cho mọi thay đổi — để quy trình review/CI xử lý việc merge lên `main`.

### Checklist trước khi Push

Trước khi thực hiện bất kỳ lệnh push nào, AI agent phải:

1. Xác nhận nhánh hiện tại KHÔNG phải nhánh được bảo vệ (`main`).
2. Kiểm tra remote và nhánh đích.

### Xử lý Sự cố

Nếu vô tình đang ở nhánh được bảo vệ với thay đổi chưa commit:

1. Stash thay đổi: `git stash`
2. Tạo nhánh mới: `git checkout -b <tên-nhánh-phù-hợp>`
3. Áp dụng thay đổi: `git stash pop`
4. Tiếp tục làm việc trên nhánh mới.

## Lý do

- Nhánh `main` đại diện cho sách trực tuyến đang xuất bản.
- Mọi thay đổi phải qua code review thông qua Pull Request trên GitHub.
- Push trực tiếp bỏ qua kiểm tra GitHub Actions CI/CD và review nội dung.
