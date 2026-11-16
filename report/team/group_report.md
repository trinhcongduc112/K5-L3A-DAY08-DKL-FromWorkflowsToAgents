# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm: **DKL** — Lớp L3A
- Thành viên: Trịnh Công Đức (N1), Lê Nhật Linh (N2), Nguyễn Tuấn Kiệt (N3) — chi tiết vai trò trong `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Nhóm hiểu bài toán là một claim flow **tuần tự**: phải có claim và order trước, phải đọc order row để tính quote, rồi mới được gọi host ghi refund. Vì vậy tiền không bao giờ do model quyết: `compute_compensation` đọc order row và policy hiện hành, còn approval, trần hoàn tiền theo đơn, idempotency và ghi ledger đều nằm trong code của host (`lab8.durability`). Model chỉ được đọc dữ liệu đã lọc và soạn câu chữ.

Chia việc theo shape và theo lớp kiểm chứng, mỗi lượt một người một commit rồi bàn giao:

| Lượt | Người | Việc |
| --- | --- | --- |
| #1–#2 | N1 | `TEAM.md`, shape workflow |
| #3 | N2 | shape single |
| #4 | N3 | shape orchestrator |
| #5 | N1 | test crash 3 điểm × 3 shape, `crash.json` |
| #6 | N2 | test approval mượn / thu hồi / thiếu |
| #7 | N3 | quét leak trace + ticket độc |
| #8 | N3 | tự kiểm hidden, chốt code |
| #9–#10 | N2, N1 | benchmark 4 shape, research card |
| #11–#12 | N2, N3 | ADR, báo cáo nhóm |
| #13–#16 | N1, N2, N3, N1 | báo cáo cá nhân, chốt nộp |

Thứ tự làm shape là A (workflow) → B (single) → C (orchestrator), sau đó mới làm test crash/approval/leak vì các test này cần đủ ba shape mới xanh.

## 3. Giải pháp

**A — Workflow (`src/lab8/shapes/workflow.py`).** Luồng cố định bằng Python: `get_claim_view` → `lookup_order` → `compute_compensation` → `issue_refund`. Claim không tồn tại → `failed`; thiếu `order_id` → `needs_info`; order ngoài principal scope → `escalated`; quote 0 → `denied` và không gọi host. Quote > 0 thì luôn gọi `issue_refund` với **quote đầy đủ** để host tự trừ phần claim trước đã nhận. Model chỉ gọi `complete(purpose="draft")` để soạn chữ, không tham gia tính tiền.

**B — Single (`src/lab8/shapes/single.py`).** Một vòng `run_react` tối đa 8 bước. `decide` có bộ nhớ riêng (dict ngoài hàm) vì `run_react` chỉ giữ kết quả tool cuối; nhờ vậy shape lưu được order rồi mới xin quote. Mọi tool trong vòng lặp là read-only (`lookup_order`, `lookup_refund_history`, `compute_compensation`); gọi tool ghi bị `dispatch` chặn bằng `ToolRejected`. Sau vòng lặp, **code** (không phải `decide`) gọi `issue_refund`.

**C — Orchestrator (`src/lab8/shapes/orchestrator.py`).** Orchestrator gọi `complete(purpose="plan")`, dựng handoff bằng `make_handoff`, rồi để `run_readonly_investigators` chạy các tool đọc. Thứ tự `allowed_tools` là `lookup_order` → `lookup_refund_history` → `compute_compensation` vì runtime chỉ tính quote từ order view đã có. Quote lấy từ `tools["compute_compensation"]["amount_vnd"]`; investigator không có quyền ghi, orchestrator mới gọi `issue_refund`.

**Ranh giới code / model.**

| Việc | Ai quyết |
| --- | --- |
| Lọc theo principal, policy hiệu lực, trần tiền | Code |
| Đọc ticket mơ hồ, có nên hỏi thêm | Model, trong bound |
| Approval, idempotency, ghi refund | Code, một transaction |
| Trần hoàn tiền theo đơn, trừ phần claim trước | Code, đọc lại tổng trong cùng transaction ghi |
| Approval có thuộc đúng claim không | Code (`claims.approval_id` → `approvals.claim_id`) |
| Retry sau crash | Ledger, không phải checkpoint đơn độc |

**Handoff.** `handoff.v1` chỉ còn các field trong `ALLOWED_FIELDS`; `facts={}` nên ticket và claim row không đi theo. `allowed_tools` chỉ chứa tool đọc; `sanitize` loại field lạ và từ chối nếu còn `CANARY_SECRET_`, `parent_secret`, `raw_headers` hoặc tool ghi. Trace chỉ ghi handoff **đã** sanitize.

**Trace.** Mỗi run ghi một file JSON đúng contract v1 (`run_id`, `events`) trên mọi nhánh return:

| Shape | Event kinds | Cố ý không ghi |
| --- | --- | --- |
| workflow | `claim`, `order`, `quote`, `host`, `decision`, `crash` | ticket, claim row thô, `parent_secret`, `Authorization`, SĐT, địa chỉ, `internal_note`, `item_name`, canary |
| single | `claim`, `order`, `quote`, `host`, `decision`, `crash` | như trên |
| orchestrator | `claim`, `handoff`, `investigator`, `quote`, `host`, `decision`, `crash` | như trên; `investigator` chỉ chứa order view đã allowlist và số liệu refund history |

`InjectedCrash` được bắt để `trace.write()` rồi **raise lại**, không bị nuốt.

## 4. Kết quả

**Bộ public** — `src/reports/benchmark.json`, 14 claim × 5 seed = 70 trial mỗi shape:

| Shape | Pass rate | pass^5 | Seed min–max | Variance | Cost/success (VND) | Mean steps | p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | ---: | ---: |
| workflow | 1.000 | 1.000 | 1.000–1.000 | 0 | 28.79 | 4.43 | 5.0 |
| single | 1.000 | 1.000 | 1.000–1.000 | 0 | 28.91 | 4.43 | 5.0 |
| orchestrator | 1.000 | 1.000 | 1.000–1.000 | 0 | 47.73 | 5.36 | 6.0 |
| swarm | 0.986 | 0.929 | 0.929–1.000 | 0.000816 | 49.57 | 2.80 | 3.0 |

Swarm chỉ trượt đúng một trial: `R-swarm-C-002-s55` (C-002, seed 55), do shape tham chiếu tin số ngày trong ticket thay vì order row. Ba shape của nhóm đều 70/70.

**Crash safety** — `src/reports/crash.json`: `20/20`. Bổ sung `test_crash_points.py` chạy 3 điểm crash × 3 shape trên C-012; sau crash chạy lại cùng DB đều `resolved`, đúng 1 dòng `refund_ledger` và 1 audit `refund_committed`.

**Bảo mật** — 0 leak / 0 approval bypass trên bộ public:
- `test_trace_leak_sweep.py`: 3 shape × 14 claim + 3 ca ticket độc = 45 test; không lộ canary, `Authorization`, SĐT, địa chỉ, `internal_note`, `SECRET_ITEM_VANG`; không in ra stdout.
- `test_shape_approval_paths.py`: 3 shape × 3 ca (approval mượn, bị thu hồi, thiếu approval) = 9 test; tất cả `denied`, proposal 35000, không có dòng ledger, có audit `policy_denied` đúng lý do.

**Tự kiểm hidden** — `src/reports/hidden_selfcheck.json`, 3 shape × 200 trial = 600 trial, không nhãn nên chỉ chấm được status/trace/leak:

| Shape | resolved | denied | escalated | needs_info | error | leak_suspect | Mean/p95 steps |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| workflow | 90 | 95 | 10 | 5 | 0 | 0 | 4.53 / 5 |
| single | 90 | 95 | 10 | 5 | 0 | 0 | 4.53 / 5 |
| orchestrator | 90 | 95 | 10 | 5 | 0 | 0 | 5.50 / 6 |

Cả 600 trial có lý do `unlabeled`, tức trace hợp lệ; ba shape cho cùng phân bố trạng thái trên hidden.

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

1. **`-k single` gom nhầm test.** Lệnh `pytest -m student -k single` kéo cả 242 test vì tên file chứa "single". Nhóm dùng `pytest -q -m student -o addopts= -k "single and not workflow and not orchestrator and not swarm"` → `62 passed`.
2. **Thứ tự tool của orchestrator.** `run_readonly_investigators` tính `compute_compensation` từ `results["lookup_order"]`; đặt quote trước `lookup_order` thì runtime bỏ qua và shape thiếu quote. Sửa bằng cách cố định thứ tự trong `allowed_tools`, kiểm lại `-k orchestrator` → `62 passed` và toàn bộ `-m student` → `248 passed`.
3. **Crash không được nuốt.** Ban đầu test crash đỏ nếu shape không truyền `crash_at` hoặc bắt `InjectedCrash`. Sửa ở cả ba shape: forward `crash_at` vào `issue_refund`, bắt để ghi trace rồi `raise`. Kết quả `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety 20/20`.
4. **Thiếu `student_drop/`.** Chạy đúng lệnh bước #8 báo `FileNotFoundError: 'student_drop\hidden_world_state.json'`. Nhóm giải nén `student_drop/` từ `instructor.zip` vào `src/student_drop/` (đã gitignore, không commit) rồi chạy lại; 600 trial, `error_trials 0`, `leak_suspect 0`.
5. **Rác pytest lọt vào git.** `src/pytest-of-Owner/` xuất hiện sau mỗi lần chạy test. Nhóm thêm `pytest-of-*/` vào `.gitignore` thay vì commit thư mục tạm.

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`:

