---
title: Border thead biến mất khi cuộn (sticky + border-collapse)
description: border-collapse:collapse + position:sticky trên thead làm border biến mất lúc cuộn dọc — chuyển sang box-shadow inset
projects: [tc-ims-fe]
type: bug
created: 2026-08-10
tags: [css, table, sticky, border-collapse]
---

**Triệu chứng:** Bảng có header "đóng băng" khi cuộn (`freezeHeader` → `<TableHeader className="sticky top-0">`)
— các đường viền (`border-l`, `border-t`, v.v.) trên `<TableHead>` hiển thị đúng lúc chưa cuộn, nhưng
**biến mất ngay khi cuộn dọc** (xem `VTTCInventoryReport.tsx`).

**Nguyên nhân gốc:** `table { border-collapse: collapse }` được áp **global** cho mọi `<table>` trong app
tại `src/index.css:298`. Đây là bug rendering đã biết của Chrome/WebKit: khi `border-collapse: collapse`
kết hợp `position: sticky` trên `<thead>`/`<th>`, trình duyệt tính toán border đã "hợp nhất" (collapse)
dựa theo **vị trí hình học gốc trong lưới bảng** (table grid layout), không phải vị trí render thực tế.
Khi phần tử sticky bị đẩy ra khỏi luồng và "dính" ở vị trí khác lúc cuộn, trình duyệt không vẽ lại đúng
border đã collapse ở vị trí mới.

Xảy ra với **mọi** bảng trong app có `freezeHeader`/sticky thead + border trên `TableHead`, không riêng
gì `VTTCInventoryReport.tsx`.

**Cách phát hiện lần sau:** Nếu thấy report/bug "border header biến mất khi cuộn" (đặc biệt Chrome), kiểm tra
ngay 2 điều kiện: (1) `<table>` có `border-collapse: collapse` (global CSS ở đây), (2) `<thead>`/`<th>` có
`position: sticky`. Có cả 2 → chính là bug này, không phải lỗi CSS cục bộ ở màn hình đó.

**Fix đã áp dụng (scoped, không sửa global):** Thay toàn bộ `border-l`/`border-t`/`border-{color}` trên
các `<TableHead>` trong `<thead>` sticky bằng `box-shadow` inset tương đương — box-shadow không bị bug
collapse-theo-grid nên vẽ đúng vị trí kể cả khi sticky. Ví dụ:
```
border-l                              → shadow-[inset_1px_0_0_0_hsl(var(--border))]
border-t border-t-blue-300 dark:...   → shadow-[inset_0_1px_0_0_theme(colors.blue.300)] dark:shadow-[...blue.800...]
```
Dùng `theme()` trong arbitrary value của Tailwind để tham chiếu màu theo token; `hsl(var(--border))` tự
đổi theo dark mode vì CSS variable đã được redefine trong `.dark` (không cần class `dark:` riêng cho màu
border mặc định).

**Why:** Sửa global `border-collapse: separate` sẽ ảnh hưởng MỌI bảng trong app (double-border, lệch 1px
ở nhiều màn khác) — rủi ro cao, cần QA rộng. Sửa scoped bằng box-shadow chỉ tác động đúng bảng đang bug,
an toàn hơn nhiều cho 1 lần sửa.

**How to apply:** Khi thêm sticky header mới hoặc gặp báo cáo tương tự, đừng cố sửa bằng cách thêm lại
`border-*` hay tăng `z-index` — không giải quyết được gốc vấn đề. Convert sang `box-shadow` inset theo
đúng pattern trên.
