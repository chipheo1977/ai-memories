---
name: playwright-e2e-testing
description: Dùng Playwright MCP mở browser thật để thực thi các TC có Assign = 🤖 AI trong file test_execution_<FEATURE_ID>.md. Ghi kết quả PASSED/FAILED và chụp screenshot. Trigger khi user nói "đchạy test", "thực thi testcase", "run playwright", "test TC AI", "mở browser test <feature>".
version: 2.0.0
---

# Playwright E2E Execution Skill

Skill giúp tester thực thi test case tự động qua Playwright MCP. Agent mở browser thật, chạy từng TC, ghi kết quả luôn vào file testcase.

---

## Các bước thực hiện

### Bước 1 — Đọc file testcase

Đọc file `test_execution_<FEATURE_ID>.md` do tester cung cấp. Lấy:

- URL môi trường test, login URL, test screen URL (từ Section 3)
- Tài khoản đăng nhập theo từng role (từ bảng credentials Section 3)
- Danh sách TC có `Assign = 🤖 AI` trong Execution Matrix (Section 2)

Bỏ qua hoàn toàn các TC có `Assign = 👤 Tester`.

### Bước 2 — Xác nhận với tester

Trước khi mở browser, liệt kê ngắn gọn:

```
Tìm thấy 48 TC (🤖 AI) trong test_execution_IMS_EQ_03.md
URL test: https://ims-staging.example.com/equipments/create
Tài khoản: admin@ims.dev (Admin), staff@ims.dev (Staff)

Chạy toàn bộ hay chỉ một nhóm?
  [1] Toàn bộ 48 TC
  [2] Chỉ Thêm mới (32 TC)
  [3] Chỉ một TC cụ thể (nhập TC ID)
```

Chờ tester xác nhận rồi mới tiến hành.

### Bước 3 — Đăng nhập

Dùng Playwright MCP đăng nhập lần lượt từng role có trong Section 3:

1. Mở Login URL
2. Điền username + password
3. Click nút đăng nhập
4. Xác nhận đã vào được trang chính (dashboard)

Sau khi đăng nhập xong, giữ nguyên session cho các TC cùng role — không đăng nhập lại giữa các TC.

### Bước 4 — Thực thi từng TC

Với mỗi TC `🤖 AI`, thực hiện theo thứ tự:

1. Điều hướng về Test Screen URL
2. Đọc cột **Các bước** và thực hiện từng action trên browser
3. Kiểm tra theo cột **Kết quả mong muốn**
4. Chụp screenshot
5. Ghi kết quả

Nếu TC thất bại → chụp ảnh lỗi, ghi error message, chuyển sang TC tiếp theo (không dừng).

Nếu TC không thể chạy (thiếu seed data, element không tồn tại) → đánh dấu BLOCKED, ghi lý do.

### Bước 5 — Cập nhật file testcase

Sau khi chạy xong, cập nhật trực tiếp vào `test_execution_<FEATURE_ID>.md`:

- Cột **Trạng thái**: `⬜ UNTESTED` → `🟩 PASSED` / `🟥 FAILED` / `🟨 BLOCKED`
- Cột **Kết quả thực tế / Bug ID / Notes**: điền error message hoặc link screenshot nếu FAILED
- **Section 1 Dashboard**: cập nhật số PASSED / FAILED / BLOCKED / UNTESTED

### Bước 6 — Báo cáo kết quả

In tóm tắt sau khi chạy xong ví dụ như:

```
===================================
  KẾT QUẢ: test_execution_IMS_EQ_03
===================================
  🟩 PASSED  : 41
  🟥 FAILED  :  5  → cần tạo bug report
  🟨 BLOCKED :  2  → thiếu seed data
  ⬜ Bỏ qua  : 17  (👤 Tester — test thủ công)

  TC FAILED:
    TC_EQ_03_003 — toast không hiện sau 15s
    TC_EQ_03_019 — button Lưu vẫn enable khi form rỗng

  File đã cập nhật: test_execution_IMS_EQ_03.md
===================================
```

---

## Cách Playwright MCP tương tác với trang

Playwright MCP có 2 cách hiểu trang web:

**Snapshot (DOM / Accessibility Tree) — ưu tiên dùng**

Playwright đọc cây element của trang, trả về danh sách có cấu trúc:

```
- button "Thêm mới" [enabled]
- textbox "Mã thiết bị" [required]
- combobox "Loại thiết bị"
- row "EQ-001 | Máy bơm | Active"
```

Agent đọc text này, biết chính xác element nào đang có → click/fill theo tên. Không cần nhìn ảnh.
Dùng snapshot cho: tìm button, fill input, click option, verify text, kiểm tra disabled/enabled.

**Screenshot — chỉ dùng khi cần thiết**

Playwright chụp ảnh → agent nhìn ảnh để xác nhận. Chậm hơn, dùng khi:
- Cần verify màu badge (đỏ/xanh/vàng) vì màu không có trong DOM
- Lưu bằng chứng TC PASSED/FAILED
- Khi snapshot không đủ thông tin để xác nhận kết quả

**Nguyên tắc:** Luôn dùng snapshot để tương tác. Chỉ chụp ảnh để lưu bằng chứng hoặc verify visual.

---

## Cách thực hiện các loại bước phổ biến

**Điều hướng**
- Mở URL: `playwright_navigate`
- Nhấn nút: `playwright_click` với tên button đọc từ snapshot

**Nhập liệu**
- Input text / number / email: `playwright_fill` theo label hoặc placeholder (đọc từ snapshot)
- Dropdown thường: click combobox → đọc snapshot xem options → click option
- Dropdown có tìm kiếm: fill keyword → chờ snapshot cập nhật list → click option
- Date picker: fill trực tiếp định dạng dd/MM/yyyy vào input
- File upload: `playwright_upload`
- Toggle / Checkbox / Radio: `playwright_click`

**Kiểm tra kết quả — ưu tiên đọc snapshot**
- Toast / thông báo thành công: đọc snapshot tìm text thông báo
- Lỗi validation: đọc snapshot tìm error message dưới field
- Chuyển trang: kiểm tra URL hiện tại
- Record trong danh sách: đọc snapshot tìm row chứa mã/tên
- Button ẩn / disabled: đọc snapshot kiểm tra trạng thái element
- Màu badge: chụp screenshot rồi xem ảnh

**Các TC đặc thù**
- **RBAC / Permission**: logout → login lại bằng role khác → đọc snapshot kiểm tra button có/không
- **Trim khoảng trắng**: nhập ` GIÁ TRỊ ` → blur → đọc snapshot kiểm tra value đã trim
- **Vượt maxLength**: nhập chuỗi dài → đọc snapshot kiểm tra value length bị chặn
- **Double-click protection**: click Lưu 2 lần nhanh → vào list đọc snapshot đếm row khớp mã

**Screenshot — chỉ dùng để lưu bằng chứng**
- TC FAILED: chụp ngay khi lỗi, lưu vào `test_cases_<ID>_screenshoot/`
- TC PASSED: chụp màn hình kết quả cuối làm bằng chứng
- Verify màu / layout visual: chụp và xem ảnh

**Test data**
- Khi cần tạo record mới: dùng mã có timestamp tránh trùng, ví dụ `EQ-AUTO-20240615143022`
- Cuối buổi test, hỏi tester có muốn xóa các record test đã tạo không
