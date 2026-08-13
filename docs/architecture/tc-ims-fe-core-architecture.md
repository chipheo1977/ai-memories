# Kiến trúc bổ sung cho `tc-ims-fe`: Core / Infrastructure / React Glue

**Ngày:** 2026-08-06
**Trạng thái:** Đề xuất đã thống nhất — áp dụng dần từng màn

---

## 1. Ý tưởng

Chia code thành **3 vòng**, kèm **quy tắc chiều phụ thuộc** (vòng trong không được import vòng ngoài):

```
┌─────────────────────────────────────────────┐
│ INFRASTRUCTURE — axios/apiClient, services, │
│   localStorage, SignalR, Date.now,          │
│   Math.random, import.meta.env, browser API │
│  ┌───────────────────────────────────────┐  │
│  │ REACT GLUE — components, hooks,       │  │
│  │   context, React Query wiring, JSX    │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │ CORE — file .ts thuần, KHÔNG    │  │  │
│  │  │ import react/axios/browser:     │  │  │
│  │  │ tính toán, validate, mapper,    │  │  │
│  │  │ luật trạng thái, build payload  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Định nghĩa vận hành được (phép thử, không phải khẩu hiệu):**

- **Core** = file `.ts` không có dòng `import react` nào, chạy được trong Vitest `environment: node` — không cần jsdom, không cần mock axios, không cần provider. Gồm: công thức tính (tiền, VAT, khối lượng, quy đổi), mapper/normalize response, luật trạng thái nghiệp vụ (`canApprove`, `canCancel`...), validate, build payload, Zod schema.
- **Infrastructure** = mọi code giao tiếp với bên ngoài process: gọi API, localStorage, SignalR, `Date.now()`, `Math.random()`, `import.meta.env`, `window`/DOM.
- **React glue** = vòng giữa: component, hook, context... Được import core; **cấm import infrastructure trực tiếp**.
- **Quy tắc chiều phụ thuộc**: core không import gì từ 2 vòng ngoài; glue import core; infrastructure đứng ngoài cùng, được service/hook gọi tới.
- **Page = điểm ghép (composition root)** — nơi duy nhất 3 vòng gặp nhau. Page không chứa logic: nó gọi hook glue (hook nối service infra + mapper/rules core) rồi đổ kết quả vào JSX. Mỗi màn có đúng 1 điểm ghép;

## 2. Lợi ích

1. **Test được phần đáng test nhất**: logic tiền/số lượng/trạng thái thành pure function có unit test chạy nhanh, không cần render component.
2. **Portability**: muốn mang nghiệp vụ một màn sang codebase khác → copy nguyên folder `core/<nghiệp-vụ>/` (zero dependency ngoài TypeScript), viết lại service + UI là xong. 
3. **File page ngắn lại, dễ đọc**: page là nơi ghép (hook glue + JSX).
5. **Cưỡng chế**: ESLint + Vitest node environment tự bắt vi phạm.

## 3. ⚠️ Nguyên tắc áp dụng — BỔ SUNG, KHÔNG ĐẬP

Đây là kiến trúc **bổ sung**. Tuyệt đối **không** refactor hàng loạt codebase hiện tại:

- **Phạm vi áp dụng:**
  - ✅ **Feature mới** — áp dụng đầy đủ ngay từ đầu.
  - ✅ **Sửa feature đơn giản** mà mình **nắm được 100% nghiệp vụ** — migrate màn đó theo công thức 5 bước.
  - ❌ Feature phức tạp chưa nắm hết nghiệp vụ — KHÔNG migrate, sửa theo cách hiện tại; chờ tới khi hiểu đủ.

## 4. Cấu trúc folder

```
src/
├── core/                              # ★ MỚI — VÒNG TRONG, chỉ file .ts
│   │                                  #   Cấm import: react, axios, @tanstack/*, sonner,
│   │                                  #   @/services, @/components, @/hooks, @/contexts,
│   │                                  #   localStorage, window, document, Date.now, Math.random
│   └── inbound/                       #   1 folder = 1 bounded context (inbound, invoice, handover...)
│       ├── types.ts                   #   Kiểu dữ liệu nghiệp vụ (tự định nghĩa, không mượn từ UI)
│       ├── mapper.ts                  #   Normalize API response → domain type; build rows
│       ├── calculations.ts            #   Công thức tính (tiền, khối lượng, quy đổi...)
│       ├── rules.ts                   #   Luật trạng thái: deriveActions(entity, permissions) → cờ can*
│       └── *.test.ts                  #   Chạy Vitest environment: node
│
├── services/                          # INFRASTRUCTURE — giữ nguyên, chỉ bổ sung method còn thiếu
│   └── features/transaction/InboundOrderApiService.ts
│
└── pages/transactions/
    ├── InboundItemDetail.tsx.tsx           # ĐIỂM GHÉP — chỉ ghép hook + JSX, ~250 dòng
    └── inbound-detail/                     # ★ MỚI — REACT GLUE riêng của màn
        ├── useInboundDetailQuery.ts        #   useQuery: queryFn = service (infra)
        ├── useInboundDetailActions.ts      #   mutations + confirm + toast + navigate
        └── columns.tsx                     #   Cột bảng JSX — render gọi core/calculations
