---
title: EF Core migration trong tc-iam-be — là gì, vì sao cần, khi nào cần (kèm bài học Phase 1 IMS_RPT_DT)
description: Giải thích migration (schema + seed dữ liệu) qua ví dụ thật đăng ký chức năng IMS_RPT_DT; cách chạy an toàn trên DB dùng chung và các bẫy đã gặp
projects: [tc-iam-be]
type: knowledge
created: 2026-10-05
tags: [extra-knowledge, migration, ef-core, sql-server, iam, permission]
---

## Tóm tắt
- Migration là **file có phiên bản, nằm trong git, mô tả một thay đổi của database** và cách hoàn tác nó. EF Core chạy các migration theo thứ tự và ghi nhớ cái nào đã chạy.
- Ở tc-iam-be, migration không chỉ đổi cấu trúc bảng mà còn là cách **đưa danh mục chức năng/phân quyền vào DB** (seed dữ liệu). Thêm báo cáo mới = thêm một migration.
- Pipeline deploy dev **không tự chạy migration**; phải áp dụng chủ động (theo chỉ đạo của leader IAM BE, chạy từ máy local).

## Migration là gì
Một migration trong repo gồm:
- `<timestamp>_<Tên>.cs` — class có hai hàm: `Up` (áp dụng thay đổi) và `Down` (hoàn tác).
- `<timestamp>_<Tên>.Designer.cs` — ảnh chụp mô hình dữ liệu tại thời điểm đó (EF dùng để so sánh).
- `ApplicationDbContextModelSnapshot.cs` — ảnh chụp mô hình mới nhất. Migration chỉ seed dữ liệu thì snapshot **không đổi**.

EF ghi lại migration đã chạy trong bảng `iam.__EFMigrationsHistory` (cột `MigrationId`). Nhờ đó chạy lại `database update` sẽ bỏ qua những migration đã có.

Có hai loại, cả hai cùng cơ chế:
| Loại | Ví dụ | Nội dung |
|---|---|---|
| Schema | `..._InitPermissionSchema` | Tạo/sửa bảng, cột, khóa ngoại |
| Dữ liệu (seed/backfill) | `..._SeedImsRpt16` | `migrationBuilder.Sql(...)` chèn dòng vào bảng đã có |

## Tại sao cần migration
Không dùng migration thì mỗi người tự chạy SQL tay vào từng DB. Hậu quả điển hình:
- DB local, dev, UAT, production **lệch nhau** mà không ai biết lệch ở đâu.
- Không có lịch sử: không rõ ai đổi gì, khi nào, vì sao; không review được trong PR.
- Không có đường lui chuẩn khi lỗi.
- Môi trường mới (máy dev mới, DB thử) không dựng lại được từ đầu.

Với migration: thay đổi được **review như code**, chạy **theo thứ tự**, có **Down**, và dựng lại được DB từ trống (đã làm đúng như vậy với DB thử `iam_test`).

### Vì sao Phase 1 bắt buộc có migration
Khai báo hằng `IMS_RPT_DT` trong code BE/FE là **chưa đủ**. Danh mục chức năng nằm trong bảng `iam.functions`, và các bảng cấp quyền (`mapping_group_function_permission`, `mapping_user_function_permission`) có **khóa ngoại** trỏ tới `functions.code`. Nếu chưa có dòng `IMS_RPT_DT` trong `functions` thì:
- không thể cấp quyền cho nhóm/user (vi phạm khóa ngoại);
- API `GET /functions` không trả chức năng này nên màn Nhóm quyền của `tcims` không hiện dòng mới.

Migration này làm 3 việc: thêm chức năng, khai báo nó chỉ hỗ trợ quyền `view`, và copy quyền từ `IMS_RPT_CP:view` sang để người đang xem Báo cáo chi phí tự xem được Báo cáo doanh thu.

## Khi nào cần / không cần migration
| Tình huống | Cần migration? |
|---|---|
| Thêm/sửa bảng, cột, khóa ngoại, index | Có |
| Thêm/đổi **danh mục** nằm trong DB (chức năng, quyền, menu nhóm) | Có |
| Backfill dữ liệu cho dữ liệu cũ (vd copy quyền) | Có |
| Chỉ đổi code logic, không đụng DB | Không |
| Đổi cấu hình `appsettings`, biến môi trường | Không |
| Sửa dữ liệu thật của một người dùng cụ thể (một lần) | Thường không — dùng thao tác nghiệp vụ/script có kiểm soát |

Quy ước của repo (ghi trong migration khởi tạo): **mỗi thay đổi catalog = một migration MỚI; không sửa migration cũ.** Mã chức năng cũng **không bao giờ đổi** (đổi tên qua cột `name`, ẩn qua `is_active`) vì quyền đã cấp được lưu theo `function_code`.

## Ví dụ thật: Phase 1 (`20261002093104_IMS_IAM_SeedImsRpt16`)
- Dùng `MERGE ... WHEN NOT MATCHED` để chèn chức năng và `INSERT ... WHERE NOT EXISTS` để copy quyền ⇒ **idempotent** (chạy lặp không tạo dòng trùng).
- `Down` xóa theo thứ tự `user mapping → group mapping → function-permission → functions` để không vi phạm khóa ngoại.
- Mẫu tham chiếu: `20260807081548_IMS_IAM_SeedImsRpt04`.

