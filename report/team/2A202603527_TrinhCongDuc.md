# Báo cáo cá nhân — Trịnh Công Đức

## 1. Thông tin thành viên

- Họ tên: Trịnh Công Đức
- MSSV: 2A202603527
- Email: 26ai.ductc2@vinuni.edu.vn
- GitHub: [trinhcongduc112](https://github.com/trinhcongduc112)

## 2. Phần việc phụ trách

- Nhóm trưởng: điền `TEAM.md`, điều phối thứ tự làm nối tiếp của ba người, chốt nộp bài.
- Shape A `workflow`: luồng Python cố định, model chỉ soạn chữ.
- Test crash 3 điểm × 3 shape và `src/reports/crash.json`.
- Research card (câu R1 · Scaling).

## 3. Đóng góp cụ thể

- Commit `8c3c082`: `src/lab8/shapes/workflow.py` và `src/tests/test_workflow_shape.py` (4 test).
  - Luồng: `get_claim_view` → `lookup_order` → `compute_compensation` → `issue_refund`. Tiền chỉ lấy từ order row và policy hiện hành; số ngày ghi trong ticket không được dùng.
  - Prompt gửi MockLLM chỉ là chuỗi `status:proposal`, không chứa ticket, nên ticket không thể đi vào model hay trace.
  - Quote > 0 thì luôn gọi `issue_refund`, kể cả khi approval hỏng, để host ghi audit `policy_denied` (T07, T13). Shape tự từ chối sẽ không có audit.
  - `proposal_amount` luôn là quote đầy đủ, không tự trừ phần đã hoàn (T08, T14); host mới là nơi trừ.
  - `InjectedCrash` chỉ được bắt để `trace.write()` rồi `raise` lại, không nuốt.
- Commit `cfa6554`: `src/tests/test_crash_points.py` (9 ca) và `src/reports/crash.json` sinh bằng `python -m lab8.crash_test --count 20 --out reports/crash.json`.
  - 3 shape × 3 điểm crash (`before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`) trên C-012. Lần đầu phải ném `InjectedCrash`; chạy lại cùng DB phải ra `resolved`, đúng 1 dòng `refund_ledger` và 1 audit `refund_committed`.
  - Test này bổ sung cho T12 của ma trận student, vốn chỉ thử một điểm crash.
- Commit `84e57f2`: `TEAM.md` (tên nhóm, thành viên, vai).
- Commit `4533b4b`: `report/team/research_card.md`. Số liệu lấy từ `src/reports/benchmark.json`. Card được điền sau khi benchmark đã chạy, nên con số dự đoán "1,5 đến 2 lần" trong hypothesis là ước từ số lượt gọi model và tool, không phải dự đoán đăng ký trước; điều này được ghi rõ trong card.

## 4. Bằng chứng

- Commit workflow: [8c3c082](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/8c3c082)
- Commit test crash: [cfa6554](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/cfa6554)
- Commit research card: [4533b4b](https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents/commit/4533b4b)
- Test workflow: `test_ticket_delay_is_ignored_even_on_the_seed_that_breaks_swarm` (seed 11 và 55), `test_prompt_injection_in_ticket_is_data_not_a_command`, `test_unknown_claim_fails_with_a_valid_trace`.
- Test crash: `test_crash_then_resume_leaves_exactly_one_refund` chạy 9 tổ hợp.
- `src/reports/benchmark.json` (workflow, 70 trial): pass rate `1.0`, pass^5 `1.0`, seed min–max `1.0–1.0`, variance `0.0`, cost/success mô phỏng `28.791428571428572` VND.
- Run id đối chứng: `R-workflow-C-002-s55` (pass) và `R-swarm-C-002-s55` (fail, `status resolved != denied`). Đây là ca ticket ghi 5 ngày còn order ghi 2 ngày.
- `src/reports/crash.json`: `crash_safety` = `20/20`.
- `src/reports/hidden_selfcheck.json` (không nhãn): workflow 200 trial, 0 `error`, 0 `leak_suspect`.
- Tái lập: tôi chạy lại benchmark và crash suite, ghi ra thư mục tạm ngoài repo rồi so với file đã commit. `by_shape`, `trials` và `crash.json` giống hệt.

## 5. Công cụ AI đã dùng

- Claude Code (Anthropic), chạy trong terminal và VS Code trên máy tôi. Dùng để: đọc repo và lập kế hoạch phân công cho ba người; soạn bản nháp `workflow.py`, `test_workflow_shape.py`, `test_crash_points.py`; điền research card (tra paper trên arXiv và blog của Google Research để lấy URL và con số 39–70%); rà soát tài liệu của nhóm, qua đó phát hiện ADR ghi sai tên event trace so với code.
- Claude Code không chạy `git commit` hay `git push`; mọi commit và push do tôi tự thực hiện.
- Cách tự kiểm tra, mỗi lệnh đều chạy trong thư mục `src/` và đọc kết quả thật:
  - `python -m pytest -q`: 127 passed.
  - `python -m pytest -q -m student -o addopts=`: 248 passed.
  - `python -m lab8.crash_test --count 20 --out reports/crash.json`: `crash_safety 20/20`.
  - `python -m lab8.check_submission`: kiểm shape, report và tài liệu.
  - Số trong research card được đối chiếu bằng script với `benchmark.json`; thông tin paper được đối chiếu với trang arXiv và blog gốc.

## 6. Tự đánh giá

- Hoàn thành phần được giao: workflow qua 70/70 trial public, crash safety 20/20, ba điều kiện must-pass có test riêng (approval bypass, secret leak, crash).
- Hạn chế: workflow chưa được chấm có nhãn trên bộ hidden, mới chỉ tự kiểm không nhãn (200 trial, 0 `error`, 0 `leak_suspect`), nên chưa có pass rate hidden.
- Chưa có test riêng cho các ca đối kháng của bộ hidden như homoglyph hay principal lẫn. Workflow bỏ qua ticket nên được kỳ vọng an toàn, nhưng điều này chưa được kiểm bằng test do nhóm tự viết.
- Kết quả dùng MockLLM, latency là `unavailable`, cost là giá mô phỏng; mẫu chỉ có 14 claim × 5 seed nên không có khoảng tin cậy.
