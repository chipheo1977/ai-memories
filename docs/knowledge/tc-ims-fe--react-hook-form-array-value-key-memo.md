---
title: Value-key thay vì array reference trong dependency của useMemo với React Hook Form
description: RHF useWatch trả về array reference mới mỗi render — dùng joined-string key để memo ổn định
projects: [tc-ims-fe]
type: knowledge
created: 2026-08-10
tags: [performance, react-hook-form, memoization]
---

RHF `useWatch` trả về array/object reference MỚI ở mỗi lần render, kể cả khi giá trị bên trong
không đổi. Nếu đưa thẳng array đó (vd `watchedDetails`, `watchedCategories`) vào dependency
array của `useMemo`/`useCallback`, memo sẽ re-run ở MỌI phím gõ — kể cả gõ vào ô không liên
quan tới giá trị đang watch.

**Pattern thay thế:** build một string key bằng cách join giá trị thực sự cần theo dõi, rồi
đưa key đó vào deps thay vì mảng gốc:

```ts
const categoriesKey = watchedCategories.join('\x01');
const detailsPathsKey = watchedDetails
  .map(d => Array.isArray(d.constructionPath) ? d.constructionPath.map(s => s.code).join('/') : '')
  .join('\x01');

const groupConfig = useMemo(() => ({ ... }), [categoriesKey, detailsPathsKey, ...]);
// eslint-disable-next-line react-hooks/exhaustive-deps
```

Dùng `\x01` (separator hiếm gặp trong dữ liệu thật) để tránh đụng key giả khi nối chuỗi.

**Why:** Khi memo phụ thuộc array reference, mọi re-render (kể cả do gõ phím ở ô khác) sẽ trigger
re-compute các phép tính đắt phía sau (re-bucket nhóm, re-flatten thứ tự windowing...). Value-key
tách biệt "identity thay đổi" khỏi "giá trị thực sự thay đổi".

**How to apply:** Áp dụng bất cứ khi nào một memo/callback tốn kém phụ thuộc vào mảng lấy từ
`useWatch`/`watch()` của RHF. Cần thêm `eslint-disable-next-line react-hooks/exhaustive-deps` vì
lint không hiểu quan hệ giữa key và mảng gốc.
Xem thêm kiến trúc liên quan: [grouped table windowing](../architecture/tc-ims-fe--grouped-table-windowing.md).
