---
title: Radix Popper components tự bơm CSS var --radix-*-trigger-width/height
description: Cách đồng bộ width/height của Popover/Select/DropdownMenu/... content với trigger qua CSS custom property Radix tự set
projects: [all]
type: knowledge
created: 2026-08-11
tags: [radix-ui, tailwind, css-variables, popover, dropdown, select]
---

Mọi component Radix xây trên nền Popper (định vị content theo 1 anchor/trigger, portal ra ngoài
DOM tree) đều tự đo kích thước trigger bằng `ResizeObserver` và set thành CSS custom property ngay
trên `style` attribute của chính content node — cập nhật động khi trigger đổi kích thước, không cần
re-render.

Quy ước đặt tên: `--radix-{component}-trigger-{width|height}` và
`--radix-{component}-content-available-{width|height}` (phần trống còn lại tới mép viewport).

| Component | Biến trigger | Biến available |
|---|---|---|
| `Popover` | `--radix-popover-trigger-width/height` | `--radix-popover-content-available-width/height` |
| `Select` | `--radix-select-trigger-width/height` | `--radix-select-content-available-width/height` |
| `DropdownMenu` | `--radix-dropdown-menu-trigger-width/height` | `--radix-dropdown-menu-content-available-width/height` |
| `ContextMenu` | `--radix-context-menu-trigger-width/height` | `--radix-context-menu-content-available-width/height` |
| `HoverCard` | `--radix-hover-card-trigger-width/height` | `--radix-hover-card-content-available-width/height` |
| `Tooltip` | `--radix-tooltip-trigger-width/height` | `--radix-tooltip-content-available-width/height` |
| `Menubar` | `--radix-menubar-trigger-width/height` | `--radix-menubar-content-available-width/height` |
| `NavigationMenu` | — | `--radix-navigation-menu-viewport-width/height` |

**Không có** ở các component không phải Popper-based / không có khái niệm trigger để đo: `Dialog`,
`AlertDialog` (modal giữa màn hình), `Accordion`, `Tabs`, `Toast`, `Switch`, `Checkbox`,
`RadioGroup`, `Slider`, `Progress`, `ScrollArea`.

Cú pháp Tailwind: khi giá trị trong `[]` bắt đầu bằng `--` (tên biến CSS), Tailwind tự bọc
`var()` — `w-[--radix-popover-trigger-width]` sinh ra `width: var(--radix-popover-trigger-width)`,
tương đương viết tường minh `w-[var(--radix-popover-trigger-width)]`.

**Why:** Content bị Radix portal ra ngoài, không phải con trực tiếp của trigger trong DOM, nên CSS
thuần (`width: 100%` theo parent) không tự đồng bộ được kích thước với trigger — phải qua kênh biến
CSS này.

**How to apply:** Muốn dropdown/popover luôn ≥ độ rộng trigger nhưng vẫn co giãn theo nội dung, dùng
pattern 3 class: `min-w-[--radix-{component}-trigger-width] w-auto max-w-[...]`. Chỉ dùng
`w-[--radix-...]` một mình (không kèm `min-w-`) nếu muốn ép cứng đúng bằng trigger, không co giãn.
Xem ví dụ bug liên quan: [SelectCombo width class xung đột](../bugs/tc-ims-fe--selectcombo-conflicting-width-class-bug.md).
