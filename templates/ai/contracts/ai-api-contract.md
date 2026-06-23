# 1. Luồng Cốt Lõi (Core Pipeline)
Tài liệu này định nghĩa một quy trình tự động hóa khép kín (closed-loop automation) bao gồm 4 bước bắt buộc:

* **Detect (Phát hiện)**: Hệ thống giám sát đẩy dữ liệu telemetry vào API `/v1/detect`.
* **Decide (Ra quyết định)**: Hạ tầng mang cảnh báo đó gọi `/v1/decide` để lấy kịch bản xử lý.
* **Execute (Thực thi)**: CDO tự động chạy các lệnh hạ tầng (không nằm trong API này).
* **Verify (Xác minh)**: CDO gọi `/v1/verify` để AI đánh giá xem sự cố đã được khắc phục hoàn toàn hay chưa.

# 2. Quy Tắc Chung & Tính Lũy Đẳng (Idempotency)

### Bảo mật và Môi trường
* **Authentication**: Mọi kết nối liên dịch vụ phải được ký bằng IAM SigV4.
* **Offline Simulation**: Vì đang dùng dữ liệu tĩnh (dataset RE2/RE3), CDO sẽ gửi các mốc thời gian giả lập vào `post_telemetry_window` để AI kiểm tra, và `tenant_id` sẽ được chèn các chuỗi định danh mô phỏng (ví dụ: `tnt-re3-simulation`).

### Bản chất thực tế của Idempotency (Tính lũy đẳng)
Đối với các API thay đổi trạng thái hạ tầng (`/v1/decide` và `/v1/verify`), CDO bắt buộc phải đính kèm một `Idempotency-Key` (UUID v4). Cơ chế này để ngăn chặn rủi ro hệ thống thực thi trùng lặp một lệnh (duplicate execution) khi xảy ra sự cố về mạng (network timeout/retries).

Kịch bản thực tế diễn ra như sau:
* **Bước 1**: CDO Platform gọi API POST `/v1/decide` để xin kịch bản xử lý lỗi, gửi kèm header `Idempotency-Key: 123...`.
* **Bước 2**: AI Engine chạy thuật toán xong và quyết định xuất ra lệnh `RESTART_DEPLOYMENT`.
* **Bước 3 (Sự cố mạng)**: Đúng lúc AI chuẩn bị trả kết quả thì đường truyền bị gián đoạn (timeout). AI đã xử lý xong nhưng CDO chưa nhận được kết quả.
* **Bước 4 (CDO Retry)**: Vì không nhận được phản hồi, CDO tự động gọi lại API (retry). Điểm mấu chốt là CDO vẫn giữ nguyên mã `Idempotency-Key: 123...` như lần đầu.
* **Bước 5 (AI Xử lý)**: AI Engine đọc header này, tra cứu trong cache và nhận ra request này đã được xử lý. Nhờ đó, AI không chạy lại thuật toán, chỉ đơn giản trả lại đúng kết quả `RESTART_DEPLOYMENT` đã lưu từ lần trước.

Nếu thiếu key này, AI có thể vô tình phát lệnh restart 2 lần liên tiếp, gây sập dịch vụ nặng hơn. Ngoài ra, nếu CDO gửi request thứ 2 khi request 1 vẫn đang chạy, AI sẽ trả mã `409 Conflict` để chặn lại.

# 3. Đặc Tả 3 API Endpoints

### 1. POST /v1/detect (Nhận diện sự cố)
* **Đầu vào**: CDO gửi mảng `telemetry_window` chứa các điểm dữ liệu (datapoints) tại thời điểm hệ thống có biến động (bao gồm timestamp, tên tín hiệu, giá trị metric hoặc log).
* **Đầu ra**: AI phân tích và trả về cờ `anomaly_detected`. Quan trọng nhất, AI cấp một `correlation_id` (định danh phiên theo dõi sự cố) để liên kết toàn bộ chuỗi hành động sau này.

### 2. POST /v1/decide (Đưa ra chiến lược)
* **Đầu vào**: CDO cung cấp `correlation_id` và ngữ cảnh lỗi từ bước trước. Chú ý cờ `dry_run_mode` để kiểm thử logic mà không kích hoạt hạ tầng thật.
* **Đầu ra**:
  * `action_plan`: Một mảng chứa trình tự các lệnh hạ tầng (ví dụ: `RESTART_DEPLOYMENT`, `SCALE_UP_PODS`).
  * `blast_radius_config`: Ràng buộc giới hạn sát thương. Ví dụ: AI chỉ cho phép khởi động lại tối đa một tỷ lệ phần trăm Pod nhất định (`max_pod_impact_pct`). Nếu lỗi vượt ngưỡng (`circuit_breaker_error_rate`), CDO phải tự ngắt tự động hóa.

### 3. POST /v1/verify (Xác minh hậu thực thi)
* **Đầu vào**: CDO gửi trạng thái lệnh vừa chạy (`action_executed`) kèm theo cửa sổ dữ liệu metrics thu thập được sau khi lệnh đã thực thi (`post_telemetry_window`).
* **Đầu ra**: AI đánh giá xem hệ thống đã ổn định chưa (`success: true`) hoặc có phát sinh suy giảm hiệu năng mới không (`regression_detected`). Nếu mọi nỗ lực thất bại, AI trả lệnh `ESCALATE` kèm theo một `escalation_bundle` chứa toàn bộ log/context để CDO bắn thẳng cảnh báo lên kênh Slack cho kỹ sư.

# 4. Cam Kết SLAs & Xử Lý Mã Lỗi (Error Handling)
Tài liệu quy định ranh giới trách nhiệm rất rõ ràng qua các mã HTTP Status để tránh hai team đổ lỗi cho nhau:

* **P99 Latency**: Hệ thống phải phản hồi dưới 500ms cho các API quyết định. Rate limit tối đa 120 req/min.
* **400 Bad Request**: CDO map sai schema JSON. CDO phải tự log và sửa code, tuyệt đối không tự động retry.
* **429 Too Many Requests**: Vượt quota. CDO phải thiết kế thuật toán Exponential Backoff (tăng dần thời gian chờ) trước khi thử lại.
* **503 Service Unavailable**: AI bị sập (engine crash). Đây là điểm chí mạng: CDO bắt buộc phải thiết kế luồng fallback nội bộ (ví dụ: tự động chạy một script runbook tĩnh dự phòng, hoặc bắn ngay alert cho SRE) thay vì để hệ thống tê liệt.
