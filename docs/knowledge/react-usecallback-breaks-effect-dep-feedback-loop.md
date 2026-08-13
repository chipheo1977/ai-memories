---
title: useCallback ổn định identity để chặn feedback loop parent↔child qua useEffect deps
description: Callback prop truyền dạng arrow literal trong JSX + nằm trong deps của 1 useEffect ở con có thể tạo vòng lặp render vô hạn nếu effect đó (trực tiếp/gián tiếp) khiến cha re-render
projects: [all]
type: knowledge
created: 2026-08-13
tags: [react, useeffect, usecallback, feedback-loop, hooks]
---

**Pattern gây bug:** Component cha truyền callback xuống con dạng arrow literal ngay trong JSX:

```tsx
// Cha
const [state, setState] = useState(...);
<Child onDone={() => setState('x')} />
```

Component con dùng callback đó trong dependency array của 1 `useEffect`, và effect đó (trực tiếp
hoặc gián tiếp qua một callback KHÁC cũng nhận từ cha) gọi một `setState` ở CHÍNH cha đó với
**object/array literal mới mỗi lần** (dù giá trị bên trong không đổi):

```tsx
// Con
useEffect(() => {
  onSomethingElse({ a, b });  // setState(newLiteral) ở cha — luôn re-render vì Object.is(new, old) = false
  if (cond) onDone();         // gọi callback không ổn định
}, [onDone, onSomethingElse, /* ... */]);
```

**Vòng lặp:** effect chạy → gọi `onSomethingElse(newLiteral)` → cha re-render (vì literal mới luôn
thắng bail-out check của React) → dòng `onDone={() => setState('x')}` được eval lại → reference
MỚI → prop mới truyền xuống con → effect thấy `onDone` đổi trong deps → chạy lại → lặp vô hạn.
Mỗi vòng lặp, nếu effect còn làm việc khác (vd `reset()` một form, ghi đè state), tác dụng phụ đó
lặp lại liên tục — có thể biểu hiện thành "giá trị vừa sửa bị reset ngay lập tức", CPU cao, hoặc
(nếu React phát hiện) cảnh báo "Maximum update depth exceeded".

**Vì sao dễ lọt qua review:** từng callback riêng lẻ trông vô hại (`() => setState('x')` là pattern
rất phổ biến). Bug chỉ lộ ra khi xâu chuỗi: callback đó phải (a) nằm trong deps của effect ở con,
VÀ (b) effect đó phải có MỘT nhánh khác cũng gọi `setState` ở đúng cha đó với giá trị không stable
(object/array literal). Thiếu 1 trong 2 điều kiện thì không có vòng lặp — nên rất dễ chỉ thấy "child
effect gọi vài callback" mà không nhận ra chúng nối vòng qua nhau ở phía cha.

**Fix:** bọc mọi callback truyền xuống mà có khả năng nằm trong dependency array của effect ở con
bằng `useCallback`:

```tsx
const onDone = useCallback(() => setState('x'), []);
<Child onDone={onDone} />
```

Vì `setState` tự thân luôn stable (React đảm bảo), `useCallback([])` cho reference bất biến vĩnh
viễn — cắt đứt vòng lặp tại bước "cha re-render sinh callback mới".

**Why:** `useEffect`/`useMemo`/`useCallback` so sánh dependency bằng `Object.is`. Một arrow
function/object/array literal viết trực tiếp trong JSX luôn là instance MỚI mỗi lần component chứa
nó render — kể cả khi hành vi/giá trị logic bên trong giống hệt lần trước. Nếu literal đó vừa là
PROP truyền xuống con vừa nằm trong DEPS của effect con, và effect con lại có khả năng khiến cha
re-render (trực tiếp gọi setState ở cha, hoặc gọi callback khác cũng dẫn tới setState ở cha), thì
đây chính là điều kiện đủ cho một vòng lặp phản hồi (feedback loop) không có điểm dừng tự nhiên.

**How to apply:** Khi review một `useEffect` có callback prop (nhận từ cha) trong dependency array:
1. Kiểm tra callback đó ở phía cha có phải arrow literal / object literal viết trực tiếp trong JSX không.
2. Nếu có, kiểm tra effect (hoặc effect khác dùng chung callback đó) có gọi MỘT callback KHÁC cũng
   dẫn tới setState ở cha, với giá trị không ổn định reference (object/array literal) không.
3. Nếu cả 2 đúng → bọc callback bằng `useCallback([])` (hoặc deps tối thiểu cần thiết) ở phía cha.
4. Không cần bọc MỌI callback mặc định — chỉ cần khi callback đó thực sự nằm trong deps của 1
   `useEffect`/`useMemo`/`useCallback` ở con. Bọc tràn lan không cần thiết làm code khó đọc mà
   không thêm giá trị.

Ca cụ thể đã gặp: [Hydrate effect reset() lặp vô hạn trong HandoverStatusForm](../bugs/tc-ims-fe--hydrate-effect-gate-callback-feedback-loop.md) (tc-ims-fe).
