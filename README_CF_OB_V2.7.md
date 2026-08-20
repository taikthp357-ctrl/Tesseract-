# CF_OB V2.7 Release Notes & Deliverables Documentation

## 1. Overview
Phiên bản **CF_OB V2.7** hoàn tất việc giải quyết triệt để lỗi phân bổ số lượng nội bộ group (Member Allocation Engine), xử lý dứt điểm bài toán `3500 + 4500 -> 0 + 8000`, bịt lỗ hổng bảo mật luồng FAST (Seam 2 Plan Lock Assertion), vô hiệu hóa cơ chế tái dùng tab PRDO làm MON (chống navigation thrash), bổ sung hàm `CF_ResetUI` cho nút LÀM MỚI và vượt qua **100% bộ kiểm thử nghiệm thu Offline**.

---

## 2. Key Architecture & Improvements

### 2.1 Member Allocation Engine (`modCFGroupAllocator`)
- **Tách biệt 2 hợp đồng độc lập:**
  - `CONTRACT_A` (PRDO Aggregate): Kiểm soát tổng số lượng trên PRDO document (`sum(MemberQty) = GroupTarget`).
  - `CONTRACT_B` (Member Allocation): Kiểm soát từng member (`RequiredCreateQty = max(TargetQty - EffectiveQtyAfterPlannedCancelReplace, 0)`).
- **Quy tắc bất biến:** `GROUP_DELTA = 0` **KHÁC** `ACTION_DELTA = 0`.
  - Case `3500 + 4500 -> 0 + 8000`:
    - Member A (Cancel 3500) $\rightarrow$ `RequiredCreate = 0`.
    - Member B (Keep 4500) $\rightarrow$ `RequiredCreate = 3500`.
    - Group Complete chỉ đạt được khi cả A và B cùng PASS.
- **Quy tắc Blank khác 0:** $3500 + 4500 \rightarrow \text{blank} + 8000 \implies$ Member A giữ nguyên 3500, Member B tạo 3500, Tổng PRDO = 11500.

### 2.2 Execution Context & Bịt lỗ hổng Seam 2 (`modCFExecContext`)
- **Khóa Read-Only tuyệt đối cho nút `CHECK`:** `SAP_WRITE = 0` ngăn chặn mọi hành vi ghi nhầm SAP khi người dùng bấm CHECK.
- **Seam 2 Guard:** Luồng FAST (`CWR_RunFastToCFCore`) bắt buộc gọi `CWR_AssertQueuePlanLock` trước khi gửi lệnh Create/Save (`lock@1300 < create@1318`).

### 2.3 Single MON Broker Hardening
- Vô hiệu hóa vĩnh viễn hàm cũ tái dùng tab PRDO làm MON (`CWR_ReuseVerifiedPRDOTabForMON` fail-closed với mã lỗi `STOP_LEGACY_MON_REUSE_DISABLED`).
- Mọi thao tác đọc MON đều đi qua broker duy nhất `CWR_GetMONSession`.

### 2.4 Giao diện 4 nút chuẩn hóa & Hàm LÀM MỚI (`modCFResetUI`)
- **4 nút độc lập:**
  1. `[1] TẢI WH` $\rightarrow$ `CF_LoadWH`
  2. `[2] CHECK` $\rightarrow$ `CF_CheckAndLoadPosting` (Read-only, `SAP_WRITE = 0`)
  3. `[3] XỬ LÝ` $\rightarrow$ `CF_ExecutePlanToCFHandoff` (Điểm ghi SAP duy nhất theo Plan đã khóa)
  4. `[4] LÀM MỚI` $\rightarrow$ `CF_ResetUI` (`CFResetUI_SelfTest = PASS`, vô hiệu hóa plan lock, dọn sạch UI, bảo toàn 100% dữ liệu lịch sử log).

---

## 3. Báo cáo kiểm thử Offline (100% PASS)

Toàn bộ các bộ kiểm thử tự động offline trên Excel COM đều đạt kết quả tuyệt đối:
- `CFAlloc_SelfTest`: **`PASS`**
- `TestBlankVsZero`: **`PASS`**
- `CFReg_RunAll` (11 Scenarios Allocation): **`REG_ALL_PASS | TOTAL=11 | PASS=11 | FAIL=0`**
- `CF_FastModuleRegression`: **`PASS`**
- `CF_Phase3_RunUnitTests`: **`11/11 PASS (SAP Write = 0)`**
- `CFResetUI_SelfTest`: **`PASS`**
- `CFExecCtx_SelfTest`: **`PASS`**
- `TestMonHardening`: **`PASS (STOP_LEGACY_MON_REUSE_DISABLED_CONFIRMED)`**

---

## 4. Package Contents
- `260820_CF_OB_V2.7.xlsm`: File Excel V2.7 hoàn chỉnh chứa 67 module VBA đã kiểm thử nghiệm thu.
- `DELIVERABLES/`: Thư mục 9 tài liệu audit và báo cáo chuyên sâu:
  1. `V2.7_BASELINE_AUDIT.md`
  2. `V2.7_CHANGELOG.md`
  3. `V2.7_FINAL_AUDIT.md`
  4. `V2.7_ITERATION_LOG.md`
  5. `V2.7_MIGRATION_AUDIT.md`
  6. `V2.7_REGRESSION_CATALOG.md`
  7. `V2.7_REGRESSION_REPORT.md`
  8. `V2.7_SOURCE_CONFLICT_LEDGER.md`
  9. `V2.7_UI_DECISIONS.md`
- `bootstrap_offline_test_report.log`: Báo cáo log chạy kiểm thử toàn diện.
