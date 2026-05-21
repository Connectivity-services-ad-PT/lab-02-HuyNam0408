# Biên Bản Đàm Phán Hợp Đồng API - Pair 02
**Hệ thống:** Smart Campus Operations Platform  
**Bên đàm phán:** Core Business (Consumer - Nguyễn Huy Nam) & AI Vision (Provider - Ngọc Chiến / Mạnh Đạt)  
**Cơ chế:** REST sync  

---

## Danh Sách Các Vấn Đề Đàm Phán (Tối thiểu 6 vấn đề)

### Vấn đề 1: Định dạng dữ liệu hình ảnh truyền lên
- **Bối cảnh:** Đầu vào cho endpoint `/vision/face-match`.
- **Đề xuất ban đầu của Provider:** Consumer truyền trực tiếp chuỗi Base64 của ảnh vào request body.
- **Phản biện của Consumer (Nam):** Truyền Base64 làm phình dung lượng request, tăng độ trễ mạng (latency) và không tối ưu cho kiến trúc REST sync của Smart Campus.
- **Thống nhất:** Sử dụng chuỗi định danh tham chiếu ảnh `imageRef` (ví dụ: `storage://...`). Phía Consumer sẽ đẩy ảnh lên Cloud Storage trước, sau đó chỉ truyền URI sang cho Provider xử lý.

### Vấn đề 2: Cấu trúc phản hồi khi độ tin cậy thấp (Low Confidence)
- **Bối cảnh:** Khi AI nhận diện được khuôn mặt nhưng chỉ số `confidence` nằm dưới ngưỡng an toàn.
- **Đề xuất ban đầu của Provider:** Trả về mã lỗi `422 Unprocessable Entity` để báo hiệu không thể định danh thành công.
- **Phản biện của Consumer (Nam):** Hệ thống AI vẫn hoạt động bình thường, không phải lỗi nghiệp vụ nhập liệu sai nên trả về đầu 4xx là không đúng bản chất. Consumer vẫn cần nhận mã `200 OK` để biết mô hình đã chạy, nhưng cấu trúc dữ liệu trả về sẽ chuyển sang nhánh cảnh báo để Core ra quyết định hạ tầng (ví dụ: yêu cầu quẹt lại thẻ).
- **Thống nhất:** Sử dụng cấu trúc đa hình `oneOf` kết hợp với `discriminator` (trường `status`). Nếu thành công trả về trạng thái `matched` kèm `userId`, nếu nghi ngờ trả về trạng thái `low_confidence` kèm thông điệp cảnh báo nhưng vẫn giữ mã HTTP `200`.

### Vấn đề 3: Quy định giá trị trống cho các trường không bắt buộc
- **Bối cảnh:** Trường hành động gợi ý (`suggestedAction`) trong tình huống nhận diện cảnh báo.
- **Đề xuất ban đầu của Provider:** Nếu không có gợi ý gì thì sẽ không trả về trường này trong JSON (bỏ trống trường).
- **Phản biện của Consumer (Nam):** Việc thiếu trường trong JSON có thể gây lỗi NullPointerException hoặc parse lỗi ở phía ứng dụng Consumer nếu không được định nghĩa rõ ràng trong OpenAPI.
- **Thống nhất:** Ép buộc sử dụng kiểu dữ liệu kết hợp với null trong OpenAPI 3.1.0 (`type: [string, "null"]`). Nếu không có giá trị, Provider bắt buộc phải trả về `"suggestedAction": null` thay vì xóa bỏ trường.

### Vấn đề 4: Cơ chế định danh chuỗi lỗi và đối soát hệ thống (Audit/Trace)
- **Bối cảnh:** Việc phân tích log lỗi khi tích hợp giữa hai hệ thống độc lập.
- **Đề xuất ban đầu của Provider:** Chỉ trả về kết quả phân tích và thời gian thực hiện.
- **Phản biện của Consumer (Nam):** Phía Core Business cần các trường thông tin bắt buộc bao gồm `modelVersion`, `timestamp`, và đặc biệt là `traceId` ở tất cả phản hồi (kể cả thành công và thất bại) để phục vụ công tác giám sát (Audit).
- **Thống nhất:** Tách toàn bộ các thông tin chung này ra và đưa vào cấu trúc Schema dùng chung, ép buộc mọi endpoint liên quan đến kết quả AI đều phải phản hồi đủ 4 trường đối soát này.

### Vấn đề 5: Chuẩn hóa cấu trúc dữ liệu phản hồi lỗi (Error Handling)
- **Bối cảnh:** Cách thức hiển thị và trả lỗi khi có sự cố 4xx hoặc 5xx xảy ra.
- **Đề xuất ban đầu của Provider:** Trả lỗi dạng chuỗi thuần túy (Plain text) hoặc object JSON tùy biến theo từng API.
- **Phản biện của Consumer (Nam):** Gây khó khăn cho việc viết bộ lọc xử lý lỗi tập trung (Global Exception Handler) ở phía Consumer.
- **Thống nhất:** Bắt buộc áp dụng nghiêm ngặt tiêu chuẩn **RFC 7807 (Problem Details)** với Header `application/problem+json` cho mọi response lỗi, cấu trúc chuẩn hóa gồm các trường: `type`, `title`, `status`, `detail`, và `instance`.

### Vấn đề 6: Giới hạn số lượng dữ liệu ở API đối soát (`/vision/results/recent`)
- **Bối cảnh:** Lấy danh sách kết quả phân tích gần đây.
- **Đề xuất ban đầu của Provider:** Trả về toàn bộ danh sách lưu trong ngày của camera.
- **Phản biện của Consumer (Nam):** Số lượng bản ghi trong ngày có thể lên tới hàng ngàn, gây quá tải bộ nhớ và nghẽn băng thông.
- **Thống nhất:** Bổ sung tham số truy vấn (query parameter) tên là `limit` kiểu số nguyên (`integer`), giá trị mặc định là `10` để phân trang và giới hạn băng thông truyền tải.

---

## Cam Kết Đã Ký (Sign-Off)
- **Đại diện Consumer:** Nguyễn Huy Nam (Đã ký duyệt cấu trúc hợp đồng)
- **Đại diện Provider:** Ngọc Chiến / Mạnh Đạt (Đã ký duyệt cấu trúc hợp đồng)
- **Phiên bản thống nhất:** v1.0.0
- **Ngày ký kết:** 21/05/2026