```

**Nguyên tắc đặt chỗ:** core theo **nghiệp vụ** (`core/inbound/` dùng chung cho Detail + Form + List của phiếu nhập), glue theo **màn** (`pages/transactions/inbound-detail/` chỉ phục vụ màn đó).

## 5. Ví dụ mẫu: màn Chi tiết phiếu nhập kho

Hiện trạng: `src/pages/transactions/InboundItemDetail.tsx` — 834 dòng, trộn cả 3 vòng.

Page sau migrate:

```tsx
export default function InboundItemDetail() {
  const { id } = useParams();
  const { transaction, rows, attachments, isLoading } = useInboundDetailQuery(id);
  const actions = useInboundDetailActions(id, transaction);
  // deriveInboundActions() (core) gọi trong hook trên — trả cờ + handler đã gói confirm/toast
  return <QueryBoundary ...>{/* JSX thuần, đọc từ 2 hook */}</QueryBoundary>;
}
```

## 6. Công thức migrate 5 bước (lặp lại cho từng màn)

1. **Tách core trước** — tạo `core/<nghiệp-vụ>/`, kéo pure logic xuống (mapper, công thức, luật trạng thái) + viết test. Bước giá trị nhất, rủi ro thấp nhất (chỉ di chuyển hàm).
2. **Đẩy fetch về service** — xóa mọi import `imsApiInstance`/axios khỏi page.
3. **Gói glue** — query + mutations + confirm/toast vào 1–2 hook trong folder con của màn.
4. **Page chỉ ghép** — hook + JSX.
5. **Shim file cũ** — utils cũ giữ lại dạng `export * from '@/core/<nghiệp-vụ>/calculations'` để màn chưa migrate không vỡ.

## 7. Cưỡng chế bằng máy

**ESLint** (`eslint.config.js` — thêm block riêng cho vòng core):

```js
{
  files: ['src/core/**'],
  rules: {
    'no-restricted-imports': ['error', { patterns: [
      { group: ['react', 'react-*', '@tanstack/*', 'axios', 'sonner'],
        message: 'core không được import framework/infra' },
      { group: ['@/services/*', '@/components/*', '@/hooks/*', '@/contexts/*', '@/pages/*'],
        message: 'core chỉ được import trong core' },
    ]}],
    'no-restricted-globals': ['error', 'localStorage', 'window', 'document', 'navigator'],
  },
}
```

**Vitest**: test của `src/core/**` chạy `environment: 'node'` — file core lén import React là test tự vỡ. Đây là phép thử "chạy trong mọi bối cảnh" được tự động hóa.

> Lưu ý: rule đặt mức `error`, không phải `warn` — bài học từ rule `EMPTY_DISPLAY` hiện tại (để `warn` nên còn 54 vi phạm).

## 8. Checklist khi review PR có áp dụng kiến trúc này

- [ ] File trong `src/core/` không import react/axios/services/components/hooks/contexts
- [ ] Core không gọi `Date.now()`, `Math.random()`, `localStorage`, `window`
- [ ] Kiểu dữ liệu trong core tự định nghĩa, không import type từ `@/components`
- [ ] Mapper/calculations/rules có file test đi kèm, chạy environment node
- [ ] Page không import `imsApiInstance`/axios trực tiếp — fetch qua service
- [ ] Unwrap envelope chỉ xảy ra một chỗ (service hoặc mapper), không rải trong component
- [ ] File utils cũ (nếu di dời) có re-export shim
