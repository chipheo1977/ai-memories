---
title: navigate(-1) sau guard trạng thái rò rỉ sang màn/phiếu không liên quan
description: Chặn form Edit khi phiếu không phải Nháp bằng navigate(-1) (history.back) thay vì điều hướng thẳng tới route theo id — user ép URL trực tiếp thì bị đưa về bất kỳ đâu, kể cả form Edit của phiếu khác
projects: [tc-ems-fe]
type: bug
created: 2026-09-11
tags: [react-router, navigate, history-back, guard, inbound, outbound, ims-4142]
---

**Triệu chứng (IMS-4142):** Form Cập nhật phiếu xuất kho (`OutboundItemForm.tsx`, tương tự
`InboundItemForm.tsx`) chặn đúng việc mở form khi phiếu KHÔNG ở trạng thái Nháp (Chờ duyệt/Đã
duyệt/Từ chối/Đã hủy) — không còn cụm nút Lưu, form không render. Nhưng **điểm điều hướng sau
khi chặn thì sai**: đứng ở màn "Hóa đơn đầu vào" rồi gõ thẳng URL
`/ems/transactions/outbound/detail/{id}/edit` (id của phiếu không phải Nháp) → bị đưa quay lại
"Hóa đơn đầu vào" (màn hoàn toàn không liên quan tới phiếu vừa mở). Tệ hơn: nếu đang đứng ở form
`/edit` của **phiếu khác**, ép URL sang `/edit` của phiếu A (không phải Nháp) → bị đưa quay lại
form `/edit` của phiếu khác đó — rò rỉ điều hướng sang bản ghi không liên quan.

**Nguyên nhân gốc:** guard dùng `navigate(-1)` (tương đương `history.back()`):

```ts
// useOutboundFormQuery.ts / useInboundFormQuery.ts (trước fix)
if (detail.status !== TransactionStatus.DRAFT) {
  navigate(-1);
  return null;
}
```

`navigate(-1)` điều hướng theo **lịch sử session của trình duyệt**, hoàn toàn không phụ thuộc
`editId` đang mở. Khi user vào bằng click trong app thì "back" tình cờ đúng (vì lịch sử liền
trước thường là Chi tiết/Danh sách của chính phiếu đó). Nhưng khi user **gõ thẳng URL** (ép URL,
bookmark, mở từ link ngoài, hoặc điều hướng chuỗi nhiều phiếu liên tiếp qua URL bar) thì lịch sử
liền trước có thể là bất kỳ màn nào — bug chỉ lộ ra trong đúng kịch bản này, nên dễ lọt qua test
chỉ thao tác bằng click.

**Fix:** điều hướng thẳng tới route Chi tiết theo đúng `editId`, dùng `replace: true` để không để
lại URL `/edit` bị chặn trong lịch sử (tránh vòng lặp khi user bấm back lần nữa):

```ts
navigate(APP_ROUTES.TRANSACTIONS.OUTBOUND_ITEM_DETAIL(editId), { replace: true });
```

Pattern đúng này đã có sẵn ở `InternalTransferForm.tsx:480` (`navigate(APP_ROUTES.TRANSFERS.INTERNAL_TRANSFER_DETAIL(editId!), { replace: true })`) — cùng guard "chỉ sửa khi DRAFT", nhưng
KHÔNG bị bug vì không dùng `navigate(-1)`.

**Cách phát hiện lần sau:** grep `navigate(-1)` trong toàn bộ form Update (`git grep "navigate(-1)"`).
Bất kỳ chỗ nào dùng `navigate(-1)`/`history.back()` làm phản ứng cho một **guard phụ thuộc dữ
liệu record cụ thể** (trạng thái, quyền, tồn tại...) đều là ứng viên bug này — guard đó phải biết
chính xác record nào đang bị chặn nên phải điều hướng theo id, không được dựa vào lịch sử trình
duyệt. `navigate(-1)` chỉ an toàn cho hành động thuần UI như nút "Hủy"/"Quay lại" do user chủ động
bấm trong đúng phiên đó, không phải cho guard chạy tự động lúc load trang.

**Phạm vi đã sửa:** `useOutboundFormQuery.ts` (issue gốc IMS-4142) và
`useInboundFormQuery.ts` (cùng bug, phát hiện thêm và sửa theo yêu cầu trong cùng phiên).
