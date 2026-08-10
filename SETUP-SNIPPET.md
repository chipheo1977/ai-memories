# Đoạn dán vào CLAUDE.md của từng project

Dán block dưới đây vào CLAUDE.md của tc-ems-fe, tc-ims-fe, ... để AI ở project đó
biết kho kiến thức này tồn tại:

```markdown
## Knowledge base (ai-memory)

Kho kiến thức chung giữa các project: `/root/workspace/ai-memory`. AI có toàn quyền
đọc/ghi kho này bất kỳ khi nào thấy cần thiết.

- **Đầu session hoặc khi gặp vấn đề lạ**: đọc `/root/workspace/ai-memory/memory/index.md`
  (mỗi kiến thức một dòng) và mở file chi tiết nếu liên quan.
- **Khi học được điều đáng ghi nhớ** (quyết định + lý do, bug khó, pattern, gotcha):
  ghi vào ai-memory theo quy ước trong `/root/workspace/ai-memory/CLAUDE.md` và cập nhật index.
- User có thể gõ `/extract-knowledge` để trích xuất kiến thức từ session hiện tại.
```
