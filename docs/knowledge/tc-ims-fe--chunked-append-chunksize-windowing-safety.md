---
title: ChunkSize lớn cho chunked-append chỉ an toàn khi có windowing
description: Tăng chunkSize của useChunkedAppend lên 250 chỉ rẻ khi bảng đang windowing, nếu không sẽ chậm hơn
projects: [tc-ims-fe]
type: knowledge
created: 2026-08-10
tags: [performance, table, windowing]
---

`useChunkedAppend` chia việc thêm nhiều dòng thành từng đợt (mặc định `chunkSize: 60`) để tránh
block UI. Tăng `chunkSize` lên 250 CHỈ rẻ hơn khi bảng đang bật windowing (row hoặc path-group);
nếu bảng KHÔNG windowing, chunk 250 dòng nghĩa là 250 dòng DOM mount cùng lúc mỗi đợt — đắt hơn
chunk 60.

Cơ chế an toàn ở `TableForm`:
```ts
bulkChunkSizeRef.current = rowWindowEnabled || pathWindowEnabled ? 250 : 50;
```
`columnBuilder` đọc `ctx.chunkSizeRef?.current ?? 50` cho bulk-fill; các màn hình gọi
`useChunkedAppend(append, { chunkSize: 250 })` trực tiếp cho luồng thêm mới — an toàn vì
`pathWindowEnabled`/`rowWindowEnabled` được tính từ `allRows.length` ngay trong CÙNG lượt render
nhận chunk, nên chunk 250 dòng đầu tiên đổ vào bảng rỗng đã kích hoạt windowing ngay khi vượt
ngưỡng `ROW_WINDOW_MIN` (120 dòng) — không có khoảnh khắc mount 250 dòng chưa được window.

**Why:** Nếu hardcode chunkSize 250 cho mọi màn hình bất kể windowing, các màn dùng
legacy `groupBy` (không phải `groupByPath`, chưa được windowing — xem
[grouped table windowing](../architecture/tc-ims-fe--grouped-table-windowing.md)) sẽ chậm hơn
trước khi tối ưu.

**How to apply:** Trước khi tăng chunkSize ở màn hình mới, xác nhận màn đó có bật windowing
(row thường >120 dòng, hoặc groupByPath >120 dòng). Nếu dùng `groupBy` legacy (không phải
`groupByPath`), giữ chunkSize mặc định 60 cho tới khi legacy mode được windowing.