- N1 Trịnh Công Đức: workflow, test crash, tích hợp và nộp bài.
- N2 Lê Nhật Linh: single, test approval, benchmark, ADR.
- N3 Nguyễn Tuấn Kiệt: orchestrator, test leak, tự kiểm hidden, báo cáo nhóm.

## 7. Hạn chế và hướng cải thiện

- **MockLLM, không phải model sản xuất.** Câu chữ và quyết định của model là quy chuẩn cố định; kết quả không đo được khả năng suy luận thật. `latency` được ghi là `unavailable`.
- **Chi phí mô phỏng.** `cost_estimate` tính theo `config/pricing.yaml`, không phải hoá đơn API thật; phần lớn cost đến từ tool call (10 VND/lần).
- **Cỡ mẫu nhỏ.** 14 claim × 5 seed = 70 trial mỗi shape trên public; không có khoảng tin cậy, không đủ để khái quát.
- **Hidden tự kiểm không nhãn.** 600 trial chỉ xác nhận `status`, trace contract và leak; không tính được pass rate trên hidden.
- **Kết luận chỉ trong phạm vi bài toán tuần tự.** Swarm thấp hơn một phần vì `REFERENCE_WEAKNESS` được cài sẵn, không phải vì topology nói chung. Quyết định chọn workflow sẽ đổi nếu claim flow chuyển sang nhiều nguồn dữ liệu song song (đơn vị vận chuyển, kho, thanh toán), hoặc nếu hidden yêu cầu suy luận phi cấu trúc mà pipeline cố định không bao phủ.
- **Chưa có timeout/cancel.** Một tool hoặc model call treo sẽ treo cả trial; chưa đo p99 hay độ trễ thật.

## 8. Tài liệu tham khảo

- `docs/architecture.md`, `docs/contracts.md`, `harness_map.yaml`, `config/tools.manifest.v1.1.yaml` — contract của lab.
- Yubin Kim và cộng sự (Google Research, Google DeepMind, MIT), *Towards a Science of Scaling Agent Systems*, arXiv:2512.08296, 2025–2026. https://arxiv.org/abs/2512.08296
- *Why Do Multi-Agent LLM Systems Fail?* (MAST), NeurIPS 2025.
- *AgentArch: A Benchmark for Agent Architecture Choices*, AACL 2026.
- *Rethinking Evaluation of Multi-Agent System Efficiency*, EMNLP 2026.
- Mã nguồn lab: `src/lab8/` (shapes, tools, durability, policy, runtime, metrics).
