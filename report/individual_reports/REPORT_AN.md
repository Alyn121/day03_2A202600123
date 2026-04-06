# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Lý Quốc An
- **Student ID**: 2A202600123
- **Date**: 06/04/2026

---

## I. Technical Contribution (15 Points)

Trong bài lab này, em tập trung vào phần cào dữ liệu (crawling), tinh chỉnh system prompt (prompting) cho mô hình ngôn ngữ lớn, đánh giá và tối ưu hóa quy trình hoạt động của agent.
- **Modules Implemented**: `src/agent/agent.py`, `src/tools/weather_forecast.py`, `src/tools/get_event.py`
- **Code Highlights**:
  - Khắc phục và tối ưu hóa hoàn toàn logic cào dữ liệu trong `weather_forecast.py`: Sử dụng BeautifulSoup để trích xuất 12 giờ dự báo thời tiết tại Hà Nội với các thông số chi tiết (nhiệt độ thực, nhiệt độ cảm nhận, tình trạng, độ ẩm, sức gió, chỉ số UV, tầm nhìn, áp suất) thông qua việc phân tích cấu trúc HTML phức tạp của thoitiet.vn.
  - Triển khai cơ chế cache chủ động cho dữ liệu thời tiết: Tự động lưu kết quả crawl ra file JSON (chỉ lấy 12 giờ tiếp theo) và ứng dụng cơ chế ưu tiên đọc cache cũ nhất gần đây (`_find_latest_cached_weather_file()`) nhằm giảm thiểu độ trễ, tiết kiệm tài nguyên mạng và tăng tốc phản hồi của tool API.
  - Nâng cấp luồng chaining-tool trong `_execute_tool()` (ví dụ luân chuyển dữ liệu từ `weather_forecast` thẳng vào `suggest_outfit`), giúp agent xử lý liền mạch các task mà không ngắt đoạn phiên reasoning.
  - Xây dựng luồng xử lý khoảng cách trong `get_nearby_places_serpapi()` (`_extract_distance`, `_distance_to_km`) để chuẩn hóa đa vị phân giải đơn vị (m/km) thành con số hệ đo liệu tiêu chuẩn, từ đó hỗ trợ thuật toán sắp xếp linh hoạt các địa điểm từ gần đến xa.
- **Prompting & Documentation**: Nâng cấp kĩ thuật prompting bằng cách tinh chỉnh system prompt thật chặt chẽ (đặt ra rule thứ tự bắt buộc: xem thời tiết -> tư vấn outfit -> gợi ý điểm hẹn). Việc này giúp agent vận hành ổn định mô hình ReAct: Model không bị "ảo giác" tự đoán mò mà luôn phân tích theo chuỗi 'Thought' để tìm 'Action' tương ứng, nhận 'Observation' đầy đủ trước khi xuất ra 'Final Answer' chỉn chu bằng tiếng Việt.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**: Ở các truy vấn có móc thời gian như "tôi muốn đi cafe vào lúc 16h" hay "tối nay 19h", agent có xu hướng dừng vòng lặp suy luận (reasoning) sớm với trạng thái lỗi `reason: "no_action"` thay vì phải kích hoạt hệ thống tool (cào dữ liệu thời tiết -> tư vấn outfit -> chọn quán).
- **Log Source**: Trích xuất từ `logs/2026-04-06.log`:
  - `{"event":"AGENT_START","data":{"input":"tôi muốn đi cafe vào lúc 16h","model":"gpt-4o-mini"}}`
  - `{"event":"AGENT_END","data":{"steps":2,"reason":"no_action"}}`
