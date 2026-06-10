# Lab Ngày 10 — Luồng Pipeline và Giải thích Artifact

Tài liệu này giải thích luồng xử lý dữ liệu trong `day10/lab`, vai trò của từng file và nơi lưu trữ log/artifact.

## 1. Luồng chạy khi pipeline clear data

Pipeline chính là `etl_pipeline.py`. Luồng dữ liệu khi clear data như sau:

```
Raw CSV (data/raw/policy_export_dirty.csv)
              |
              v
     transform/cleaning_rules.py
      load_raw_csv(path)
              |
              v
     transform/cleaning_rules.py
        clean_rows(rows)
              |
      +------------------------------+
      |                              |
      v                              v
  cleaned rows                   quarantine rows
      |                              |
      |                              +--> transform/cleaning_rules.py
      |                              |      write_quarantine_csv()
      |                              |      -> artifacts/quarantine/quarantine_<run_id>.csv
      v
  transform/cleaning_rules.py
  write_cleaned_csv()
      -> artifacts/cleaned/cleaned_<run_id>.csv
              |
              v
  quality/expectations.py
  run_expectations(cleaned)
              |
      +-------+-------+
      |               |
     pass            fail
      |               |
      v               v
  embed sang Chroma   dừng nếu không có --skip-validate
      |               |
      v               |
  etl_pipeline.py     |
  ghi manifest       |
  -> artifacts/manifests/manifest_<run_id>.json
              |
              v
  monitoring/freshness_check.py
  check_manifest_freshness()
              |
              v
  ghi log tất cả
  -> artifacts/logs/run_<run_id>.log
```

### Những bước chính trong `clean_rows()`
- đọc từng dòng từ `data/raw/policy_export_dirty.csv`
- chỉ cho phép `doc_id` nằm trong `ALLOWED_DOC_IDS`
- chuẩn hoá `effective_date` thành `YYYY-MM-DD`
- quarantine các dòng có ngày không hợp lệ, thiếu text, bản HR cũ, hoặc nội dung noisy
- loại bỏ duplicate theo text đã chuẩn hoá
- sửa nội dung refund từ `14 ngày` thành `7 ngày làm việc`
- sinh `chunk_id` ổn định để đảm bảo embed idempotent

## 2. Giải thích các file chính

### `etl_pipeline.py`
- entrypoint chính của ETL
- định nghĩa 2 command:
  - `run`: ingest → clean → validate → embed
  - `freshness`: kiểm tra freshness của manifest
- ghi log, CSV cleaned, CSV quarantine, manifest và embed dữ liệu

### `transform/cleaning_rules.py`
- chứa hàm xử lý CSV raw và rules clean data
- quyết định dòng nào giữ lại, dòng nào chỉnh sửa, dòng nào quarantine
- trả về dữ liệu cleaned cho bước embed
- viết file cleaned và quarantine

### `quality/expectations.py`
- định nghĩa expectation suite cho dữ liệu đã clean
- kiểm tra số dòng tối thiểu, doc_id, refund stale, noise, và `access_control_sop`
- trả về danh sách kết quả và flag halt

### `grading_run.py`
- đánh giá vector store với bộ câu hỏi grading chính thức
- ghi ra file JSONL kết quả grading

### `eval_retrieval.py`
- đánh giá retrieval với bộ câu hỏi test
- ghi ra file CSV kết quả retrieval

### `monitoring/freshness_check.py`
- kiểm tra manifest có thỏa SLA freshness không
- được dùng bởi `etl_pipeline.py run` và `etl_pipeline.py freshness`

## 3. Thư mục artifact và nơi lưu

### `artifacts/logs/`
- lưu log của pipeline
- định dạng file: `run_<run_id>.log`
- chứa các dòng như `raw_records`, `cleaned_records`, `quarantine_records`, kết quả expectation, hành động embed, và trạng thái freshness

### `artifacts/cleaned/`
- lưu file CSV dữ liệu đã clean
- định dạng file: `cleaned_<run_id>.csv`
- đây là dữ liệu dùng để embed vector

### `artifacts/quarantine/`
- lưu file CSV các dòng bị quarantine
- định dạng file: `quarantine_<run_id>.csv`
- chứa các dòng bị loại kèm `reason`

### `artifacts/manifests/`
- lưu metadata của mỗi run
- định dạng file: `manifest_<run_id>.json`
- chứa `raw_records`, `cleaned_records`, `quarantine_records`, `latest_exported_at`, `no_refund_fix`, `skipped_validate`, đường dẫn cleaned CSV, và thông tin Chroma

### `artifacts/eval/`
- lưu báo cáo đánh giá
- `grading_run.jsonl`: kết quả grading chính thức
- `after_fix_eval.csv`: kết quả retrieval sau khi fix pipeline

## 4. Cách log được ghi

`etl_pipeline.py` dùng helper `log()` để:
- in ra stdout
- ghi cùng một dòng vào `artifacts/logs/run_<run_id>.log`

Các mục quan trọng trong log:
- `run_id` và các bộ đếm: `raw_records`, `cleaned_records`, `quarantine_records`
- đường dẫn: `cleaned_csv`, `quarantine_csv`, `manifest_written`
- kết quả expectation: `expectation[...] OK/FAIL`
- hành động embed: `embed_prune_removed`, `embed_upsert count`
- kết quả freshness: `freshness_check=...`
- cuối cùng: `PIPELINE_OK` hoặc `PIPELINE_HALT`

## 5. Tóm tắt hành vi

Một lệnh `python etl_pipeline.py run` thành công sẽ:
- clean dữ liệu raw
- validate dữ liệu đã clean
- embed chunk cleaned vào ChromaDB
- ghi artifact vào thư mục `artifacts/`
- tạo manifest và log file
- thực hiện kiểm tra freshness

Nếu expectation fail và không dùng `--skip-validate`, pipeline sẽ dừng trước bước embed. Nếu dùng `--skip-validate`, pipeline vẫn tiếp tục embed dù expectation fail.

## 6. Ví dụ lệnh chạy

```bash
cd day10/lab
python etl_pipeline.py run
python etl_pipeline.py run --no-refund-fix --skip-validate
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json
python grading_run.py --out artifacts/eval/grading_run.jsonl
python eval_retrieval.py --out artifacts/eval/after_fix_eval.csv
```

## 7. Chỗ cần xem khi debug

- lỗi clean: `artifacts/quarantine/*.csv`
- lỗi validate: `artifacts/logs/run_<run_id>.log`
- lỗi embed: `artifacts/logs/run_<run_id>.log` và thư mục `chroma_db`
- lỗi freshness: `artifacts/manifests/manifest_<run_id>.json`
- lỗi grading: `artifacts/eval/grading_run.jsonl`
