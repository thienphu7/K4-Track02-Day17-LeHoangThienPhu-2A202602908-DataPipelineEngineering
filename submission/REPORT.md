# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Lê Hoàng Thiên Phú / 2A202602908
**Repo:** https://github.com/thienphu7/K4-Track02-Day17-LeHoangThienPhu-2A202602908-DataPipelineEngineering
**Commit bài nộp:** _Cập nhật SHA của commit cuối sau khi commit._
**AI đã dùng và phạm vi hỗ trợ (hoặc không dùng):** Dùng Codex để đọc đề, xác định ba lỗi.
**Nguồn tham khảo khác (nếu có):** README, RUBRIC và mã nguồn của đề bài.

## 1. Ba lỗi

Mỗi lỗi 4 dòng. Triệu chứng = thứ bạn *thấy* đầu tiên (check nào fail, số nào lạ,
checksum nào lệch) — không phải cách sửa.

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | Check Silver báo `silver_tickets` có nhiều hàng cho cùng một `ticket_id`; rerun ngày cũ làm Gold checksum lệch. | Check Gold không khớp full recompute; event offline của u05 ngày 08-12 không được tính cho 08-12. | Check Silver/Gold báo T-97 chưa là tombstone và vẫn còn trong training set/RAG index. |
| **Nguyên nhân gốc** | Hàm ghi dữ liệu dùng `INSERT`, không upsert theo khoá và không chặn LSN cũ ghi đè trạng thái mới. | `LOOKBACK_DAYS = 0` dù P99 lateness đo từ Bronze là 3 ngày. | Staging chỉ lấy `ticket_id` từ `after`; bản ghi Debezium `op = 'd'` có `after = null`. |
| **Cách sửa** (file, vài dòng) | `pipeline/silver.py`: thay `INSERT` bằng `MERGE` theo `ticket_id`, chỉ `UPDATE` khi `s._lsn > t._lsn`. | `pipeline/config.py`: đặt `LOOKBACK_DAYS = 3`, bằng `ceil(P99)`. | `pipeline/staging.py`: dùng `coalesce(after.ticket_id, before.ticket_id)` để delete tạo tombstone có khoá. |
| **Khái niệm trên slide** | Silver keyed write; LSN guard giúp rerun/backfill idempotent. | Late data: overwrite partition theo event time với lookback đo từ Bronze. | Debezium CDC delete/tombstone; xoá phải lan xuống Gold. |

## 2. Các con số

- P99 lateness đo từ Bronze: `3.00` ngày → `LOOKBACK_DAYS = 3`
- `submission/checksums.txt`: PASS — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: PARITY

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- MERGE theo khoá cho `silver_tickets`, overwrite-partition cho `gold_feature_daily`: MERGE giữ một trạng thái hiện tại cho mỗi ticket, còn aggregate theo ngày có thể tính lại an toàn trong cửa sổ event-time.
- Tombstone thay vì xoá hẳn hàng trong Silver: tombstone giữ khoá và trạng thái xoá để bản ghi cũ hoặc rerun không vô tình tạo lại ticket đã bị xoá.
- Snapshot training dựng lại từ Bronze "as of" ngày đó, không sửa snapshot cũ: cách này tái lập được dữ liệu huấn luyện và tránh dùng thông tin của tương lai.
- DuckDB (lite) / dbt (track dbt) cho bài toán cỡ này, chứ không phải Spark: DuckDB chạy local, zero-key và đủ cho dữ liệu nhỏ; dbt kiểm chứng cùng logic bằng SQL declarative.

## 4. Hai câu hỏi suy ngẫm

1. Snapshot `v2026-08-12`..`v2026-08-14` vẫn chứa văn bản của T-97 (đã bị xoá ngày
   08-15). "Snapshot bất biến" và "quyền được xoá dữ liệu" mâu thuẫn — bạn xử lý thế nào?
   Trong production, tôi lưu lineage và chính sách retention cho từng snapshot, thu hồi quyền truy cập ngay khi có yêu cầu xoá, rồi purge hoặc rebuild các artifact bị ảnh hưởng theo yêu cầu pháp lý; không coi bất biến là quyền giữ PII vô thời hạn.
2. Regex che được email và số điện thoại, nhưng tên "Nguyễn Văn An" vẫn còn. Bạn sẽ
   đặt chốt PII nào, ở tầng nào, và đo nó ra sao?
   Tôi đặt PII gate ngay sau Bronze trước khi dữ liệu vào Silver/Gold, kết hợp schema/classification, regex và NER để tokenize hoặc quarantine dữ liệu; đo tỷ lệ phát hiện, false positive/false negative qua mẫu audit và thời gian xử lý leak.

## 5. Output (dán nguyên văn)

```text
$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe -m scripts.verify
RESULT: 18/18 checks — ALL PASS

$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe -m pytest
..................................                                       [100%]
34 passed in 6.05s

$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe -m scripts.rerun_check
fresh build             8630e04a61d1  9370ca77af23  cb9ebd12fdcc  39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1  9370ca77af23  cb9ebd12fdcc  39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1  9370ca77af23  cb9ebd12fdcc  39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1  9370ca77af23  cb9ebd12fdcc  39e115c510ecdf526800eac227158a4f
RESULT: PASS — 3 re-runs, identical checksums

$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe main.py --lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ C:\Users\ADMIN\miniforge3\envs\labs\Scripts\dbt.exe build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17
Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19

$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe -m scripts.parity
RESULT: PARITY — both implementations agree
```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.

Bonus B1:

```text
$ C:\Users\ADMIN\miniforge3\envs\labs\python.exe -m scripts.bonus_llm
=== bonus: LLM labelling of 11 live tickets ===
  cost estimate before running: ~484 tokens = $0.0010 per full run
  [OK ] first run labels every live ticket
  [OK ] re-run with same model + prompt makes 0 LLM calls
  [OK ] every Gold label is bug / billing / other
  [OK ] off-schema answers go to llm_label_quarantine
  [OK ] new prompt version re-labels on purpose
  [OK ] labels carry their prompt version
BONUS PASS
```

Bonus B2: `bonus/DESIGN.md`
