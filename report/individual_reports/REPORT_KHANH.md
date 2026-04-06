# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Nguyễn Quốc Khánh
- **Student ID**: 2A202600200
- **Date**: 06/04/2026

---

## I. Technical Contribution (15 Points)

Trong bài lab này, em tập trung vào việc hoàn thiện luồng ReAct cho bài toán gợi ý đi cafe theo ngữ cảnh thời tiết và địa điểm, đồng thời tối ưu dữ liệu đầu vào/đầu ra của các tool để agent trả lời ổn định hơn.

- **Modules Implementated**: `src/agent/agent.py`, `src/tools/weather_forecast.py`, `src/tools/get_event.py`, `src/tools/suggest_outfit.py`
- **Code Highlights**:
  - Hoàn thiện vòng lặp Thought -> Action -> Observation trong `ReActAgent.run()`, gồm parse `Action: tool_name(args)`, gọi tool, ghi lịch sử, dừng theo `Final Answer` hoặc `max_steps`.
  - Bổ sung cơ chế truyền kết quả giữa các tool trong `_execute_tool()` (ví dụ `suggest_outfit(weather_forecast)`), giúp chain tool tự động trong cùng phiên reasoning.
  - Cải thiện chất lượng câu trả lời cuối với `_finalize_response()` và `_compose_natural_cafe_answer()`: tự bổ sung dữ liệu thời tiết, outfit, danh sách cafe khi truy vấn liên quan "cafe".
  - Xây dựng xử lý khoảng cách trong `get_nearby_places_serpapi()` (`_extract_distance`, `_distance_to_km`) để chuẩn hóa đơn vị m/km và sắp xếp quán theo gần -> xa.
  - Tối ưu `weather_forecast.py` bằng cơ chế ưu tiên đọc cache JSON mới nhất trước khi crawl lại, giúp giảm phụ thuộc mạng và tăng tốc phản hồi.
- **Documentation**: Các thay đổi trên giúp agent vận hành đúng kiến trúc ReAct: LLM suy nghĩ và chọn hành động, tool trả về observation có cấu trúc, agent dùng observation cho bước tiếp theo rồi tổng hợp thành câu trả lời tiếng Việt theo đúng định dạng đầu ra của đề bài.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**: Ở một số truy vấn như "tôi muốn đi cafe vào lúc 16h" hoặc "tôi muốn đi cafe vào 19h tối", agent kết thúc sớm với trạng thái `reason: "no_action"` thay vì gọi tool để lấy thời tiết + gợi ý outfit + quán gần.
- **Log Source**: `logs/2026-04-06.log` có các phiên điển hình:
  - `{"event":"AGENT_START","data":{"input":"tôi muốn đi cafe vào lúc 16h","model":"gpt-4o-mini"}}`
  - `{"event":"AGENT_END","data":{"steps":2,"reason":"no_action"}}`
  - `{"event":"AGENT_START","data":{"input":"tôi muốn đi cafe vào 19h tối","model":"gpt-4o-mini"}}`
  - `{"event":"AGENT_END","data":{"steps":2,"reason":"no_action"}}`
- **Diagnosis**: Mô hình đôi lúc trả lời dạng hội thoại trực tiếp, không theo đúng mẫu `Action: tool_name(args)`. Khi regex không parse được Action trong `run()`, nhánh fallback bị kích hoạt sớm. Nguyên nhân chính nằm ở ràng buộc prompt chưa đủ chặt cho các truy vấn có mốc thời gian, kết hợp với tính ngẫu nhiên của model.
- **Solution**: Em xử lý theo 2 lớp:
  1. Củng cố system prompt trong `get_system_prompt()` với rule rõ ràng về thứ tự gọi tool cho bài toán cafe.
  2. Thêm cơ chế hồi phục trong `_finalize_response()`: nếu là truy vấn cafe mà thiếu dữ liệu, agent tự gọi `weather_forecast` -> `suggest_outfit` -> `get_nearby_places_serpapi`, sau đó tổng hợp lại bằng `_compose_natural_cafe_answer()`.
  Kết quả: giảm đáng kể trường hợp trả lời thiếu dữ liệu khi model "quên" xuất Action.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**: `Thought` giúp agent tách bài toán thành nhiều bước rõ ràng (xác định thời tiết -> chọn outfit -> tìm quán gần). So với chatbot trả lời trực tiếp, ReAct giảm hiện tượng "đoán mò", vì mỗi bước đều có observation từ tool để làm bằng chứng.
2. **Reliability**: Agent có thể kém hơn chatbot khi model không tuân thủ format Action hoặc tool bị lỗi mạng/API key. Khi đó chatbot vẫn tạo được câu trả lời trôi chảy, còn agent có nguy cơ dừng sớm hoặc trả về nội dung chưa hoàn chỉnh nếu thiếu fallback.
3. **Observation**: Observation là tín hiệu phản hồi quan trọng để agent tự điều chỉnh bước tiếp theo. Ví dụ khi dữ liệu địa điểm đã có khoảng cách, agent có thể sắp thứ tự quán gần -> xa và tạo câu trả lời hữu ích hơn thay vì danh sách ngẫu nhiên.

---

## IV. Future Improvements (5 Points)

- **Scalability**: Tách tầng tool execution thành hàng đợi bất đồng bộ (task queue) để xử lý nhiều user đồng thời, đồng thời thêm cache TTL cho dữ liệu thời tiết và địa điểm.
- **Safety**: Thêm lớp validator cho Action trước khi thực thi (whitelist tên tool, schema args) và thêm supervisor check để chặn output sai định dạng hoặc thông tin thiếu.
- **Performance**: Chuẩn hóa dữ liệu tool thành JSON schema thống nhất, giảm parse text tự do; bổ sung retry + timeout policy cho API ngoài, và log metrics (latency mỗi tool, tỉ lệ `no_action`) để tối ưu theo dữ liệu thực tế.

---