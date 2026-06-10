# Data contract — Lab Day 10

Tài liệu này định nghĩa hợp đồng dữ liệu cho tập dữ liệu tri thức CS + IT Helpdesk dùng cho RAG.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn (doc_id) | Phương thức ingest | Failure mode chính | Metric / alert |
|----------------|-------------------|-------------------|----------------|
| `policy_refund_v4` | Batch export (CSV) | Trộn lẫn chunk stale 14 ngày của v3 | `refund_no_stale_14d_window` |
| `sla_p1_2026` | Batch export (CSV) | Tiền tố bẩn `Escalation P1:` làm giảm retrieval | `no_unclear_content_prefix` |
| `it_helpdesk_faq` | Batch export (CSV) | Thiếu thông tin ngày hiệu lực, trùng lặp text | `missing_effective_date`, `no_duplicate_chunk_text` |
| `hr_leave_policy` | Batch export (CSV) | Xung đột phiên bản nghỉ phép năm (2025 vs 2026) | `hr_leave_no_stale_10d_annual` |
| `access_control_sop` | Batch export (CSV) | Nguồn dữ liệu chưa được đăng ký trong pipeline | `unknown_doc_id` |

---

## 2. Schema cleaned

Dữ liệu sau khi làm sạch phải tuân thủ schema cấu trúc sau trước khi nạp vào vector database:

| Cột | Kiểu | Bắt buộc | Ghi chú / Ràng buộc |
|-----|------|----------|---------|
| `chunk_id` | string | Có | ID duy nhất, sinh bằng cách hash: `[doc_id]_[seq]_[sha256]` |
| `doc_id` | string | Có | Khóa tài liệu, thuộc allowlist quy định trong `data_contract.yaml` |
| `chunk_text` | string | Có | Nội dung văn bản sạch, độ dài tối thiểu $\ge 8$ ký tự |
| `effective_date` | date | Có | Ngày hiệu lực của tài liệu, định dạng ISO `YYYY-MM-DD` |
| `exported_at` | datetime | Có | Thời gian xuất bản ghi, dạng ISO 8601 |

---

## 3. Quy tắc quarantine vs drop

Dữ liệu bị lỗi trong quá trình ETL sẽ được xử lý theo các quy tắc sau:
- **Quarantine (Cách ly):** 
  - Các bản ghi có định dạng ngày hiệu lực sai, thiếu ngày hiệu lực, doc_id lạ, hoặc chứa nội dung stale (như chính sách phép 10 ngày cũ) sẽ bị chuyển vào thư mục cách ly `artifacts/quarantine/quarantine_<run_id>.csv` kèm theo cột `reason`.
  - Bản ghi cách ly sẽ KHÔNG được nạp vào vector store.
  - Đội ngũ Data Ops sẽ review các bản ghi bị quarantine định kỳ để sửa chữa nguồn hoặc cấu hình pipeline.
- **Drop (Loại bỏ):**
  - Các bản ghi trùng lặp nội dung (`duplicate_chunk_text`) chỉ giữ bản đầu tiên xuất hiện, các bản ghi trùng sau sẽ bị quarantine để giám sát nhưng không được xử lý tiếp.
  - Các bản ghi trống nội dung (`missing_chunk_text`) sau khi làm sạch sẽ bị loại bỏ hoàn toàn.

---

## 4. Phiên bản & canonical

- **Source of truth cho chính sách hoàn tiền:** `data/docs/policy_refund_v4.txt` (phiên bản 4 mới nhất hiệu lực từ 2026-02-01, quy định hoàn tiền trong 7 ngày làm việc). Tất cả các thông tin hoàn tiền 14 ngày của v3 đều bị coi là stale và bị sửa đổi hoặc loại bỏ.
- **Source of truth cho nghỉ phép năm:** `data/docs/hr_leave_policy.txt` (bản 2026 hiệu lực từ 2026-01-01, quy định nghỉ phép 12 ngày cho nhân viên dưới 3 năm kinh nghiệm). Tất cả thông tin 10 ngày nghỉ phép của bản 2025 đều bị loại bỏ.
- **Cấu hình phiên bản động:** Được định nghĩa trong `contracts/data_contract.yaml` dưới khóa `policy_versioning` để tránh hardcode trong mã nguồn.
