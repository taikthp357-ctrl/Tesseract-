# CF_OB V2.5 Release Notes & Documentation

## 1. Overview
Phiên bản **CF_OB V2.5** hoàn thành việc tái cấu trúc engine thực thi Outbound WT (**CFX Engine**), giải quyết triệt để lỗi false stop (điển hình như trường hợp WT `5042889690` do MON grid trả về trạng thái cache cũ), đảm bảo tương thích 64-bit toàn diện và vượt qua bộ kiểm thử nghiệm thu 22/22 kịch bản offline.

---

## 2. Key Architecture & Improvements

### 2.1 Deterministic CFX Engine (7 Modules)
- **`modCFX_State`**: RAM-truth state machine lưu trữ trạng thái WT và Run Meta (không phụ thuộc vào sheet cache).
- **`modCFX_SapAdapter`**: Lớp Adapter phân tách rõ ràng giữa chế độ `REPLAY` (giả lập bộ nhớ) và `LIVE` (SAP GUI). Đảm bảo tính bất biến: MON session != PRDO session.
- **`modCFX_Planner`**: Bộ lập kế hoạch determinism, gom nhóm theo document (`QtyGroupId`), tính toán Target/Current/Delta và khóa fingerprint.
- **`modCFX_Gates`**: Hệ thống 3 cổng kiểm soát an toàn tối giản:
  1. *Input Gate*: Kiểm tra tính đầy đủ và tính hợp lệ của mã bin/batch trước khi khóa plan. Chấp nhận các định dạng bin hợp lệ như `APFL-CC01-01-01`.
  2. *Action Gate*: Kiểm tra xung đột WT dựa trên RAM truth (thay thế logic lỗi cũ ở `CWR_BatchAssertNoActiveOutboundWT`).
  3. *Post-Verify Gate*: Xác thực OpenQty và PRDO target sau khi ghi.
- **`modCFX_Executor`**: Thực thi theo thứ tự pha cố định không thay đổi:
  `Cancel -> WT Settle -> Action Gate -> PRDO One-Save -> Post-Verify -> Create -> WHTask Verify -> CF Manual Handoff`.
- **`modCFX_Replay`**: Bộ suite kiểm thử nghiệm thu gồm 22 kịch bản (S01 đến S22).
- **`modCFX_Entry`**: Cầu nối Button #4 với Feature Flag an toàn.

### 2.2 64-bit Win32 API Audit
Tất cả các hàm Windows API trong 65 modules VBA (như `Sleep`, `FindWindowEx`, `GetWindowText`...) đều được khai báo `#If VBA7 Then Declare PtrSafe ... #Else Declare ... #End If`, hoạt động mượt mà trên cả Excel 32-bit và 64-bit.

### 2.3 Feature Flag & Safe Handoff
- **Defined Name**: `CFX_UseNewEngine` (Mặc định: `FALSE`).
  - Khi `FALSE`: Nút #4 (`CF_ExecutePlanToCFHandoff`) chạy trên luồng kế thừa đã được kiểm chứng.
  - Khi `TRUE`: Nút #4 chuyển sang engine mới `CFX_ExecutePlanToCFHandoff_Live()`.
- **Nguyên tắc bất biến**: Nút #4 luôn dừng lại ở bước `CF_USER_HANDOFF` (người dùng tự xác nhận trong SAP), tuyệt đối không tự động post/confirm CF ngoài quyền kiểm soát.

---

## 3. Replay Test Evidence (22/22 PASS)

Bộ 22 kịch bản nghiệm thu offline đã được chạy trực tiếp trên file Excel và ghi nhận kết quả:
**`REPLAY_ALL_PASS | TOTAL=22 | PASS=22 | FAIL=0`**

| Scenario | Result | Groups | Members | DocOpen | PRDOSaves | Creates | HardStops | Final WT States | Notes |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **S01** | **PASS** | 1 | 1 | 1 | 0 | 0 | 0 | 5040000001=ACTIVE | CF_ONLY / No writes |
| **S02** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000002=ACTIVE, 9000000002=ACTIVE | Increase qty / Delta-add 20 |
| **S03** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000003=ACTIVE, 9000000002=ACTIVE | Large delta-add 400 |
| **S04** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000042=ACTIVE, 9000000003=ACTIVE | Delta with prior split |
| **S05** | **PASS** | 1 | 3 | 1 | 0 | 0 | 0 | 3 WTs ACTIVE | Multi-WT 1 line aggregate 600 |
| **S06** | **PASS** | 1 | 1 | 1 | 1 | 0 | 0 | 5040000061=CANCELLED_ZERO | Cancel to zero / PRDO 0 |
| **S07** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000071=CANCELLED_ZERO, 9000000002=ACTIVE | Decrease qty / Cancel + create 400 |
| **S08** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5042889690=CANCELLED_ZERO, 9000000002=ACTIVE | **Fix regression WT 5042889690** |
| **S09** | **PASS** | 1 | 1 | 1 | 0 | 1 | 0 | 5040000091=CANCELLED_ZERO, 9000000002=ACTIVE | Bin change only |
| **S10** | **PASS** | 1 | 1 | 1 | 0 | 1 | 0 | 5040000010=CANCELLED_ZERO, 9000000002=ACTIVE | Exact bin APFL-CC01 accepted |
| **S11** | **PASS** | 1 | 1 | 1 | 0 | 0 | 0 | 5040000011=CANCELLED_PENDING | Old WT A unused / no stop |
| **S12** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000012=CANCELLED_PENDING, 9000000002=ACTIVE | Old WT A + repick |
| **S13** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5040000013=CANCELLED_PENDING, 9000000002=ACTIVE | Old WT A + repick new bin |
| **S14** | **PASS** | 1 | 4 | 1 | 1 | 2 | 0 | 4 members mixed | 4 members / 1 group / 1 PRDO save |
| **S15** | **PASS** | 1 | 2 | 1 | 1 | 2 | 0 | 2 members | Increase & Decrease / 1 PRDO save |
| **S16** | **PASS** | 1 | 2 | 1 | 0 | 2 | 0 | 2 members | 2 Creates distinct bins/batches |
| **S17** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5042889690=CANCELLED_PENDING, 9000000002=ACTIVE | Rerun after cancel (cancels=0) |
| **S18** | **PASS** | 1 | 1 | 1 | 0 | 1 | 0 | 5040000181=ACTIVE, 9000000002=ACTIVE | Rerun after save (saves=0) |
| **S19** | **PASS** | 1 | 1 | 1 | 0 | 0 | 0 | 5040000191=ACTIVE | Rerun after create (creates=0, no dup) |
| **S20** | **PASS** | 1 | 1 | 1 | 0 | 0 | 0 | 5040000020=DONE | Existing C terminal |
| **S21** | **PASS** | 1 | 1 | 1 | 1 | 1 | 0 | 5042889690=CANCELLED_PENDING, 9000000002=ACTIVE | Stale cache refreshed once (no stop) |
| **S22** | **PASS** | 1 | 1 | 0 | 0 | 0 | 1 | 5040000022=ACTIVE | Session role collision stop before write |

---

## 4. Package Contents
- `CF_OB_V2.5_WORKING.xlsm`: File Excel hoàn chỉnh đã nạp toàn bộ 65 module VBA và Defined Name `CFX_UseNewEngine`.
- `evidence/REPLAY_EVIDENCE.csv`: Báo cáo dữ liệu thực thi chi tiết 22 kịch bản nghiệm thu.
- `README_CF_OB_V2.5.md`: Bản mô tả và hướng dẫn chi tiết.
