# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Đông Anh  
**Vai trò:** Ingestion / Cleaning / Embed / Monitoring (All-in-One Owner)  
**Ngày nộp:** 2026-06-10  

---

## 1. Tôi phụ trách phần nào?

Trong bài tập lab này, do làm việc độc lập, tôi tự phụ trách toàn bộ các công đoạn phát triển và tích hợp của ETL pipeline:
- **Ingestion & Data Contract:** Thiết lập file cấu hình [data_contract.yaml](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/contracts/data_contract.yaml) để khai báo các nguồn dữ liệu chính thức, bao gồm việc thêm tài liệu nguồn mới `access_control_sop` để có thể vượt qua câu hỏi đánh giá số 10.
- **Cleaning & Transform:** Viết 4 quy tắc làm sạch mới và cơ chế đọc dynamic cutoff versioning từ yaml trong [cleaning_rules.py](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/transform/cleaning_rules.py).
- **Expectation Suite:** Bổ sung 3 luật kiểm định chất lượng mới (E7, E8, E9) trong [expectations.py](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/quality/expectations.py) giúp kiểm soát tính hợp lệ của dữ liệu trước khi lưu trữ.
- **Embedding & Evaluation:** Thực hiện nạp dữ liệu idempotent vào ChromaDB và chạy các bài test đánh giá.

---

## 2. Một quyết định kỹ thuật

Tôi quyết định lựa chọn cơ chế **Dynamic Policy Versioning** (Đọc cutoff date động) cho chính sách nghỉ phép của HR từ file data contract thay vì hardcode giá trị `2026-01-01` trong mã nguồn. 
Quyết định này mang lại các ưu điểm lớn:
- **Tính linh hoạt:** Nếu bộ phận nhân sự thay đổi thời điểm áp dụng chính sách nghỉ phép mới (ví dụ dời sang `2026-03-01`), người vận hành chỉ cần chỉnh sửa file [data_contract.yaml](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/contracts/data_contract.yaml) mà không cần can thiệp hay sửa đổi dòng code Python nào trong [cleaning_rules.py](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/transform/cleaning_rules.py).
- **Tính đồng bộ:** Giúp data contract luôn đóng vai trò là "Single Source of Truth" duy nhất cho cả tài liệu kiến trúc, hệ thống QA và mã nguồn thực thi.

---

## 3. Một lỗi hoặc anomaly đã xử lý

Trong lần chạy thử đầu tiên sau khi cài đặt môi trường, pipeline báo lỗi dừng tiến trình (`halt`) ở Expectation số 6 (`hr_leave_no_stale_10d_annual`) với 2 vi phạm:
```
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
PIPELINE_HALT: expectation suite failed (halt).
```
- **Nguyên nhân:** Khi phân tích tệp xuất thô `policy_export_dirty.csv`, tôi phát hiện có 2 dòng bản ghi chứa nội dung chính sách nghỉ phép phép 10 ngày cũ của năm 2025 nhưng lại có ngày hiệu lực bị ghi đè thành `2026-03-30` (ví dụ dòng số 7 trong CSV). Do ngày hiệu lực giả mạo này vượt qua bộ lọc ngày cutoff `eff_norm >= 2026-01-01`, chúng được nạp vào cleaned data và gây ra lỗi xung đột version nghiêm trọng.
- **Cách xử lý:** Tôi bổ sung thêm một quy tắc lọc nội dung stale trong [cleaning_rules.py](file:///Users/nguyendonganh/Lecture-Day-08-09-10/day10/lab/transform/cleaning_rules.py). Nếu dòng dữ liệu thuộc `hr_leave_policy` và chứa cụm từ `"10 ngày phép năm"`, nó sẽ lập tức bị cách ly vào quarantine với lý do `"stale_hr_policy_text"`. Kết quả là lượt chạy tiếp theo đã đạt `violations=0` và pipeline kết thúc thành công (Exit 0).

---

## 4. Bằng chứng trước / sau

Trong kịch bản tiêm nhiễm dữ liệu lỗi hoàn tiền của Sprint 3 (`run_id=inject-bad`), dữ liệu chứa thông tin stale `"14 ngày làm việc"` được nạp vào vector store.
Dưới đây là so sánh kết quả truy xuất từ tệp `after_inject_bad.csv` (trước fix) và `eval_after_fix.csv` (sau fix):

- **Trước khi fix (inject-bad):**
  ```csv
  q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền...,policy_refund_v4,Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc...,yes,yes,yes,3
  ```
- **Sau khi fix (clean_run):**
  ```csv
  q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền...,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm...,yes,no,yes,3
  ```
Lỗi cấm `hits_forbidden` đã được sửa đổi hoàn toàn từ `yes` về `no`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ tích hợp thư viện **Great Expectations** thực tế hoặc xây dựng một model **Pydantic** để xác thực kiểu dữ liệu đầu vào và các trường ràng buộc cấu trúc của file cleaned CSV một cách tự động và chuẩn hóa, thay vì tự viết các vòng lặp kiểm tra thủ công bằng mã nguồn Python thuần.
