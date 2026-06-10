# Báo cáo Cá nhân — Trần Văn Huỳnh

**Vai trò:** Cleaning & Quality Owner
**Run id:** `2026-06-10T08-04Z`

## 1. Tóm tắt công việc tôi làm
- Cập nhật `transform/cleaning_rules.py`: thêm allowlist `access_control_sop`, thêm rule quarantine cho nội dung noisy, chuẩn hoá refund 14→7 và loại bỏ HR stale.
- Cập nhật `quality/expectations.py`: thêm `no_noisy_chunk_text` và `access_control_sop_ingested` (cả hai severity=`halt`).
- Chạy pipeline end-to-end, kiểm tra manifest và artifacts.

## 1. Tôi phụ trách phần nào?

Tôi chịu trách nhiệm chính phần `Cleaning` và `Quality` (module chính: `transform/cleaning_rules.py` và `quality/expectations.py`). Công việc cụ thể của tôi:
- Thiết kế và triển khai các rule làm sạch (quarantine noisy chunks, chuẩn hoá ngày, sửa ngữ cảnh refund 14→7).
- Cập nhật expectation suite để phát hiện các anomaly quan trọng (thêm `no_noisy_chunk_text`, `access_control_sop_ingested` với severity=`halt`).

Kết nối với các thành viên khác: tôi phối hợp với người phụ trách Embed để đảm bảo các chunk được upsert vào Chroma chỉ khi sạch; với Monitoring để đưa metric freshness và số liệu quarantine vào manifest.

Bằng chứng (commit / file): sửa tại `transform/cleaning_rules.py` và `quality/expectations.py`; run id được ghi trong manifest: `artifacts/manifests/manifest_2026-06-10T08-04Z.json`.

## 2. Một quyết định kỹ thuật

Quyết định chính tôi đưa ra là chọn chiến lược "halt on critical expectations" (severity=`halt`) cho những expectation có thể dẫn tới sai lệch nghiệp vụ (ví dụ: ingestion thiếu `access_control_sop` hoặc presence of noisy chunks). Lý do: nếu pipeline tiếp tục với dữ liệu noisy sẽ ảnh hưởng lan toả tới vector index và làm sai lệch retrieval cho nhiều câu grading.

Thực thi: expectation critical sẽ trả về lỗi và dừng pipeline; các expectation ít nghiêm trọng hơn chỉ log cảnh báo. Cách này giúp đảm bảo idempotency và chất lượng đầu vào cho embedding, đồng thời dễ debug (fail-fast). Việc này thể hiện trong `quality/expectations.py` và logs của run `2026-06-10T08-04Z`.

## 3. Một lỗi hoặc anomaly đã xử lý

Triệu chứng: nhiều chunk chứa ký tự marker, note nội bộ, hoặc thông tin HR cũ (`"10 ngày phép năm"`) làm kết quả retrieval trỏ tới bản HR cũ thay vì HR 2026. Kiểm tra manifest cho thấy `raw_records=247`, ban đầu cleaned ~40 nhưng vẫn có HR stale.

Phát hiện: expectation `hr_leave_no_stale_10d_annual` báo fail trên run test đầu tiên. Hành động: tôi thêm rule nhận diện "stale HR text" và chuyển các chunk này vào quarantine; đồng thời chuẩn hoá refund windows (`14 ngày` → `7 ngày làm việc`).

Kết quả: cleaned giảm còn `36` record và quarantine tăng thành `211` (xem `artifacts/manifests/manifest_2026-06-10T08-04Z.json`). Sau fix, expectation liên quan pass và grading hộp kiểm top-1 khớp.

## 4. Bằng chứng trước / sau (2 dòng trích từ grading_run)

Run `2026-06-10T08-04Z`, file: `artifacts/eval/grading_run.jsonl` (2 dòng ví dụ):

{"id": "gq_d10_01", "question": "...hoàn tiền...", "top1_doc_id": "policy_refund_v4", "contains_expected": true, "top1_doc_matches": true}

{"id": "gq_d10_10", "question": "...Admin Access...", "top1_doc_id": "access_control_sop", "contains_expected": true, "top1_doc_matches": true}

Nhận xét: cả hai dòng đều `contains_expected=true` và `top1_doc_matches=true` — chứng tỏ rule clean + expectations đã bảo toàn thông tin nghiệp vụ cần thiết cho retrieval.

## 5. Cải tiến tiếp theo (nếu có thêm 2 giờ)

1) Viết test unit cho từng rule trong `transform/cleaning_rules.py` (fixtures nhỏ). 2) Thực hiện `inject-bad` run để sinh `after_inject_bad.csv` và tạo bảng so sánh metric_impact (precision@1, contains_expected, quarantine_changes).

---

_Báo cáo này được tạo dựa trên run thực tế và các artifact trong thư mục `artifacts/` — chỉnh thông tin cá nhân trước khi nộp chính thức._

## 2. Kết quả grading với `grading_questions.json`
Artifact grading: `artifacts/eval/grading_run.jsonl` (tạo bởi `python grading_run.py`). Dưới đây là tóm tắt kết quả cho 10 câu grading:

| ID câu | Top-1 doc | Contains expected | Top1 doc matches |
|---|---|---:|---:|
| gq_d10_01 | policy_refund_v4 | true | true |
| gq_d10_02 | policy_refund_v4 | true | true |
| gq_d10_03 | policy_refund_v4 | true | true |
| gq_d10_04 | sla_p1_2026 | true | true |
| gq_d10_05 | sla_p1_2026 | true | true |
| gq_d10_06 | sla_p1_2026 | true | true |
| gq_d10_07 | it_helpdesk_faq | true | true |
| gq_d10_08 | it_helpdesk_faq | true | true |
| gq_d10_09 | hr_leave_policy | true | true |
| gq_d10_10 | access_control_sop | true | true |

Tất cả 10 câu grading đều trả về `contains_expected=true` và `top1_doc_matches=true` theo file `grading_run.jsonl`.

## 3. Bằng chứng & artifact
- Grading artifact: `artifacts/eval/grading_run.jsonl`
- Eval retrieval CSV: `artifacts/eval/after_fix_eval.csv`
- Manifest: `artifacts/manifests/manifest_2026-06-10T08-04Z.json`

