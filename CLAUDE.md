# ai-memory — Kho kiến thức chung cho AI

Repo này là bộ nhớ dài hạn dùng chung giữa các project (tc-ems-fe, tc-ims-fe, ...).
AI có toàn quyền đọc/ghi repo này bất kỳ khi nào thấy cần thiết.

## Điểm vào

- **Luôn đọc [memory/index.md](memory/index.md) trước** — mỗi kiến thức một dòng. Chỉ mở file chi tiết khi dòng index cho thấy nó liên quan.
- Không bao giờ ghi nội dung kiến thức vào index; index chỉ chứa pointer.

## Phân loại

| Folder | Dùng cho |
|---|---|
| `docs/knowledge/` | Pattern, cách làm, gotcha tái sử dụng được |
| `docs/decisions/` | Quyết định kỹ thuật + lý do (ADR ngắn: bối cảnh → quyết định → hệ quả) |
| `docs/bugs/` | Bug khó: triệu chứng → nguyên nhân gốc → cách phát hiện lần sau |
| `docs/architecture/` | Tổng quan hệ thống, ranh giới module/service |
| `docs/tasks/` | Việc dở dang, kế hoạch dài hạn bàn giao giữa các session |

## Quy ước file

- Tên file: kebab-case. Kiến thức gắn với một project cụ thể thì prefix tên project:
  `tc-ems-fe--token-refresh-race.md`. Kiến thức chung thì không prefix.
- Mỗi file = MỘT mẩu kiến thức độc lập, template:

```markdown
---
title: <tiêu đề ngắn>
description: <1 dòng — dùng để quyết định độ liên quan khi tra cứu>
projects: [tc-ems-fe]        # hoặc [all] nếu là kiến thức chung
type: knowledge | decision | bug | architecture | task
created: 2026-08-10          # ngày tuyệt đối, không dùng "hôm qua"
tags: [auth, react-query]
---

<nội dung. Ngắn gọn, đủ để một AI chưa từng thấy session gốc áp dụng được.>

**Why:** <lý do / bối cảnh — phần quan trọng nhất, code không tự nói được>
**How to apply:** <khi nào và cách dùng>
```

- Liên kết kiến thức liên quan bằng đường dẫn tương đối: `[xem thêm](../bugs/xxx.md)`.

## Quy tắc ghi

1. **Chỉ ghi thứ không suy ra được từ code**: quyết định + lý do, bug khó + nguyên nhân gốc,
   quy ước team, gotcha thư viện, ngữ cảnh nghiệp vụ. KHÔNG chép cấu trúc code, không tóm tắt
   những gì git history / CLAUDE.md của project đã ghi.
2. **Trước khi tạo file mới, kiểm tra index** — nếu đã có file tương tự thì cập nhật file đó,
   không tạo trùng.
3. **Sau khi ghi/sửa file, cập nhật [memory/index.md](memory/index.md)** ngay trong cùng lượt.
4. Kiến thức sai hoặc lỗi thời: **sửa hoặc xóa**, kèm xóa dòng index. Kho nhỏ mà đúng
   giá trị hơn kho to mà nhiễu.
5. Ngày tháng luôn viết tuyệt đối (2026-08-10), không viết tương đối.

## Trích xuất từ session

Dùng skill `/extract-knowledge` (đã cài ở `~/.claude/skills/extract-knowledge/`) —
nó lọc kiến thức đáng nhớ từ session hiện tại, ghi theo template trên và cập nhật index.