### Cách kiểm tra đã dùng (và nên dùng lại)
1. Dựng DB trống trên SQL Server Express local, áp dụng migration tới ngay trước migration mới.
2. Nạp dữ liệu thử gồm cả trường hợp **đối chứng** (nhóm/user không có `IMS_RPT_CP` không được nhận quyền mới).
3. Chạy migration, đếm các bảng; chạy lại phần SQL của `Up` để kiểm tra idempotent (kỳ vọng `0 rows affected`); chạy `Down` và đếm về 0.

## Cách chạy trên môi trường dev (và rủi ro)
- `Database:AutoMigrate` mặc định `false` và workflow deploy chỉ chạy một script ngoài repo ⇒ **không có migration nào tự chạy khi deploy**.
- Xem trước SQL mà không đụng DB: `dotnet ef migrations script <từ> <đến> --output file.sql`.
- Áp dụng: `dotnet ef database update <tên migration đích>` hoặc chạy file script trong SSMS. Nên chỉ định **migration đích** để lỡ có migration chờ của người khác thì không bị áp dụng theo.
- Connection string mặc định trong `appsettings.json` trỏ vào **DB dev/UAT dùng chung**. Ghi đè bằng biến môi trường `ConnectionStrings__ConnectionStrings` (hai dấu gạch dưới) thay vì sửa file.

## Bẫy đã gặp
1. **`migrations script` không thay đổi DB nào.** Nó chỉ sinh file `.sql`. Báo "Build succeeded" không có nghĩa migration đã chạy. Muốn thấy dữ liệu phải thực sự áp dụng (script hoặc `database update`).
2. **Biến môi trường dính trong terminal.** Đặt biến trỏ `iam_test` rồi quên: lệnh sau lại chạy vào DB đó (hoặc ngược lại, mở terminal khác thì quay về DB dùng chung). Luôn chạy `dotnet ef dbcontext info` để xem `Database name` / `Data source` **trước khi** `database update`.
3. **Nhận ra nhầm DB nhờ dữ liệu.** API local từng trả 9 nhóm thật với hàng chục thành viên thay vì 3 nhóm thử ⇒ nó đang nối DB dùng chung. Dấu hiệu "dữ liệu quá thật" là cảnh báo.
4. **`sqlcmd` cần cờ `-I`** khi ghi vào bảng có filtered index (lỗi `QUOTED_IDENTIFIER`), và `-f 65001` để không hỏng dấu tiếng Việt. SSMS/Azure Data Studio mặc định đã ổn.
5. **Đổi mã sau khi đã mở PR.** Mã `IMS_RPT_16` đổi thành `IMS_RPT_DT` sau khi có PR ⇒ phải amend + force-push cả 3 repo. Chốt tên mã trước khi viết migration. Lưu ý: **tên file migration không đổi theo** (`...SeedImsRpt16`) vì sau khi migration đã áp dụng ở đâu đó, `MigrationId` đã nằm trong lịch sử của DB — đổi tên tức là EF coi như migration khác.
6. **Quyền trễ tới ~10 phút** sau khi áp dụng do cache Redis của quyền người dùng; không phải lỗi migration.
7. **Số "quyền" hiển thị không bằng số dòng trong DB.** `permissionCount` cộng thêm 1 cho ô "Quản trị" khi một chức năng được cấp đủ mọi quyền nó hỗ trợ (ô này là quyền ảo, không lưu DB). Báo cáo chỉ có `view` nên 2 chức năng báo cáo = 4.
8. **`dotnet ef` bản 10 chạy được với EF Core 8.0.14** của dự án (đã dùng `migrations add`/`script`/`database update` thành công).
9. **`Down` cũng xóa quyền `IMS_RPT_DT` đã gán tay sau deploy.** Chỉ rollback khi thật sự cần.

## Tự kiểm tra
- Vì sao khai báo hằng trong code chưa đủ để cấp quyền được? *(thiếu dòng trong `iam.functions`, khóa ngoại chặn.)*
- Idempotent là gì và migration này đạt bằng cách nào? *(chạy nhiều lần cho cùng kết quả; `MERGE`/`NOT EXISTS`.)*
- Vì sao `Down` xóa bảng mapping trước bảng `functions`? *(khóa ngoại.)*
- Làm sao biết `database update` sắp chạy vào DB nào? *(`dotnet ef dbcontext info`.)*
- Có nên sửa một migration đã áp dụng lên dev không? *(không; tạo migration mới.)*

## Liên quan
- [Core/Infra/React-glue architecture](../architecture/tc-ims-fe-core-architecture.md) — kiến trúc FE của cùng dự án.

**Why:** Migration là điểm mà lỗi nhỏ (nhầm DB, nhầm tên, quên thứ tự xóa) gây hậu quả trên môi trường dùng chung, và rất nhiều bước kiểm tra an toàn không suy ra được từ code.
**How to apply:** Trước khi viết hoặc chạy migration IAM, đọc các mục "Cách chạy" và "Bẫy đã gặp"; luôn kiểm tra DB đích bằng `dbcontext info` và chỉ định migration đích.
