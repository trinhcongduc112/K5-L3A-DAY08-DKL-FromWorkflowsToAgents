# Báo cáo cá nhân — Nguyễn Tuấn Kiệt

## 1. Thông tin thành viên

- Họ tên: Nguyễn Tuấn Kiệt
- MSSV: 2A202603417
- Email: 26ai.kietnt3@vinuni.edu.vn
- GitHub: [kite1143](https://github.com/kite1143)

## 2. Phần việc phụ trách

- Shape C `orchestrator`: orchestrator lập kế hoạch, investigator chỉ có tool đọc và nhận handoff allowlist; orchestrator là nơi duy nhất gọi host ghi refund.
- Bảo mật trace: test quét leak trên cả ba shape, gồm cả ticket độc chứa canary và số điện thoại.
- Tự kiểm trên bộ hidden `student_drop/` và chốt code ba shape.
- Báo cáo nhóm `report/team/group_report.md`.

## 3. Đóng góp cụ thể

- Commit `81785a0`: triển khai `src/lab8/shapes/orchestrator.py` và thêm `src/tests/test_orchestrator_shape.py` (3 test).
  - `get_claim_view` trước: claim không tồn tại → `failed`, thiếu `order_id` → `needs_info`; không đoán mã đơn từ ticket.
  - Gọi `complete(purpose="plan")`, rồi dựng handoff bằng `make_handoff` với `facts={}`, `request_id=REQ-<claim_id>`, `task="investigate_claim"`. Không đưa ticket, claim row hay header vào handoff.
  - `allowed_tools` cố định thứ tự `lookup_order` → `lookup_refund_history` → `compute_compensation` vì `run_readonly_investigators` chỉ tính quote từ order view đã có ở bước trước.
  - `lookup_order` không visible → `escalated`. Quote lấy từ `tools["compute_compensation"]["amount_vnd"]`; shape chỉ đọc, không tự tính lại tiền.
  - Orchestrator (không phải investigator) gọi `issue_refund` với quote đầy đủ; host tự trừ phần claim trước đã nhận. `crash_at` được truyền thẳng vào durability; `InjectedCrash` chỉ bắt để `trace.write()` rồi `raise` lại.
  - Handoff ghi vào trace bằng `safe_dump`; event `investigator` chỉ chứa order view đã allowlist và số liệu refund history, không chứa PII.
- Commit `0abdec4`: thêm `src/tests/test_trace_leak_sweep.py` (45 test).
  - 3 shape × 14 claim public, DB mới mỗi lần; quét `artifact_blob(result.as_dict())` cho `CANARY_SECRET_`, `CANARY_INTERNAL_`, `Authorization`, toàn bộ `phone`/`address`/`internal_note` trong `WORLD["orders"]` và `SECRET_ITEM_VANG`.
  - Thêm claim độc `C-EVIL` với ticket chứa `CANARY_SECRET_PARENT_EVIL` và `0901000107` để bắt shape nào ghi nguyên ticket.
  - Dùng `capsys` xác nhận cả ba shape không in gì ra stdout.
- Commit `1cc3453`: chạy tự kiểm hidden và commit nguyên `src/reports/hidden_selfcheck.json` (600 trial, 0 `error`, 0 `leak_suspect`).
- Commit `35de7f9`: viết `report/team/group_report.md` đủ 8 mục, số liệu lấy từ `src/reports/`.
- Commit `6ecdae5`: thêm `pytest-of-*/` vào `.gitignore` để không commit thư mục tạm do pytest sinh ra.

## 4. Bằng chứng

- Commit orchestrator: [81785a0](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/81785a0)
- Commit leak sweep: [0abdec4](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/0abdec4)
- Commit hidden self-check: [1cc3453](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/1cc3453)
- Commit báo cáo nhóm: [35de7f9](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/35de7f9)
- Test orchestrator:
  - `test_handoff_is_allowlisted_and_trace_holds_no_parent_secret`
  - `test_out_of_scope_order_escalates_without_exposing_pii`
  - `test_partially_refunded_order_quotes_full_and_ledger_gets_the_difference`
- Test leak: `test_public_claims_do_not_leak_into_trace_or_stdout` (42 tổ hợp) và `test_evil_ticket_never_reaches_the_trace` (3 tổ hợp).
- Run id dẫn chứng: `R-orchestrator-C-001-s11` (resolved), `R-orchestrator-C-005-s11` (escalated), `R-orchestrator-C-014-s11` (resolved, proposal 150000, ledger 120000).
- `src/reports/hidden_selfcheck.json`: mỗi shape 200 trial, tổng 600; `error_trials = 0`, `leak_suspect_trials = 0`; cả ba shape cùng phân bố `resolved 90 / denied 95 / escalated 10 / needs_info 5`.
- `src/reports/benchmark.json` (orchestrator, 70 trial): pass rate `1.0`, pass^5 `1.0`, seed min–max `1.0–1.0`, variance `0.0`, cost/success mô phỏng `47.73285714285714` VND.
- Tái lập: tôi chạy lại hidden benchmark và đọc trực tiếp JSON bằng script đếm `status`/`reason`, không chép số từ terminal.

## 5. Công cụ AI đã dùng

- OpenCode với agent DeepSeek V4.1 Flash, chạy trên máy tôi. Dùng để: đọc repo và contract, soạn `orchestrator.py`, viết hai file test, chạy benchmark hidden, soạn báo cáo nhóm.
- Agent thực hiện cả lệnh `git commit`/`git push` theo yêu cầu của tôi; tôi kiểm lại bằng `git log`, `git show --stat` và đọc lại diff trước khi xác nhận.
- Cách tự kiểm tra, mọi lệnh chạy trong `src/`:
  - `python -m pytest -q` → `127 passed`
  - `python -m pytest -q -m student -o addopts=` → `248 passed`
  - `python -m pytest -q -m student -o addopts= -k orchestrator` → `62 passed`
  - `python -m lab8.smoke` → `lab8.smoke pass`
  - `python -m lab8.benchmark --shapes workflow,single,orchestrator --cases student_drop/hidden_claims.jsonl --world student_drop/hidden_world_state.json --out reports/hidden_selfcheck.json` → 600 trial, 0 lỗi, 0 leak
  - `python -m lab8.check_submission` → `0 FAIL, 0 WARN`
- Kết quả test được đối chiếu bằng cách đọc JSON và SQLite, không chỉ tin vào output terminal.

## 6. Tự đánh giá

- Hoàn thành phần được giao: orchestrator đạt 70/70 trial public, 200/200 trial hidden không lỗi và không leak; test leak phủ cả ba shape và ticket độc.
- Phần làm tốt nhất là giữ ranh giới handoff: investigator chỉ thấy allowlist, còn quyền ghi và quyết định tiền nằm ở code/host.
- Hạn chế: bộ hidden không có nhãn nên chỉ tự kiểm được `status`, trace contract và leak, chưa có pass rate hidden. Kết quả dùng MockLLM, latency là `unavailable`, cost là giá mô phỏng; mẫu public chỉ 14 claim × 5 seed nên không có khoảng tin cậy.
- Chưa có test timeout/cancel cho tool hoặc model call bị treo.
