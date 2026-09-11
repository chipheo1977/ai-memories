# Index — mỗi kiến thức một dòng

Format: `- [Tiêu đề](../docs/<folder>/<file>.md) — mô tả 1 dòng (projects)`

## Knowledge

- [RHF array value-key memo](../docs/knowledge/tc-ims-fe--react-hook-form-array-value-key-memo.md) — dùng joined-string key thay array ref trong deps để tránh re-run mỗi phím gõ (tc-ims-fe)
- [ChunkSize + windowing safety](../docs/knowledge/tc-ims-fe--chunked-append-chunksize-windowing-safety.md) — tăng chunkSize append chỉ rẻ khi bảng đang windowing (tc-ims-fe)
- [Radix trigger size CSS vars](../docs/knowledge/radix-popper-trigger-size-css-vars.md) — --radix-*-trigger-width/height tự bơm bởi mọi Popper-based component, cách dùng với Tailwind (all)
- [useCallback chặn feedback loop qua effect deps](../docs/knowledge/react-usecallback-breaks-effect-dep-feedback-loop.md) — callback prop arrow literal nằm trong deps effect con + effect gọi setState literal ở cha → vòng lặp render vô hạn (all)

## Decisions

(chưa có)

## Bugs

- [Border thead biến mất khi cuộn](../docs/bugs/tc-ims-fe--sticky-thead-border-collapse-bug.md) — border-collapse:collapse + sticky thead làm border biến mất lúc cuộn, fix bằng box-shadow inset (tc-ims-fe)
- [SelectCombo width class xung đột](../docs/bugs/tc-ims-fe--selectcombo-conflicting-width-class-bug.md) — w-[--radix-trigger-width] + w-auto cùng string, w-auto âm thầm thắng, fix bằng min-w- (tc-ims-fe)
- [Hydrate effect reset() lặp vô hạn](../docs/bugs/tc-ims-fe--hydrate-effect-gate-callback-feedback-loop.md) — HandoverStatusForm: onAccessAllowed arrow literal trong deps effect hydrate → vòng lặp reset() xoá edit user ở Update/Clone (tc-ims-fe)
- [navigate(-1) rò rỉ sang màn không liên quan](../docs/bugs/tc-ems-fe--navigate-back-leaks-unrelated-screen.md) — guard trạng thái Nháp dùng history.back() thay vì điều hướng theo id, lộ khi ép URL trực tiếp (IB/OB, IMS-4142) (tc-ems-fe)

## Architecture

- [Core/Infra/React-glue + quy ước JSX](../docs/architecture/tc-ims-fe-core-architecture.md) — 3 vòng phụ thuộc, công thức migrate 5 bước; §9: không dùng `?:`/`&&` trong JSX, tách component + lookup map, kiểm tra generic đã có trước khi tạo mới (tc-ims-fe, tc-ems-fe)
- [Grouped table windowing](../docs/architecture/tc-ims-fe--grouped-table-windowing.md) — windowing cho TableForm khi groupByPath, pathGroupKey, hoisted collapse state (tc-ims-fe)
- [Tree-select components](../docs/architecture/tc-ims-fe--tree-select-components.md) — SelectTree/MultipleSelectTree là 2 generic, API parentSelect (none/self/cascade), rủi ro migrate getConstructionTree (tc-ims-fe)

## Tasks

(chưa có)
