# Bản Phân Tích Độc Lập - Phía Consumer (Core Business)
**Mã cặp:** Pair 02 (Core Business -> AI Vision)  
**Sinh viên thực hiện:** Nguyễn Huy Nam  
**Vai trò:** Consumer  

## 1. Hiểu Về Nhu Cầu Nghiệp Vụ & Kỹ Thuật
- **Mục đích:** Core Business cần gửi dữ liệu hình ảnh hoặc thông tin định danh khuôn mặt sang hệ thống AI Vision để thực hiện so khớp (face-match) hoặc lấy kết quả phân tích hình ảnh, nhằm đưa ra quyết định nghiệp vụ (ví dụ: cho phép qua cổng, kích hoạt cảnh báo, chấm công...).
- **Cơ chế tương tác:** REST sync (Yêu cầu và phản hồi đồng bộ qua HTTP).
- **Các thông tin bắt buộc phải nhận được từ Provider để phục vụ đối soát (Audit):**
  - `confidence`: Độ tin cậy của kết quả phân tích (kiểu số thực, ví dụ: 0.95).
  - `modelVersion`: Phiên bản của mô hình AI được sử dụng (ví dụ: "yolov8-face-v2").
  - `timestamp`: Thời gian hệ thống AI xử lý xong.
  - `traceId`: Mã định danh chuỗi hệ thống để kiểm tra log lỗi khi cần.

## 2. Đề Xuất Các Endpoints Trọng Tâm
Dựa trên nhu cầu, phía Consumer đề xuất hệ thống Provider phải cung cấp tối thiểu 4 endpoints sau:
1. `POST /vision/face-match`: Gửi ảnh hoặc tham chiếu ảnh để so khớp khuôn mặt.
2. `GET /vision/detections/{detectionId}`: Lấy chi tiết một kết quả phân tích theo ID.
3. `GET /vision/results/recent`: Lấy danh sách các kết quả phân tích gần đây nhất.
4. `GET /health`: Kiểm tra trạng thái hoạt động của dịch vụ AI Vision.

## 3. Các Trường Hợp Lỗi Dự Kiến & Phương Án Xử Lý (Problem Details)
Phía Consumer yêu cầu tất cả các phản hồi lỗi từ Provider (4xx, 5xx) phải tuân theo chuẩn **RFC 7807 (Problem Details)** gồm các trường: `type`, `title`, `status`, `detail`, và `instance`.
- **Lỗi 400 Bad Request:** Khi Consumer gửi sai định dạng dữ liệu hoặc thiếu các trường bắt buộc.
- **Lỗi 401 Unauthorized / 403 Forbidden:** Thiếu hoặc sai Token xác thực trong Header (`Authorization`).
- **Lỗi 422 Unprocessable Entity:** Định dạng ảnh đúng nhưng không thể xử lý (ví dụ: Ảnh quá mờ, không tìm thấy khuôn mặt trong ảnh).
- **Lỗi 504 Gateway Timeout:** Hệ thống AI downstream bị nghẽn, không trả kết quả kịp thời.

## 4. Các Câu Hỏi Chuẩn Bị Cho Phiên Đàm Phán (Bước 4)
Để thống nhất hợp đồng API, Consumer sẽ chất vấn Provider 3 câu hỏi sau:
1. Khi gọi `POST /vision/face-match`, Core Business sẽ gửi dữ liệu ảnh thô (base64/binary) hay chỉ cần gửi một chuỗi tham chiếu ảnh (`imageRef`) đã lưu trên Cloud Storage?
2. Ngưỡng độ tin cậy (`confidence`) tối thiểu là bao nhiêu (ví dụ: 0.80 hay 0.85) thì hệ thống AI Vision mới tính là so khớp thành công (`match: true`)?
3. Trong trường hợp mô hình AI hoạt động nhưng độ tin cậy quá thấp (Low confidence), hệ thống sẽ trả về mã lỗi `422` hay vẫn trả về `200 OK` kèm trạng thái cảnh báo trong body?