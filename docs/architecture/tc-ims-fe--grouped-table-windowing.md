---
title: TableForm windowing cho chế độ nhóm theo path (groupByPath)
description: Kiến trúc windowing khi bảng có gom nhóm theo path — pathGroupKey, flattenVisiblePathOrder, hoisted collapse state
projects: [tc-ims-fe]
type: architecture
created: 2026-08-10
tags: [performance, table, windowing, grouping]
---

Khi `groupByPath` được set và số dòng > 120 (`ROW_WINDOW_MIN`), `TableForm` bật windowing
theo nhóm thay vì windowing phẳng. Chỉ mount ~40 dòng trong viewport + overscan; các dòng
ngoài cửa sổ được thay bằng `<tr aria-hidden>` spacer để giữ chiều cao cuộn.

**Thành phần cốt lõi** (`src/components/formed-combo/table/pathGrouping.ts`):
- `pathGroupKey(prefix, segment)` → `` `${prefix}/${segment.code}` `` — key định danh nhóm
  ổn định theo VỊ TRÍ trong cây, dùng chung cho cả state đóng/mở lẫn windowing. Nếu build key
  bằng cách khác ở hai nơi, windowing sẽ tính sai tập dòng hiển thị của nhóm đang đóng.
- `flattenVisiblePathOrder(rows, indices, accessor, depth, expectedPaths, sort, isCollapsed, prefix)`
  → `{ order: number[]; groupCount: number }` — thứ tự DFS các dòng THẬT SỰ được render (bỏ qua
  subtree của nhóm đang đóng, không đệ quy vào, không đếm header con).

**State ở TableForm:**
- `pathCollapsed: ReadonlyMap<string, boolean>` — trạng thái đóng/mở được hoist từ
  `useState` cục bộ trong từng `TableFormGroup` lên `TableForm` cha, key bằng `pathGroupKey`.
- Quy tắc skipHeader: nếu `getPathGroupLabel(segment, rows, depth)` trả về `null` (nhóm không
  render header), nhóm đó PHẢI luôn coi là đang mở — nếu không windowing sẽ ẩn dòng dưới một
  header không tồn tại.

**Bù sai số scrollTop:** header nhóm chiếm chiều cao cuộn nhưng không phải dòng dữ liệu, làm
phép chia `scrollTop / rowHeight` ước lượng lố vị trí dòng. Bù bằng
`winOverscanExtraRef.current = Math.min(pathVisible.groupCount, 80)` cộng thêm vào
`ROW_WINDOW_OVERSCAN`.

**An toàn khi append theo chunk lớn (250 dòng):** `pathWindowEnabled` được tính từ
`allRows.length` ngay trong CÙNG lượt render nhận chunk, nên chunk 250 dòng đầu tiên đổ vào
bảng đã vượt ngưỡng 120 thì windowing kích hoạt ngay lập tức — không có khoảnh khắc mount cả
250 dòng chưa được window.

**Why:** Bảng nhóm 300–700 dòng trước đây mount toàn bộ vì windowing bị tắt hoàn toàn khi có
`groupConfig` (lý do: index dòng không map thẳng tới vị trí cuộn khi có header xen kẽ, và state
đóng/mở nằm cục bộ trong từng nhóm nên không biết dòng nào đang thực sự hiển thị). 3 vấn đề gốc
này được giải quyết bằng: hoist state lên cha (biết chính xác dòng nào hiện), DFS
order theo cây có bỏ nhóm đóng (biết đúng vị trí cuộn), và bù overscan theo số header.

**How to apply:** Khi thêm màn hình mới có `groupByPath`, không cần làm gì thêm — windowing tự
kích hoạt khi rows > 120. Khi debug windowing sai (dòng biến mất/lặp khi cuộn), kiểm tra trước:
(1) `pathGroupKey` build key có nhất quán giữa nơi lưu collapse state và nơi tính
`flattenVisiblePathOrder` không, (2) nhóm không có header có bị coi nhầm là "đóng được" không.