- **Diagnosis**: Thông qua logs, em nhận thấy LLM đôi lúc "quên" vai trò agent và đưa ra câu trả lời trực tiếp. Cú pháp không theo mẫu bắt buộc `Action: tool_name(args)` làm hàm regex không thể giãi mã (parse) lệnh, kích hoạt cơ chế fallback và dừng luồng chạy. Gốc rễ của vấn đề là do hệ thống prompt chưa có các ràng buộc đủ cứng rắn (hard-rules).
- **Solution**: Quá trình fix lỗi được thiếp lập qua 2 lớp phòng vệ:
  1. **Prompt Engineering**: Viết lại system prompt trong `get_system_prompt()`, định nghĩa rõ các Rule ép buộc thứ tự thực thi (ví dụ: bắt buộc gọi `weather_forecast` trước `suggest_outfit`).
  2. **Code Recovery**: Thêm cơ chế tự hồi phục trong `_finalize_response()`: Nếu agent thiếu dữ liệu thiết yếu, hệ thống can thiệp và cưỡng chế kích hoạt toolchain `weather_forecast` -> `suggest_outfit` -> `get_nearby_places_serpapi` để vá lỗi dữ liệu trước khi ghép thành final answer bằng `_compose_natural_cafe_answer()`.
  **Kết quả**: Giảm triệt để hiện tượng vứt bỏ Action, agent trở nên kỷ luật và thu tập đủ dự liệu mới tổng hợp câu trả lời.

---

## III. Personal Insights: Chatbot vs ReAct Agent (10 Points)

1. **Reasoning (Tư duy luân phiên)**: Chìa khóa của ReAct là tiến trình `Thought` giúp chia nhỏ các task phức tạp thành các phân mảnh nhỏ hơn (xác định thời tiết -> chọn đồ -> tìm vị trí). So với chatbot truyền thống vốn dễ mắc lỗi "hallucinate" (ảo giác), ReAct tạo ra một phác đồ tư duy minh bạch và dựa trên cơ sở dữ liệu thực (evidence-based).
2. **Reliability (Độ tin cậy của quy trình)**: Trong khi chatbot có thể gen-text rất trôi chảy dù thiếu kiến thức, ReAct Agent mong manh hơn trước các lỗi kỹ thuật (ví dụ: lỗi mạng, tool crawl website bị block, trả về sai định dạng Action). Sự ổn định của agent đòi hỏi sự phối hợp chặt chẽ giữa logic code cứng (bổ sung file cache JSON, fallback recovery) lẫn các prompt chặt chẽ.
3. **Observation (Biến số định hướng)**: Observation không đơn thuần là string trả về từ tool, mà là định hướng để agent tự hiệu chỉnh tư duy (`Thought`). Ví dụ: nếu thông số trả về từ tool weather báo động có tia UV quá cao, agent sẽ tự thay đổi logic gọi tool outfit để ưu tiên quần áo chống nắng, thay vì làm việc theo một workflow cứng nhắc.

---

## IV. Future Improvements (5 Points)

Để mang agent này gần hơn với cấp độ môi trường thực tế (Production-ready), em đề xuất một số cải thiện:
- **Scalability**: Dịch chuyển việc gọi tool sang các thead bất đồng bộ / hàng đợi độc lập (task queue) để đảm bảo đáp ứng được lượng truy cập lớn. Đồng thời tiến hành nâng cấp hệ thống Local JSON cache hiện tại cho tool thời tiết sang cấu trúc In-memory datastore (ví dụ Redis cache TTL).
- **Safety & Guardrails**: Bổ sung bộ xác thực (Validator) để kiểm tra các chỉ số Action trước khi thực thi tool (chặn các argument đầu vào độc hại - Prompt Injection), đồng thời thiết lập giám sát cấp độ supervisor để ngăn thông tin trả về bị rò rỉ.
- **Performance & Observability**: Chuyển đổi định dạng Action truyền thống thành `Structured JSON Output` hoặc tích hợp thẳng công nghệ `Function Calling` bản địa hoá của model để dứt điểm việc parse regex lỗi. Tích hợp thêm các bộ đếm số liệu đo latency chuyên sâu của mỗi tool nhằm tối ưu tốc độ crawl / API.

---