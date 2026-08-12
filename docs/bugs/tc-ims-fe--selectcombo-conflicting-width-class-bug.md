---
title: SelectCombo dropdown không theo width trigger do 2 class w- xung đột
description: className có cả w-[--radix-popover-trigger-width] và w-auto trong cùng string → w-auto âm thầm thắng
projects: [tc-ims-fe]
type: bug
created: 2026-08-11
tags: [tailwind, css, selectcombo, popover]
---

**Triệu chứng:** `PopoverContent` của `SelectCombo` không tuân theo độ rộng của trigger như kỳ vọng
— dù đã có `w-[--radix-popover-trigger-width]` trong className.

**Nguyên nhân gốc:** `src/components/inputed-combo/SelectCombo.tsx:202` có:

```
className={cn('w-[--radix-popover-trigger-width] w-auto max-w-[500px] p-0', popoverClassName)}
```

Hai class cùng set property `width` (`w-[--radix-popover-trigger-width]` và `w-auto`) xuất hiện
trong cùng 1 chuỗi. Đây là bẫy dễ nhầm: nhìn vào source code, trực giác nghĩ `w-auto` (viết sau)
"đè" `w-[--radix-...]` theo thứ tự xuất hiện trong string — và đúng là vậy, nhưng **không phải vì
thứ tự trong JSX string quyết định**, mà vì cả 2 sinh ra 1 class CSS riêng biệt trong stylesheet
Tailwind, và class nào được **generate sau trong file CSS cuối** (theo thứ tự lần đầu Tailwind gặp
class đó khi build, không nhất thiết khớp thứ tự trong JSX) sẽ thắng do cùng specificity. Kết quả:
`w-auto` thắng, `min-width` theo trigger không có tác dụng.

Toàn bộ các component khác trong repo (`SelectTree.tsx`, `MultipleSelectTree.tsx`,
`MultipleSelectCombo.tsx`, và hàng chục `popoverClassName=` ở call site) đều dùng đúng pattern
`min-w-[--radix-popover-trigger-width] w-auto max-w-[...]` — không xung đột vì `min-w-` và `w-` là
2 property CSS khác nhau. `SelectCombo.tsx` là chỗ DUY NHẤT lệch pattern.

**Cách phát hiện lần sau:** grep `w-\[--radix-.*-trigger-width\] w-auto` (thiếu `min-` ở trước) —
nếu thấy `w-[...]` và `w-auto` cùng xuất hiện trong 1 className string mà không phải
`min-w-[...]`, đó là bug tương tự.

**Fix đã áp dụng:** đổi `w-[--radix-popover-trigger-width]` thành
`min-w-[--radix-popover-trigger-width]`.

**Why:** Muốn dropdown ≥ độ rộng trigger nhưng vẫn co giãn theo nội dung dài hơn (tới trần
`max-w-[500px]`) thì bắt buộc phải tách 2 property khác nhau (`min-width` + `width: auto`), không
thể dùng chung 1 property `width` cho cả 2 mục đích.

**How to apply:** Khi thấy/viết pattern "trigger-width + auto + max-width" cho bất kỳ Popper-based
component nào (xem [Radix trigger size CSS vars](../knowledge/radix-popper-trigger-size-css-vars.md)),
luôn dùng `min-w-[--radix-...]`, không bao giờ dùng trần `w-[--radix-...]` kèm `w-auto`.
