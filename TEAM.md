# Team

## Thông tin nhóm

- Tên nhóm: DKL
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-DKL-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/trinhcongduc112/K5-L3A-DAY08-DKL-FromWorkflowsToAgents, nhánh `main`, commit chốt: `35de7f9`

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Trịnh Công Đức | 2A202603527 | 26ai.ductc2@vinuni.edu.vn | [trinhcongduc112](https://github.com/trinhcongduc112) | Nhóm trưởng; workflow + tích hợp, nộp bài |
|   2 | Lê Nhật Linh | 2A202603514 | 26ai.linhln@vinuni.edu.vn | NhatLinh056 | single + benchmark, ADR |
|   3 | Nguyễn Tuấn Kiệt | 2A202603417 | 26ai.kietnt3@vinuni.edu.vn | [kite1143](https://github.com/kite1143) | orchestrator + bảo mật trace, báo cáo nhóm |

Gợi ý chia vai:

- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Trịnh Công Đức | workflow + tích hợp, nộp bài | 8c3c082, cfa6554 | src/tests/test_workflow_shape.py; src/tests/test_crash_points.py |
| Lê Nhật Linh | single + benchmark, ADR | 6f59b95 | src/tests/test_single_shape.py; src/tests/test_shape_approval_paths.py |
| Nguyễn Tuấn Kiệt | orchestrator + bảo mật trace, báo cáo nhóm | 81785a0, 0abdec4 | src/tests/test_orchestrator_shape.py; src/tests/test_trace_leak_sweep.py |

## Cá nhân

Mỗi thành viên đồng thời tạo một báo cáo riêng trong `report/` theo dạng:

```text
report/team/MSSV_HoTen.md
```

Ví dụ: `report/team/20241234_NguyenVanAn.md`.

Sao chép mục dưới đây cho từng thành viên.

## 2A202603527_TrinhCongDuc

### Phần việc phụ trách

- Nhóm trưởng. Shape A `workflow`, test crash 3 điểm × 3 shape, research card R1, tích hợp và nộp bài.

### Đóng góp cụ thể

- `8c3c082`: `src/lab8/shapes/workflow.py` và `src/tests/test_workflow_shape.py`.
- `cfa6554`: `src/tests/test_crash_points.py` (9 ca) và `src/reports/crash.json` do `lab8.crash_test` sinh.
- `84e57f2`: `TEAM.md`; `4533b4b`: `report/team/research_card.md`.

### Bằng chứng

- `report/team/2A202603527_TrinhCongDuc.md`. Workflow đạt 70/70 trial public, crash safety 20/20.

## 2A202603514_LeNhatLinh

### Phần việc phụ trách

- Shape B `single` (ReAct), test approval 3 shape, benchmark 4 shape × 5 seed, ADR.

### Đóng góp cụ thể

- `6f59b95`: `src/lab8/shapes/single.py` và `src/tests/test_single_shape.py`.
- `78cc808`: `src/tests/test_shape_approval_paths.py`. `64d65ec`: `src/reports/benchmark.json`. `a1e71c9`: `report/team/decision_adr.md`.

### Bằng chứng

- `report/team/2A202603514_LeNhatLinh.md`. Single đạt 70/70 trial public.

## 2A202603417_NguyenTuanKiet

### Phần việc phụ trách

- Shape C `orchestrator` (handoff allowlist, investigator chỉ đọc), test quét leak, tự kiểm hidden, báo cáo nhóm.

### Đóng góp cụ thể

- `81785a0`: `src/lab8/shapes/orchestrator.py` và `src/tests/test_orchestrator_shape.py`.
- `0abdec4`: `src/tests/test_trace_leak_sweep.py` (45 test). `1cc3453`: `src/reports/hidden_selfcheck.json`. `35de7f9`: `report/team/group_report.md`.

### Bằng chứng

- `report/team/2A202603417_NguyenTuanKiet.md`. Orchestrator đạt 70/70 trial public; tự kiểm hidden 600 trial, 0 `error`, 0 `leak_suspect`.
