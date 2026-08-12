---
title: Tree-select components trong tc-ims-fe — bản đồ + API parentSelect
description: SelectTree (chọn 1) + MultipleSelectTree (chọn nhiều) là 2 generic tree-select dùng chung; các component khác là wrapper/đặc thù dùng chúng bên trong
projects: [tc-ims-fe]
type: architecture
created: 2026-08-11
tags: [tree-select, select-tree, multiple-select-tree, item-group-tree-select, parent-select]
---

## Bản đồ component

Chỉ có **2 generic tree-select** thực sự dùng lại được, cả hai ở `src/components/inputed-combo/`:

- `SelectTree.tsx` — chọn 1 (`value: string | null`)
- `MultipleSelectTree.tsx` — chọn nhiều (`value: string[]`)

Các component khác liên quan "tree select" trong repo **không phải** generic, mà là wrapper/đặc
thù nghiệp vụ dùng 1 trong 2 component trên bên trong:

- `ItemGroupTreeSelect.tsx` (`src/components/app/`) — wrapper fetch `useItemGroupTree` + map sang
  `TreeOption`, render `SelectTree` bên trong. KHÔNG tự vẽ UI riêng (trước 2026-08-11 nó tự
  implement UI bằng `<button>` thô, đã refactor để dùng chung `SelectTree`).
- `TreeFilterSelect` (`src/components/reports/reportFilterPrimitives.tsx`) — **không** dùng
  `SelectTree`/`MultipleSelectTree`, tự implement UI riêng vì model dữ liệu khác hẳn: mỗi node có
  `codes: string[]` (nhiều mã / 1 node), còn `TreeOption` của 2 component generic chỉ có 1
  `value: string` / node. Không migrate được nếu không đổi model dữ liệu ở BE/call site — đã cân
  nhắc và quyết định KHÔNG migrate (xem phần "Quyết định không migrate" bên dưới).

## `SelectTree` dùng `VciComboTrigger` (đổi 2026-08-11)

`SelectTree` trước đây tự vẽ trigger bằng `<Button>` thô; đã đổi sang dùng chung
`VciComboTrigger` (`src/components/app/VciComboTrigger.tsx`) — cùng 1 trigger với
`MultipleSelectTree`, `MultipleSelectCombo`, `SelectCombo`. Đây là thay đổi **diện rộng**: `SelectTree`
có 25+ call site trên toàn app (reports, master-data, transactions, expenses...).

Height tùy ý mà các call site truyền (`h-8`, `h-[32px]`, mặc định `h-[36px]`) map lossless sang 2
mode cố định của `VciComboTrigger` (`cell` → `h-8`/32px, mặc định → `h-9`/36px) vì **32px và 36px
trùng khớp chính xác** với giá trị pixel của `h-8`/`h-9` trong thang spacing mặc định của Tailwind —
đã kiểm tra toàn bộ giá trị `height=` thực tế đang dùng trong repo trước khi đổi, không có giá trị
lệch (vd `h-7`/28px) nào gây mất chính xác.

**Why:** Đồng bộ UI/UX trigger toàn app, tận dụng style filled/invalid/pending/badge có sẵn của
`VciComboTrigger` thay vì mỗi component tự vẽ lại.

## API `parentSelect` — hành vi click node cha (2026-08-11)

Cả 2 component có prop `parentSelect` để cấu hình hành vi khi click 1 node CÓ CON:

- `SelectTree`: `parentSelect?: 'none' | 'self'` (mặc định `'self'`)
- `MultipleSelectTree`: `parentSelect?: ParentSelectMode` = `'none' | 'self' | 'cascade'` (mặc định
  `'self'`; prop cũ `cascadeParent?: boolean` vẫn hoạt động, đã đánh `@deprecated`, tự map
  `true` → `'cascade'`)

| mode | Click node cha | Checkbox/check cha |
|---|---|---|
| `'self'` (default) | toggle chính giá trị của cha như node thường | bình thường |
| `'cascade'` (chỉ Multiple) | chọn/bỏ TOÀN BỘ lá hậu duệ; cha KHÔNG bao giờ vào `value` | tri-state dẫn xuất từ lá; dòng "chọn tất cả" hiện nếu có `selectAllLabel` |
| `'none'` | no-op, không chọn được | **vẫn hiện** nhưng disabled (tái dùng đúng style của `TreeOption.disabled` có sẵn — không phải style riêng); ở Multiple: dòng "chọn tất cả" **ẩn** |

Quyết định quan trọng: **default vẫn là `'self'`, KHÔNG phải `'cascade'`** dù yêu cầu gốc ghi cascade
"sẽ là mặc định" — lý do: đổi default sẽ phá hành vi của 8 call site `MultipleSelectTree` đang ăn
default `cascadeParent=false` ngầm (không truyền prop). Giữ default an toàn, cascade là opt-in.

Migrate kèm theo: bỏ cờ `disabled: true` set thủ công cho mọi node có con ở
`useConstructionItems.ts` (`getConstructionTree`) — đây là cách "làm `parentSelect='none'` bằng
tay ở tầng dữ liệu" trước khi có prop. Điểm rủi ro nhất khi migrate: **`InvoiceLineDetailReport.tsx`
có 1 `MultipleSelectTree` dùng cây từ `getConstructionTree` nhưng KHÔNG truyền `cascadeParent`** —
hành vi "cha không chọn được" của nó đến 100% từ cờ `disabled` cũ, không phải từ prop nào. Bỏ cờ ở
nguồn mà quên truyền `parentSelect="none"` ở đúng call site này sẽ khiến node cha đột nhiên chọn
được và lọt vào `value` → sai kết quả lọc báo cáo mà không có lỗi biên dịch nào báo hiệu.

Chi tiết đầy đủ quá trình quyết định (Q&A, ma trận hành vi, checklist từng call site đã sửa) nằm ở
`/root/workspace/check-list/checklist-tree-select-parent-mode.md` trong repo `tc-ims-fe` (không
phải ai-memory) — đọc file đó nếu cần audit lại toàn bộ quá trình hoặc mở rộng thêm mode mới.

**Why đáng nhớ:** việc gắn hành vi UI (cha chọn được hay không) vào 1 field dữ liệu (`disabled`)
thay vì 1 prop cấu hình tường minh tạo ra coupling ẩn — sửa ở "nguồn" (hook build tree) có thể âm
thầm đổi hành vi ở "ngọn" (màn hình dùng tree đó) mà không qua review rõ ràng. `parentSelect` tường
minh hơn nhưng vẫn cần audit kỹ mọi consumer của cùng 1 hàm build-tree trước khi đổi field `disabled`
ở nguồn.

**How to apply:** Trước khi sửa 1 hàm build `TreeOption[]` dùng chung (vd thêm/bớt field `disabled`),
grep TOÀN BỘ call site của hàm đó và xác nhận từng nơi consume field này theo cách nào — component
generic đọc trực tiếp, hay call site tự map lại (auto strip field) — trước khi kết luận đổi ở nguồn
là an toàn.
