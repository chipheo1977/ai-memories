---
name: extra-knowledge
description: Viết "kiến thức bổ trợ" (extra knowledge) giải thích các khái niệm nền tảng mà dev cần hiểu để làm một phase/task — theo mạch Là gì → Tại sao → Khi nào → Ví dụ thật từ dự án → Bẫy đã gặp. Ghi vào ai-memories/docs/knowledge/ và cập nhật index. Trigger khi user nói "extra knowledge", "extra-knowledge", "kiến thức bổ trợ", "giải thích khái niệm của phase này", "X là gì? tại sao cần X? khi nào cần X?", "viết tài liệu học cho phase/task".
version: 1.0.0
---

# Extra knowledge — kiến thức bổ trợ cho một phase/task

Mục tiêu: sau một phase/task, dev (kể cả người chưa từng làm mảng đó) đọc một file là hiểu **khái niệm đứng sau việc đã làm** — không chỉ "chạy lệnh gì" mà "vì sao phải làm vậy". Đây là tài liệu học, nhưng vẫn phải tuân quy ước của kho `ai-memories` (xem `CLAUDE.md` của repo).

## Khi nào dùng
- User hỏi một chuỗi câu kiểu "X là gì / tại sao cần X / khi nào cần X" quanh việc vừa làm.
- Kết thúc phase GSD có khái niệm mới với người làm (migration, idempotent, cache quyền, faceted filter, web worker, ...).
- User nói "tự suggest nội dung" → bạn đề xuất dàn ý, không hỏi lại từng mục.

## Các bước

### 1. Xác định phạm vi
- Đọc artifact của phase/task: `.planning/phases/*/` (CONTEXT, PLAN, SUMMARY, VERIFICATION, RELEASE-NOTES), PR, code đã đổi.
- Liệt kê **3–6 khái niệm** người làm cần hiểu. Ưu tiên khái niệm mà user vừa hỏi hoặc từng vướng.
- Nếu user chưa chỉ rõ chủ đề, đề xuất dàn ý (tiêu đề + 1 dòng mỗi mục) trong 5–8 dòng rồi viết luôn, không chờ duyệt từng mục.

### 2. Kiểm tra trùng
- Mở `memory/index.md`. Nếu đã có file cùng chủ đề → **cập nhật file đó**, không tạo mới.

### 3. Viết file `docs/knowledge/<project>--<chu-de>.md`
Frontmatter theo template của repo, thêm tag `extra-knowledge`:

```markdown
---
title: <tiêu đề ngắn>
description: <1 dòng — dùng để quyết định độ liên quan khi tra cứu>
projects: [<tên project>]
type: knowledge
created: <YYYY-MM-DD tuyệt đối>
tags: [extra-knowledge, <chủ đề>]
---
```

Thân bài, theo thứ tự (bỏ mục không áp dụng, không bịa cho đủ):
1. **Tóm tắt 3 dòng** — đọc xong dòng này là nắm ý chính.
2. **X là gì** — định nghĩa bằng lời thường, 1 ví dụ đời thường nếu giúp ích.
3. **Tại sao cần X** — so sánh với cách không dùng X (hậu quả cụ thể), và vì sao *dự án này* cần.
4. **Khi nào cần / không cần X** — bảng "tình huống → cần hay không".
5. **Ví dụ thật từ dự án** — file, lệnh, kết quả đã xảy ra (giữ đường dẫn tương đối, không chép nguyên file).
6. **Bẫy đã gặp & cách tránh** — chỉ những gì thực sự gặp trong dự án. Đây là phần giá trị nhất.
7. **Tự kiểm tra** — 3–5 câu hỏi ngắn kèm đáp án 1 dòng.
8. **Liên quan** — link tương đối tới file khác trong kho.

Kết file bằng cặp **Why / How to apply** (1–2 dòng) như quy ước repo.

### 4. Cập nhật index
Thêm một dòng vào mục `## Knowledge` của `memory/index.md`: `- [Tiêu đề](../docs/knowledge/<file>.md) — mô tả 1 dòng (project)`.

### 5. Báo user
Nêu đường dẫn file, danh sách mục chính (3–6 gạch đầu dòng), và việc đã cập nhật index. **Không tự commit/push** — hỏi trước.

## Quy tắc chất lượng
- **Giải thích "vì sao" trước "làm thế nào".** Lệnh chỉ để minh họa.
- **Dùng sự kiện thật của dự án**, không văn mẫu giáo khoa. Mỗi khái niệm gắn ít nhất một ví dụ từ phase/task đang xét.
- **Viết cho người đọc chưa có ngữ cảnh**: không giả định đã biết GSD, tên người, hay lịch sử chat. Thuật ngữ lần đầu nêu nghĩa.
- **Tiếng Việt**, giữ nguyên thuật ngữ kỹ thuật tiếng Anh (migration, idempotent, seed...).
- **Ngắn gọn**: 80–160 dòng. Quá dài thì tách file theo khái niệm.
- **Không ghi bí mật**: tuyệt đối không chép connection string, mật khẩu, token, khóa JWT, tên host DB dùng chung. Chỉ mô tả "RDS dùng chung", "biến môi trường".
- **Ngày tuyệt đối** (2026-10-05), không viết "hôm qua", "tuần trước".
- **Chỉ khẳng định điều đã xác minh** (đã chạy hoặc đã đọc code). Điều chưa chắc ghi rõ "chưa kiểm chứng".
- Không biến thành changelog hay tóm tắt code; những gì đọc code là ra thì không ghi.

## Ví dụ kích hoạt
- "extra knowledge về Phase 1 IAM BE migration"
- "viết kiến thức bổ trợ cho phase 3: web worker và pivot"
- "giải thích idempotent là gì, ghi vào kho kiến thức"
