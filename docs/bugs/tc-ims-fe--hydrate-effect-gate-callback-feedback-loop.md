---
title: Hydrate effect reset() lặp vô hạn do gate-callback inline arrow trong deps
description: onAccessAllowed truyền dạng arrow literal → mỗi lần effect hydrate gọi setState khác khiến parent re-render, sinh callback mới, effect hydrate tự chạy lại — reset() xoá edit của user liên tục
projects: [tc-ims-fe]
type: bug
created: 2026-08-13
tags: [react, useeffect, usecallback, react-hook-form, feedback-loop, handoverstatus]
---

**Triệu chứng:** Màn Update/Clone "Phiếu hiện trạng bàn giao"
(`src/pages/transfers/handoverStatus/HandoverStatusForm.tsx`) — bảng "Đánh giá tình trạng vật
tư" (`items`) không sửa được: mỗi lần đổi giá trị 1 cell (giá đề nghị, giá thu hồi, SL tình
trạng...), giá trị lập tức bị **reset về giá trị gốc đã tải từ server**. Chỉ xảy ra ở Update/Clone,
KHÔNG xảy ra ở Create.

**Nguyên nhân gốc — vòng lặp phản hồi (feedback loop) 2 chiều giữa parent/child:**

Component ngoài `HandoverStatusForm` (default export) render component trong
`HandoverStatusFormContent` và truyền 2 callback "gate" dạng arrow literal viết trực tiếp trong
JSX:

```tsx
onRecordForbidden={() => setAccessGate('forbidden')}
onAccessAllowed={() => setAccessGate('allowed')}
```

`HandoverStatusFormContent` có 1 `useEffect` "hydrate" (nạp `loadedReport` từ API vào form qua
`reset({...})`, dòng ~569-644) với dependency array gồm cả `onAccessAllowed` (dùng ở nhánh
clone-mode) VÀ luôn gọi `onHeaderMetaChange({ code, status })` (= `setHeaderMeta` của outer) mỗi
lần effect chạy — object literal MỚI mỗi lần, dù giá trị bên trong giống hệt lần trước.

Chuỗi lặp:
1. `loadedReport` tải xong → effect hydrate chạy → gọi `onHeaderMetaChange({code, status})` (object mới).
2. `setHeaderMeta(newLiteral)` không bao giờ `Object.is`-equal state cũ (object mới) → outer LUÔN re-render, không bail-out.
3. Outer re-render → dòng `onAccessAllowed={() => setAccessGate('allowed')}` được eval lại → **function reference mới**.
4. Reference mới này truyền xuống làm prop cho `HandoverStatusFormContent`.
5. Effect hydrate có `onAccessAllowed` trong deps → thấy đổi → **effect chạy lại**.
6. Quay lại bước 1 → lặp vô hạn.

Mỗi vòng lặp, effect hydrate gọi lại `reset({ ..., items: hsrFormRowsFromHSRDetail(loadedReport) })`
— tức nạp lại đúng snapshot GỐC từ server. User gõ giá trị mới → `setValue` ghi thành công trong
khoảnh khắc đó → nhưng vòng lặp kế tiếp `reset()` đè lại → giá trị vừa gõ biến mất gần như ngay lập tức.

**Tại sao chỉ Update/Clone, không Create:** `accessGate` khởi tạo
`editId || cloneFromId ? 'pending' : 'allowed'`. Ở Create, `accessGate` đã là `'allowed'` sẵn →
`setAccessGate('allowed')` gọi lại là no-op (state y hệt) → React bail-out, không re-render, không
có vòng lặp. Ở Update/Clone, cú chuyển `'pending' → 'allowed'` (khi `onAccessAllowed()` được gọi
lần đầu, từ trong `queryFn` của `loadedReport` hoặc trong chính effect hydrate ở nhánh clone) là
state change THẬT — chính là cú re-render "mồi" khởi động vòng lặp ở bước 3.

**Fix đã áp dụng:** bọc cả 2 callback bằng `useCallback(..., [])` ở outer component thay vì arrow
literal trong JSX:

```tsx
const onRecordForbidden = useCallback(() => setAccessGate('forbidden'), []);
const onAccessAllowed = useCallback(() => setAccessGate('allowed'), []);
// ...
<HandoverStatusFormContent onRecordForbidden={onRecordForbidden} onAccessAllowed={onAccessAllowed} .../>
```

Vì `setAccessGate` tự thân đã ổn định (setState function của React luôn stable), `useCallback([])`
cho ra reference bất biến vĩnh viễn → phá đứt bước 3-4-5, effect hydrate không còn tự kích hoạt lại.

**Cách phát hiện lần sau:** grep pattern `on[A-Z]\w*=\{\(\) =>` (arrow literal truyền thẳng vào
prop) ở các component cha có `useState` bên trong, RỒI kiểm tra prop đó có nằm trong dependency
array của 1 `useEffect` ở component con hay không — nếu effect đó (trực tiếp hoặc gián tiếp qua 1
callback khác) gọi 1 setState nhận **object/array literal mới mỗi lần** ở đúng component cha đó, đó
là ứng viên vòng lặp phản hồi.

Xem thêm: [useCallback + useEffect deps chống feedback loop](../knowledge/react-usecallback-breaks-effect-dep-feedback-loop.md).
