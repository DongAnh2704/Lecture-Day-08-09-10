# Runbook — Lab Day 10 (incident tối giản)

Tài liệu hướng dẫn vận hành và khắc phục sự cố cho đường ống dữ liệu RAG Knowledge Base.

---

## Symptom (Triệu chứng)

- Người dùng cuối hoặc các Agent RAG (từ Day 09) nhận được câu trả lời sai hoặc lỗi thời. Ví dụ:
  - Trả lời khách hàng được hoàn tiền trong "14 ngày làm việc" thay vì "7 ngày làm việc".
  - Trả lời nhân viên dưới 3 năm kinh nghiệm được nghỉ phép "10 ngày" thay vì "12 ngày".
  - Agent không thể trả lời hoặc lấy thông tin từ tài liệu mới như `access_control_sop` (không trả lời được quyền Level 4 Admin phê duyệt bởi ai).

---

## Detection (Phát hiện)

Sự cố có thể được phát hiện qua các hệ thống giám sát:
1. **Pipeline Halt (Lỗi nghiêm trọng):** Lệnh `python etl_pipeline.py run` bị lỗi dừng (exit code 2) do vi phạm một trong các expectations nghiêm trọng (như phát hiện dữ liệu thô chứa thông tin stale hoặc định dạng ngày sai).
2. **Freshness SLA Alert:** Lệnh `python etl_pipeline.py freshness` báo trạng thái `FAIL` khi tuổi dữ liệu (age_hours) vượt quá ngưỡng SLA 24 giờ.
3. **Retrieval Eval Alert:** Chạy `python eval_retrieval.py` cho thấy cột `hits_forbidden` trả về `"yes"` hoặc `contains_expected` trả về `"no"`.

---

## Diagnosis (Chẩn đoán)

Nhân viên vận hành thực hiện các bước sau để tìm nguyên nhân:

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| **1** | Kiểm tra file log chạy gần nhất trong `artifacts/logs/run_*.log` | Tìm dòng thông báo lỗi `expectation[...] FAIL (halt)` để xác định luật kiểm định nào bị vi phạm. |
| **2** | Đọc tệp cách ly mới nhất trong `artifacts/quarantine/quarantine_*.csv` | Xem các dòng bị cách ly và cột `reason` để biết tại sao dữ liệu bị gạt ra ngoài (Ví dụ: `missing_effective_date`, `stale_hr_policy_text`). |
| **3** | Kiểm tra manifest tương ứng trong `artifacts/manifests/manifest_*.json` | Kiểm tra timestamp `latest_exported_at` để xem dữ liệu nguồn được xuất từ bao giờ, có bị chậm trễ không. |
| **4** | Chạy kiểm tra retrieval tự động: `python eval_retrieval.py --out artifacts/eval/temp_eval.csv` | Kiểm tra file `temp_eval.csv` xem câu nào bị lỗi `hits_forbidden=yes` hoặc `contains_expected=no`. |

---

## Mitigation (Khắc phục tạm thời)

1. **Khắc phục lỗi định dạng / dữ liệu cũ:**
   - Nếu lỗi do dữ liệu thô bị xuất sai định dạng ngày, yêu cầu hệ thống nguồn xuất lại hoặc cập nhật file thô đúng chuẩn ISO.
   - Nếu có thông tin stale (ví dụ: bản hoàn tiền 14 ngày), chạy pipeline với cờ sửa đổi tự động hoặc sửa đổi trực tiếp dữ liệu thô nếu nguồn bị sai.
2. **Rollback dữ liệu ChromaDB:**
   - Nếu pipeline chạy thành công nhưng dữ liệu embed bị lỗi nghiêm trọng, chạy pipeline với file cleaned của lượt chạy tốt trước đó để phục hồi trạng thái vector database (Idempotent upsert & prune sẽ tự động xóa các vector lỗi).
3. **Bỏ qua kiểm định khẩn cấp (Chỉ dùng khi được phê duyệt):**
   - Nếu cần cập nhật dữ liệu khẩn cấp và lỗi được xác định là không nghiêm trọng, chạy pipeline với cờ `--skip-validate` để tiếp tục embed bất chấp cảnh báo.

---

## Prevention (Phòng ngừa)

1. **Tự động hóa kiểm tra trước khi commit:** Tích hợp pipeline chạy thử nghiệm trên CI/CD trước khi cập nhật dữ liệu tri thức lên production.
2. **Cập nhật định kỳ data contract:** Đảm bảo mọi tài liệu nguồn mới đều được đăng ký đầy đủ trong `contracts/data_contract.yaml` để tránh bị gạt sang quarantine do `unknown_doc_id`.
3. **Theo dõi freshness chủ động:** Lên lịch cron job chạy `etl_pipeline.py freshness` định kỳ mỗi 6 giờ để cảnh báo sớm nếu dữ liệu tri thức không được cập nhật đúng hạn.
